# Layer 9 — Rate Limiting

> Dokumen teknis mendalam untuk lapisan **Rate Limiting** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 9).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan **Rate Limiting** mengatur pembatasan jumlah request dalam periode tertentu untuk melindungi server dari **penyalahgunaan (abuse)**, **brute force**, atau **overload**.

Sesuai `ARCHITECTURE.md` Layer 9, tanggung jawab:
1. Membatasi frekuensi request API.
2. Melindungi **endpoint sensitif** (login, upload, dsb.) dari brute force.
3. Menjaga **stabilitas & ketersediaan** layanan.

> **STATUS PENTING**: Pada kode saat ini, **rate limiting BELUM diimplementasikan**. Tidak ada middleware rate limiter, tidak ada library rate limit (mis. `golang.org/x/time/rate` atau `tollbooth`). Lapisan ini adalah **area pengembangan/roadmap**. Dokumen ini menjabarkan:
> - Mekanisme anti-abuse yang **SUDAH ADA** (ReCaptcha login, batas body, dll).
> - **Titik-titik integrasi** yang direkomendasikan (endpoint sensitif, middleware, cache Redis) beserta rencana implementasi.
> - Seluruh struktur file terkait (yang ada + yang disarankan).

---

## 2. Status Implementasi Rate Limiting

### 2.1 Yang sudah ada (anti-abuse bawaan)

| Mekanisme | Lokasi | Fungsi |
|-----------|--------|--------|
| **ReCaptcha** di login | `auth/json.go` (`ReCaptcha.Ok`) | Mencegah bot brute-force login |
| Body decoding terbatas | `auth/json.go` | Decode JSON login langsung; body oversized/dll → `os.ErrPermission` |
| Respon seragam | `auth/json.go` | `dummyHash` bcrypt (anti timing/user-enumeration) — menjaga konsistensi kecepatan |
| Timeout graceful shutdown | `cmd/root.go` | Shutdown 10s mencegah overload saat restart |

> **Belum ada**: middleware rate limiter, limiter per-endpoint, limiter berbasis IP/user/token, kuota upload, atau limiter ter-distribusi.

### 2.2 Yang belum ada (gap / roadmap)

- Middleware `http.RateLimit` di `http/http.go`.
- Library rate limiter (mis. `golang.org/x/time/rate`, `golang.org/x/net/x/net/...`, atau `tollbooth`).
- Rate limit per **IP**, per **user** (userId/token), per **endpoint**.
- Kuota **upload / bandwidth** per user.
- State ter-distribusi via **Redis** untuk multi-instance.

---

## 3. Endpoint Sensitif yang Perlu Dilindungi

Berdasarkan peta rute di `http/http.go`:

### 3.1 Prioritas TINGGI (brute-force / abuse)

| Endpoint | Method | Handler | Alasan |
|----------|--------|---------|--------|
| `/api/login` | POST | `loginHandler` | Target brute-force kredensial |
| `/api/signup` | POST | `signupHandler` | Register spam |
| `/api/renew` | POST | `renewHandler` | Token refresh abuse |
| `/api/tus/*` (POST/PATCH) | POST/PATCH | `tusPostHandler`/`tusPatchHandler` | Upload bandwith abuse |
| `/api/resources` (POST/PUT/PATCH/DELETE) | variasi | `resource*Handler` | Operasi tulis/hapus massal |

### 3.2 Prioritas SEDANG (load protection)

| Endpoint | Method | Handler |
|----------|--------|---------|
| `/api/resources` (GET) | GET | `resourceGetHandler` |
| `/api/resources/recursive` | GET | `resourceGetRecursiveHandler` |
| `/api/search` | GET | `searchHandler` |
| `/api/preview/{size}/...` | GET | `previewHandler` (resize gambar — CPU berat) |
| `/api/raw` | GET | `rawHandler` |
| `/api/share` (GET) | GET | `shareGetsHandler` |
| `/api/public/dl/`, `/api/public/share/` | GET | `publicDlHandler`/`publicShareHandler` |
| `/api/command` | GET | `commandsHandler` (WebSocket) |

### 3.3 Prioritas RENDAH (admin/self ops)

- `/api/users/*`, `/api/settings`, `/api/shares`, `/api/usage`, `/health`, `/static`.

