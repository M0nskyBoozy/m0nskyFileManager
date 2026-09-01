# Layer 12 — Error Tracking & Logs

> Dokumen teknis mendalam untuk lapisan **Error Tracking & Logs** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 12).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan **Error Tracking & Logs** adalah lapisan **observability**: menangkap, mencatat, dan menganalisis error serta log aplikasi untuk pemantauan & debugging.

Sesuai `ARCHITECTURE.md` Layer 12, tanggung jawab:
1. Mencatat log **request**, **error**, dan **event** aplikasi.
2. Menangani error secara **konsisten** (status code & pesan terstruktur).
3. **Rotasi & penyimpanan** log yang aman.
4. (Roadmap) Audit log untuk riwayat aksi (action history) & backup.

Komponen & teknologi nyata:
- **`log`** (Go standard library).
- **lumberjack** (`gopkg.in/natefinch/lumberjack.v2`) — log rotation di `cmd/root.go`.
- Tipe error terpusat di **`errors/`** (paket `fberrors`).
- Peta status → error (`errToStatus`) di `http/utils.go`.
- `realip` untuk mencatat IP client asli.

> **STATUS**: Structured logging & trace ID **belum** ada (roadmap). Saat ini memakai **log standar (text)** dengan dampak pesan cukup informatif (path, status, IP, error).

---

## 2. Peta Struktur File Error Tracking & Logs (Lengkap)

```
# Error terpusat (domain errors)
└── errors/
    └── errors.go                # paket fberrors: daftar error + ErrShortPassword struct

# Logging setup & rotasi
└── cmd/
    └── root.go                  # setupLog (stdout/stderr/lumberjack), log startup/shutdown

# Handler wrapper: log request error + HTTP status
└── http/
    ├── data.go                  # func handle: set header, panggil handler, log error, http.Error
    ├── utils.go                 # errToStatus (error→status), renderJSON, stripPrefix, slashClean
    ├── commands.go              # wsErr (error WebSocket)
    └── (semua handler)          # adalah handleFunc (int, error)

# Frontend error views
└── frontend/src/views/Errors.vue   # halaman 403/404/500 (map error→pesan/i18n)

# Dependency
└── go.mod                       # gopkg.in/natefinch/lumberjack.v2, github.com/tomasen/realip
```

---

## 3. Domain Errors Terpusat — `errors/errors.go`

Paket `github.com/filebrowser/filebrowser/v2/errors` (alias `fberrors`).

```go
var (
	ErrEmptyKey, ErrExist, ErrNotExist,
	ErrEmptyPassword, ErrEasyPassword, ErrEmptyUsername,
	ErrEmptyRequest, ErrScopeIsRelative, ErrInvalidDataType,
	ErrIsDirectory, ErrInvalidOption, ErrInvalidAuthMethod,
	ErrPermissionDenied, ErrInvalidRequestParams, ErrSourceIsParent,
	ErrRootUserDeletion, ErrCurrentPasswordIncorrect,
	ErrShareRequiresDownload error
)
```

| Error | Makna |
|-------|-------|
| `ErrExist` / `ErrNotExist` | Resource ada / tidak ada |
| `ErrEmptyPassword` / `ErrEasyPassword` / `ErrEmptyUsername` | Validasi kredensial |
| `ErrInvalidAuthMethod` | Metode auth tidak sah |
| `ErrPermissionDenied` | Akses ditolak |
| `ErrRootUserDeletion` | Admin tunggal tak bisa dihapus |
| `ErrCurrentPasswordIncorrect` | Password saat ini salah |
| `ErrShareRequiresDownload` | Share butuh permission download |
| `ErrScopeIsRelative`, `ErrIsDirectory`, `ErrSourceIsParent`, `ErrInvalidRequestParams`, `ErrInvalidOption`, `ErrEmptyRequest`, `ErrInvalidDataType`, `ErrEmptyKey` | Validasi parameter/operasi |

### `ErrShortPassword` (struct dengan data)

```go
type ErrShortPassword struct{ MinimumLength uint }
func (e ErrShortPassword) Error() string {
	return fmt.Sprintf("password is too short, minimum length is %d", e.MinimumLength)
}
```

