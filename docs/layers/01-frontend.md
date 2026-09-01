# Layer 1 — Frontend (Web File Manager)

> Dokumen teknis mendalam untuk lapisan **Frontend** dari `m0nskyFileManager`.
> Ini adalah dokumen *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 1).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Frontend adalah **Single Page Application (SPA)** yang dibangun dengan **Vue 3** (`<script setup>` Composition API), **TypeScript**, **Vite** (build tool), **Pinia** (state management), **Vue Router 5** (routing), dan **Vue I18n** (internasionalisasi) dengan JSON messages.

Frontend sepenuhnya **dibundel + di-embed ke dalam binary Go** melalui `go:embed` (`frontend/assets.go`), sehingga deploy menghasilkan satu binary tunggal tanpa server frontend terpisah. Selama development, Vite dev server digunakan; saat production, hasil `vite build` (folder `dist/`) di-embed.

### 1.1 Posisi dalam Arsitektur

```
┌─────────────────────────────────────────────────────────┐
│ LAYER 1: FRONTEND (Vue SPA)                             │
│  - views (halaman) / components (UI)                    │
│  - stores (Pinia state) / api (HTTP client)             │
│  - router / i18n / utils / types                        │
│           │                                             │
│           ▼ HTTP (REST JSON + WebSocket + upload TUS)   │
│ LAYER 2-5: Backend Go (auth, files, share, users)       │
└─────────────────────────────────────────────────────────┘
```

Frontend **tidak pernah berkomunikasi langsung** dengan filesystem — semua operasi file lewat API backend.

---

## 2. Technological Stack & Dependencies

### 2.1 Versi Utama

| Teknologi | Peran |
|-----------|-------|
| Vue 3 | Framework UI (Composition API + `<script setup>`) |
| TypeScript | Type safety |
| Vite | Build tool & dev server |
| Pinia | State management |
| Vue Router 5 | Client-side routing |
| Vue I18n | i18n (JSON messages, lazy loading) |
| Material Icons | Set ikon (font ligature) |

### 2.2 Dependensi Runtime Penting (dari `package.json`)

- `ace-builds` — editor teks kode (untuk halaman Editor).
- `marked` + `marked-katex-extension` — renderer Markdown + dukungan math Katex.
- `dompurify` — sanitasi HTML untuk preview Markdown/HTML (keamanan XSS).
- `epubjs` + `vue-reader` — reader dokumen EPUB.
- `tus-js-client` — protokol upload chunk TUS (Tus Resumable Upload).
- `pinia` — state.
- `vue-i18n`, `vue-router`.
- `xmldom` — parse XML (untuk SVG dll.).
- `clipboard`, `mime`, `pretty-bytes`, `lodash-es` (util).
- `@intlify/unplugin-vue-i18n` — plugin Vite untuk kompilasi i18n.

### 2.3 Dev Dependencies

- `typescript`, `vite`, `vue-tsc` (type check).
- `@vue/tsconfig`, `eslint` + config Vue/TS (lint).
- `postcss` + `autoprefixer` (CSS preprocessing).

---

## 3. Struktur Direktori & Alur Build

### 3.1 Pohon Struktur Frontend

