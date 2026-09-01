# Layer 5 — Hosting & Deployment

> Dokumen teknis mendalam untuk lapisan **Hosting & Deployment** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 5).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan ini mengatur **bagaimana `m0nskyFileManager` dibangun, dikemas, dan dijalankan** di berbagai lingkungan produksi. Mencakup:

1. **Build** — kompilasi binary Go + bundling aset frontend (Vue) menjadi satu, lalu dikemas sebagai image Docker.
2. **Packaging** — release matrix multi-platform (`.goreleaser.yml`), arsip `tar.gz`/`zip`, dan image Docker multi-arch dengan buildx.
3. **Runtime & container** — `Dockerfile` (alpine) dan `Dockerfile.s6` (s6-overlay), non-root user, healthcheck, init script, default config.
4. **Orkestrasi** — `compose.yaml` (compose + Redis shared cache).
5. **Konfigurasi server** — CLI flags, file config, variabel lingkungan `FB_*`, TLS/socket, logging, graceful shutdown.
6. **Bootstrap database** — quick setup / `config init` dan `config export/import` untuk backup config & user antar instance.

Empat jalur deployment utama:
- **Biner langsung** (bare metal / VM / cloud) — jalankan `./filebrowser`.
- **Docker standalone** — image `filebrowser/filebrowser`.
- **Docker Compose** — multi-service (filebrowser + redis).
- **Di belakang reverse proxy / TLS terminator** — baseURL + ProxyAuth/JSONAuth.

---

## 2. Peta Struktur File Hosting & Deployment (Lengkap)

```
# Build & packaging (repo root)
├── main.go                    # entry point Go → panggil cmd.Execute()
├── go.mod                     # module github.com/filebrowser/filebrowser/v2, Go 1.25
├── .goreleaser.yml            # release: binary matrix + docker buildx + manifests + homebrew
├── Taskfile.yml               # task build, build:frontend, build:backend, release, docs:cli
├── .dockerignore              # file/dir yang dikecualikan saat build context Docker
├── Dockerfile                 # image produk (alpine, multistage, non-root, tini)
├── Dockerfile.s6              # image alternatif berbasis LinuxServer.io baseimage (s6-overlay)
└── compose.yaml               # docker compose: filebrowser + redis (shared cache)

# Script runtime dalam image
├── docker/
│   ├── alpine/
│   │   ├── healthcheck.sh     # healthcheck untuk image alpine
│   │   └── init.sh            # entrypoint: seed config + eksekusi filebrowser
│   ├── common/
│   │   ├── healthcheck.sh     # healthcheck bersama (pakai jq)
│   │   └── defaults/
│   │       └── settings.json  # default config /config/settings.json
│   └── s6/
│       ├── custom-cont-init.d/
│       │   └── 20-config      # s6 init: seed config + chown
│       └── etc/services.d/
│           └── filebrowser/run # s6 service: jalankan filebrowser sbg user abc

# Runtime binder server (engine dekstop/bare-metal)
├── cmd/
│   ├── root.go                # server run: flags, listener (TCP/TLS/socket), log, shutdown
│   ├── utils.go               # initViper (config file + env FB_*), DB bootstrap, marshal
│   ├── config.go              # getServerSettings, auther selection
│   ├── config_init.go         # `filebrowser config init`
│   ├── config_cat.go          # `filebrowser config cat`
│   ├── config_set.go          # `filebrowser config set`
│   ├── config_export.go       # `filebrowser config export` (backup/port)
│   ├── config_import.go       # `filebrowser config import`
│   └── version.go             # `filebrowser version`
└── version/
    └── version.go             # Version & CommitSHA (ldflags)

# Frontend build (embedded ke binary saat `go:build !dev`)
└── frontend/
    ├── vite.config.ts         # build SPA + template StaticURL (deploy subpath)
    ├── package.json           # scripts build/typecheck/lint/test, node/pnpm engines
    ├── assets.go              # //go:embed dist/* (disusun ke binary)
    └── dist/                  # hasil build (di-embed)
```

---

## 3. Build & Kompilasi

### 3.1 Langkah build lengkap (`Taskfile.yml`)

`task build` = `build:frontend` + `build:backend`:

1. **`build:frontend`** (di `frontend/`):
   - `pnpm install --frozen-lockfile`
   - `pnpm run build` → `pnpm run typecheck && vite build`
   - Vite memproduksi `frontend/dist/` (SPA) — **di-embed ke binary** via `frontend/assets.go` (`//go:embed dist/*`).

