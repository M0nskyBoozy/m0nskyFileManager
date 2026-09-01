# Layer 2 — Database & Storage

> Dokumen teknis mendalam untuk lapisan **Database & Storage** dari `m0nskyFileManager`.
> Ini adalah dokumen *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 2).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Gambaran Umum Dua Jenis Penyimpanan (Two-Tier Storage)

`m0nskyFileManager` (fork File Browser) memiliki **dua lapisan penyimpanan yang terpisah dan jelas bedanya**:

| Aspek | (A) Database Metadata | (B) File System (Data Aktual) |
|-------|----------------------|-------------------------------|
| Isi | Users, Share links, Settings, Auth + versi DB | Isi file & direktori milik pengguna |
| Teknologi | **BoltDB** (via **Storm v3** ORM/ODM) | Sistem file OS via **afero** abstraction |
| Lokasi | Satu file binary `filebrowser.db` | Di bawah `root` scope (biasa `/srv/...`) |
| Junctions | `storage.Storage` (composite) + `users.User.Fs` | `files.ScopedFs` / `afero.BasePathFs` |
| Karakter | Metadata kecil, cepat di-query | Data besar, diakses langsung |

> **Poin kunci**: "Database" (BoltDB) hanya menyimpan **metadata** (user, share, settings, auth).
> **File aktual** pengguna tinggal di **filesystem OS** dan diakses lewat filesystem abstraction (`files.ScopedFs`, berbasis `afero`). Keduanya dijembatani di `storage.Storage` (untuk metadata) dan `users.User.Fs` (untuk akses file aktual).

---

## 2. Arsitektur Penyimpanan & Alur Inisialisasi

### 2.1 Diagram Alur

```
┌─────────────────────────────────────────────────────────────────────┐
│ cmd/ (Cobra CLI)                                                     │
│  cmd/utils.go: initStorage + withViperAndStore                      │
│                                                                     │
│  1. Resolve path database (flag --database / -d)                    │
│  2. cek dbExists (expectsNoDatabase / allowsNoDatabase)             │
│  3. storm.Open(path, storm.BoltOptions(0640, nil))                  │
│       └── buka file filebrowser.db (BoltDB)                        │
│  4. bolt.NewStorage(db)  ──▶ *storage.Storage                       │
│                                                                     │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│ storage/storage.go: Storage (struct composite)                      │
│   .Users    users.Store     ── users.Storage (backend: usersBackend)│
│   .Share    *share.Storage  ── backend: shareBackend                │
│   .Auth     *auth.Storage   ── backend: authBackend                 │
│   .Settings *settings.Storage ── backend: settingsBackend            │
└─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼ (untuk akses file aktual)
┌─────────────────────────────────────────────────────────────────────┐
│ users.User.Fs (afero.Fs) ── files.NewFs / files.NewScopedFs         │
│   └── membungkus afero.NewOsFs + BasePathFs / ScopedFs              │
│        membatasi akses ke scope/home user                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Depedensi Utama (dari `go.mod`)

| Library | Versi | Peran |
|---------|-------|-------|
| `go.etcd.io/bbolt` | v1.5.0 | KV-store embedded, single-file |
| `github.com/asdine/storm/v3` | v3.2.1 | ORM/ODM di atas bbolt (struct → bucket) |
| `github.com/spf13/afero` | v1.15.0 | Filesystem abstraction (untuk file aktual) |
| `golang.org/x/crypto/bcrypt` | — | Hashing password (via users) |

---

## 3. Struktur File Database & Storage (Peta Lengkap)

Berikut seluruh file yang membentuk lapisan ini, dikelompokkan per paket, dengan tujuan masing-masing.

### 3.1 Paket `storage/` — Komposisi & Backend BoltDB

```
storage/
├── storage.go              # Tipe Storage (composite): Users, Share, Auth, Settings
└── bolt/                   # Implementasi backend metada berbasis BoltDB
    ├── bolt.go             # NewStorage(db) → merakit semua backend + simpan versi "2"
    ├── auth.go             # authBackend: Get/Save Auther
    ├── config.go           # settingsBackend: Get/Save Settings & Server
    ├── share.go            # shareBackend: query & mutasi share.Link
    ├── users.go            # usersBackend: CRUD User, GetByScope, CountAdmins
    ├── utils.go            # helper get(db,"config",name) & save(db,"config",name)
    ├── users_test.go       # test GetByScope case-insensitive
    └── share_test.go       # test DeleteWithPathPrefix (regresi CVE/GHSA)