```
frontend/
├── index.html                 # Template HTML untuk dev (Placeholder window.FileBrowser)
├── vite.config.ts             # Konfigurasi Vite (alias @/, plugin i18n, proxy, base static URL)
├── package.json               # Manifest dependensi & scripts
├── tsconfig.json              # Root TS config (references node + app)
├── tsconfig.app.json          # TS config untuk src (Vue/TS DOM)
├── tsconfig.node.json         # TS config untuk file Node (vite.config)
├── eslint.config.js           # Konfigurasi ESLint
├── postcss.config.cjs         # PostCSS + autoprefixer
├── env.d.ts                   # Deklarasi tipe env Vite/client
├── assets.go                  # EMBED dist/ ke binary Go (!! production)
├── public/                    # Template HTML final (diproses Go template + di-embed)
│   └── index.html             # HTML production dengan [{[ .Var }]} Go templates
├── dist/                      # Hasil build (di-embed, tidak di-commit)
└── src/
    ├── main.ts                # Entry point aplikasi
    ├── App.vue                # Root component
    ├── index.d.ts             # Deklarasi tipe global tambahan
    ├── api/                   # HTTP client layer (REST/TUS/WebSocket)
    │   ├── index.ts           # re-export semua modul api
    │   ├── utils.ts           # fetchURL, formatURL, error handling
    │   ├── files.ts           # operasi file CRUD
    │   ├── commands.ts        # WebSocket command (shell/exec)
    │   ├── search.ts          # pencarian streaming
    │   ├── pub.ts             # share public
    │   ├── share.ts           # share management
    │   ├── settings.ts        # global settings
    │   ├── users.ts           # user management
    │   └── tus.ts             # TUS upload client
    ├── stores/                # Pinia stores
    │   ├── index.ts
    │   ├── auth.ts            # sesi pengguna
    │   ├── file.ts            # state listing file
    │   ├── layout.ts          # UI/shell/prompt state
    │   ├── upload.ts          # antrian upload
    │   ├── clipboard.ts       # operasi copy/move
    │   └── router.ts          # sync route → store
    ├── router/
    │   └── index.ts           # definisi route & guards
    ├── i18n/
    │   └── index.ts           # inisialisasi Vue I18n + lazy locales
    │   └── locales/*.json     # file terjemahan per bahasa
    ├── types/
    │   ├── api.d.ts           # tipe request/response API
    │   ├── file.d.ts          # Resource, File, dll.
    │   ├── user.d.ts          # IUser, perms
    │   ├── global.d.ts        # deklarasi global window.FileBrowser
    │   ├── layout.d.ts
    │   ├── settings.d.ts
    │   ├── upload.d.ts        # UploadTask
    │   └── toast.d.ts         # tipe notifikasi toast
    ├── utils/                 # utilitas murni (framework-agnostic)
    │   ├── constants.ts       # konstanta dari window.FileBrowser
    │   ├── auth.ts            # helper login/logout/token
    │   ├── index.ts           # util umum
    │   ├── url.ts             # manipulasi path/jalur
    │   ├── theme.ts           # tema terang/gelap editor
    │   ├── upload.ts          # logika upload
    │   ├── encodings.ts       # encoding (base64, dll.)
    │   ├── css.ts
    │   ├── cookie.ts          # query params dari cookie
    │   ├── clipboard.ts
    │   ├── buttons.ts
    │   ├── previews.ts
    │   ├── download.ts
    │   └── files.ts
    ├── components/            # komponen reusable
    │   ├── Shell.vue          # terminal web (WebSocket exec)
    │   ├── Sidebar.vue        # menu navigasi samping
    │   ├── Breadcrumbs.vue    # navigasi jalur
    │   ├── Search.vue         # input pencarian
    │   ├── ProgressBar.vue    # bar penggunaan disk
    │   ├── ContextMenu.vue    # menu klik kanan
    │   ├── header/            # komponen header bar
    │   │   ├── HeaderBar.vue
    │   │   ├── Action.vue     # tombol aksi header
    │   │   └── Title.vue      # judul halaman
    │   └── prompts/           # modal dialog (BaseModal + varian)
    │       ├── Prompts.vue    # dispatcher modal berdasarkan state
    │       ├── BaseModal.vue
    │       ├── Info.vue, Help.vue
    │       ├── Delete.vue, Rename.vue, Move.vue, Copy.vue
    │       ├── NewFile.vue, NewDir.vue
    │       ├── Download.vue, Replace.vue, Upload.vue
    │       ├── Share.vue, ShareDelete.vue
    │       ├── DeleteUser.vue
    │       ├── DiscardEditorChanges.vue
    │       ├── ResolveConflict.vue
    │       └── CurrentPassword.vue
    ├── views/                 # halaman (routing target)
    │   ├── Layout.vue         # layout umum (Header + Prompts)
    │   ├── Files.vue          # halaman utama file manager
    │   ├── Login.vue          # halaman login/signup
    │   ├── Share.vue          # halaman share publik
    │   ├── Settings.vue       # shell pengaturan (tab navigasi)
    │   ├── Errors.vue         # halaman error (403/404/500/dll.)
    │   ├── files/
    │   │   ├── FileListing.vue  # grid/daftar file
    │   │   ├── Editor.vue       # editor teks/kode
    │   │   └── Preview.vue      # preview media (pdf/epub/csv/image/audio/video)
    │   └── settings/
    │       ├── Profile.vue   # pengaturan profil
    │       ├── Users.vue     # manajemen pengguna (admin)
    │       ├── Global.vue    # pengaturan global (admin)
    │       └── Shares.vue    # manajemen share
    ├── assets/                # asset statis (logo, icons)
    └── css/                   # file stylesheet global
        └── ...                # SCSS/CSS komponen & tema
```