2. **`build:backend`**:
   ```sh
   go build -ldflags='-s -w \
     -X "github.com/filebrowser/filebrowser/v2/version.Version={{.VERSION}}" \
     -X "github.com/filebrowser/filebrowser/v2/version.CommitSHA={{.GIT_COMMIT}}"' \
     -o filebrowser .
   ```
   - `VERSION` = `git describe --tags --abbrev=0 --match=v* | cut -c 2-`
   - `GIT_COMMIT` = `git log -n 1 --format=%h`
   - Hasil: satu biner `filebrowser` (sudah berisi frontend).

**Persyaratan toolchain:**
- Go `1.25` (go.mod)
- Node.js `>= 24.0.0`, pnpm `>= 10.0.0` (untuk frontend)
- Task (`github.com/go-task/task`)

### 3.2 Biner mandiri

`go build ./...` menghasilkan binary mandiri karena frontend di-embed (`frontend.Assets()`). Tidak perlu `static` file terpisah saat runtime. `go:build !dev` memastikan embed hanya pada build produksi.

---

## 4. Packaging & Release (`goreleaser`)

`.goreleaser.yml` (v2) mengotomasi rilis:

- **Builds** — `main: main.go`, binary `filebrowser`, `CGO_ENABLED=0`, ldflags (-s -w + Version/CommitSHA). Matriks OS: `darwin, linux, windows, freebsd, openbsd`; arch: `amd64, 386, arm, arm64, riscv64`; goarm: `5,6,7`. Beberapa kombinasi di-ignore (mis. windows/arm deprecated di Go 1.24-1.26).
- **Archives** — `tar.gz` (zip untuk windows), nama `{{.Os}}-{{.Arch}}{{if .Arm}}v{{.Arm}}{{end}}-filebrowser`.
- **Dockers** (buildx, multi-arch) — 6 image config:
  - `Dockerfile` alpine: amd64, arm64, arm/v6, arm/v7.
  - `Dockerfile.s6`: amd64, arm64.
  - Tag per image: `{{ .Tag }}-{arch}` dan `v{{ .Major }}-{arch}` (+ `-s6` untuk varian s6).
  - Label OCI (created, name, revision, version, source).
- **Docker manifests** — menggabungkan multi-arch menjadi tag manifest: `latest`, `{{ .Tag }}`, `v{{ .Major }}`, dan varian `s6`.
- **Homebrew cask** — publish ke `filebrowser/homebrew-tap`, dengan `post.install` menghapus `com.apple.quarantine`.

---

## 5. Image Docker — `Dockerfile` (varian alpine produksi)

### 5.1 Analisis stage

**Stage 1 `fetcher`** (`alpine:3.23`):
- `apk add ca-certificates mailcap tini-static`
- Download `JSON.sh` (parsing JSON shell untuk healthcheck) pada commit pasti (reproducible).

**Stage 2 `busybox:1.37.0-musl`** (runtime ringan):
- `ENV UID=1000 GID=1000`
- Buat user/group `user` non-root: `addgroup -g $GID user && adduser -D -u $UID -G user user`.
- Copy: `filebrowser → /bin/filebrowser`, `docker/common/ → /`, `docker/alpine/ → /`, `tini-static → /bin/tini`, CA certs, `mime.types`, `ssl`.
- Buat dir data: `/config /database /srv`, chown `user:user`, `chmod +x /healthcheck.sh`.
- `HEALTHCHECK --start-period=2s --interval=5s --timeout=3s CMD /healthcheck.sh`
- `USER user`, `VOLUME /srv /config /database`, `EXPOSE 80`.
- `ENTRYPOINT ["tini","--","/init.sh"]` — **tini** sebagai PID 1 agar sinyal SIGTERM diteruskan dengan benar (zombie reaping).

### 5.2 Alasan desain keamanan/operasional

- **Non-root** (`USER user`) — proses tidak berjalan sebagai root.
- **Tini** — init yang benar untuk container (forward signal, reap zombie).
- **Volume terpisah** `/srv` (data), `/config` (settings), `/database` (BoltDB) — pemisahan data & konfigurasi agar mudah backup/migrasi.
- **Healthcheck** terhadap `/health` endpoint bawaan server.

---

## 6. Image Docker — varian `Dockerfile.s6` (s6-overlay)

