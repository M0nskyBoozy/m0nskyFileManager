# Layer 6 — Cloud Compute

> Dokumen teknis mendalam untuk lapisan **Cloud Compute** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 6).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan **Cloud Compute** merepresentasikan **infrastruktur komputasi (server/instance)** tempat aplikasi berjalan, dan **bagaimana kapasitas komputasi dieksploitasi/dikelola** oleh aplikasi itu sendiri.

Fokus utama lapisan ini (sesuai `ARCHITECTURE.md` Layer 6):

> *"Lapisan infrastruktur komputasi (server/instance) tempat aplikasi berjalan dan bagaimana akses aman dicapai (SSH tunneling)."*

Secara konkret, "cloud compute" di repositori ini mencakup **dua wajah**:

1. **Compute host (platform server)** — bagian program yang mendefinisikan bagaimana instance server berjalan: alamat/port pengikatan (`bind`), TLS, unix socket, root path, dan konfigurasi server (`settings.Server`) yang dimuat `cmd/root.go`.
2. **Compute workloads (eksekusi)** — kemampuan aplikasi untuk **menjalankan pekerjaan di server**: eksekusi command (blocking/non-blocking), WebSocket terminal/exec, work pool untuk image processing, serta cache lokal disk / Redis untuk pekerjaan multi-instance.

Penting: fitur-fitur "cloud-native" tingkat lanjut yang tertulis di `PRD.md` — **SSH tunneling**, **Rootless Podman dashboard/manager**, **resource limit per container** — **belum diimplementasikan** dan merupakan **roadmap** (tanda `(Roadmap)`). Dokumen ini menjabarkan:
- Modul kode nyata yang **sudah ada** (menjadi fondasi compute host & execution).
- Bagian yang **harus dibangun** (roadmap) beserta titik integrasinya.

---

## 2. Peta Struktur File Cloud Compute (Lengkap)

```
# Compute Host — inisialisasi & konfigurasi instance server
├── cmd/
│   ├── root.go                # server run: listener TCP/TLS/socket, log, graceful shutdown,
│   │                          #   quick setup, cache, image worker, settings.Server
│   └── utils.go               # initViper (config file + env FB_*), DB bootstrap
├── settings/
│   └── settings.go            # type Server (Root/BaseURL/Socket/TLS/Port/Address/...) + Settings
├── version/
│   └── version.go             # Version & CommitSHA (identitas build/deployment compute instance)
└── main.go                    # entry point → cmd.Execute()

# Compute Workloads — eksekusi pekerjaan pada server
├── runner/
│   ├── runner.go              # Runner: eksekusi command blocking/non-blocking + hooks
│   ├── commands.go            # SplitCommandAndArgs (Unix/Windows shell parsing)
│   └── parser.go              # ParseCommand (shell vs direct binary)
├── http/
│   └── commands.go            # WebSocket handler: terminal/command streaming (perm.Execute)
├── img/
│   ├── service.go             # Service: image processing work pool (semaphore workers)
│   └── service_enum.go        # Format enum (jpeg/png/gif/tiff/bmp)
├── diskcache/
│   ├── cache.go               # Interface diskcache.Interface
│   ├── file_cache.go          # FileCache — cache lokal disk (--cacheDir)
│   └── noop_cache.go          # NoOpCache — cache nonaktif (default)
└── http/upload.go             # UploadCache — cache upload via Redis (--redisCacheUrl)

# Deployment (dikonsumsi compute instance) — dirujuk dari Layer 5
├── Dockerfile / Dockerfile.s6 / compose.yaml / .goreleaser.yml
└── docs/layers/05-hosting-deployment.md
```

---

## 3. Compute Host — Inisialisasi Instance Server (`cmd/root.go`)

Ini adalah **modul kode nyata utama** lapisan Cloud Compute: bagian yang menghidupkan server pada infra komputasi.

### 3.1 Alur `RunE` (root command)

```
withViperAndStore(...)
  1. initViper → config file + env FB_* (utils.go)
  2. Database: quick setup (jika baru) | buka DB (storm/Bolt)
  3. imageService = img.New(imageProcessors)   → work pool pemroses gambar
  4. fileCache    = cacheDir? FileCache : NoOp  → cache lokal disk
  5. uploadCache  = fbhttp.NewUploadCache(redisCacheUrl) → cache upload Redis
  6. server       = getServerSettings(v, st.Storage)  → settings.Server
  7. setupLog(server.Log) → lumberjack/stdout/stderr
  8. root = filepath.Abs(server.Root)
  9. Listener: Socket | TLS | TCP   (lihat 3.3)
  10. http.Server{Serve(listener)} di goroutine
  11. signal.Notify → graceful shutdown (10s)
```