### 3.2 Alur Build (Production) — Embedding ke Go

1. `npm run build` → Vite menghasilkan `frontend/dist/` (HTML, JS, CSS ter-bundle).
2. `frontend/assets.go` (`//go:embed dist/*`) men-embed seluruh `dist/` ke binary Go.
3. Template `public/index.html` diproses server Go (`[{[ .Json ]}`, `[{[ .StaticURL ]}]`, dll.) untuk **injeksi konfigurasi** `window.FileBrowser`.
4. Binary hasil `go build` berisi seluruh frontend → deploy satu file executable.

> Beda kunci `index.html` vs `public/index.html`:
> - `index.html` (dev): `window.FileBrowser` diisi nilai placeholder statis.
> - `public/index.html` (production): `window.FileBrowser = [{[ .Json ]}]` di-inject dari backend, plus reCaptcha/CSS opsional (`.ReCaptcha`, `.CSS`).

---

## 4. Konfigurasi Runtime (Konstanta `window.FileBrowser`)

Backend meng-inject objek `window.FileBrowser` (lihat `utils/constants.ts`). Ini adalah **single source of truth** konfigurasi yang diteruskan backend → frontend:

| Field | Tipe | Fungsi |
|-------|------|--------|
| `AuthMethod` | string | Metode auth (`json`, `proxy`, `noauth`) |
| `BaseURL` | string | Prefix URL aplikasi |
| `CSS` | boolean | Aktifkan custom CSS |
| `Color` | string | Warna aksen/theme-color |
| `DisableExternal` | boolean | Sembunyikan link eksternal (File Browser repo) |
| `DisableUsedPercentage` | boolean | Matikan indikator penggunaan disk |
| `EnableExec` | boolean | Aktifkan terminal/exec |
| `EnableThumbs` | boolean | Aktifkan thumbnail |
| `LogoutPage` | string | URL redirect logout |
| `LoginPage` | boolean | Tampilkan halaman login |
| `Name` | string | Nama instance (judul PWA) |
| `NoAuth` | boolean | Mode tanpa autentikasi |
| `ReCaptcha` | boolean/objek | Konfigurasi reCAPTCHA |
| `ResizePreview` | boolean | Izinkan resize preview image |
| `Signup` | boolean | Aktifkan registrasi |
| `StaticURL` | string | URL prefix asset statis |
| `Theme` | string | Tema CSS |
| `TusSettings` | object | `{ chunkSize, retryCount }` upload |
| `Version` | string | Versi aplikasi |

Konstanta ini dibaca sekali saat init (`utils/constants.ts`) dan diekspor sebagai nilai statis yang dipakai seluruh aplikasi (mis. `baseURL`, `staticURL`, `signup`, `noAuth`, `enableExec`, dst.).

---

## 5. Entry Point & Bootstrap

### 5.1 `src/main.ts` — urutan inisialisasi

1. Import stylesheet global (CSS).
2. Inisialisasi **Vue I18n** (default locale + lazy-load locales).
3. Buat **Pinia** store instance.
4. Buat router; sync ke store router jika menggunakan `useRouterStore`.
5. Buat root Vue app.
6. `app.use(pinia)`, `app.use(router)`, `app.use(i18n)`.
7. `app.mount("#app")`.
8. Hapus elemen spinner `#loading` setelah mount (memberikan transisi loading awal).

### 5.2 `src/App.vue`

