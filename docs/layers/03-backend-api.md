# Layer 3 — Backend Logic & API

> Dokumen teknis mendalam untuk lapisan **Backend Logic & API** dari `m0nskyFileManager`.
> Ini adalah dokumen *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 3).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan ini adalah **HTTP layer (REST API) + logika bisnis backend** — jembatan antara Frontend (Layer 1) dan Database/Storage (Layer 2). Berisi:

- **Router & routing** (Gorilla Mux) — definisi seluruh endpoint.
- **Middleware & auth guard** — JWT, admin, permission, rule checking.
- **Handlers** — logika tiap endpoint (files, users, settings, share, upload TUS, search streaming, terminal WebSocket, preview, raw download/archive, static assets).
- **Service pendukung** — image processing (`img`), disk cache (`diskcache`), upload cache (memory/redis), search engine, file operations utility (`fileutils`), runner (hooks).

Dibangun dengan **Go 1.25**, menggunakan:
- `github.com/gorilla/mux` — router.
- `github.com/golang-jwt/jwt/v5` — autentikasi token JWT HS256.
- `github.com/gorilla/websocket` — terminal (exec) WebSocket.
- `github.com/redis/go-redis/v9` — upload cache multi-instance (opsional).
- `github.com/disintegration/imaging` + `go-exif` — image resize.
- `github.com/mholt/archives` — kompresi/arsip (zip/tar/...).
- `github.com/asticode/go-astisub` — konversi subtitle.

---

## 2. Arsitektur & Alur Request

```
 Browser/Frontend (Vue SPA)
   │  HTTP (JSON / WebSocket / TUS / streaming)
   ▼
┌──────────────────────────────────────────────────────────────┐
│ http.NewHandler(...)  (http/http.go)                        │
│  1. server.Clean() + deteksi CaseInsensitiveFs               │
│  2. mux.NewRouter() + CSP header middleware                  │
│  3. Route: /health, /static, /{index}                        │
│  4. /api subrouter: login, users, resources, tus, share,     │
│     settings, raw, preview, command, search, subtitle, public│
│  5. stripPrefix(BaseURL)                                     │
└───────────────────────────┬──────────────────────────────────┘
                            │ handle(fn, prefix, store, server)
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ http/data.go: handle() wraps handler                        │
│  - set globalHeaders (Cache-Control)                         │
│  - load Settings dari store                                  │
│  - buat &data{Runner, store, settings, server}               │
│  - panggil fn(w,r,d) → (status, err)                         │
│  - log error + tulis HTTP status                            │
└───────────────────────────┬──────────────────────────────────┘
                            │ Guard middleware (dalam handler)
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ withUser / withAdmin / withSelfOrAdmin / withPermShare       │
│  • JWT verify via X-Auth header / cookie                      │
│  • load user → d.user (Clean → setup Fs)                      │
│  • canonicalizeRequestPath                                     │
└───────────────────────────┬──────────────────────────────────┘
                            ▼  akses data & logika bisnis
┌──────────────────────────────────────────────────────────────┐
│ store (storage.Storage)  +  d.user.Fs (afero/ScopedFs)       │
│ helper: files.*, fileutils.*, runner.RunHook, disk/usage     │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Struktur File Backend Logic & API (Peta Lengkap)

### 3.1 Paket `http/` — HTTP handlers & routing (inti lapisan ini)

```
http/
├── http.go                  # NewHandler(): router Mux + definisi SEMUA endpoint
├── data.go                  # type data, handle() wrapper, rules Check/CheckRules
├── auth.go                  # JWT login/renew/signup, withUser/withAdmin, token
├── users.go                 # CRUD user (GET/POST/PUT/DELETE /api/users)
├── settings.go              # GET/PUT /api/settings (admin)
├── share.go                 # share management (list/gets/post/delete)
├── public.go                # share publik (dl/share via hash) + health
├── resource.go              # operasi file: GET/DELETE/POST/PUT/PATCH + recursive + usage
├── raw.go                   # download raw file / arsip (zip/tar/...) + query algo
├── preview.go               # preview image (thumb/big) via imgSvc + fileCache
├── preview_enum.go          # enum PreviewSize (go-enum generated)
├── commands.go              # terminal WebSocket (exec command)
├── search.go                # pencarian streaming (NDJSON + heartbeat)
├── subtitle.go              # konversi subtitle → VTT
├── tus_handlers.go          # TUS upload: POST/HEAD/PATCH/DELETE
├── upload_cache_memory.go   # UploadCache interface + implementasi in-memory (ttlcache)
├── upload_cache_redis.go    # UploadCache implementasi Redis (multi-instance)
├── static.go                # serve index.html (inject config) + static assets (gzip)
├── headers.go               # globalHeaders (dbuild tag !dev)
├── utils.go                 # slashClean, renderJSON, errToStatus, stripPrefix
└── *_test.go                # unit tests (auth, raw, resource, share, tus, public, dll.)
```

### 3.2 Paket pendukung (service/business logic)

```
img/
├── service.go               # ImgService: resize image (imaging+exif), semaphore worker
├── service_enum.go          # enum Format (go-enum) 
└── testdata/                # fixture gambar uji

