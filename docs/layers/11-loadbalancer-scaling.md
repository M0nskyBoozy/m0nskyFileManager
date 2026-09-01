# Layer 11 — Load Balancer & Scaling

> Dokumen teknis mendalam untuk lapisan **Load Balancer & Scaling** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 11).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan **Load Balancer & Scaling** menangani **distribusi beban** dan **penskalaan** aplikasi seiring bertambahnya pengguna/beban.

Sesuai `ARCHITECTURE.md` Layer 11, tanggung jawab:
1. Mendistribusikan request ke beberapa instance aplikasi.
2. Menyinkronkan **state antar-instance** (terutama cache & session).
3. (Roadmap) Mendukung **horizontal scaling** pada beban tinggi.

Komponen & teknologi:
- **Reverse proxy / load balancer** (nginx, traefik, dsb.) — *external*.
- **Redis** untuk state ter-distribusi (cache/upload cache).
- **Stateless design** pada aplikasi Go (state di BoltDB + Redis).

> **STATUS**: Aplikasi **sudah didesain stateless & scale-friendly**, namun **arsitektur scaling penuh (multi-instance aktif, session affinity, konfigurasi LB)** belum diimplementasi sebagai komposisi (roadmap). Dokumen ini menjabarkan fondasi nyata yang memungkinkan scaling, plus rencana arsitektur LB.

---

## 2. Prinsip Stateless Design (fondasi scaling)

Aplikasi `m0nskyFileManager` merupakan server **stateless per-request** — state persisten disimpan di luar memori process, sehingga **beberapa instance dapat berjalan paralel** di belakang load balancer tanpa berbagi memori.

```
           ┌── frontend (embedded SPA, stateless)
           │
 Request ──┼──▶ Instance A (filebrowser:80)  ─┐
 LB        ┼──▶ Instance B (filebrowser:80)  ─┼──▶ BoltDB  (/database)   [single-file shared via volume/BD]
           │                                  └──▶ Redis   (cache shared)  ─┐
           └── Instance C (filebrowser:80)                                  │
                                                                            ▼
                                                        upload cache TUS (multi-instance)
```

### 2.1 State yang ter-distribusi / disimpan

| State | Lokasi | Scaling-friendly |
|-------|--------|------------------|
| User, settings, share link, dsb. | **BoltDB** (`/database/filebrowser.db`) | Ya (single-file; perlu shared storage NFS/nas bila multi-instance) |
| Upload cache TUS | **Redis** (`--redisCacheUrl`) | Ya — dirancang khusus multi-instance |
| Thumbnail/preview cache | **disk** (`--cacheDir`, lokal per-instance) | Ya — regenerable (cache hanya percepatan) |
| Auth session/JWT | **Web token (JWT)** client-side | Ya — token dibawa client, tidak tersimpan server |
| Real-time WebSocket | **runner/commands** (per-koneksi) | Perlu **stickiness** (lihat §6) |

> **Catatan penting**: Untuk **reporting/upload besar** (WebSocket & upload panjang), load balancer sebaiknya memakai **session affinity (sticky)** atau memastikan WebSocket ter-routing konsisten, karena koneksi WS terikat instance.

---

## 3. Redis untuk State Ter-Distribusi

Redis adalah kunci untuk multi-instance. Terdiri atas:

- **`http/upload_cache_redis.go`** — `redisUploadCache`:
  - Key `filebrowser:upload:<path>`, `SET/GET/DEL/EXPIRE` dengan TTL `3 min`.
  - Dipilih otomatis saat `--redisCacheUrl` di-set (`NewUploadCache`).
  - Memungkinkan **slice TUS upload** dari beberapa instance berbagi state ukuran upload → instance A mulai upload, instance B lanjut part berikutnya.
- **Komposisi** di `compose.yaml`:
  ```yaml
  filebrowser:
    image: filebrowser/filebrowser:latest
    environment:
      - FB_REDIS_CACHE_URL=redis://default:filebrowser@redis:6379
  redis:
    image: redis:latest
    command: ... redis-server --aclfile ...   # ACL password
  ```