Komponen akar. Merender `<router-view>` untuk halaman aktif, dan pada mode tertentu merender komponen global seperti `Prompts` (modal) dan shell. Mengamati state auth untuk memutuskan tampilan.

---

## 6. Routing (Vue Router 5) — `src/router/index.ts`

Struktur route utama:

| Path | View | Akses | Keterangan |
|------|------|-------|------------|
| `/login` | `Login.vue` | publik (jika `LoginPage`) | Auth |
| `/files/:path?` | `Files.vue` | login-only | Halaman utama file manager |
| `/share/:hash/:path?` | `Share.vue` | publik (hash share) | Share publik & ber-password |
| `/settings/profile` | `Settings` + `Profile.vue` | login-only | Profil |
| `/settings/shares` | `Settings` + `Shares.vue` | `perm.share` | kelola share |
| `/settings/global` | `Settings` + `Global.vue` | `perm.admin` | settings global |
| `/settings/users` | `Settings` + `Users.vue` | `perm.admin` | kelola user |
| `/settings/users/:id` | `Settings` + `Users` (detail) | `perm.admin` | edit user |
| fallback `*` | `Errors.vue` | publik | 404 |

**Navigation Guards**: menjaga `meta.requiresAuth` dan `meta.requiresAdmin`/`perm`; redirect ke `/login` bila belum autentikasi, atau ke `Errors` bila tak punya izin.

---

## 7. Layer API Client — `src/api/*`

Semua komunikasi backend dikapsulkan di folder `api/`. Tidak ada komponen yang memanggil `fetch` langsung ke file ops selain lewat modul ini.

### 7.1 `utils.ts` — fondasi HTTP
- `fetchURL(url, args)` — wrapper `fetch` dengan:
  - Header `X-Auth` berisi token JWT (jika ada).
  - Prepends `baseURL`.
  - Parsing JSON response.
  - **Error handling seragam** → melempar objek `{ status, message }` yang dipetakan ke pesan i18n.
- `formatURL` — menggabungkan URL dengan benar.
- `throwError` / `getErrorMessage` — memetakan kode HTTP ke kunci i18n.

### 7.2 `files.ts` — operasi file (CRUD)
- `list(path)` → ambil isi direktori.
- `newFile(newFileName, path)` → buat file.
- `newDir(newDirName, path)` → buat folder.
- `remove(files)` → delete.
- `rename(file, newName)` → rename.
- `copy(files)` / `move(files)` → salin/pindah (via clipboard store).
- `download(url, filename)` / `downloadByPath(path)` → unduh.
- `getContent(path)` → ambil isi file (untuk editor).
- `putContent(url)` → simpan isi file (upload/editor).
- `compress(path, files, algorithm, outName)` → arsip zip/tar.
- `extract(file, destination)` → ekstrak arsip (jika backend mendukung).
- `raw(path)` → stream raw.
- `usage(path)` → statistik penggunaan disk.
- `checksum` → hash file.
- `patch` (partial) → operasi parsial sesuai perm.

### 7.3 `search.ts` — pencarian streaming
Pencarian dilakukan lewat **StreamingResponse**: fetch POST ke endpoint search, hasil di-streaming baris-per-baris (buffer `NDJSON`/sse-chunks), di-decode dan dikirim ke callback progresif (mendukung pencarian besar tanpa memuat semua sekaligus).

### 7.4 `commands.ts` — WebSocket executor (terminal)
- `commands(path, cmd, onEvent, onClose)`:
  - Membuka **WebSocket** ke endpoint exec.
  - Mengirim command untuk dieksekusi di backend (path working dir).
  - `onEvent(data)` dipanggil per chunk output (untuk terminal real-time).
  - `onClose()` saat selesai (digunakan `Shell.vue`).

### 7.5 `tus.ts` — upload resumable (TUS)
- Memakai `tus-js-client` dengan `TusSettings` (`chunkSize`, `retryCount`).
- Mendukung upload besar dengan resume, progres callback (`onProgress`), dan endpoint `POST /api/tus`.