---

## 4. Peta Struktur File Rate Limiting

### 4.1 Struktur NYATA (saat ini)

```
# Anti-abuse yang ada saat ini
├── auth/
│   └── json.go                  # ReCaptcha.Ok + dummyHash (anti brute-force/timing)
├── http/
│   ├── http.go                  # Router: definisi endpoint (titik pemasangan middleware kelak)
│   ├── auth.go                  # loginHandler / signupHandler / renewHandler
│   ├── resource.go              # resource*Handler (upload/patch/delete/get)
│   ├── tus_handlers.go          # tus*Handler (upload TUS - perlu kuota/rate)
│   └── preview.go               # previewHandler (CPU berat - resize gambar)
├── cmd/
│   └── config.go                # getJSONAuth (config ReCaptcha: host/key/secret)
└── go.mod                       # (library rate limit BELUM ada)
```

### 4.2 Struktur SARANAN (yang akan ditambahkan — roadmap)

```
# Rate limiting baru
├── http/
│   ├── ratelimit.go            # middleware RateLimit + limiter store (in-memory/Redis)
│   └── http.go                 # r.Use(rateLimitMiddleware) sebagai global / per-endpoint
├── rate/                        # (paket baru opsional)
│   ├── limiter.go              # RateLimiter, MiddlewareOption, Memoized per key
│   └── limiter_test.go
└── cmd/
    └── config.go               # flag --rate-...
```

---

## 5. Mekanisme Anti-Abuse yang Ada — Detail

### 5.1 ReCaptcha (`auth/json.go`)

```go
type ReCaptcha struct {
	Host   string `json:"host"`
	Key    string `json:"key"`
	Secret string `json:"secret"`
}

func (r *ReCaptcha) Ok(response string) (bool, error) {
	// POST r.Host + "/recaptcha/api/siteverify"
	// body: secret + response → {"success": bool}
}
```

- Hanya aktif jika `JSONAuth` dan `ReCaptcha.Secret != ""` (`a.ReCaptcha != nil && a.ReCaptcha.Secret != ""`).
- Jika `!ok` → `os.ErrPermission` (403).
- Konfigurasi CLI:
  - `--recaptcha.host` (default `https://www.google.com`; `recaptcha.net` untuk Tiongkok).
  - `--recaptcha.key` (site key).
  - `--recaptcha.secret`.
- Ini adalah satu-satunya proteksi bot pada login saat ini.

### 5.2 Anti timing / user-enumeration

```go
const dummyHash = "$2a$10$O4mEMeOL/nit6zqe.WQXauLRbRlzb3IgLHsa26Pf0N/GiU9b.wK1m"

u, err := usr.Get(...)
hash := dummyHash
if err == nil { hash = u.Password }
if !users.CheckPwd(cred.Password, hash) { return nil, os.ErrPermission }
```

- Selalu menjalankan bcrypt (baik user ada maupun tidak) → waktu respon konsisten.
- Tidak membocorkan eksistensi user via timing.

### 5.3 Keterbatasan saat ini

- Tidak **melacak jumlah attempt** per IP/user → brute force lambat (slow) tetap mungkin.
- Tidak ada **lockout** setelah N gagal.
- Tidak ada **backoff** atau **kuota**.

> Rate limiting akan **melengkapi** semua ini di level HTTP, bukan hanya di login.

---

## 6. Rencana Implementasi Rate Limiting (Roadmap)

### 6.1 Arsitektur yang direkomendasikan

```
 Request ──▶ middleware RateLimit ──▶ router (http/http.go)
                 │
                 ├─ key: [IP]   (global per alamat)
                 ├─ key: [userID/token]  (per user autentikasi)
                 └─ key: [IP:endpoint]   (per endpoint sensitif)
                 │
                 ▼
           Limiter Store
           ├─ in-memory (jellydator/ttlcache) → instance tunggal
           └─ Redis (go-redis)               → multi-instance (shared)
                 │
                 ▼
         Bila melebihi limit → 429 Too Many Requests
```

### 6.2 Algoritma yang cocok

1. **Token Bucket** (`golang.org/x/time/rate.Limiter`) — laju per detik + burst. Perlu menambah dependency `golang.org/x/time`.
2. **Fixed/Sliding Window via Redis** — struktur `INCR` + `EXPIRE` / `ZADD`+count untuk window ter-distribusi.
3. **Leaky Bucket** — untuk upload throughput tracking.