### 3.2 `settings.Server` (fondasi komputasi host)

```go
type Server struct {
    Root          string `json:"root"`
    BaseURL       string `json:"baseURL"`
    Socket        string `json:"socket"`
    TLSKey        string `json:"tlsKey"`
    TLSCert       string `json:"tlsCert"`
    Port          string `json:"port"`
    Address       string `json:"address"`
    Log           string `json:"log"`
    EnableThumbnails, ResizePreview, EnableExec bool
    TypeDetectionByHeader, ImageResolutionCal, FollowExternalSymlinks bool
    AuthHook            string
    TokenExpirationTime string
    CaseInsensitiveFs   bool `json:"-"`
}
```

- `CaseInsensitiveFs` **tidak dipersist** — dideteksi dari `Root` saat startup (untuk rule checker case-insensitive).
- `Clean()` menghapus trailing `/` pada `BaseURL`.
- `GetTokenExpirationTime(fallback)` — parse durasi `tokenExpirationTime`, fallback jika invalid.

### 3.3 Listener (pengikatan network disk)

Dipilih via `switch` di `root.go`:

| Kondisi | Listener | Catatan Cloud Compute |
|---------|----------|----------------------|
| `server.Socket != ""` | `net.Listen("unix", socket)` + `chmod(socketPerm)` | Unix socket untuk akses lokal/proxy di host |
| `TLSKey != "" && TLSCert != ""` | `tls.Listen("tcp", adr, {MinVersion: TLS1.2, ...})` | TLS langsung di instance |
| default | `net.Listen("tcp", address:port)` | Bind TCP; `--address 0.0.0.0` untuk expose |

- `adr = server.Address + ":" + server.Port`.
- Konflik `--socket` dengan `address/port/key/cert` → **error** (validasi eksplisit).
- **Resource/security note**: default bind `127.0.0.1:8080` (lokal); untuk akses via SSH tunneling justru **tidak perlu** membuka port publik.

### 3.4 Hasil komputasi & caching pada instance

Diwakili `FileCache` (`--cacheDir`) dan `UploadCache` (`--redisCacheUrl`) — garis besar pembagian kerja komputasi antar instance (multi-instance berbagi Redis). Detail di Layer 10 / Layer 2.

---

## 4. Compute Workloads — Eksekusi Command (`runner/`)

Lapisan compute yang membuat instance **menjalankan pekerjaan**. Dibagi dua jalur:

1. **`runner.RunHook`** — eksekusi sebagai *side-effect* event (before/after) pada operasi file (via `http/data.go`).
2. **WebSocket terminal (`http/commands.go`)** — eksekusi interaktif/streaming yang dipicu user dari UI.

### 4.1 `runner.Runner` (`runner/runner.go`)

```go
type Runner struct {
    Enabled bool
    *settings.Settings
}
func (r *Runner) RunHook(fn func() error, evt, path, dst string, user *users.User) error
```

- Jika `Enabled` (`server.EnableExec`) dan ada `Commands["before_"+evt]`: jalankan semua command sebelum operasi; error membatalkan `fn`.
- Jalankan `fn` (operasi file).
- Jika ada `Commands["after_"+evt]`: jalankan setelah.
- `exec`:
  - `&` suffix → **non-blocking** (`cmd.Start` + async `Wait`).
  - Command di-parse via `ParseCommand`.
  - **Env mapping**: `FILE`, `SCOPE`, `TRIGGER`, `USERNAME`, `DESTINATION` di-set ke child process.
  - `stdin/stdout/stderr` diteruskan ke proses server.

### 4.2 `ParseCommand` (`runner/parser.go`)

- `name, args, err := SplitCommandAndArgs(raw)`.
- Jika `settings.Shell` kosong → jalankan **binary langsung** (`name args...`).
- Jika ada shell → jalankan `shell... raw` (via shell).

### 4.3 `SplitCommandAndArgs` (`runner/commands.go`)

- Parser shell-style lintas-OS:
  - Unix → `go-shlex.Split`.
  - Windows → parser manual (perlakuan backslash & quote).
- Dipakai memastikan command & argumen terpisah aman sebelum dieksekusi.

---