### 7.6 `pub.ts`, `share.ts`, `settings.ts`, `users.ts`, `index.ts`
- `pub.ts` — akses share publik via hash.
- `share.ts` — CRUD share (list, create, delete) + validasi password share.
- `settings.ts` — ambil/ubah pengaturan global (admin).
- `users.ts` — CRUD pengguna (admin).
- `index.ts` — re-export semua sub-modul sebagai `api` namespace (mis. `import { files as api } from "@/api"`).

---

## 8. State Management (Pinia) — `src/stores/*`

### 8.1 `auth.ts` — autentikasi & sesi
- `state`: `user` (IUser), `isLoggedIn`, `isAdmin`, dsb.
- `actions`: `login`, `logout`, `setUser`, `loadUser`.
- `getters`: cek permission pengguna.
- Menyimpan/melepas **token JWT** dan mengatur header `X-Auth`.

### 8.2 `file.ts` — state file listing
- `state`: `files` (daftar Resource), `selected` (file terpilih), `req` (request/path saat ini), `isFiles`.
- `getters`: `selectedCount`, `isSelected`, `isRootDirectory`.
- `actions`: `fetch`, `select`, `deselect`, `deselectAll`, `addFiles`, `setRequst`, dsb.
- Menangani sorting, navigasi folder, dan *selected state* untuk operasi batch.

### 8.3 `layout.ts` — UI/global layout
- `state`: `loading`, `showShell`, `currentPrompt`, `currentPromptName`, `headerTitle`.
- `actions`: `showHover(prompt)`, `closeHovers`, `toggleShell`, `setLoading`.
- Menentukan modal yang tampil (`currentPromptName` dibaca `Prompts.vue`).

### 8.4 `upload.ts` — antrian upload
- `state`: `uploads` (daftar `UploadTask`).
- `getters`: `percentage` (progres agregat).
- `actions`: `update(id, ...)`, `add`, `retry`, `delete`.
- Memakai **`UPLOADS_LIMIT = 5`** — maksimal 5 upload berjalan paralel (antrian di belakangnya).

### 8.5 `clipboard.ts` — operasi copy/move
- `state`: `files`, `clip`, `copy` (boolean; false = move).
- `actions`: `copy`, `move`, `reset`.
- Menyimpan daftar file yang dipilih untuk operasi cut/copy antar folder.

### 8.6 `router.ts` — sinkronisasi route
- Menjaga state terkini dari route aktif untuk keperluan store lain (opsional; beberapa build memakai langsung `useRoute`).

---

## 9. Views (Halaman) — `src/views/*`

### 9.1 `Files.vue` — halaman inti
- Menampilkan `FileListing.vue` + `Breadcrumbs.vue` di dalam layout.
- Mengelola path aktif, sorting, dan *virtual* navigation.
- Header menampilkan aksi kontekstual (upload, new folder, search, list/grid toggle).
- Memuat data direktori via store `fileStore.fetch`.

### 9.2 `FileListing.vue` — grid/list file
- Merender setiap `Resource` sebagai entri (folder/file).
- Dukungan **multi-selection** (checkbox/klik, select-all), selection persistence.
- *Context menu* (klik kanan) dengan aksi: rename, move, copy, download, delete, info, compress, extract, dan lainnya (disesuaikan permission pengguna).
- Dukungan drag & drop upload, dan navigation folder.

### 9.3 `Share.vue` — share publik
- Menampilkan file/direktori yang di-share via hash.
- Jika share ber-password → form password (`error.status === 401`).
- Aksi: download, multi-select, copy link download.
- Keamanan: tak menampilkan operasi tulis; hanya sesuai permission share.

### 9.4 `Settings.vue` — shell pengaturan
- Layout tab: Profile / Shares (jika `perm.share`) / Global (admin) / Users (admin).
- Menampilkan `router-view` untuk sub-halaman.
- Kartu "sunset" (peringatan keamanan dari upstream) untuk admin.

