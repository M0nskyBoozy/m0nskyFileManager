# Layer 8 — Role Level Security (RBAC)

> Dokumen teknis mendalam untuk lapisan **Role Level Security** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 8).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan ini mengimplementasikan **keamanan berbasis peran (RBAC — Role-Based Access Control)**: mengatur tingkat akses yang berbeda antara **admin** dan **pengguna biasa**, serta permission granular untuk operasi file.

Sesuai `ARCHITECTURE.md` Layer 8, tanggung jawab:
1. Menentukan **permission granular**: Admin, Execute, Create, Rename, Modify, Delete, Share, Download.
2. Membedakan **hak akses admin vs non-admin** di frontend (route guard) dan backend (authorization di setiap handler).
3. (Roadmap) Menambah role untuk operasi **Podman** / deployment.

**Prinsip inti (defense in depth):**
- Level keamanan **diputuskan di backend** (server) sebagai sumber kebenaran, saat **setiap request**.
- Frontend hanya **menyembunyikan/menonaktifkan** UI berdasarkan permission — **bukan** pengamanan utama.
- **Admin** adalah super-permission: dapat mengelola users/settings/global/share milik siapa pun.

Modul kode nyata: `users/permissions.go` + `frontend/src/router/index.ts` + guard backend di `http/`.

---

## 2. Model Permission — `users/permissions.go`

```go
// Permissions describe a user's permissions.
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

### 2.1 Makna setiap permission

| Field | JSON | Makna | Dampak bila `true` |
|-------|------|-------|--------------------|
| `Admin` | `admin` | Super-user / admin | Akses penuh: kelola semua user, settings global, semua share, bypass `withSelfOrAdmin` |
| `Execute` | `execute` | Jalankan command runner | Mengaktifkan terminal/command di UI (`EnableExec` juga harus `true`) |
| `Create` | `create` | Buat file/dir | Upload, buat folder, copy (dest baru) |
| `Rename` | `rename` | Ganti nama | Rename & move |
| `Modify` | `modify` | Ubah isi file | Edit/save file di editor |
| `Delete` | `delete` | Hapus | Hapus file/dir |
| `Share` | `share` | Buat/kelola share | Membuat & mengelola public link |
| `Download` | `download` | Unduh/lihat isi | Download, preview, raw view |

### 2.2 Dependensi antar permission

- **`Share` mensyaratkan `Download`** — validasi `ErrShareRequiresDownload` (backend) dan auto-set di frontend (`Permissions.vue`).
- **`Admin` menonaktifkan semua toggle lain** di UI (admin otomatis berhak segalanya).

---

## 3. Peta Struktur File Role Level Security (Lengkap)

```
# Definisi model & storage permission
├── users/
│   ├── permissions.go        # struct Permissions (8 permission)
│   ├── users.go              # model User (memuat Perm), Clean, Rules
│   ├── storage.go            # IsUniqueAdmin, CountAdmins, Delete protection
│   └── storage/bolt/users.go # persistenci user + permission (DB, lihat Layer 2)

# Enforcement BACKEND (kebenaran utama) — guard & validasi per handler
├── http/
│   ├── auth.go               # withUser (JWT→user), withAdmin, userInfo, signup hardening
│   ├── users.go              # withSelfOrAdmin, NonModifiableFieldsForNonAdmin, validasi share↔download
│   ├── share.go              # withPermShare (Share && Download)
│   ├── commands.go           # Perm.Execute check + whitelist user.Commands
│   ├── resource.go / subtitle.go / raw.go / preview.go / tus_handlers.go
│   │                         # Perm.Download/Create/Rename/Modify/Delete per endpoint
│   └── public.go             # share publik: pemilik harus Share && Download

# Enforcement FRONTEND (UX / hide-disable, bukan keamanan utama)
└── frontend/
    ├── src/router/index.ts     # guard global: requiresAuth, requiresAdmin → /403
    ├── src/stores/auth.ts      # state user (dengan perm)
    └── src/views/ + components/
        ├── views/files/FileListing.vue    # toolbar disabilitas per perm
        ├── views/files/Preview.vue        # tombol rename/edit/delete/download
        ├── views/files/Editor.vue         # save (perm.modify)
        ├── views/settings/Settings.vue    # menu admin/share
        ├── views/settings/Users.vue / User.vue
        ├── components/settings/Permissions.vue  # form toggle permission (UI)
        └── views/Layout.vue / Sidebar.vue / components/prompts/...