## 5. WebSocket Terminal / Command Streaming (`http/commands.go`)

Handler **execution real-time** atas kapasitas komputasi instance:

```
Client (WebSocket) ── raw command ──▶ commandsHandler ──▶ runner.ParseCommand
        ◀── stdout/stderr stream ──  (scanner io.MultiReader)
```

Alur `commandsHandler` (`withUser`):
1. Upgrade WebSocket (`upgrader`, buffer 1024/1024).
2. Baca satu pesan `raw` (trim).
3. **Fail-fast authorization**: `if !server.EnableExec || !user.Perm.Execute` → kirim `Command not allowed.` — **aplikasi harus mengaktifkan "Command Runner"** untuk menggunakan compute eksekusi ini.
4. `ParseCommand` → error dikirim balik.
5. Whitelist per-user: `slices.Contains(user.Commands, name)` — hanya command yang diizinkan user.
6. `exec.Command(...)`, `cmd.Dir = user.FullPath(r.URL.Path)` (jalankan di dalam scope user).
7. Pipe stdout+stderr → **scanner** → stream `TextMessage` real-time ke client.
8. `cmd.Wait()` menunggu proses selesai; error dilaporkan via `wsErr`.

Konstanta: `WSWriteDeadline = 10s`, `cmdNotAllowed = "Command not allowed."`.

> Hubungannya dengan Cloud Compute: mekanisme ini adalah **fondasi eksekusi pada instance**. Untuk roadmap Podman/website-deployer, pola WebSocket streaming serupa akan dipakai untuk log container build/deploy.

---

## 6. Compute Workloads — Work Pool Image Processing (`img/`)

`img.Service` membatasi **paralelisme** pekerjaan berat (resize/thumbnail) dengan **semaphore**:

```go
type Service struct { sem semaphore.Semaphore }
func New(workers int) *Service { return &Service{sem: semaphore.New(workers)} }
```

- Jumlah worker ditentukan `--imageProcessors` (default **4**), di-root.go: `imgWorkersCount < 1 → error`.
- **Batas dimensi**: `MaxImageWidth/MaxImageHeight = 10000` — melindungi instance dari Out-of-Memory (server crash) saat thumbnail besar.
- Guard batas ini adalah contoh **resource protection** di sisi komputasi (mengelola alokasi CPU/memori instance).
- Format: `jpeg, png, gif, tiff, bmp` (`service_enum.go`).

---

## 7. Compute Storage — Cache pada Compute Node (`diskcache/`)

Lapisan compute lokal untuk data sementara:

- `diskcache.Interface` (antarmuka cache).
- `FileCache` — cache di **disk lokal instance** (`--cacheDir`); dibuat dengan `os.MkdirAll(cacheDir, 0700)`.
- `NoOpCache` — default (nonaktif jika `--cacheDir` kosong).

`NewUploadCache(redisCacheUrl)` (Layer 2) memilih Redis untuk cache multi-instance. Ini membedakan **compute-local** (FileCache) vs **shared** (Redis).

---

## 8. Eksekusi pada Root Process (`cmd/root.go` — logging & shutdown)

Bagian operasional penting bagi compute instance panjang umur (daemon):

- **`setupLog(server.Log)`**: `stdout`/`stderr`/kosong, atau **lumberjack** rotation (MaxSize 100MB, MaxAge 14 hari, MaxBackups 10) bila berupa path file.
- **Graceful shutdown**: tangkap `SIGINT/SIGTERM/SIGHUP/SIGQUIT` → `srv.Shutdown(ctx 10s)` → "Graceful shutdown complete."
- Ini menjaga instance compute bisa di-*rolling restart* dengan aman oleh orkestrator.

---

## 9. Access Aman — SSH Tunneling (Roadmap)

Sesuai `PRD.md` & `ARCHITECTURE.md`, akses aman utama **belum diimplementasi** sebagai modul kode; saat ini akses dicapai lewat konfigurasi bind lokal + tunnel eksternal. Status:

**Sudah ada (fondasi):**
- Default bind `127.0.0.1` memungkinkan hanya visible via localhost → aman untuk tunnel.
- `--address`, `--port`, `--baseURL`, `--cert/--key`, `--socket` memberikan kontrol penuh bind.
- `ProxyAuth` (Layer 4) memungkinkan identitas dari header proxy (cocok saat tunnel belakangnya ada gatekeeper).