### 9.5 `files/Editor.vue` — editor teks
- Berbasis **Ace Editor** (`ace-builds`) dengan syntax highlighting, `ext-modelist`, `ext-language_tools`.
- Upload via `PUT` (`putContent`).
- Fitur: fontSize +/- , preview Markdown (`marked` + `marked-katex-extension`, di-sanitasi `DOMPurify`), command palette, copy/cut/paste.
- Tema editor dari `utils/theme.ts` (dark/light mengikuti konfigurasi).
- Aksi save didasarkan `perm.modify`.

### 9.6 `files/Preview.vue` — preview media
- Menangani tipe: **image** (resize toggle), **pdf**, **csv**, **audio**, **video**, **epub** (`vue-reader`/`epubjs`).
- Header menampilkan aksi: rename (`perm.rename`), edit-as-text (csv), delete (`perm.delete`), download (`perm.download`), open-direct, info.
- `@touchmove.prevent`/`@wheel.prevent` mencegah scroll halaman saat memanipulasi preview.
- Memakai `ResizePreview` config untuk image.

### 9.7 `Login.vue` — autentikasi
- Form login; mendukung signup (`Signup`) dan mode `NoAuth` otomatis.
- Menyimpan token, mengatur redirect setelah sukses.
- Menampilkan error kredensial salah via i18n.

### 9.8 `Errors.vue` — halaman error
Memetakan kode → ikon + pesan i18n:
- `0` → "connection" (`cloud_off`), `403` → forbidden, `404` → notFound, `500` → internal (default).

---

## 10. Komponen UI — `src/components/*`

### 10.1 `Shell.vue` — terminal web
- Terminal interaktif yang memanggil **`commands()`** (WebSocket exec).
- Input `contenteditable` + riwayat command (arrow up/down).
- Command khusus lokal: `clear`, `exit` (toggle shell).
- Output stream di-render real-time; ANSI color di-strip sementara.
- **Draggable resize** divider untuk menyesuaikan tinggi terminal.
- Rendering hanya relevan jika `isFiles`.

### 10.2 `Sidebar.vue` — navigasi samping
- Menu: username (→ profile), **My Files**, New Folder / New File (`perm.create`), Settings (admin), logout.
- Versi logout: tombol login/signup.
- Menampilkan **penggunaan disk** (`api.usage`) via `ProgressBar` + teks `used/total` (kecuali `disableUsedPercentage`).
- Footer: versi + link help/external.

### 10.3 `Breadcrumbs.vue`
- Menampilkan jalur navigasi (`/files/...`) sebagai breadcrumb, klik untuk navigasi.
- Dasar path dapat diubah (mis. `/share/:hash`).

### 10.4 `ContextMenu.vue`
- Menu konteks klik-kanan; menerima posisi (x,y) dan daftar aksi sesuai permission.
- Menutup saat klik di luar / tombol esc.

### 10.5 `header/*`
- `HeaderBar.vue` — bar header umum: tombol menu (sidebar), logo, slotted actions (`<template #actions>`).
- `Action.vue` — tombol aksi dengan ikon Material + label i18n + optional counter.
- `Title.vue` — judul halaman.

### 10.6 `prompts/*` — modal
- `Prompts.vue` — dispatcher: membaca `currentPromptName` dari layout store, merender modal terkait via `<component :is>` di dalam `BaseModal`.
- Daftar dialog: Info, Help, Delete, Rename, Move, Copy, NewFile, NewDir, Download, Replace, Upload, Share, ShareDelete, DeleteUser, DiscardEditorChanges, ResolveConflict, CurrentPassword.

---

## 11. Utilitas — `src/utils/*`

| File | Fungsi |
|------|--------|
| `constants.ts` | Baca `window.FileBrowser` → konstanta global |
| `auth.ts` | Login/logout, token header |
| `url.ts` | Manipulasi path URL (join, normalize, base) |
| `theme.ts` | Pemetaan tema editor (dark/light) |
| `upload.ts` | Logika upload (chunking, progress) |
| `encodings.ts` | Encoding base64/UTF-8 |
| `css.ts` | Helper CSS |
| `cookie.ts` | Ekstrak param dari cookie (untuk pre-auth) |
| `clipboard.ts` | Clipboard native + fallback |
| `buttons.ts` | Definisi tombol aksi terstandar |
| `previews.ts` | Deteksi tipe preview |
| `download.ts` | Trigger unduhan browser |
| `files.ts` | Util manipulasi file |