```

---

## 4. Enforcement BACKEND (sumber kebenaran)

Backend adalah **penegak akhir** Role Level Security. Setiap endpoint dilindungi guard; permission diperiksa saat request.

### 4.1 Guard utama di `http/`

| Guard | Definisi | Efek |
|-------|----------|------|
| `withUser` | Verifikasi JWT, muat user (`d.user`) | Semua request terautentikasi |
| `withAdmin` | `withUser` + `d.user.Perm.Admin` | Hanya admin → else `403` |
| `withSelfOrAdmin` | `withUser` + `d.user.ID == id || Admin` | User hanya mengelola dirinya, kecuali admin |
| `withPermShare` | `withUser` + `Perm.Share && Perm.Download` | Untuk operasi share |

Alur `withUser` (`http/auth.go:116`):
1. Ekstrak token JWT (header/cookie).
2. Validasi HS256, expiry, issuer.
3. `d.store.Users.Get(...)` → muat user sesuai token.
4. Set `d.user`, `d.user.Fs` (ScopedFs confine).

Alur `withAdmin` (`http/auth.go:146`):
```go
func withAdmin(fn handleFunc) handleFunc {
	return withUser(func(...) {
		if !d.user.Perm.Admin {
			return http.StatusForbidden, nil   // 403
		}
		return fn(...)
	})
}
```

### 4.2 `withSelfOrAdmin` (`http/users.go:57`)

- `getUserID(r)` → id target.
- Jika `d.user.ID != id` dan **bukan admin** → `403`.
- Melindungi: `userGetHandler`, `userDeleteHandler`, `userUpdateHandler`.

### 4.3 `NonModifiableFieldsForNonAdmin` (`http/users.go:22`)

```go
NonModifiableFieldsForNonAdmin = []string{
	"Username", "Scope", "LockPassword", "Perm", "Commands", "Rules",
}
```

- Non-admin **tidak boleh** mengubah field sensitif ini (username, scope, lock password, **permission**, commands, rules) — hanya admin.
- `userPutHandler` memverifikasi: tiap field sensitif → wajib current password (JSONAuth) + admin untuk field admin-only.
- Validasi `Share && !Download` → `ErrShareRequiresDownload` (create & update).

### 4.4 Permission per endpoint (contoh enforcement)

| Permission | Titik backend |
|-----------|---------------|
| `Execute` | `http/commands.go:64` `!EnableExec || !Perm.Execute` → tolak + whitelist `user.Commands` |
| `Create` | `resourcePostHandler`, `tus*`, copy-create, folder-create |
| `Rename` | `resourcePatchHandler` rename |
| `Modify` | `resourcePutHandler`, editor save, override |
| `Delete` | `resourceDeleteHandler`, `tusDeleteHandler` |
| `Download` | `raw`, `preview`, `subtitle`, `resource.Get` (Content), share |
| `Share` | `withPermShare`, share handlers, public |

### 4.5 Protector admin (storage) — `users/storage.go`

- `Delete(id)` memanggil `IsUniqueAdmin(user)` sebelum menghapus.
- `IsUniqueAdmin(user)` (`users/storage.go:183`):
  - `!user.Perm.Admin` → `false` (boleh hapus).
  - `CountAdmins() <= 1` → `true` (admin terakhir, **dilarang hapus** → `ErrRootUserDeletion`).
- Menjamin **selalu ada minimal satu admin** (`ErrRootUserDeletion`).

### 4.6 Hardening signup/proxy/hook provisioning (fork m0nsky)

- `auth/proxy.go:49-51`: user provisioning **dipaksa** `Perm.Admin = false`, `Perm.Execute = false`, `Commands = []string{}`:
  ```go
  user.Perm.Admin = false
  user.Perm.Execute = false
  user.Commands = []string{}
  ```
- `http/auth.go:213`: pada signup, `user.Perm.Admin = false` (tidak pernah jadi admin via self-registrasi).
- User default admin hanya lewat **quick setup** (`cmd/root.go:530` `user.Perm.Admin = true`).

### 4.7 Eksposur permission via `userInfo` (`http/auth.go`)

- `userInfo` (payload JWT & respon login) memuat `Perm` (untuk frontend render), tapi **tidak** memuat `Password`/`Scope` non-admin.
- `userGetHandler`: untuk non-admin, `u.Scope = ""` disamarkan (`http/users.go:101`).

---

## 5. Enforcement FRONTEND (UX & route guard)

Frontend **bukan** pengaman utama — hanya menyembunyikan/menonaktifkan UI sesuai `user.perm`. Jika user menyerang langsung endpoint, backend tetap menolak.

### 5.1 Route guard global (`frontend/src/router/index.ts`)

- Meta route:
  - `requiresAuth: true` pada `/files` & `/settings`.
  - `requiresAdmin: true` pada `/settings/global`, `/settings/users`, `/settings/users/:id`.
- `router.beforeResolve`:
  - Kalau `to.meta.requiresAuth` & tidak login → redirect `/login?redirect=...`.
  - Kalau juga `requiresAdmin` & `!user.perm.admin` → redirect `/403` (`router/index.ts:210-214`).
- `initAuth()`: validasi login awal (loginPage) atau auto-login (noauth/proxy).

### 5.2 Penggunaan permission di UI

| Komponen | Kondisi permis | Efek UI |
|----------|----------------|---------|
| `FileListing.vue:479-489` | create/download/execute/delete/rename/share+download | Toolbar upload/download/shell/delete/rename/share/move/copy |
| `Preview.vue` | rename / modify / delete / download / share | Tombol aksi preview |
| `Editor.vue` | modify | Tombol/save edit |
| `Settings.vue` | share / admin | Menu share management / global / user |
| `Permissions.vue` | admin → semua toggle disabled; share→auto download | Form permission saat create/edit user |
| `Layout.vue`, `Sidebar.vue` | execute / admin | Tombol terminal / menu admin |

---

## 6. Alur Otorisasi Lengkap (per request)

```
 Request ──▶ withUser (JWT → d.user + Perm)
   │
   ├─ (butuh admin?)         ──▶ withAdmin        → 403 bila !Perm.Admin
   ├─ (butuh user/id sendiri?)─▶ withSelfOrAdmin   → 403 bila bukan admin & id≠self
   ├─ (operasi share?)       ──▶ withPermShare     → 403 bila !(Share&&Download)
   └─ (operasi file?)        ──▶ handler → cek Perm.(Create|Rename|Modify|Delete|Download|Execute)
                                   + rules (Allow/Deny path) + ScopedFs confine
   │
   ▼
   Success / 403