- Basis `ghcr.io/linuxserver/baseimage-alpine:3.23` (LinuxServer.io).
- `apk add ca-certificates mailcap jq libcap`.
- User `abc`; dir `/config /database /srv` chown `abc:abc`.
- `setcap 'cap_net_bind_service=+ep' /bin/filebrowser` — memberi izin bind port <1024 (80) tanpa root penuh.
- Copy `docker/common/` + `docker/s6/`.
- `HEALTHCHECK` sama; `VOLUME /srv /config /database`; `EXPOSE 80`.
- Tata kelola service s6:
  - `custom-cont-init.d/20-config`: seed `/config/settings.json` dari `/defaults/settings.json` jika belum ada + chown `abc` pada config/database/srv.
  - `etc/services.d/filebrowser/run`: `exec s6-setuidgid abc filebrowser -c /config/settings.json` (drop ke user `abc`, pakai config-file).

Kelebihan s6: manajemen long-running service, integrasi umrah LinuxServer, dan `cap_net_bind_service` memungkinkan bind port 80 sebagai non-root.

---

## 7. Script Runtime & Default Konfigurasi (`docker/`)

### 7.1 `docker/alpine/init.sh` (entrypoint alpine)

Alur:
1. Jika `/config/settings.json` belum ada → salin dari `/defaults/settings.json`.
2. Ekstrak path config dari args (`-c`/`--config` atau `-c=X`). Jika tidak diberikan → sisipkan `--config=/config/settings.json`.
3. `exec filebrowser "$@"` — ganti proses (PID 1 = filebrowser, dikelola tini).

### 7.2 Default `docker/common/defaults/settings.json`

```json
{
  "port": 80,
  "baseURL": "",
  "address": "",
  "log": "stdout",
  "database": "/database/filebrowser.db",
  "root": "/srv"
}
```

Jadi di container: listen `:80`, DB di `/database/filebrowser.db`, file di `/srv`, log ke stdout, tanpa baseURL.

### 7.3 Healthcheck

- **alpine** (`docker/alpine/healthcheck.sh`): baca `PORT`/`ADDRESS` dari env `FB_PORT`/`FB_ADDRESS` atau parse `/config/settings.json` via `JSON.sh`; lalu `wget -q --spider http://$ADDRESS:$PORT/health`.
- **common** (`docker/common/healthcheck.sh`): sama namun pakai `jq -r .port/.address`.

---

## 8. Orkestrasi — `compose.yaml`

`compose.yaml` mendefinisikan dua service dalam satu network + volume:

- **`filebrowser`**:
  - `image: filebrowser/filebrowser:latest`
  - `ports: 8000:80` (host:container)
  - `volumes: filebrowser:/flux/vault` — catatan: di fork m0nsky, **host volume `/flux/vault`** dialihkan ke volume bernama `filebrowser`. Ini bagian dari desain internal m0nsky (disk vault terpusat).
  - `environment: FB_REDIS_CACHE_URL=redis://default:filebrowser@redis:6379` — memakai **Redis shared upload cache** (multi-instance), komentar `rediss://` untuk SSL.
- **`redis`**: `image: redis:latest`, command menulis **ACL file** yang mendefinisikan user `default` dengan password `filebrowser` dan izin `~* +@all`, lalu `redis-server --aclfile`. Ini mengamankan Redis agar tidak akses tanpa kredensial.
- `networks: filebrowser`, `volumes: filebrowser`.

Ini mencontohkan deployment **multi-instance** (lebih dari satu server filebrowser) berbagi cache upload Redis (lihat `--redisCacheUrl`), berbagi storage `/flux/vault` untuk konsistensi TUS chunk antar instance.

---

## 9. Konfigurasi Server saat Run (`cmd/root.go`)

### 9.1 Server flags (persistent + runtime)

| Flag | Default | Fungsi |
|------|---------|--------|
| `--config`/`-c` | (none) | Path file config |
| `--database`/`-d` | `./filebrowser.db` | Path database Bolt |
| `--address`/`-a` | `127.0.0.1` | Bind address |
| `--port`/`-p` | `8080` | Bind port |
| `--cert`/`-t`, `--key`/`-k` | (none) | TLS cert/key (untuk TLS langsung) |
| `--root`/`-r` | `.` | Root path yang dilayani |
| `--socket` | (none) | Unix socket (tidak bisa dengan address/port/cert/key) |
| `--baseURL`/`-b` | (none) | Base URL (deploy di subpath) |
| `--socketPerm` | `0666` | Permission socket |
| `--cacheDir` | (none) | Cache file lokal (nonaktif jika kosong) |
| `--redisCacheUrl` | (none) | Redis cache URL (multi-instance) |
| `--imageProcessors` | `4` | Jumlah worker resize gambar |
| `--tokenExpirationTime` | `2h` | Masa sesi JWT |
| `--disableThumbnails`/`disablePreviewResize`/... | vary | Fitur switch |
| `--followExternalSymlinks` | `false` | Ikuti symlink keluar scope (unsafe) |