- Satu-satunya error dengan **konteks terstruktur** (jumlah min length) — contoh pola untuk error domain.

> Konsistensi: seluruh paket memakai error sentral ini (dengan `errors.Is`) → perbandingan & mapping status seragam.

---

## 4. Logging Setup & Rotasi — `cmd/root.go` (`setupLog`)

```go
func setupLog(logMethod string) {
	switch logMethod {
	case "stdout":   log.SetOutput(os.Stdout)
	case "stderr":   log.SetOutput(os.Stderr)
	case "":         log.SetOutput(io.Discard)
	default:         // path file → lumberjack rotation
		log.SetOutput(&lumberjack.Logger{
			Filename:   logMethod,
			MaxSize:    100,   // MB
			MaxAge:     14,    // hari
			MaxBackups: 10,    // jumlah file backup
		})
	}
}
```

- `--log` (persisten): `stdout` / `stderr` / kosong (`io.Discard`) / **path file**.
- Bila path file → **rotation otomatis** via lumberjack:
  - max 100 MB per file, simpan 14 hari, 10 backup → mencegah disk penuh pada deployment panjang.
- Dijalankan di `rootCmd.RunE`: `setupLog(server.Log)` sebelum server listen (`cmd/root.go:193`).

### Log penting di startup/shutdown

| Lokasi | Pesan |
|--------|-------|
| `utils.go:126` | `No config file used` / `Using config file: ...` |
| `utils.go:174` | `Using database: ...` |
| `root.go:252` | `Listening on <addr>` |
| `root.go:275` | `Got signal: <sig>` |
| `root.go:283` | `Graceful shutdown complete.` |
| `root.go:195-198` | Notice wind-down (discontinued project) |
| `root.go:377+/384+/395+` | `WARNING:` Command Runner / symlink / signup risk |

---

## 5. Handler Wrapper — Log & Error (`http/data.go`)

Semua handler dibungkus oleh `handle`:

```go
func handle(fn handleFunc, prefix string, store, server) http.Handler {
	handler := http.HandlerFunc(func(w, r) {
		// 1. set global headers
		// 2. ambil settings (fatal jika gagal)
		status, err := fn(w, r, &data{...})
		// 3. log error
		if status >= 400 || err != nil {
			clientIP := realip.FromRequest(r)
			log.Printf("%s: %v %s %v", r.URL.Path, status, clientIP, err)
		}
		// 4. tulis HTTP error ke client
		if status != 0 {
			txt := http.StatusText(status)
			if status == http.StatusBadRequest && err != nil {
				txt += " (" + err.Error() + ")"
			}
			http.Error(w, strconv.Itoa(status)+" "+txt, status)
		}
	})
	return stripPrefix(prefix, handler)
}
```

### Format log error request

```
<path>: <status> <clientIP> <error>
```
Contoh: `/api/resource/foo.txt: 403 203.0.113.5 permission denied`

- **Path**, **status code**, **IP client asli** (via `realip.FromRequest`, membaca `X-Forwarded-For` di balik proxy), dan **error**.
- Hanya dicetak saat `status >= 400` atau ada error → fokus pada kegagalan.
- `http.StatusText(status)` memberi teks standar (mis. "Forbidden").
- Untuk `400 Bad Request` ditambahkan detail `err.Error()`.

> Catatan `renderJSON` (`utils.go:67-71`): jika tulis JSON gagal (client hang up) → kembalikan `(0, err)` tanpa menulis header kedua (sudah di wire) — hanya dilaporkan ke log.

---

## 6. Peta Error → Status — `errToStatus` (`http/utils.go`)

```go
func errToStatus(err error) int {
	switch {
	case err == nil:                                        return http.StatusOK
	case os.IsPermission(err):                              return http.StatusForbidden
	case os.IsNotExist(err), errors.Is(err, ErrNotExist):   return http.StatusNotFound
	case os.IsExist(err), errors.Is(err, ErrExist):         return http.StatusConflict
	case errors.Is(err, ErrPermissionDenied):               return http.StatusForbidden
	case errors.Is(err, ErrInvalidRequestParams):           return http.StatusBadRequest
	case errors.Is(err, ErrRootUserDeletion):               return http.StatusForbidden
	case errors.Is(err, imgErrors.ErrImageTooLarge):        return http.StatusRequestEntityTooLarge
	default:                                                return http.StatusInternalServerError
	}
}
```