```

### 3.2 Paket definisi `Store` (interface) & `Storage` (logika)

Setiap domain mendefinisikan **dua abstraksi**:
- `StorageBackend` (interface di paket domain) — kontrak primitif yang harus diimplementasi backend DB.
- `Storage` (struct di paket domain) — membungkus backend + menambahkan logika verifikasi, cleanup, keamanan.

```
users/
├── storage.go             # users.StorageBackend (interface) + users.Store + users.Storage
├── users.go               # model User + Clean() (validasi field & setup Fs)
├── permissions.go         # type Permissions struct
├── password.go            # bcrypt hashing + validasi password
├── assets.go              # embed common-passwords.txt → map
└── assets/
    └── common-passwords.txt  # daftar password umum (dicek saat signup)
```

```
share/
├── storage.go             # share.StorageBackend (interface) + share.Storage (auto-expire)
└── share.go               # model Link + CreateBody
```

```
settings/
├── storage.go             # settings.StorageBackend (interface) + settings.Storage (defaults)
├── settings.go            # model Settings + Server + GenerateKey
├── defaults.go            # UserDefaults
├── tus.go                 # Tus settings + konstanta default
├── branding.go            # Branding settings
└── dir.go                 # MakeUserDir / CreateUserHome
```

```
auth/
├── storage.go             # auth.StorageBackend (interface) + auth.Storage
└── auth.go                # interface Auther (Auth + LoginPage)
```

### 3.3 Paket pendukung penyimpanan fisik (File System)

```
files/
├── scoped.go              # ScopedFs (confine ke scope + anti-symlink escape) + NewFs
├── file.go                # FileInfo/FileOptions — representasi file untuk listing/API
├── listing.go             # Listing — struktur direktori & sorting
├── sorting.go             # Sorting options
├── case.go                # case-insensitivity fs detection
├── mime.go                # deteksi tipe MIME
└── utils.go               # util filesystem
```

### 3.4 Konfigurasi & Pembukaan DB (CLI)

```
cmd/
├── root.go                # flag persistent: --database / -d (default "./filebrowser.db")
└── utils.go               # initStorage, withViperAndStore, dbExists
```

### 3.5 Pendukung lain

```
errors/errors.go           # definisi error domain (ErrExist, ErrNotExist, ...)
runner/                    # eksekusi command hooks (bukan DB, tapi dijalankan saat event storage)
```

---

## 4. Model Data (Struktur yang Disimpan di BoltDB)

BoltDB menyimpan data sebagai **buckets**; Storm memetakan **struct Go** ke bucket dengan nama = nama tipe (`reflect.TypeOf(T).Name()`).

### 4.1 Bucket: `User` — satu record per user

Field di-serialize dari `users.User` (lihat `users/users.go`):

| Field | Tipe | Tag Storm | Keterangan |
|-------|------|-----------|------------|
| `ID` | uint | `storm:"id,increment"` | Primary key auto-increment |
| `Username` | string | `storm:"unique"` | Unik, dipakai lookup `GetBy(string)` |
| `Password` | string | — | Hash bcrypt |
| `Scope` | string | — | Batas home dir; **di-index untuk `GetByScope`** |
| `Locale` | string | — | Preferensi bahasa |
| `LockPassword` | bool | — | Kunci ubah password |
| `ViewMode` | ViewMode | — | `list` / `mosaic` |
| `SingleClick` | bool | — | Buka file sekali klik |
| `RedirectAfterCopyMove` | bool | — | Redirect setelah copy/move |
| `Perm` | Permissions | — | Hak akses (lihat 4.6) |
| `Commands` | []string | — | Perintah khusus user |
| `Sorting` | files.Sorting | — | Preferensi sortir |
| `Fs` | afero.Fs | `json:"-"` | **TIDAK disimpan** (runtime-only, di-set ulang saat load) |
| `Rules` | []rules.Rule | — | Aturan allow/deny path user |
| `HideDotfiles` | bool | — | Sembunyikan dotfiles |
| `DateFormat` | bool | — | Format tanggal |
| `AceEditorTheme` | string | — | Tema editor |

> `storm:"id,increment"` → BoltDB auto-number ID.
> `storm:"unique"` pada `Username` → gagal (ErrExist) bila duplikat.

### 4.2 Bucket: `Link` (Share Links)

Dari `share/share.go`:

| Field | Tipe | Tag Storm | Keterangan |
|-------|------|-----------|------------|
| `Hash` | string | `storm:"id,index"` | Primary key (hash URL) |
| `Path` | string | `storm:"index"` | Path yang di-share |
| `UserID` | uint | — | Pemilik link |
| `Expire` | int64 | — | Timestamp kedaluwarsa (0 = permanent) |
| `PasswordHash` | string | `omitempty` | Hash password share |
| `Token` | string | `omitempty` | Token acak untuk unduhan ber-password |

> `CreateBody` (request body saat membuat share): `{ password, expires, unit }` — bukan bagian tersimpan; hanya payload API.

### 4.3 Bucket config: `Settings`

Dari `settings/settings.go`, disimpan sebagai pasangan `config` → `"settings"`:

| Field | Tipe | Keterangan |
|-------|------|------------|
| `Key` | []byte | Kunci enkripsi (512 bit / `GenerateKey`) |
| `Signup` | bool | Izinkan registrasi |
| `HideLoginButton` | bool | Sembunyikan tombol login |
| `CreateUserDir` | bool | Buat home dir per user otomatis |
| `UserHomeBasePath` | string | Default `"/users"` (defaults.go) |
| `Defaults` | UserDefaults | Default field user baru |
| `AuthMethod` | AuthMethod | json/proxy/hook/noauth |
| `LogoutPage` | string | Default `"/login"` |
| `Branding` | Branding | Nama, warna, tema, dsb. |
| `Tus` | Tus | chunk upload (default 10MB, retry 5) |
| `Commands` | map[string][]string | Hooks sebelum/sesudah event |
| `Shell` | []string | Shell untuk runner |
| `Rules` | []rules.Rule | Rules global |
| `MinimumPasswordLength` | uint | Default `12` |
| `FileMode` | fs.FileMode | Default `0640` |
| `DirMode` | fs.FileMode | Default `0750` |
| `HideDotfiles` | bool | Global hide dotfiles |

### 4.4 Bucket config: `Server`

Dari `settings.Server` (disimpan `config` → `"server"`):

| Field | Tipe | Keterangan |
|-------|------|------------|
| `Root` | string | Root filesystem yang dilayani |
| `BaseURL` | string | Prefix URL (di-clean saat `SaveServer`) |
| `Socket` | string | Unix socket |
| `TLSKey` / `TLSCert` | string | TLS |
| `Port` / `Address` | string | Bind address |
| `Log` | string | File log |
| `EnableThumbnails` | bool | Thumbnail |
| `ResizePreview` | bool | Resize preview |
| `EnableExec` | bool | Aktifkan exec/terminal |
| `TypeDetectionByHeader` | bool | Deteksi tipe dari header |
| `ImageResolutionCal` | bool | Hitung resolusi gambar |
| `AuthHook` | string | URL hook auth |
| `TokenExpirationTime` | string | Durasi token (parse via `GetTokenExpirationTime`) |
| `FollowExternalSymlinks` | bool | Ikuti symlink keluar scope |
| `CaseInsensitiveFs` | bool | `json:"-"` → **tidak pernah dipersist**; dideteksi saat startup |

### 4.5 Bucket config: `auther`

Disimpan `config` → `"auther"`. Menyimpan auther aktif sesuai `settings.Server` / metode auth:

- `JSONAuth` (`auth/json.go`) — login via form JSON.
- `ProxyAuth` (`auth/proxy.go`) — delegasi ke reverse proxy/header.
- `HookAuth` (`auth/hook.go`) — panggil URL eksternal.
- `NoAuth` (`auth/none.go`) — tanpa autentikasi.

`authBackend.Get(t)` membuat struct kosong sesuai metode, lalu `get(db, "auther", auther)` mengisi dari DB.

### 4.6 Bucket: `Permissions` (terbentuk dalam User)

Dari `users/permissions.go`:

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

---

## 5. Versi Database & Migrasi

`bolt.NewStorage` memanggil `save(db, "version", 2)` — menulis **versi DB = 2** ke bucket `config`.

- Ditulis ke `config` → `"version"`.
- Menandai skema metadata saat ini.
- Walaupun `storm` menangani banyak perubahan skema secara otomatis (bucket per tipe), versi ini menjadi penanda manual untuk migrasi di masa depan.

---

## 6. Pembukaan Database di CLI (`cmd`)

`withViperAndStore` di `cmd/utils.go`:

```go
path := filepath.Abs(v.GetString("database"))       // flag -d / --database (default ./filebrowser.db)
exists := dbExists(path)
// validasi: expectsNoDatabase (mis. "config init" harus baru)
//           allowsNoDatabase (mis. "config set" boleh belum ada)
db, err := storm.Open(path, storm.BoltOptions(0640, nil))
storage, err := bolt.NewStorage(db)
```

- **Mode file**: `0640` (owner rw, group r, no other).
- `dbExists(path)` digunakan untuk pesan peringatan "Initialing in ..." bila file belum ada.

Database path default: `./filebrowser.db` (`cmd/root.go:84`).

---

## 7. Interface & Logika Storage per Domain

### 7.1 Users — `users/storage.go`

**StorageBackend interface** (kontrak primitif):
```go
type StorageBackend interface {
    GetBy(interface{}) (*User, error)
    GetByScope(scope string) (*User, error)
    Gets() ([]*User, error)
    Save(u *User) error
    Update(u *User, fields ...string) error
    DeleteByID(uint) error
    DeleteByUsername(string) error
    CountAdmins() (int, error)
}
```

**Store interface** (kontrak level layanan, apa yang dipakai handler):
```go
type Store interface {
    Get(baseScope string, followExternalSymlinks bool, id interface{}) (*User, error)
    GetByScope(scope string) (*User, error)
    Gets(baseScope string, followExternalSymlinks bool) ([]*User, error)
    Update(user *User, fields ...string) error
    Save(user *User) error
    SaveProvisioned(user *User, derivedScope bool) error
    Delete(id interface{}) error
    LastUpdate(id uint) int64
}
```

**Logika penting di `users.Storage`**:

- `Get(baseScope, follow, id)` → panggil `back.GetBy(id)`, lalu `user.Clean(baseScope, follow)` yang **membangun ulang `Fs`** (setup filesystem user).
- `Save` → `user.Clean("", false)` lalu `back.Save`.
- `Update(user, fields...)` → `Clean` per-field + `back.Update` (partial update) + catat timestamp di peta `updated` (dipakai `LastUpdate`).
- `Delete` → cek `IsUniqueAdmin` (tolak hapus admin tunggal → `ErrRootUserDeletion`) via `CountAdmins`.
- `SaveProvisioned(user, derivedScope)`:
  - Untuk user yang di-provisioning (signup/proxy/hook) dengan **scope derived dari username**:
  - **Race-condition guard**: dibawah `provision sync.Mutex`, cek `GetByScope(user.Scope)` dulu; jika sudah ada → `ErrExist` (cegah 2 username share 1 home dir). Jika bebas → `Save`.
  - `LastUpdate(id)` → timestamp update terakhir (dipakai untuk invalidation/cache).

### 7.2 Share — `share/storage.go`

`share.Storage` membungkus `shareBackend` dan menambahkan **auto-expiration**:

- `All()`, `FindByUserID(id)`, `Gets(path, id)` → ambil daftar, lalu **hapus link yang `Expire != 0 && Expire <= now`** (delete on-read) sebelum dikembalikan.
- `GetByHash(hash)` → jika kedaluwarsa, hapus & kembalikan `ErrNotExist`.
- `GetPermanent(path, id)` → cari `Path==path && Expire==0 && UserID==id`.
- `DeleteWithPathPrefix(pathPrefix, userID)` → hapus semua link milik user di bawah prefix path (untuk delete direktori). Backend melakukan trim trailing slash & prefix query.

### 7.3 Settings — `settings/storage.go`

`settings.Storage.Get()` mengisi **default** bila field kosong:
- `UserHomeBasePath` → `/users`
- `LogoutPage` → `/login`
- `MinimumPasswordLength` → 12
- `Tus` → `{ChunkSize: 10MB, RetryCount: 5}`
- `FileMode` → 0640, `DirMode` → 0750

`Save(set)` validasi:
- `Key` tidak boleh kosong → `ErrEmptyKey`.
- Defaults `Locale` → `en`, `ViewMode` → `mosaic`, `Commands` → `[]`.
- Memastikan semua events hooks `before_*`/`after_*` ada (defaultEvents: save, copy, rename, upload, delete).

### 7.4 Auth — `auth/storage.go`

`auth.Storage` (`NewStorage(back, userStore)`) — menyimpan userStore reference untuk auth flow. `Get(t)` memangil backend yang membuat Auther sesuai metode.

---

## 8. Backend BoltDB (`storage/bolt/`)

Setiap backend adalah struct kecil membungkus `db *storm.DB`, mengimplementasi interface domain.

### 8.1 `usersBackend` (`users.go`)

- `GetBy(i)` — `i` `uint` → lookup by `"ID"`; `i` `string` → lookup by `"Username"`. `storm.ErrNotFound` → `ErrNotExist`, tipe lain → `ErrInvalidDataType`.
- `GetByScope(scope)` — **case-insensitive** regex `(?i)^<escaped>$` via `q.Re` terhadap field `Scope`. Ini mencegah dua user dengan scope yang hanya beda kapital (mis. `/users/Alice` vs `/users/alice`) — pada filesystem case-insensitive keduanya menunjuk home yang sama (kolisi). Diperkuat unit test.
- `Gets()` — semua user via `db.All`.
- `Update(user, fields...)` — `db.UpdateField` per field (partial). Field tak valid → error.
- `Save(user)` — `db.Save`; `storm.ErrAlreadyExists` → `ErrExist`.
- `DeleteByID` / `DeleteByUsername` — `db.DeleteStruct`.
- `CountAdmins()` — iterasi bucket `User` via bare bbolt cursor, unmarshal tiap record, hitung `Perm.Admin == true`. Dipakai untuk proteksi admin tunggal.

### 8.2 `shareBackend` (`share.go`)

- `All()`, `FindByUserID(id)`, `GetByHash(hash)`, `GetPermanent(path,id)`, `Gets(path,id)` — query Storm (`q.Eq`, `q.Re`...).
- `DeleteWithPathPrefix(pathPrefix, userID)`:
  - Trim trailing slash, `db.Prefix("Path", prefix)`.
  - Saring hanya milik `userID` dan `Path == prefix || HasPrefix(prefix+"/")`.
  - Hapus tiap link (agar `/abc` sibling tidak ikut terhapus).
  - Test regresi di `share_test.go` menutup bug stale-share (GHSA-pp88-jhwj-5qh5).

### 8.3 `authBackend` (`auth.go`)

- `Get(settings.AuthMethod)` → buat Auther sesuai metode, isi dari bucket config `"auther"`.
- `Save(a)` → `save(db, "auther", a)`.

### 8.4 `settingsBackend` (`config.go`)

- `Get()` / `Save()` → bucket config key `"settings"`.
- `GetServer()` / `SaveServer()` → bucket config key `"server"`.

### 8.5 `utils.go` (helper)

```go
func get(db *storm.DB, name string, to interface{}) error { return db.Get("config", name, to) }
func save(db *storm.DB, name string, from interface{}) error { return db.Set("config", name, from) }
```
- Semua nilai config tunggal disimpan dalam bucket `"config"` dengan key `name`.
- `storm.ErrNotFound` → `ErrNotExist`.

---

## 9. Penyimpanan File Aktual (File System Layer)

Meski "database" adalah BoltDB, **data file user disimpan di filesystem OS**. Ini bagian penting dari lapisan storage.

### 9.1 Afero & `User.Fs`

- Setiap `User` memiliki `Fs afero.Fs` (field runtime, tidak dipersist).
- `Clean()` pada user membangun `Fs`:
  ```go
  scope := filepath.Join(baseScope, filepath.Join("/", scope))
  u.Fs = files.NewFs(afero.NewOsFs(), scope, followExternalSymlinks)
  ```
- `User.FullPath(path)` → `afero.FullBaseFsPath(files.BasePath(u.Fs), path)` untuk path absolut di disk.

### 9.2 `files.NewFs` & `ScopedFs` (`files/scoped.go`)

- `NewFs(source, path, followExternal)`:
  - `followExternal == true` → `afero.NewBasePathFs` (boleh ikuti symlink keluar scope).
  - `followExternal == false` → `NewScopedFs` (default, lebih aman).
- **`ScopedFs`** membungkus `*afero.BasePathFs` dan menambahkan **per-operation scope guard**:
  - Setiap operasi (Create, Open, Remove, Rename, Stat, Chmod, Chown, Chtimes, Mkdir, ...) memanggil `guard()/within()`.
  - `within(p)` — `filepath.EvalSymlinks` target, lalu pastikan tetap dalam root scope. **Menolak symlink yang pointing keluar scope** (cegah symlink escape).
  - Menangani path baru (belum ada) dengan validasi ancestor terdekat; **dangling symlink** diikuti dan divalidasi di targetnya (cegah penulisan keluar scope lewat link).
  - `maxSymlinkHops = 255` membatasi rantai symlink (mirip MAXSYMLINKS kernel).
- `BasePath(fs)` & `ScopedFs.RealPath` — mengambil path fisik aktual (dipakai mis. `disk.UsageWithContext`).

### 9.3 Case-Insensitive Filesystem

- `files/case.go` — deteksi apakah `Root` berada di filesystem case-insensitive.
- Diteruskan ke `Server.CaseInsensitiveFs` (field `json:"-"`, tidak dipersist) & dipakai oleh rule checker untuk mencocokkan path case-insensitive (lihat `rules/rules.go` `Matches(path, fold)`, dan `users.go` `GetByScope`).

---

## 10. Keamanan & Konsistensi Data (Poin Penting)

1. **Scope confinement** — `ScopedFs` mencegah user mengakses/menulis di luar home directory, termasuk via symlink (anti path traversal & symlink escape).
2. **Symlink keluar scope** — ditolak kecuali `FollowExternalSymlinks` diaktifkan (opsi Server).
3. **Kolisi scope** — `GetByScope` case-insensitive + `SaveProvisioned` lock → cegah dua user berbagi satu home dir secara tak sengaja.
4. **Admin tunggal** — `CountAdmins` + `IsUniqueAdmin` → cegah hapus admin terakhir.
5. **Password** — bcrypt (default cost), validasi panjang minimal + daftar password umum (`common-passwords.txt` di-embed via `//go:embed`).
6. **Share expire** — auto-delete on read.
7. **Permission check** — metadata permission disimpan per user; backend memvalidasi setiap request (UI hanya menyembunyikan).
8. **DB file permission** — `0640` saat dibuat.
9. **Case-insensitive rules** — rule/path dicocokkan case-insensitive pada fs demikian agar tak mudah di-evade.