diskcache/
├── cache.go                 # Interface: Store/Load/Delete
├── file_cache.go            # FileCache: cache thumbnail ke disk (SHA1 key, scoped lock)
├── noop_cache.go            # NoOp: cache non-aktif (disable)
└── file_cache_test.go

fileutils/
├── file.go                  # MoveFile, CopyFile, CommonPrefix
├── copy.go                  # Copy (recursive) 
├── dir.go                   # util direktori
└── *_test.go

search/
├── search.go                # engine pencarian (afero.Walk + conditions + terms)
└── conditions.go            # parser query → conditions/terms

runner/
├── runner.go                # Runner: RunHook (before_/after_ events), exec
├── commands.go              # parse command (Unix/Windows shlex)
├── parser.go                # ParseCommand (shell wrap)
└── *_test.go
```

### 3.3 Paket domain (dipakai handler — lapisan backend/data)

```
files/        # FileInfo, Listing, sorting, mime, ScopedFs (representasi file & fs)
users/        # model User, Permissions, password (bcrypt), storage
settings/     # model Settings/Server/Tus/Branding/UserDefaults, storage
share/        # model Link, storage
auth/         # interface Auther + 4 implementasi (json/proxy/hook/noauth)
storage/      # composite Storage + backend BoltDB (lihat Layer 2)
rules/        # Rule allow/deny + Checker
version/      # version.Version
```

### 3.4 Entry & server startup

```
main.go       # main → cmd.Execute()
cmd/root.go   # Build server: init storage, img service, fileCache, uploadCache,
              #   listener (socket/TLS/TCP), assetsFs → NewHandler → http.Server