- Dependency: `github.com/redis/go-redis/v9 v9.21.0` (sudah ada di go.mod).

> Dengan Redis, **beberapa instance filebrowser dapat berbagi cache upload** tanpa konflik — prasyarat scaling aktif.

---

## 4. Identitas di Belakang Proxy / Load Balancer

### 4.1 ProxyAuth (`auth/proxy.go`)

```go
type ProxyAuth struct { Header string `json:"header"` }
func (a ProxyAuth) Auth(r *http.Request, usr ...) {
	username := r.Header.Get(a.Header)  // mis. X-Remote-User
	user, err := usr.Get(..., username)
	if errors.Is(err, fberrors.ErrNotExist) { return a.createUser(...) }
	...
}
```

- Metode `proxy` (`MethodProxyAuth`): **identitas diambil dari header** (mis. `X-Remote-User`), di-set oleh load balancer/SSO/IdP di depan aplikasi.
- `--auth.header` mengonfigurasi nama header.
- **BSangat cocok untuk LB**: LB meng-autentikasi user, lalu meneruskan identitas; aplikasi tidak perlu sesi server-side.
- User baru di-provision otomatis (password acak, `LockPassword=true`, **bukan** admin/execute — hardening).

### 4.2 realip — IP client asli (logging)

`http/data.go:107`:
```go
clientIP := realip.FromRequest(r)
log.Printf("%s: %v %s %v", r.URL.Path, status, clientIP, err)
```
- Dependency `github.com/tomasen/realip` (go.mod).
- `realip.FromRequest` membaca IP dari header proxy (`X-Forwarded-For` dll.) → log menampilkan **IP client asli**, bukan IP LB. Krusial untuk observability & rate limiting per-IP (Layer 9) di deployment bertingkat.

---

## 5. Healthcheck untuk LB (`/health`)

```go
func healthHandler(w http.ResponseWriter, _ *http.Request) {
	w.WriteHeader(http.StatusOK)
	_, _ = w.Write([]byte(`{"status":"OK"}`))
}
```

- Endpoint publik `GET /health` → `200 {"status":"OK"}`.
- Di-root `r.HandleFunc("/health", healthHandler)` (`http/http.go:45`) — tidak butuh auth.
- Digunakan load balancer / orchestrator untuk **health probe** (menentukan instance mana yang menerima traffic).
- Healthcheck container juga memanggil endpoint ini (`docker/{alpine,common}/healthcheck.sh` → `wget .../health`, Layer 5).

---

## 6. WebSocket & Session Affinity (`/api/command`)

- `http/commands.go` — handler WebSocket untuk command runner (terminal).
- WebSocket adalah **koneksi panjang stateful per-instance** → load balancer perlu **rasa `Upgrade`/`Connection: upgrade`** (HTTP/1.1 upgrade) dan **session affinity** untuk command streaming.
- Namun auth **JWT stateless** tetap ada di header `X-Auth` (client-side), sehingga request HTTP biasa bisa di-round-robin tanpa sticky.

---

## 7. Konfigurasi Bind & Deploy untuk Scaling (`cmd/root.go`)

- `--address` / `--port` → bind listener. Di deployment LB, instance biasanya bind `127.0.0.1:8080` (internal) dan **tidak di-expose publik**; LB yang expose.
- `--socket` → Unix socket untuk proxy lokal.
- Semua flag dapat diset via **env `FB_*`** (Layer 5) → pemberian konfigurasi identik per-instance mudah (mis. `FB_REDIS_CACHE_URL`, `FB_AUTH_HEADER`, `FB_BASE_URL`).
- `tokenExpirationTime` → JWT expiry; konsisten antar instance (kalau beda → token valid) — signature pakai `settings.Key` yang **harus sama** antar instance (dari DB yang sama).

---

## 8. Arsitektur Load Balancer (Rekomendasi/Roadmap)