### 9.2 Precedence konfigurasi

```
Flags  >  Environment variables (FB_*)  >  Config file  >  Database values  >  Defaults
```

Hanya server-related options yang bisa dari config file/env; sisanya (users, permissions, branding, dll.) ada di database, diatur via `config set` / `config import`.

### 9.3 File config & lingkungan (`cmd/utils.go`)

- `initViper`:
  - Jika `--config` diberikan → gunakan file tsb (`viper.SetConfigFile`).
  - Jika tidak → cari `.filebrowser.{json,toml,yaml,yml}` di: `./`, `$HOME/`, `/etc/filebrowser/`.
  - **Variabel lingkungan**: prefix `FB_` (`viper.SetEnvPrefix("FB")`), `AutomaticEnv`, `SetEnvKeyReplacer` dengan `generateEnvKeyReplacements` — sehingga `--branding.disableExternal` bisa via `FB_BRANDING_DISABLE_EXTERNAL` (uppercase snake). Deprecated `FB_BASEURL` dipetakan ke `FB_BASE_URL`.
- Path database di-absolutkan; direktori dibuat bila perlu (`dbExists`).

### 9.4 Listener (TCP / TLS / socket)

Pilih oleh `switch` di `root.go`:
- `Socket != ""` → `net.Listen("unix", socket)` + chmod `socketPerm`.
- `TLSKey != "" && TLSCert != ""` → `tls.LoadX509KeyPair` + `tls.Listen` dengan `MinVersion: TLS1.2`.
- default → `net.Listen("tcp", address:port)`.
- Konflik `socket` vs `address/port/key/cert` → error.

### 9.5 Graceful shutdown

- `signal.Notify` untuk `SIGINT, SIGTERM, SIGHUP, SIGQUIT`.
- Setelah `http.Server.Serve(listener)` berjalan di goroutine, tunggu sinyal → `srv.Shutdown(ctx)` dengan timeout 10 detik → *graceful shutdown complete*. Ini penting untuk deployment (orchestrator mengirim SIGTERM saat rolling update/stop).

### 9.6 Logging (`setupLog`)

`--log`:
- `stdout` / `stderr` / kosong (io.Discard).
- Path file → **lumberjack rotation**: MaxSize 100MB, MaxAge 14 hari, MaxBackups 10. Cocok untuk long-running deployment.

---

## 10. Bootstrap Database & Migrasi Config

- **Quick setup** (root.go): jika database tidak ada → `quickSetup` membuat database baru, user pertama, default settings. Flag `--noauth`, `--username`, `--password`, defaults permission (admin=true untuk user pertama). Mode ini penting untuk deployment pertama kali.
- **`filebrowser config init`** — inisialisasi config eksplisit (menyimpan auther ke DB). `config_cat`, `config_set`, `config_import`, `config_export` untuk melihat/mengedit/backup/pindah config (lihat Layer 3 untuk detail CLI).
- `version.go` / `version/version.go`: `filebrowser version` mencetak `File Browser v{Version}/{CommitSHA}` (diisi ldflags saat build).

---

## 11. Deployment Frontend / Subpath (`vite.config.ts`)

- Build production menggunakan template `[{[ .StaticURL ]}]` — asset URL di-template server saat render HTML (untuk `renderBuiltUrl`), memungkinkan deploy di **baseURL/subpath** tanpa hardcode path.
- `manualChunks`: `dayjs` dan `i18n` dipisah jadi chunk terpisah untuk caching.
- `compression` plugin untuk `.js` (gzip).
- Dev server mem-proxy `/api` → `127.0.0.1:8080` dan `/api/command` (WS) — hanya untuk pengembangan lokal, tidak dipakai di produksi.

---

## 12. Strategi Deployment (Panduan Utama)

### A. Biner langsung (VM / cloud / bare metal)