### 6.3 Dependency yang tersedia & yang perlu

| Dependency | Status | Kegunaan untuk rate limit |
|-----------|--------|---------------------------|
| `github.com/redis/go-redis/v9 v9.21.0` | **Sudah ada** | Storage rate counter ter-distribusi (multi-instance) |
| `github.com/jellydator/ttlcache/v3 v3.4.1` | **Sudah ada** | In-memory TTL cache untuk limiter instance tunggal |
| `golang.org/x/time` | **Perlu ditambah** | `rate.Limiter` token bucket standar |
| (plugins tollbooth) | opsional | Middleware siap pakai |

> Karena `go-redis` & `ttlcache` sudah ada di proyek (dipakai upload cache), integrasi rate limiter **tidak memerlukan redis stack baru** — tinggal tambah `golang.org/x/time`.

### 6.4 Pemasangan middleware (untuk `http/http.go`)

```go
// contoh (roadmap)
r.Use(rateLimit.Middleware(rateLimit.Config{
    Store:  limiterStore,            // ttlcache (single) atau Redis (multi)
    GeneralIP: rate.Limit(20),       // 20 req/detik per IP
    Login:      rate.Limit(1),       // endpoint login (per ReCaptcha)
    UploadMBPS: rate.Limit(5),       // kuota upload
}))
```

- Middleware Global untuk semua rute → `r.Use(...)`.
- Atau per-subrouter sensitif (`/api/login`, `/api/signup`, `/api/tus/*`).
- Kembalian `429 Too Many Requests` (+ `Retry-After` header).

---

## 7. Keamanan Rate Limiting (Catatan Khusus)

1. **Jangan expose di single node bila multi-instance** — gunakan Redis agar counter konsisten lintas instance.
2. **Jangan block berbasis IP untuk CDN/proxy** — identifikasi via `X-Forwarded-For` dengan hati-hati (spoofing).
3. **Prioritaskan endpoint auth & public share** — mereka paling terpapar.
4. **Kombinasikan** dengan ReCaptcha (layer login) + aturan allow/deny (Layer 4) + RBAC (Layer 8).
5. **Tambahkan header** `Retry-After` & `X-RateLimit-*` untuk UX dan observability.
6. **Test keamanan brute-force** perlu memperhitungkan rate limiter.

---

## 8. Unit Tests yang Disarankan (Roadmap)

- `http/ratelimit_test.go` — limit per IP, per user, per endpoint; kembalian 429; reset window.
- `rate/limiter_test.go` — token bucket / sliding window, overflow, concurrency.
- `http/login_ratelimit_test.go` — N attempt gagal → lockout sementara.
- Integrasi Redis (opsional) — counter ter-distribusi multi-instance.

---

## 9. Catatan Pengembangan (Roadmap)

1. **Mulai dari middleware global in-memory** (ttlcache) untuk single node → kemudian upgrade ke Redis untuk multi-instance.
2. **Konfigurasi via env/flag** (`--rate-...`, prefix `FB_`) memakai pola Viper yang sudah ada (`cmd/utils.go`).
3. **Tambahkan dependency** `golang.org/x/time` (atau implement window sendiri via Redis INCR/EXPIRE).
4. **Endpoint prioritas**: `/api/login`, `/api/signup`, `/api/tus/*`, `/api/preview/*` (CPU berat), `/api/command`.
5. Selaras `STYLEGUIDE.md` (belum ada) serta `docs/AGENTS.md` branch `m0nskyBuildAI`.

---

## 10. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 9: Rate Limiting (navigator & roadmap).
- `docs/layers/04-auth-authorization.md` — ReCaptcha, `dummyHash`, login flow.
- `docs/layers/03-backend-api.md` — semua endpoint (login/signup/upload/share), handler.
- `docs/layers/02-database-storage.md` — Redis cache (`UploadCache`), potensi share state.
- `docs/layers/08-role-security.md` — RBAC endpoint sensitif.
- `docs/PRD.md` — kebutuhan keamanan, target user, roadmap Podman.
- `docs/AGENTS.md` — workflow pengembangan branch `m0nskyBuildAI`.