```
                 User
                  │ (HTTPS, public)
                  ▼
        ┌─────────────────┐
        │  Load Balancer   │  ←─ nginx / traefik / Caddy / cloud LB
        │  (TLS termination)│     healthcheck: GET /health
        │  X-Forwarded-*   │     realip untuk log
        └────────┬─────────┘
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
  Instance A   Instance B   Instance C   (filebrowser:80, bind internal)
      │          │          │
      └─────┬────┴─────┬────┘
            │          │
      ┌─────▼────┐  ┌──▼────────┐
      │  BoltDB   │  │  Redis     │  (shared state / upload cache)
      │ (shared)  │  │ (cache)    │
      └──────────┘  └───────────┘

Routing strategy:
  - REST API           : round-robin / least-conn (stateless, JWT)
  - /api/command (WS)  : sticky session / IP-hash (atau special route)
  - /static, /preview  : cache (public max-age) → bisa di-CDN (Layer 10)
```

### Strategi routing yang disarankan
- **REST** (`/api/*`): round-robin / least-conn — aman karena JWT & Redis state.
- **WebSocket** (`/api/command`): **sticky** (IP-hash/cookie) agar koneksi tidak putus antar-instance.
- **Login/proxy**: jika pakai ProxyAuth, pastikan LB yang set header identitas dengan benar (hapus header dari client, jangan trust sembarangan).

---

## 9. Horizontal vs Vertical Scaling

| Tipe | Deskripsi | Kebutuhan |
|------|-----------|-----------|
| **Vertical** | Naik CPU/RAM/memori satu instance | Sederhana; cek `imageProcessors`, cache |
| **Horizontal** | Tambah instance di belakang LB | **Redis** (upload cache), **BoltDB shared** (volume), **JWT key sama**, healthcheck `/health` |

### Prasyarat horizontal scaling
1. Semua instance **berbagi DB** (BoltDB file di volume bersama — perlu perhatian pada single-file write concurrency; alternatif gunakan satu writer / NFS dengan lock).
2. Semua instance **berbagi Redis** untuk upload cache (`FB_REDIS_CACHE_URL`).
3. **`settings.Key`** (JWT signing) diambil dari DB yang sama → identik → token lintas-instance valid.
4. Probe `/health` di tiap instance.
5. (Opsional) `--cacheDir` lokal per-instance untuk thumbnail (regenerable), atau simpan ke volume bersama.

---

## 10. Keamanan Load Balancer

1. **TLS termination** di LB; instance bind internal (jangan expose publik).
2. **ProxyAuth**: jangan pernah mempercayai header identitas dari client; LB harus **menimpa/menghapus** header sebelum diteruskan.
3. **realip**: hati-hati spoofing `X-Forwarded-For`; pastikan hanya LB tepercaya yang menambahkan.
4. **Redis** dengan ACL (password) — jangan expose Redis publik.
5. Health endpoint aman (`/health` bebas auth, tidak membocorkan data sensitif).

---

## 11. Unit Tests & Catatan Roadmap

- `http/upload_cache_memory_test.go` — cache upload (basis single/sharded).
- Test Redis (roadmap): multi-instance berbagi upload cache.
- (Roadmap) implementasi penuh: session affinity, konfigurasi LB template, roadmap horizontal scaling.

---

## 12. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 11: Load Balancer & Scaling (navigator & roadmap).
- `docs/layers/10-cache-cdn.md` — Redis upload cache, static/CDN caching.
- `docs/layers/05-hosting-deployment.md` — compose.yaml, bind address, healthcheck, env FB_*.
- `docs/layers/04-auth-authorization.md` — JWT stateless, ProxyAuth, `settings.Key`.
- `docs/layers/09-rate-limiting.md` — realip & per-IP limiting di balik LB.
- `docs/layers/13-backend-static.md` / Layer 1 — frontend embedded (stateless, siap CDN).
- `docs/PRD.md` — roadmap scaling & infrastruktur.
- `docs/AGENTS.md` — workflow branch `m0nskyBuildAI`.