---

## 11. Unit Tests terkait Storage

| File | Cakupan |
|------|---------|
| `storage/bolt/users_test.go` | `GetByScope` case-insensitive; tidak match superstring; scope beda → `ErrNotExist` |
| `storage/bolt/share_test.go` | `DeleteWithPathPrefix`: tidak menyentuh user lain / sibling `/abc`; regresi trailing slash (GHSA); no-op saat kosong |
| `users/storage_test.go` | Logika `users.Storage` (SaveProvisioned, IsUniqueAdmin, dsb.) |
| `auth/proxy_test.go`, `auth/hook_test.go` | Backend auth proxy/hook |
| `settings/dir_test.go` | `MakeUserDir`/`CreateUserHome` |

---

## 12. Catatan Pengembangan (untuk perpanjangan — selaras PRD)

- Untuk **Podman dashboard / website deployer**, data baru (proyek, konfigurasi deploy, tunnel SSH, dsb.) dapat ditambahkan sebagai **bucket baru + domain baru** mengikuti pola yang sama:
  1. Definisikan struct model di paket domain baru.
  2. Definisikan `StorageBackend` interface + `Storage` (logika).
  3. Implementasikan backend nyata di `storage/bolt/` (mis. `xxx.go`).
  4. Registrasikan di `bolt.NewStorage` & tambahkan ke `storage.Storage`.
- Jaga versi DB (`save(db, "version", N)`).
- Tetap gunakan `afero`/`ScopedFs` untuk segala akses filesystem agar security confinement konsisten.
- Simpan metadata besar (mis. isi file) di filesystem, bukan BoltDB — BoltDB untuk metadata & index.

---

## 13. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 2: Database & Storage (navigator).
- `docs/PRD.md` — kebutuhan fitur yang memengaruhi skema data (Podman, deployer, SSH tunnel).
- `docs/layers/01-frontend.md` — sisi client yang mengkonsumsi data tersimpan.
- `docs/AGENTS.md` — panduan workflow (konvensi, branching `m0nskyBuildAI`).