```sh
# 1. Bangun biner
task build        # membangun frontend + backend

# 2. Jalankan pertama kali (quick setup → buat DB + user admin)
./filebrowser --address 0.0.0.0 --port 8080 --root /srv

# Untuk restart, jadikan service (systemd) — jalankan tanpa quick setup lagi.
```
- Simpan DB (`filebrowser.db`), config, dan root data sebagai volume persisten.
- Reverse proxy memidi jwt auth; gunakan `--baseURL` bila di subpath, atau ProxyAuth di belakang gatekeeper.

### B. Docker standalone

```sh
docker run -d \
  -p 8080:80 \
  -v /my/srv:/srv \
  -v /my/config:/config \
  -v /my/db:/database \
  filebrowser/filebrowser:latest
```
- Image alpine non-root (user id 1000). Pastikan permission host volume sesuai.
- Gunakan varian `-s6` bila menginginkan integrasi LinuxServer (cap_net_bind_service, service manager).

### C. Docker Compose (multi-instance + Redis)

```sh
docker compose up -d
```
- Menjalankan `filebrowser` (port 8000) + `redis` (dengan ACL password).
- Cocok bila >1 instance berbagi cache upload TUS (Redis) dan storage terpusat `/flux/vault`.

### D. Di belakang reverse proxy / TLS terminator

- Selesaikan TLS di proxy (Nginx/Caddy/Traefik); aplikasi `--port 8080` HTTP.
- `--baseURL` jika di subpath (mis. `/files`).
- Auth: JSONAuth (form) atau ProxyAuth (`--auth.header`) jika proxy mengasert identitas.
- Pastikan `followExternalSymlinks` tetap `false` kecuali benar-benar diperlukan (risk scope escape).

---

## 13. Keamanan Hosting (Checklist)

1. **Non-root** di container & drop user di s6 (`USER user` / `s6-setuidgid abc`).
2. **TLS min TLS1.2** bila memakai TLS langsung (`tini`/cert); sebaiknya terminasi TLS di proxy.
3. **Tidak mengekspose `--socket`** tak terpakai; jika dipakai, batasi `--socketPerm`.
4. **Command Runner** (`--disableExec`) nonaktif secara default (`true`); hanya aktifkan dengan pemahaman risiko (known vulns — lihat `WARNING` di root.go).
5. **`--followExternalSymlinks`** `false` (default) — mencegah scope escape.
6. **Signup**: jika diaktifkan, hati-hati — root.go memperingatkan risiko bila `Signup && !CreateUserDir && scope=root`.
7. **Volume persistence**: `/srv`, `/config`, `/database` terpisah untuk backup & recovery.
8. **Redis** dilindungi ACL (password) — jangan share tanpa kredensial.
9. **Backup config/user** via `config export` / `config import` untuk disaster recovery & pindah instance.
10. **Healthcheck** `/health` untuk orkestrator (rollout sehat).

---

## 14. Catatan Pengembangan (Sesuai PRD m0nsky)

Roadmap menambah **Podman dashboard/manager**, **website deployer**, dan **SSH tunneling** — semuanya memengaruhi lapisan hosting:

1. **Podman manager** — aplikasi akan mengelola container lain; butuh integrasi dengan daemon/API Podman, manajemen image/volume/network, dan guard authorization baru (Layer 4).
2. **Website deployer** — proses deploy harus menjalankan build/hook script dari binary/container; pertimbangkan memakai script `init.sh`/healthcheck sebagai pola untuk lifecycle custom container.
3. **SSH tunneling** — remote access; pertimbangkan memakai listener custom & `--socket`-like pattern; pastikan hanya aktif bila dikonfigurasi admin.
4. Setiap service baru harus menerapkan pola **non-root + healthcheck + graceful shutdown** yang sama agar konsisten dengan `Dockerfile`/`root.go`.
5. `compose.yaml` dapat diperluas menjadi template layanan (filebrowser + storage + service pendukung).

---

## 15. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 5: Hosting & Deployment (navigator).
- `docs/layers/01-frontend.md` — build SPA Vite, `StaticURL` template, embed `frontend/dist`.
- `docs/layers/02-database-storage.md` — BoltDB single file `/database/filebrowser.db`, quick setup bootstrap.
- `docs/layers/03-backend-api.md` — `/health` endpoint, config CLI subcommands.
- `docs/layers/04-auth-authorization.md` — auth method config (JSON/Proxy/Hook/NoAuth) saat deployment.
- `docs/PRD.md` — roadmap Podman/website deployer/SSH tunnel yang memperluas lapisan ini.
- `docs/AGENTS.md` — workflow pengembangan branch `m0nskyBuildAI`.