| Error / kondisi | Status |
|-----------------|--------|
| `nil` | `200 OK` |
| `os.IsPermission` / `ErrPermissionDenied` | `403 Forbidden` |
| `os.IsNotExist` / `ErrNotExist` | `404 Not Found` |
| `os.IsExist` / `ErrExist` | `409 Conflict` |
| `ErrInvalidRequestParams` | `400 Bad Request` |
| `ErrRootUserDeletion` | `403 Forbidden` |
| `img.ErrImageTooLarge` | `413 Request Entity Too Large` |
| default | `500 Internal Server Error` |

- Memanfaatkan `errors.Is` → **error wrap** tetap terdeteksi (bukan hanya `==`).
- Digunakan luas di handler (mis. `auth.go`, `resource.go`, `users.go`, `share.go`) via `return errToStatus(err), err`.

---

## 7. Error WebSocket — `wsErr` (`http/commands.go`)

```go
func wsErr(ws *websocket.Conn, r *http.Request, status int, err error) {
	txt := http.StatusText(status)
	if err != nil || status >= 400 {
		log.Printf("%s: %v %s %v", r.URL.Path, status, r.RemoteAddr, err)
	}
	if err := ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt), ...); err != nil {
		log.Print(err)
	}
}
```

- Untuk WebSocket command runner: log path/status/RemoteAddr/err, lalu kirim **close frame** dengan text status.
- `WSWriteDeadline = 10s` untuk write control.

---

## 8. Frontend Error Views — `frontend/src/views/Errors.vue`

- Menampilkan halaman error dengan **map errorCode → pesan i18n**:
  - `502` → `errors.connection`
  - `403` → `errors.forbidden`
  - `404` → `errors.notFound`
  - `500` → `errors.internal`
- `errorCode?` default `500`; fallback ke `500` bila tak dikenal (`errors[500]`).
- Rute terkait (`router/index.ts`): `/403`, `/404`, `/500` → component `Errors` dengan `props.errorCode`.
- **Route guard**: `/settings/global|users` tanpa admin → redirect `/403` (lihat Layer 8).

---

## 9. Keamanan Log

1. **Jangan log password/kredensial** — handler auth mengembalikan `os.ErrPermission` tanpa detail.
2. **Waspada terhadap sensitive path** — `renderJSON` tidak menulis error detail ke client (hanya log server).
3. **Rotasi** (lumberjack) mencegah log membludak di disk.
4. **`realip`** — jangan sampai log IP spoof dari client; hanya trust proxy tepercaya.
5. **Jangan expose log intern** via `/health` atau endpoint publik.

---

## 10. Roadmap — Audit Log, Structured Logging & Trace ID

Sesuai ARCHITECTURE, roadmap:
1. **Audit log (action history)** — catat riwayat aksi (upload/delete/modify/share) per user untuk monitoring & backup — sesuai PRD §2.4.
2. **Structured logging** — JSON key-value (bukan text) agar mudah di-parse oleh log aggregator (ELK/Loki).
3. **Trace ID** — tambahkan `X-Request-Id`/trace di setiap request untuk korelasi antar-instance (Layer 11).
4. Adaptasi dengan `cmd/root.go` `setupLog` (tambahkan mode structured).

---

## 11. Unit Tests terkait Error/Logging

- `http/utils_test.go`, `http/auth_test.go` — verifikasi status & error mapping.
- `cmd/cmd_test.go` — konfigurasi server/log.
- (Roadmap) test audit log & structured logging.

---

## 12. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 12: Error Tracking & Logs (navigator & roadmap).
- `docs/layers/03-backend-api.md` — handler wrapper, error/status codes, endpoint `/health`.
- `docs/layers/08-role-security.md` — 403 handling, Error.vue, route guard.
- `docs/layers/11-loadbalancer-scaling.md` — realip (IP client asli) untuk logging lintas proxy.
- `docs/layers/05-hosting-deployment.md` — setupLog/lumberjack, log output di container.
- `docs/PRD.md` — §2.4 audit log & backup (roadmap).
- `docs/AGENTS.md` — workflow branch `m0nskyBuildAI`.
