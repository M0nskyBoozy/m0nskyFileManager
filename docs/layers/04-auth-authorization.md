# Layer 4 — Authentication & Authorization

> Dokumen teknis mendalam untuk lapisan **Authentication & Authorization** dari `m0nskyFileManager`.
> Ini adalah dokumen *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 4).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan ini mengatur **siapa yang boleh mengakses** (`authentication`) dan **apa yang boleh mereka lakukan** (`authorization`) pada `m0nskyFileManager`.

Dua konsep utama:

1. **Authentication** — memverifikasi identitas pengguna. Didukung **4 metode auth** (plugin `Auther`):
   - `JSONAuth` (`json`) — login via form JSON (username/password + opsional reCAPTCHA).
   - `ProxyAuth` (`proxy`) — identitas dipercaya dari header reverse proxy.
   - `HookAuth` (`hook`) — verifikasi via command/script eksternal (SSO/kustom).
   - `NoAuth` (`noauth`) — tanpa autentikasi (autentikasi sebagai user ID 1).

2. **Authorization** — menentukan hak akses pengguna. Terdiri atas:
   - **Permissions** per user (`users.Permissions`): admin, execute, create, rename, modify, delete, share, download.
   - **Rules** allow/deny per path (`rules.Rule`, global & per-user).
   - **Scope confinement** (`ScopedFs`) — batas filesystem yang dapat diakses.
   - **Guard middleware** HTTP (`withUser`, `withAdmin`, `withSelfOrAdmin`, `withPermShare`) + `withHashFile` untuk share publik.

Output autentikasi sukses adalah **JWT token** (HS256) yang dibawa client pada request berikutnya.

---

## 2. Arsitektur Auth & Authorization (Alur)