```

---

## 7. Roadmap & Ekstensi (sesuai PRD)

- **Role Podman** (roadmap): permission baru untuk operasi container, volume, image, dsb. — mis. `Perm.Podman` atau `Perm.Deploy`. Titik integrasi: tambah field di `Permissions`, guard di handler Podman, form di `Permissions.vue`, dan route guard baru.
- **RBAC deployment**: hanya user berperan tertentu dapat mendeploy website/kelola Podman.
- **Distinguish role**: saat ini model adalah **binary (admin vs non-admin)**; roadmap menuju role yang lebih granular untuk fitur Podman/cloud.

---

## 8. Unit Tests terkait Role Level Security

- `auth/proxy_test.go`, `auth/hook_test.go` — provisioning tidak pernah admin/execute.
- `users/storage_test.go` — `SaveProvisioned`, `IsUniqueAdmin`/admin protection.
- `storage/bolt/users_test.go` — persistensi user/permission.
- `http/auth_test.go` — guard & JWT permission.
- `http/users_test.go` (jika ada) — `withSelfOrAdmin`, non-modifiable fields.
- `http/resource_*test.go`, `rules_*test.go`, `tus_*test.go` — enforcement permission & rules.

---

## 9. Catatan Pengembangan (Saran)

1. Untuk fitur Podman, pertimbangkan permission baru (`Podman`, `Deploy`, `Tunnel`) + guard ad-hoc (pola `withPermShare`/`withAdmin`).
2. Pastikan setiap endpoint baru **tetap mengecek permission di backend** (bukan hanya UI).
3. Jaga invariant "selalu ada satu admin" (`IsUniqueAdmin`) untuk akun produksi.
4. Pertimbangkan `STYLEGUIDE.md` (belum ada) untuk menyeragamkan konvensi guard/permission.

---

## 10. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 8: Role Level Security (navigator & roadmap).
- `docs/layers/04-auth-authorization.md` — autentikasi, JWT, signup/proxy hardening, `Perm.Execute` & whitelist.
- `docs/layers/03-backend-api.md` — guard handler (`withUser`/`withAdmin`), endpoint & authorization.
- `docs/layers/01-frontend.md` — `user.perm.*`, route guard, UI permission.
- `docs/layers/06-cloud-compute.md` — roadmap role untuk operasi Podman.
- `docs/PRD.md` — §2.1/2.2 kebutuhan RBAC Podman; target user beginner/advanced.
- `docs/AGENTS.md` — workflow pengembangan branch `m0nskyBuildAI`.