**Roadmap (akan dibangun):**
- Integrasi **SSH tunneling** langsung dalam program (encrypted payload).
- Autentikasi **SSH Key (Ed25519/RSA)** dan password.
- **Session control**: auto-logout pada session idle (idle timeout).
- **Audit log & action history** + one-click backup/restore.

---

## 10. Rootless Podman & Resource Limit (Roadmap)

Sesuai `ARCHITECTURE.md` dan `PRD.md`, integrasi Podman adalah **visi utama** namun **belum menjadi modul kode**:

| Kebutuhan (PRD) | Status | Titik integrasi (ketika dibangun) |
|---|---|---|
| GUI monitor container (Running/Stopped/Paused/Exited) | Roadmap | Frontend view baru; backend API baru (Layer 3) |
| Start/Stop/Restart/Pause/Delete container | Roadmap | Podman REST API client |
| Live resource monitoring (CPU/Mem/Disk I/O/Network) | Roadmap | Backend polling + WebSocket streaming |
| Streaming logs container | Roadmap | Pola WebSocket (`http/commands.go`) |
| **Rootless Podman** (isolation & keamanan) | Roadmap | Jalankan podman sebagai user non-root |
| **Resource limit** (CPU & RAM) per container | Roadmap | Opsi `--cpus`/`--memory` + RBAC (Layer 4) |

Modul yang akan ditambahkan (sesuai ARCHITECTURE roadmap):
- **Podman client/service** — akses `Podman REST API` via socket `/run/podman/podman.sock`.
- **Podman Wrapper / Website Deployer** — deploy website via podman rootless.
- **Reverse proxy otomatis + Let's Encrypt / Auto-SSL**.
- **Persistent volume Podman** sebagai sumber data (browse/backup/restore).

---

## 11. Keamanan Cloud Compute (Checklist)

1. **Bind lokal default** (`127.0.0.1`) — jangan expose port publik bila tidak perlu; akses via SSH tunnel/Layer 6.
2. **TLS min TLS 1.2** bila memakai TLS langsung; idealnya terminasi di proxy (Layer 5).
3. **Command Runner** nonaktif default (`--disableExec=true`); aktifkan hanya dengan pemahaman risiko (known vulns) — lihat `WARNING` di `getServerSettings`.
4. **`FollowExternalSymlinks=false`** — mencegah scope escape pada compute host.
5. **Resource protection** gambar: `MaxImageWidth/Height = 10000` + `imageProcessors` work pool (mencegah OOM).
6. **Non-root** di container (Layer 5) & rootless podman (roadmap) demi isolasi.
7. **Rotasi log** (lumberjack) agar disk instance tidak penuh pada deployment panjang.
8. **Whitelist command per-user** (`user.Commands`) & `Perm.Execute` — compute execution hanya untuk user berwenang.
9. **Redis** dilindungi ACL (compose) bila pakai cache multi-instance.
10. **Graceful shutdown** agar orkestrator dapat restart instance tanpa korupsi data.

---

## 12. Catatan Pengembangan (Roadmap Jangka Pendek)

1. **Mulai dari Podman REST client** — modul baru yang berkomunikasi dengan `podman.sock` (mirip cara `runner` berkomunikasi dengan OS).
2. **Perluas RBAC** (Layer 4) untuk operasi Podman (mis. hanya admin yang boleh manage container) dan resource limit.
3. **SSH tunnel + session control**: bangun modul SSH server-internal + idle timeout → bangun di atas listener `--socket`/`--address` yang sudah ada.
4. **Resource limit visual**: backend perlu mengumpulkan data `/sys`/cgroup per container → tampilkan via WebSocket (fondasi dapat dipinjam dari `http/commands.go`).
5. Semua penambahan harus **menaati `runner` / worker pool / non-blocking** pattern agar tidak menahan compute host.

---

## 13. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 6: Cloud Compute (navigator & roadmap).
- `docs/PRD.md` — §2.4 Keamanan & SSH Tunneling; §2.1 & §2.2 Podman; target user.
- `docs/layers/03-backend-api.md` — handler API, `withUser`, WebSocket endpoint.
- `docs/layers/04-auth-authorization.md` — `Perm.Execute`, whitelist command, RBAC.
- `docs/layers/05-hosting-deployment.md` — container, binding port, host/instance deployment.
- `docs/layers/02-database-storage.md` — cache Redis/disk, `UploadCache`.
- `docs/AGENTS.md` — workflow pengembangan branch `m0nskyBuildAI`.