```
 Client request (dengan / tanpa token)
   │
   ▼
┌──────────────────────────────────────────────────────────────┐
│ AUTHENTICATION                                               │
│  Endpoint publik: /api/login, /api/signup, /api/renew        │
│                                                              │
│  store.Auth.Get(AuthMethod) → Auther                         │
│  auther.Auth(req, store.Users, settings, server)             │
│    ├─ JSONAuth: decode JSON → (reCAPTCHA?) → bcrypt verify   │
│    ├─ ProxyAuth: header → Get user | auto-provision          │
│    ├─ HookAuth:  JSON → exec hook → action(auth/block/pass)  │
│    └─ NoAuth:    selalu user ID 1                            │
│  → sukses: printToken (JWT HS256)                           │
└──────────────────────────────────────────────────────────────┘
   │ JWT (X-Auth header / cookie)
   ▼
┌──────────────────────────────────────────────────────────────┐
│ AUTHORIZATION (middleware per request)                       │
│  withUser        → verify JWT, muat user (Clean→Fs),         │
│                    canonicalize path                          │
│  withAdmin        → + Perm.Admin                              │
│  withSelfOrAdmin  → + ID == id OR admin                       │
│  withPermShare    → + Perm.Share && Perm.Download             │
│  withHashFile     → share publik (hash+password/token)        │
│  data.Check()/CheckRules() → Rules allow/deny                 │
│  ScopedFs confine → path di luar scope ditolak                │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Struktur File Authentication & Authorization (Peta Lengkap)

### 3.1 Paket `auth/` — plugin autentikasi

```
auth/
├── auth.go          # interface Auther (Auth + LoginPage) — kontrak semua metode
├── json.go          # JSONAuth (MethodJSONAuth="json") + ReCaptcha
├── proxy.go         # ProxyAuth (MethodProxyAuth="proxy") + auto-provision
├── hook.go          # HookAuth (MethodHookAuth="hook") + hookFields parser
├── none.go          # NoAuth (MethodNoAuth="noauth")
├── storage.go       # auth.Storage (wrapper backend + reference user store)
└── *_test.go        # proxy_test.go, hook_test.go (security & isolation)
```

### 3.2 Paket `users/` — model user, permission, password

```
users/
├── users.go         # model User, Clean(), FullPath(), Rules (GetRules)
├── permissions.go   # type Permissions (8 boolean hak)
├── password.go      # bcrypt hash/compare, ValidateAndHashPwd, RandomPwd
├── storage.go       # users.Storage (Get/GetByScope/Gets/Update/Save/SaveProvisioned/Delete/LastUpdate)
├── assets.go        # embed common-passwords.txt
├── assets/
│   └── common-passwords.txt   # daftar password umum (anti weak password)
└── *_test.go        # users_test.go, storage_test.go
```

### 3.3 Rule (authorization per path)

```
rules/
├── rules.go         # Rule{Regex,Allow,Path,Regexp}, Checker, MatchHidden, Matches
└── rules_test.go
```

### 3.4 HTTP guard & auth handlers (bagian authorization enforcement)

```
http/
├── auth.go          # loginHandler, signupHandler, renewHandler, withUser, withAdmin,
│                    #   extractor, renewableErr, proxyAsserts, printToken
├── users.go         # user CRUD + permission enforcement (withSelfOrAdmin, NonModifiableFields)
├── share.go         # withPermShare, share handlers
├── public.go        # withHashFile (share publik auth), authenticateShareRequest, health
└── data.go          # data.Check / CheckRules (rule enforcement), checkerPrefix
```

### 3.5 Konfigurasi auth (CLI)

```
cmd/
├── config.go        # getAuthentication, getJSONAuth, getProxyAuth, getHookAuth, getNoAuth
└── config_init.go   # simpan auther ke storage saat init
```

---

## 4. Interface `Auther` & Pemilihan Metode

### 4.1 `auth/auth.go` — kontrak autentikasi

```go
type Auther interface {
    Auth(r *http.Request, usr users.Store, stg *settings.Settings, srv *settings.Server) (*users.User, error)
    LoginPage() bool
}
```

- `Auth(...)` — verifikasi request → kembalikan `*users.User` yang terautentikasi, atau error.
  - `os.ErrPermission` → 403 (kredensial salah).
  - error lain → 500 (kesalahan sistem).
- `LoginPage() bool` — apakah metode ini memerlukan halaman login di frontend.

Metode dipilih dari `settings.AuthMethod` (`json` / `proxy` / `hook` / `noauth`).

### 4.2 `auth/storage.go` — penyimpanan auther

```go
type Storage struct { back StorageBackend; users *users.Storage }
type StorageBackend interface { Get(settings.AuthMethod) (Auther, error); Save(Auther) error }
```

- `Store` disimpan di DB (bucket config `"auther"`, lihat Layer 2).
- Constructor `NewStorage(back, userStore)` — menyimpan reference `userStore` untuk dipakai auth flow (terutama hook).

---

## 5. Metode Autentikasi (detail)

### 5.1 `JSONAuth` (`auth/json.go`)

- Konstanta `MethodJSONAuth = "json"`.
- Body JSON: `{ "username", "password", "recaptcha" }`.
- **Anti user-enumeration timing attack**: selalu verifikasi bcrypt — saat user tidak ditemukan, memakai `dummyHash` (hash bcrypt valid yang tetap) agar waktu respon konsisten; baru setelah itu user tidak ditemukan → `os.ErrPermission`.
- reCAPTCHA opsional: bila `a.ReCaptcha != nil && Secret != ""`, validasi `ReCaptcha.Ok(cred.ReCaptcha)` via POST ke `Host + "/recaptcha/api/siteverify"`.
- `LoginPage()` → `true`.

### 5.2 `ProxyAuth` (`auth/proxy.go`)

- Konstanta `MethodProxyAuth = "proxy"`.
- Membaca `username` dari **header** (`a.Header`, mis. `X-Remote-User` / `X-Forwarded-User`).
- Jika user tidak ada → **auto-provision** (`createUser`):
  - Buat password acak (panjang `DefaultMinimumPasswordLength + 10`), di-hash, `LockPassword: true`.
  - Terapkan `settings.Defaults.Apply(user)`.
  - **Hardening (fork m0nsky)**: paksa `Perm.Admin = false`, `Perm.Execute = false`, `Commands = []` — user proxified tak pernah admin/execute walau default mengaturnya.
  - `settings.CreateUserHome` + `SaveProvisioned` (isolasi scope, guard kolisi).
- Jika user ada → kembalikan user.
- `LoginPage()` → `false` (login ditangani proxy).

### 5.3 `HookAuth` (`auth/hook.go`)

- Konstanta `MethodHookAuth = "hook"`.
- Body JSON: `{ "username", "password" }`.
- Menjalankan **command eksternal** (`a.Command`) dengan kredensial hanya lewat env (`USERNAME`, `PASSWORD`) — **tidak pernah** interpolasi ke string command (cek CWE-78/CWE-88 di test).
- Output command parsing `key=value` (`GetValues`) — hanya field yang ada di `validHookFields` yang diterima (`hookFields.IsValid`).
- **Action** (`hook.action`):
  - `auth` → save/lakukan update user (`SaveUser`) lalu autentikasi.
  - `block` → `os.ErrPermission` (tolak).
  - `pass` → verifikasi bcrypt password terhadap user existing.
- `SaveUser`: buat user baru (defaults) atau update existing (field dari hook, mis. `user.perm.*`, `user.scope`); scope eksplisit dari hook menang atas derivasi home auto.
- `LoginPage()` → `true`.

### 5.4 `NoAuth` (`auth/none.go`)

- Konstanta `MethodNoAuth = "noauth"`.
- `Auth` selalu mengembalikan `usr.Get(root, follow, uint(1))` — autentikasi sebagai **user ID 1** (user pertama/admin bootstrap).
- `LoginPage()` → `false`.

---

## 6. Permissions (Authorization — hak akses) — `users/permissions.go`

```go
type Permissions struct {
    Admin    bool `json:"admin"`
    Execute  bool `json:"execute"`
    Create   bool `json:"create"`
    Rename   bool `json:"rename"`
    Modify   bool `json:"modify"`
    Delete   bool `json:"delete"`
    Share    bool `json:"share"`
    Download bool `json:"download"`
}
```

Makna & titik enforcement (di `http/*`):

| Permission | Makna | Utama di-enforce di |
|------------|-------|---------------------|
| `Admin` | Akses penuh (users, settings, semua share) | `withAdmin`, `withSelfOrAdmin` (admin bypass), `users.go`, `settings.go`, `share.go` |
| `Execute` | Jalankan terminal/command | `commands.go` (`!EnableExec || !Execute` → tolak) |
| `Create` | Upload & buat file/dir | `resourcePostHandler`, `tus*`, `patchAction` copy |
| `Rename` | RENAME file/dir | `resourcePatchHandler` rename, Preview rename button |
| `Modify` | Tulis/simpan/ubah file | `resourcePutHandler`, `resourcePostHandler` override, Preview edit |
| `Delete` | Hapus file/dir | `resourceDeleteHandler`, `tusDeleteHandler` |
| `Share` | Buat/kelola share | `withPermShare`, `share*`, public (butuh juga Download) |
| `Download` | Unduh/lihat isi file | `raw`, `preview`, `subtitle`, `resource.Get` (Content), share |

**Poin penting**:
- `Share` **mensyaratkan** `Download` (validasi `ErrShareRequiresDownload` di `users.go` saat create/update user).
- Enforcement dilakukan **di backend** setiap request (bukan sekadar sembunyikan tombol di UI).

---

## 7. Model User & Setup Filesystem (Authorization batas)

Dari `users/users.go`:

- `User` menyimpan `Scope` (batas home) + `Rules []rules.Rule` (aturan custom user).
- `Clean(baseScope, followExternalSymlinks)`:
  - Validasi field (username, password, viewMode, commands, sorting, rules).
  - **Membangun `Fs`**: `files.NewFs(afero.NewOsFs(), scope, followExternalSymlinks)` → menghasilkan `ScopedFs` (default, symlink-confined) atau `BasePathFs` (jika `followExternalSymlinks`).
- `FullPath(path)` — path absolut di disk dari path relatif user.
- `GetRules()` — implement `rules.Provider` (user rules untuk `data.CheckRules`).

**Scope** adalah batas otorisasi fisik: operasi di luar scope ditolak `os.ErrPermission` (lihat Layer 2 `scoped.go`).

---

## 8. Rules (Authorization per path) — `rules/rules.go`

```go
type Rule struct {
    Regex  bool    `json:"regex"`   // true: cocokkan sebagai regex
    Allow  bool    `json:"allow"`   // true: izinkan, false: tolak
    Path   string  `json:"path"`
    Regexp *Regexp `json:"regexp"`
}
type Checker interface { Check(path string) bool }
```

- `Matches(path, fold)`:
  - `Regex` → pakai `Regexp.MatchString` (tidak di-fold).
  - non-regex → cocokkan `path` terhadap `rule.Path` (prefix-aware, dengan normalisasi fold case-insensitive bila `CaseInsensitiveFs`).
- `MatchHidden(path)` — deteksi basename berawalan `.` (untuk `HideDotfiles`).
- Rules disimpan di **dua level**: `settings.Rules` (global) dan `user.Rules` (per user).

### Enforcement di `http/data.go`

```go
func (d *data) Check(path string) bool {
    if d.user.HideDotfiles && rules.MatchHidden(d.rulePath(path)) { return false }
    return d.CheckRules(path)
}
func (d *data) CheckRules(path string) bool {
    path = d.rulePath(path)  // canonicalize + checkerPrefix (public share)
    allow := true
    for _, rule := range d.settings.Rules { if rule.Matches(path, CaseInsensitiveFs) { allow = rule.Allow } }
    for _, rule := range d.user.Rules  { if rule.Matches(path, CaseInsensitiveFs) { allow = rule.Allow } }
    return allow
}
```

- Rules dievaluasi **berurutan; aturan terakhir yang cocok menang**.
- `rulePath` — normalisasi `\`→`/` dan, untuk public share yang di-rebase, menambahkan `checkerPrefix` agar deny rules tetap diterapkan pada scope asli (anti-bypass).
- `checkDescendants` (di `resource.go`) memvalidasi rules pada **seluruh subtree** untuk delete/copy/rename — mencegah bypass via operasi pada parent.

---

## 9. Password Hashing & Policy — `users/password.go`

- `HashPwd` / `CheckPwd` — `golang.org/x/crypto/bcrypt` (`bcrypt.DefaultCost`).
- `ValidateAndHashPwd(password, minimumLength)`:
  - Cek panjang ≥ `minimumLength` → `ErrShortPassword`.
  - Cek terhadap `commonPasswords` (daftar password umum dari `assets/common-passwords.txt`, di-embed via `//go:embed`) → `ErrEasyPassword`.
  - Hash.
- `RandomPwd(len)` — password acak (dipakai ProxyAuth & provisioning) via `crypto/rand` + base64 URL.
- `LockPassword` flag user — mencegah user mengubah password sendiri (dicatat di `users.go` guard).

---

## 10. Guard Middleware HTTP (Authorization enforcement)

### 10.1 `http/auth.go`

**JWT verification (`withUser`):**
- Token diekstrak (`extractor`) dari `X-Auth` header, fallback cookie `auth` (GET only), harus berisi 2 titik (JWT format).
- Parser `jwt.NewParser(WithValidMethods([HS256]), WithExpirationRequired())`.
- `renewableErr` — jika gagal hanya karena expired & metode = ProxyAuth & logout page bukan default → `proxyAsserts` (verifikasi proxy masih mengasert identitas sama) → izinkan; jika tidak → 401.
- `expiresSoon` (< 1 jam) atau `updated` (user di-update sejak token) → set `X-Renew-Token: true`.
- Muat user via `store.Users.Get(root, follow, tk.User.ID)` → `d.user`. canonicalize path.

**`withAdmin`** = `withUser` + `d.user.Perm.Admin` (else 403).

**Token issuance (`printToken`):**
- Membangun claims `authToken{ User userInfo, RegisteredClaims{IssuedAt, ExpiresAt, Issuer:"File Browser"} }`.
- Sign HS256 dengan `d.settings.Key` (512-bit, dari DB).
- `userInfo` hanya berisi field non-sensitif (tidak termasuk Password/Scope).

### 10.2 `http/users.go`

- `withSelfOrAdmin` = `withUser` + `d.user.ID == id || Admin` (else 403). Menjaga user hanya mengubah dirinya sendiri kecuali admin.
- `NonModifiableFieldsForNonAdmin = [Username, Scope, LockPassword, Perm, Commands, Rules]` — field ini hanya bisa diubah admin.
- `userPutHandler`: sensitive fields (password/perm/scope/lockPassword/commands/username) mensyaratkan current password (JSONAuth) + admin untuk field admin-only.
- `Share` memerlukan `Download` (`ErrShareRequiresDownload`).

### 10.3 `http/share.go`

- `withPermShare` = `withUser` + `Perm.Share && Perm.Download` (else 403).
- `shareListHandler`/`shareGetsHandler`: admin → semua share / all paths; non-admin → hanya share miliknya / path-nya.
- `shareDeleteHandler`: hanya pemilik atau admin.

### 10.4 `http/public.go` — `withHashFile` (share publik)

- Redirect authorization ke konteks pemilik share:
  - `store.Share.GetByHash(hash)` → `authenticateShareRequest` (password/token).
  - `store.Users.Get(...)` si pemilik → butuh `Perm.Share && Perm.Download`.
  - `d.user = pemilik`; fs di-rebase ke path share (`files.NewFs`), `checkerPrefix = basePath` (rules tetap berlaku).
  - **Symlink-confined** (`ScopedFs`) → share tak bisa lewat symlink keluar scope.

---

## 11. Konfigurasi Auth di CLI (`cmd/config.go`)

- `getAuthMethod(flags, defaults...)` — pilih `AuthMethod` dari flag.
- `getAuthentication(flags, defaults...)` → `(method, auther, err)`:
  - `MethodNoAuth` → `auth.NoAuth{}`.
  - `MethodProxyAuth` → `getProxyAuth` (baca `--auth.header`).
  - `MethodJSONAuth` → `getJSONAuth` (baca reCAPTCHA config).
  - `MethodHookAuth` → `getHookAuth` (baca `--auth.command`).
  - default → `ErrInvalidAuthMethod`.
- `config_init.go` → `st.Auth.Save(auther)` menyimpan auther ke DB saat inisialisasi.
- `cmd/root.go` flag `--noauth` untuk quick setup.

---

## 12. Keamanan di Lapisan Auth/Authorization

1. **Anti user-enumeration** (JSONAuth): dummy bcrypt hash saat user tidak ada → waktu respon konsisten.
2. **bcrypt cost default** + validasi panjang minimum & password umum (common-passwords).
3. **JWT HS256** dengan kunci 512-bit, `ExpirationRequired`, `Issuer`, `X-Renew-Token`.
4. **Token bocor paska-expired** tidak bisa self-authenticate (ProxyAuth renew hanya via `proxyAsserts`).
5. **Signup/proxy/hook auto-provision** tidak pernah admin/execute (hardening fork).
6. **Scope confinement** (`ScopedFs`) — pembatasan filesystem tingkat OS.
7. **Rules** global & per-user, dievaluasi di backend, termasuk subtree (`checkDescendants`).
8. **Permission check di backend** setiap request (bukan hanya UI).
9. **Hook injection** — kredensial hanya via env, tidak pernah interpolasi ke command (CWE-78/88, diuji).
10. **Kolisi scope** dicegah `GetByScope` case-insensitive + `SaveProvisioned` lock (Layer 2).
11. **Admin tunggal** tak bisa dihapus (`ErrRootUserDeletion` via `CountAdmins`/`IsUniqueAdmin`).

---

## 13. Unit Tests terkait Auth/Authorization

| File | Cakupan |
|------|---------|
| `auth/proxy_test.go` | auto-provision tidak admin/execute; `CreateUserDir` mengisolasi scope tiap user |
| `auth/hook_test.go` | anti credential injection (CWE-78/88); kredensial via env; isolated scope; explicit scope menang |
| `users/users_test.go` | `Clean` membangun `ScopedFs` vs `BasePathFs` sesuai `followExternalSymlinks` |
| `users/storage_test.go` | `SaveProvisioned` atomic (cek-then-save), reject konflik scope; shared explicit scope diperbolehkan |
| `storage/bolt/users_test.go` | `GetByScope` case-insensitive (Layer 2) |
| `http/*_test.go` | guard handlers, rules path/recursive, public share auth (Layer 3) |

---

## 14. Catatan Pengembangan (Perluasan sesuai PRD)

Untuk roadmap (Podman dashboard, website deployer, SSH tunnel), pertimbangan auth/authorization:

1. **Granular role-level security**: `users.Permissions` dapat diperluas dengan flag baru (mis. `Podman`, `Deploy`, `Tunnel`) — tambahkan field + enforcement guard baru + tipe frontend (sudah di-layer 1).
2. **Hooks/SSO**: `HookAuth` sudah mendukung provisioning kustom — dapat dimanfaatkan untuk mengintegrasikan SSO/IdP eksternal.
3. **ProxyAuth** cocok untuk reverse proxy/LB (Layer 11) mengasert identitas via header.
4. **Rules** dapat diperluas untuk membatasi path yang boleh di-deploy/di-tunnel.
5. Setiap endpoint/toolbar baru harus diverifikasi dengan guard perm yang sesuai dan validasi rule (`data.Check`).

---

## 15. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 4: Auth & Authorization (navigator).
- `docs/layers/02-database-storage.md` — penyimpanan user/auther/settings (DB), `ScopedFs`.
- `docs/layers/03-backend-api.md` — guard handler & enforcement (withUser, withAdmin, dst.).
- `docs/layers/01-frontend.md` — permission `user.perm.*` di UI & token `X-Auth`.
- `docs/PRD.md` — kebutuhan role-level security (Layer 7) yang menambah permission.
- `docs/AGENTS.md` — panduan workflow (branch `m0nskyBuildAI`).