---

## 12. i18n — `src/i18n/*`

- `index.ts` — inisialisasi `createI18n`; detector bahasa browser; **lazy-load** file locale per bahasa.
- `locales/*.json` — pesan terjemahan (en, id, dll.) diorganisir per namespace: `files.*`, `buttons.*`, `login.*`, `settings.*`, `sidebar.*`, `errors.*`, `prompts.*`, dst.
- Plugin `@intlify/unplugin-vue-i18n` meng-kompilasi JSON saat build (kecil & efisien).

---

## 13. Types — `src/types/*`

| File | Definisi kunci |
|------|----------------|
| `api.d.ts` | Tipe request/response API (ListRequest, dll.) |
| `file.d.ts` | `Resource`, `Directory`, tipe file |
| `user.d.ts` | `IUser`, `Permissions` |
| `global.d.ts` | Deklarasi `window.FileBrowser` & `window.__prependStaticUrl` |
| `layout.d.ts`, `settings.d.ts`, `upload.d.ts`, `toast.d.ts` | Tipe terkait masing-masing domain |

**Catatan permission penting**: tipe `Permissions` frontend mencakup field yang menjadi bahan ekspansi masa depan (di luar set backend default), mis. flag untuk `copy`/`move`/`upload`/`shell` — relevan dengan roadmap PRD (granular role-level security, Layer 7).

---

## 14. Keamanan di Sisi Frontend

1. **XSS**: Preview Markdown/HTML di-sanitasi dengan `DOMPurify` (Editor.vue, Preview.vue).
2. **Auth**: Token JWT dikirim via header `X-Auth` (bukan URL param) saat komunikasi API.
3. **Share password**: Share publik ber-password ditangani; kredensial salah → ulang, tidak pernah bocorkan konten.
4. **No sensitive in client**: Frontend hanya menerima data yang diizinkan permission backend; UI menyembunyikan aksi yang bukan hak pengguna (`v-if="user.perm.*"`).
5. **Output terminal**: ANSI escape di-strip (prevent render injection) sampai di-handle lebih baik.

> **IMPORTANT**: Keamanan otoritatif tetap di **backend**. Frontend hanya *menyembunyikan* UI; backend wajib memvalidasi permission setiap request (lihat Layer 7 & Layer 3).

---

## 15. Konvensi & Catatan Pengembangan

- Component baru: **Composition API + TypeScript** (`<script setup lang="ts">`).
- Beberapa file lama masih **Options API** non-TS (`Shell.vue`, beberapa `prompts/*`) — dikecualikan dari `tsconfig.app.json` (lihat komentar "can be removed once those files are properly migrated").
- Import alias `@/` → `src/`.
- Seluruh teks UI lewat **i18n**, jangan hardcode string (kecuali label teknis).
- Material Icons via kelas `material-icons` (font ligature).
- Build production = `npm run build` → `dist/` di-embed via `assets.go`.

---

## 16. Roadmap Alignment (Langkah Berikutnya terkait Frontend)

Berdasarkan PRD, ekspansi frontend yang relevan di masa depan:

1. **Podman Dashboard/Manager** — view baru (mis. `views/podman/*`) + API client Podman pada `src/api/`.
2. **Website Deployer Wizard** — form multi-langkah; komponen stepper; integrasi reverse proxy & SSL config.
3. **SSH Tunneling UI** — panel manajemen koneksi SSH tunnel; direkomendasikan memakai WebSocket (pola yang sama dengan `Shell.vue`/`commands.ts`).
4. **Granular Permissions** — perbaikan UI untuk flags permission baru yang sudah ada di tipe frontend (`copy`, `move`, `upload`, `shell`), selaras dengan Layer 7.
5. Setiap fitur baru tetap mengikuti pola: **view + component + store + api module + i18n keys + types**.

---

## 17. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 1: Frontend (navigator).
- `docs/PRD.md` — kebutuhan fitur yang memandu evolusi frontend.
- `docs/AGENTS.md` — panduan workflow agen (konvensi penulisan, branching).