```

---

## 4. Router & Pendaftaran Endpoint (`http/http.go`)

`NewHandler(imgSvc, fileCache, uploadCache, store, server, assetsFs)`:

- `server.Clean()` + set `server.CaseInsensitiveFs` dari `files.CaseInsensitive(os, root)`.
- `mux.NewRouter()`; middleware global set **Content-Security-Policy** `default-src 'self'; style-src 'unsafe-inline';`.
- `index, static := getStaticHandlers(...)`.
- `monkey(fn, prefix)` = `handle(fn, prefix, store, server)` → membungkus handler dengan `data` + error handling.
- `r.NotFoundHandler = index` → semua route tak dikenal dilayani SPA index (untuk frontend routing).

### 4.1 Daftar endpoint API lengkap

| Method | Path | Handler | Auth | Keterangan |
|--------|------|---------|------|------------|
| GET | `/health` | `healthHandler` | publik | health check `{"status":"OK"}` |
| GET | `/static` | `static` | publik | asset statis (gzip) |
| POST | `/api/login` | `loginHandler` | publik | login → JWT |
| POST | `/api/signup` | `signupHandler` | publik (jika signup) | registrasi |
| GET | `/api/renew` | `renewHandler` | user | perbarui token |
| GET | `/api/users` | `usersGetHandler` | admin | daftar user |
| POST | `/api/users` | `userPostHandler` | admin | buat user |
| GET | `/api/users/{id}` | `userGetHandler` | self/admin | detail user |
| PUT | `/api/users/{id}` | `userPutHandler` | self/admin | update user (partial) |
| DELETE | `/api/users/{id}` | `userDeleteHandler` | self/admin | hapus user |
| GET | `/api/resources/recursive/...` | `resourceGetRecursiveHandler` | user | listing recursive |
| GET | `/api/resources/...` | `resourceGetHandler` | user | list/file info + checksum |
| POST | `/api/resources/...` | `resourcePostHandler` | user | upload/buat dir |
| DELETE | `/api/resources/...` | `resourceDeleteHandler` | user | hapus |
| PUT | `/api/resources/...` | `resourcePutHandler` | user | tulis ulang file (save) |
| PATCH | `/api/resources/...` | `resourcePatchHandler` | user | copy/rename/move |
| POST | `/api/tus/...` | `tusPostHandler` | user | inisiasi upload |
| HEAD/GET | `/api/tus/...` | `tusHeadHandler` | user | status offset/length |
| PATCH | `/api/tus/...` | `tusPatchHandler` | user | kirim chunk |
| DELETE | `/api/tus/...` | `tusDeleteHandler` | user | batalkan upload |
| GET | `/api/usage/...` | `diskUsage` | user | statistik disk |
| GET | `/api/shares` | `shareListHandler` | perm.share | daftar share |
| GET | `/api/share/...` | `shareGetsHandler` | perm.share | share per path |
| POST | `/api/share/...` | `sharePostHandler` | perm.share | buat share |
| DELETE | `/api/share/...` | `shareDeleteHandler` | perm.share | hapus share |
| GET | `/api/settings` | `settingsGetHandler` | admin | baca settings |
| PUT | `/api/settings` | `settingsPutHandler` | admin | tulis settings |
| GET | `/api/raw/...` | `rawHandler` | user | download file / arsip |
| GET | `/api/preview/{size}/{path}` | `previewHandler` | user | preview image |
| GET | `/api/command/...` | `commandsHandler` | user + execute | terminal WebSocket |
| GET | `/api/search/...` | `searchHandler` | user | pencarian streaming |
| GET | `/api/subtitle/...` | `subtitleHandler` | user | subtitle → VTT |
| GET | `/api/public/dl/{hash}/...` | `publicDlHandler` | publik (hash) | download share publik |
| GET | `/api/public/share/{hash}/...` | `publicShareHandler` | publik (hash) | lihat share publik |
| `*` | (fallback) | `index` | — | SPA index.html |

---

## 5. Sistem Guard / Middleware Auth (`http/auth.go`, `http/users.go`, `http/share.go`)

### 5.1 Token & extractor

- **JWT HS256** (`jwt.SigningMethodHS256`), kunci `d.settings.Key` (dari DB, Layer 2).
- Claims `authToken{ User userInfo, jwt.RegisteredClaims }`; `Issuer: "File Browser"`.
- `extractor` membaca token dari:
  1. Header `X-Auth`.
  2. Cookie `auth` (hanya untuk method GET, kompatibilitas).
  - Hanya jika string berisi **2 titik** (format JWT valid; menghindari konflik basic auth lama).
- Parser: `jwt.WithValidMethods([HS256])`, `WithExpirationRequired()` (token wajib ada expiry).
- Default expiry: `DefaultTokenExpirationTime = time.Hour * 2`; dapat di-override via `Server.TokenExpirationTime`.

### 5.2 `withUser` (auth inti)

1. Verifikasi JWT (`/renew` dan request).
2. **`renewableErr`**: bila gagal hanya karena token *expired* dan metode = ProxyAuth (dan logout page bukan default), izinkan via `proxyAsserts` (proxy masih mengasert identitas yang sama) — **security guard**: token bocor yang sudah expired tidak bisa self-authenticate selamanya.
3. Set header `X-Renew-Token: true` bila:
   - `expiresSoon` (tersisa < 1 jam), ATAU
   - `updated` (user di-update sejak token diterbitkan — `store.Users.LastUpdate(id)`).
4. Muat user penuh via `store.Users.Get(root, followExt, id)` (ini mem-bangun `Fs`).

### 5.3 `withAdmin`

`withUser` + cek `d.user.Perm.Admin` → 403 bila bukan admin.

### 5.4 `withSelfOrAdmin`

`withUser` + userID dari path (`/users/{id}`); cek `d.user.ID == id || Admin`. `d.raw = id`.

### 5.5 `withPermShare`

`withUser` + cek `Perm.Share && Perm.Download` → 403 bila tidak.

### 5.6 canonicalizeRequestPath (di `withUser`)

Normalisasi path request ke bentuk virtual standar (lihat `utils.go`), agar rule checker, hooks, share record, dan cache key thumbnail **menggunakan string yang sama**. Trailing slash dipertahankan (handler membedakan `/dir` vs `/dir/`).

---

## 6. Handler Data Context & Rule Checking (`http/data.go`)

### 6.1 `type data`

```go
type data struct {
    *runner.Runner
    settings *settings.Settings
    server   *settings.Server
    store    *storage.Storage
    user     *users.User
    raw      interface{}        // nilai sementara antar-layer (id, FileInfo, ...)
    checkerPrefix string        // untuk public share (rebased path → rule path asli)
}
```

### 6.2 `handle()` wrapper

- Set `globalHeaders` (`Cache-Control: no-cache, no-store, must-revalidate`).
- `store.Settings.Get()` → settings (dengan defaults).
- Bangun `&data{Runner: {Enabled: server.EnableExec, Settings: settings}, store, settings, server}`.
- Panggil `fn(w, r, d)` → `(status, err)`.
- Log error (via `realip.FromRequest`).
- Bila `status != 0` → `http.Error` dengan teks status (+ deskripsi err untuk 400).

### 6.3 Rule Checking (`Check` / `CheckRules`)

- `Check(path)` → `HideDotfiles` (jika aktif, blokir hidden) **DAN** `CheckRules`.
- `CheckRules(path)`:
  - Normalisasi `rulePath` (slashClean + `checkerPrefix` join untuk public share).
  - Iterasi `d.settings.Rules` lalu `d.user.Rules`; tiap rule `Matches(path, CaseInsensitiveFs)` → set `allow = rule.Allow` (aturan terakhir menang).
  - Return `allow`.
- `rulePath` — memastikan route yang di-rebase (public share) tetap dicocokkan terhadap path scope asli user agar deny rules tak ter-bypass.

---

## 7. Handler Auth (`http/auth.go`)

### 7.1 `loginHandler`

- Max body 1 MiB (`maxAuthBodySize`).
- Ambil `Auther` sesuai `settings.AuthMethod`.
- `auther.Auth(r, store.Users, settings, server)` → user.
  - `os.ErrPermission` → 403.
  - sukses → `printToken`.
- `printToken` membangun `authToken` (userInfo) + mint & sign JWT, tulis sebagai plain text.

### 7.2 `signupHandler`

- Hanya jika `settings.Signup`; body max 1 MiB.
- Terapkan `settings.Defaults.Apply(user)`.
- **Security hardening (fork m0nsky)**: selalu set `Perm.Admin = false` dan `Perm.Execute = false` + `Commands = []` — user signup tidak pernah jadi admin & tidak mewarisi exe dari default.
- `ValidateAndHashPwd` (bcrypt + min length + common passwords).
- `settings.CreateUserHome` → buat home dir; `store.Users.SaveProvisioned` (guard kolisi scope).
- Duplikat → 409.

### 7.3 `renewHandler`

`withUser` + selalu set `X-Renew-Token: false` + cetak token baru.

---

## 8. Handler Users (`http/users.go`)

- `modifyUserRequest{modifyRequest, Data *users.User}`; `modifyRequest.What == "user"`.
- `usersGetHandler` (admin): `store.Users.Gets` → **zero-kan Password** → urutkan by ID.
- `userGetHandler` (self/admin): `Get` by ID; Password = ""; non-admin → Scope = "".
- `userDeleteHandler` (self/admin): wajib body `current_password`; bila JSONAuth → cek `users.CheckPwd`; `store.Users.Delete` (proteksi admin tunggal).
- `userPostHandler` (admin): verifikasi current password (JSONAuth), hash password, validasi `Perm.Share → Perm.Download`, `MakeUserDir`, `Save`.
- `userPutHandler` (self/admin): dukungan **partial update** (`Which` fields):
  - Sensitive fields (all/username/password/scope/lockPassword/commands/perm) → cek current password (JSONAuth).
  - `Perm.Share` tanpa `Download` → 400.
  - "all" (admin only): hash password bila diisi; isi password lama bila kosong.
  - Per-field: judul English `cases.Title`; `Password` → re-hash; guard `NonModifiableFieldsForNonAdmin` (Username, Scope, LockPassword, Perm, Commands, Rules) untuk non-admin.
- `withSelfOrAdmin` membatasi akses (self atau admin).

---

## 9. Handler Files/Resources (`http/resource.go`)

Semua dibangun di atas `files.NewFileInfo(&FileOptions{...})` yang memvalidasi, menerapkan rules, dan menyiapkan tipe/MIME.

### 9.1 `resourceGetHandler` (GET)
- File/dir info; dir → `ApplySort()` dari `user.Sorting`.
- `X-Encoding: true` + tipe text → stream isi file (dengan `Perm.Download`).
- `?checksum=` → hitung checksum (`file.Checksum`) dengan `Perm.Download`.

### 9.2 `resourceDeleteHandler` (DELETE)
- Tolak `/` atau tanpa `Perm.Delete`.
- `checkDescendants(d, path, "")` — validasi rules untuk **seluruh subtree** (cegah bypass rule via parent).
- Hapus share terkait: `store.Share.DeleteWithPathPrefix(path, userID)`.
- `delThumbs` — hapus thumbnail dari cache.
- `RunHook(..., "delete")` → `Fs.RemoveAll`.

### 9.3 `resourcePostHandler` (POST)
- Perlukan `Perm.Create` + `Check(path)`.
- Suffix `/` → `MkdirAll` (buat dir), hook `upload`.
- File: cek konflik (409 bila ada & tanpa `override`), `override` butuh `Perm.Modify`.
- `writeFile` + `RunHook(..., "upload")`; Set `ETag`.
- Error → `Fs.RemoveAll` rollback.

### 9.4 `resourcePutHandler` (PUT)
- `Perm.Modify` + `Check`; hanya file (bukan dir) → `MethodNotAllowed` bila dir.
- Wajib file ada → 404 bila tidak.
- `writeFile` + hook `save`; Set `ETag`.

### 9.5 `resourcePatchHandler` (PATCH) — copy/rename/move
- Query: `destination`, `action` (copy/rename), `override`, `rename` (auto version-suffix).
- Surah: `Check(src)` & `Check(dst)`, larang root, `checkParent` (cegah source jadi parent dst).
- Konflik handling: `override` (butuh Modify) / `rename` (auto suffix `name(1).ext`).
- `checkDescendants(src, dst)` — validasi seluruh subtree source & dst.
- `patchAction`:
  - `copy` → butuh `Perm.Create` → `fileutils.Copy`.
  - `rename` → butuh `Perm.Rename` → `delThumbs` + `fileutils.MoveFile`.
  - lain → `ErrInvalidRequestParams`.

### 9.6 `resourceGetRecursiveHandler` (recursive listing)
- Daftar **flat** seluruh file/dir di bawah path via `afero.Walk` (server-side, 1 HTTP call).
- Hormati konteks (stop bila client gone), hormati rules (`SkipDir` untuk dir terlarang), normalisasi path `filepath.ToSlash`.

### 9.7 `diskUsage` (usage)
- Untuk dir → `disk.UsageWithContext(RealPath)` → `{total, used}`.

### Helper `writeFile`
`MkdirAll` + `OpenFile(O_RDWR|O_CREATE|O_TRUNC)` + `io.Copy` + **`file.Sync()`** (cegah korupsi) + `Stat`.

---

## 10. Handler Raw & Arsip (`http/raw.go`)

### 10.1 `rawHandler`
- Butuh `Perm.Download`.
- Named pipe (`files.IsNamedPipe`) → set content-disposition saja.
- File → `rawFileHandler` (`http.ServeContent` + `script-src 'none'` + `nosniff`).
- Dir → `rawDirHandler` (arsip).

### 10.2 `rawDirHandler` (arsip)
- `parseQueryFiles` (`?files=a,b,c`).
- `parseQueryAlgorithm` (`?algo=zip|tar|targz|tarbz2|tarxz|tarlz4|tarsz|tarbr|tarzst`).
- `getFiles` rekursif membangun `[]archives.FileInfo`:
  - **Defense-in-depth arsip traversal**: nama dalam arsip dinormalisasi, `\` (legal di POSIX) → `_`, dan diverifikasi `gopath.Clean` tidak keluar root arsip.
  - Hormati `d.Check(path)`.
- `archiver.Archive(w, allFiles)`.

---

## 11. Handler Preview & Cache (`http/preview.go`, `diskcache/`)

### 11.1 `previewHandler`
- Hanya tipe `image` (lain → 501).
- `ParsePreviewSize(vars["size"])` → `thumb` (256x256) / `big` (1080x1080).
- Jika `big` & `!ResizePreview`, atau `thumb` & `!EnableThumbnails` → `rawFileHandler`.
- Format tak didukung / GIF → raw.
- Cache lookup via `fileCache.Load(previewCacheKey)`; miss → `createPreview` + store asynchronously.

### 11.2 `ImgService` (img)
- Resize dengan `imaging`, orientasi via EXIF, batasi dimensi `MaxImageWidth/Height=10000` (cegah crash server), semaphore worker (`imageProcessors`).
- `Format` enum: jpeg/png/gif/tiff/bmp.

### 11.3 `FileCache` (diskcache)
- `Interface{Store, Load, Delete}`.
- `FileCache` (disk): key → **SHA1** → path `first/next2/full-hash`; scoped lock per key; `MkdirAll 0700`.
- `NoOp` (disable): saat `cacheDir` kosong.

### 11.4 Cache key
`previewCacheKey = hex(RealPath)+hex(ModTime.Unix)+size`.

---

## 12. Handler Terminal WebSocket (`http/commands.go`)

- Upgrade HTTP → **WebSocket** (`gorilla/websocket`).
- Baca satu pesan teks (command) dari client.
- Fail-fast: `!server.EnableExec || !user.Perm.Execute` → kirim `Command not allowed.`
- `runner.ParseCommand` → pecah argument (shell wrap sesuai `Settings.Shell`).
- **Allowlist**: command **harus ada di `user.Commands`** (`slices.Contains`), jika tidak → `Command not allowed.`.
- `exec.Command`, `cmd.Dir = user.FullPath(path)` (scope-confined).
- Stream stdout+stderr line-by-line via WebSocket.
- `WSWriteDeadline = 10s`.

---

## 13. Handler Search Streaming (`http/search.go`, `search/`)

- `searchHandler`: jalankan `search.Search` di **goroutine**, tulis hasil per baris (NDJSON) ke response dengan **heartbeat** tiap 5s (cegah timeout koneksi) + `http.Flusher`.
- Setiap hasil `{dir, path}`.
- Pembatalan konteks pada error/abort user.
- `search.Search`: `afero.Walk` scope, hormati `checker`, cocokkan `Conditions` + `Terms` (case-sensitive configurable) pada nama file; kembalikan path relatif.

---

## 14. Handler Subtitle (`http/subtitle.go`)

- Butuh `Perm.Download`; dir → 400.
- Hanya subtitle file (`files.IsSupportedSubtitle`: `.vtt/.srt/.ass/.ssa`).
- `.srt` → `ReadFromSRT` (normalize `<br>` → newline via regex); `.ass/.ssa` → `ReadFromSSA`.
- Konversi ke **WebVTT** (`WriteToWebVTT`), serve `text/vtt`.
- `.vtt` asli → serve langsung.
- `script-src 'none'` + `nosniff`.

---

## 15. Handler Upload TUS (`http/tus_handlers.go`, `upload_cache_*`)

Protokol **Tus resumable upload** (chunked, offset-based).

### 15.1 `tusPostHandler` (POST) — inisiasi
- `Perm.Create` + `Check`.
- Buat file bila belum ada (mkdir parent); file ada & `override=true` → `O_TRUNC` (butuh `Perm.Modify`); tanpa override → 409.
- `Upload-Length` header wajib.
- `cache.Register(RealPath, length, remove)` — `remove` memakai **scoped Fs** (eviction tidak bisa symlink escape).
- Response `Location: /api/tus<path>` + `201`.

### 15.2 `tusHeadHandler` (HEAD)
- `Upload-Offset` (ukuran file saat ini), `Upload-Length` dari cache.

### 15.3 `tusPatchHandler` (PATCH)
- Content-Type harus `application/offset+octet-stream`.
- `Upload-Offset` harus cocok `file.Size` (409 bila tidak).
- `keepUploadActive` — touch cache tiap 2s (cegah eviction saat transfer).
- `io.Copy` dengan `io.LimitReader(body, remaining+1)` — **cegah over-length write**; bila lebih → `Truncate` rollback + 413.
- `file.Sync()`; set `Upload-Offset` baru.
- Selesai (`newOffset >= length`) → `cache.Complete` + hook `upload`.
- Rejected chunk (`status >= 400`) → `drainRequestBody` (baca sisa body agar koneksi reusable, max 32MB).

### 15.4 `UploadCache` (interface)
```go
type UploadCache interface {
    Register(filePath string, fileSize int64, remove func() error)
    Complete(filePath string)
    GetLength(filePath string) (int64, error)
    Touch(filePath string)
    Close()
}
```
- `memoryUploadCache`: `ttlcache` (TTL 3 mnt); saat expired → panggil `remove` (hapus partial file via scoped Fs).
- `redisUploadCache`: key `filebrowser:upload:<path>`; digunakan untuk **multi-instance** (`redisCacheUrl`); `remove` diabaikan (tidak hapus partial file).
- `NewUploadCache(redisURL)`: kosong → memory; isi → redis.

### 15.5 `tusDeleteHandler` (DELETE)
- `Perm.Delete`; tolak root; hapus file & `cache.Complete`.

---

## 16. Handler Static & SPA Inject (`http/static.go`)

### 16.1 `index` (SPA fallback)
- GET only; `x-xss-protection: 1; mode=block`, `nosniff`.
- `handleWithStaticData`:
  - Bangun `data` map (Name, Color, BaseURL, Version, StaticURL, Signup, NoAuth, AuthMethod, LogoutPage, LoginPage, CSS, ReCaptcha, Theme, EnableThumbs, ResizePreview, EnableExec, TusSettings, HideLoginButton).
  - Custom CSS bila `Branding.Files/custom.css` ada.
  - ReCaptcha dari `JSONAuth` bila aktif.
  - Marshal JSON → `data["Json"]` untuk `window.FileBrowser = ...`.
  - Eksekusi **Go template** `public/index.html` (delim `[{[`, `}]}`).
- Ini **jembatan ajektif** antara backend & frontend config (Layer 1 `window.FileBrowser`).

### 16.2 `static` (assets)
- Cache-Control `public, max-age=86400`; `nosniff`.
- Override branding `img/...` & `custom.css` dari `Branding.Files`.
- **JS files**: serve versi `.gz` bila client Accept-Encoding gzip (batch ter-kompresi saat build).

### 16.3 `globalHeaders`
`Cache-Control: no-cache, no-store, must-revalidate` (build tag `!dev`).

---

## 17. Handler Public Share (`http/public.go`)

### 17.1 `withHashFile` (middleware share publik)
- Parse `id` + subpath dari URL (`ifPathWithName`; guard index-out-of-range).
- `store.Share.GetByHash(id)`.
- `authenticateShareRequest`:
  - No password → izin.
  - Token query `?token=` → `subtle.ConstantTimeCompare`.
  - Header `X-SHARE-PASSWORD` → `bcrypt.CompareHashAndPassword`.
  - Gagal → 401.
- Muat pemilik share `store.Users.Get(link.UserID)`; butuh `Perm.Share && Perm.Download`.
- `d.user = pemilik`.
- **Rebase fs** ke `link.Path`: `d.user.Fs = files.NewFs(...)`; `d.checkerPrefix = basePath` — sehingga deny rules tetap berlaku pada scope asli, dan share juga **symlink-confined** (`ScopedFs`).
- Susun `FileInfo`; dir → nama dari basename.

### 17.2 `publicShareHandler`
- Dir → list (sortir by name desc); file → info.

### 17.3 `publicDlHandler`
- File → `rawFileHandler`; dir → `rawDirHandler` (arsip).

### 17.4 `healthHandler`
`GET /health` → `{"status":"OK"}` (default codep).

---

## 18. Service Pendukung Lain

### 18.1 `runner` (hooks)
- `Runner.RunHook(fn, evt, path, dst, user)`:
  - Jalankan `Commands["before_"+evt]`, lalu `fn()`, lalu `Commands["after_"+evt]`.
  - `evt` ∈ {save, copy, rename, upload, delete} (default di `settings.Save`).
- `exec`: parse command, perluas env mapping (`FILE`, `SCOPE`, `TRIGGER`, `USERNAME`, `DESTINATION`), blocking/non-blocking (suffix `&`).
- `ParseCommand`: tanpa `Shell` → panggil binary langsung; dengan `Shell` → bungkus.

### 18.2 `fileutils`
- `Copy` (rekursif), `MoveFile` (rename → fallback copy+remove), `CopyFile`, `CommonPrefix`.

### 18.3 `files` (representasi file)
`FileInfo`, `Listing`, `Sorting`, MIME/type, `ScopedFs` — dipakai semua handler untuk konsistensi rule + confinement (detail di Layer 2).

---

## 19. Server Startup & Wiring (`cmd/root.go`)

`withViperAndStore` → inisialisasi storage (Layer 2), lalu:

```go
imageService := img.New(imgWorkersCount)          // resize worker
var fileCache diskcache.Interface = diskcache.NewNoOp()
if cacheDir != "" { fileCache = diskcache.New(afero.NewOsFs(), cacheDir) }
uploadCache, _ := fbhttp.NewUploadCache(redisCacheURL)  // memory | redis
server := getServerSettings(v, st.Storage)
assetsFs, _ := fs.Sub(frontend.Assets(), "dist")
handler, _ := fbhttp.NewHandler(imageService, fileCache, uploadCache, st.Storage, server, assetsFs)
```

Listener: `Socket` (unix) / `TLS` (TLS 1.2+) / `TCP`. `http.Server{ReadHeaderTimeout: 60s}`.

---

## 20. Konvensi Handler (Pola `handleFunc`)

```go
type handleFunc func(w http.ResponseWriter, r *http.Request, d *data) (int, error)
```

- Handler mengembalikan `(status, err)`; `handle()` yang menulis header/status & log.
- `status == 0` berarti telah menulis respons sendiri (contoh `renderJSON`, streaming, WebSocket).
- `errToStatus(err)` memetakan error domain → HTTP:
  - `os.ErrPermission`/`ErrPermissionDenied` → 403
  - `os.IsNotExist`/`ErrNotExist` → 404
  - `os.IsExist`/`ErrExist` → 409
  - `ErrInvalidRequestParams` → 400
  - `ErrRootUserDeletion` → 403
  - `ErrImageTooLarge` → 413
  - default → 500

---

## 21. Keamanan di Lapisan Backend/API

1. **JWT HS256** dengan `settings.Key` (512-bit) & expiry wajib; `X-Renew-Token`.
2. **ProxyAuth renew** hanya via `proxyAsserts` (identitas sama) → cegah token bocor expired.
3. **Permission check di backend** (bukan hanya UI): `Perm.*` di-validasi di tiap handler.
4. **Rule allow/deny** global + user, diterapkan pada seluruh request (termasuk recursive & descendants) → anti bypass.
5. **ScopedFs** confinement + anti symlink-escape (termasuk dalam share & upload eviction).
6. **Path canonicalization** (`slashClean`) → mencegah perbedaan separator (Windows `\`) yang bisa menembus rule.
7. **Arsip traversal defense** (`raw.go`): normalisasi nama, cegah `..` escape.
8. **TUS over-length guard** + body drain.
9. **Signup tidak pernah admin/execute** (hardening fork).
10. **CSP header**, `nosniff`, `x-xss-protection`, `script-src 'none'` pada konten user.
11. `Upload-Length` dihormati; `file.Sync()` mencegah korupsi.

---

## 22. Unit Tests terkait (menunjukkan perilaku & registrasi)

| File | Cakupan |
|------|---------|
| `http/auth_test.go` | login/renew, proxy renew guard |
| `http/public_symlink_test.go`, `public_test.go` | share publik, symlink + auth |
| `http/raw_test.go` | download & arsip, traversal |
| `http/resource_test.go`, `resource_checksum_test.go`, `resource_recursive_test.go` | CRUD file, checksum, recursive |
| `http/rules_path_test.go`, `rules_recursive_test.go` | rule enforcement (path & subtree) |
| `http/share_test.go` | share management |
| `http/subtitle_test.go` | subtitle→VTT |
| `http/tus_*.test.go` | TUS multi-chunk, symlink, upload-length |
| `http/upload_cache_memory_test.go` | memory cache eviction |
| `http/utils_test.go` | slashClean/errToStatus |
| `fileutils/*_test.go`, `diskcache/*_test.go`, `search`/`img` tests | service layer |

---

## 23. Catatan Pengembangan (untuk perpanjangan — selaras PRD)

Untuk fitur PRD (Podman dashboard, website deployer, SSH tunnel), penambahan endpoint mengikuti pola:

1. Buat model + storage baru (Layer 2) jika ada data persist.
2. Buat service (mis. paket `podman`, `tunnel`) untuk logika bisnis murni.
3. Tambahkan handler di paket `http/` (modul handler baru), daftarkan di `http.go` dengan `monkey(...)`.
4. Alur WebSocket untuk terminal deployer/SSH — tiru pola `commands.go` (allowlist command, scope confinement, streaming).
5. Terapkan guard perm yang sesuai (`withUser`/`withAdmin`/perm baru), dan validasi rule/path via `data` context.

---

## 24. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 3: Backend Logic & API (navigator).
- `docs/layers/02-database-storage.md` — `storage.Storage`, model data & `files.ScopedFs`.
- `docs/layers/01-frontend.md` — sisi client & `window.FileBrowser` config (dari `static.go`).
- `docs/PRD.md` — kebutuhan fitur (podman/deployer/SSH) yang menambah endpoint.
- `docs/AGENTS.md` — panduan workflow (branch `m0nskyBuildAI`).
