# Layer 7 — CI/CD & Version Control

> Dokumen teknis mendalam untuk lapisan **CI/CD & Version Control** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 7).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan ini mengelola **siklus pengembangan perangkat lunak** — dari kontrol versi kode, otomatisasi build & test, hingga pembuatan release & distribusi.

Tanggung jawab utama (sesuai `ARCHITECTURE.md` Layer 7):
1. **Version control** — mengelola riwayat kode via **Git** di **GitHub**.
2. **Automation build** — membangun frontend (Vite/pnpm) & backend (Go).
3. **Versioning & release** — mengelola versi dan membuat release.
4. **Quality gate** — menjaga kualitas kode via **linting** & **testing**.

Komponen & teknologi nyata:
- **Git & GitHub** (`M0nskyBoozy/m0nskyFileManager`).
- **Taskfile.yml** (orchestrator task: build:frontend, build:backend, build, release, docs:cli).
- **.golangci.yml** (lint Go).
- **.goreleaser.yml** (release automation & distribusi).
- **Test**: Go test + Vitest (frontend) + lint frontend (ESLint/Vue) + typecheck (vue-tsc).

**Catatan**: lapisan ini **belum** memiliki pipeline CI asli (`.github/workflows` belum ada). Otomatisasi saat ini melalui `Taskfile.yml` + `goreleaser` + `commit-and-tag-version` yang dijalankan lokal/developer. Pipeline CI/CD penuh (termasuk integrasi Podman) merupakan **roadmap**.

---

## 2. Peta Struktur File CI/CD & Version Control (Lengkap)

```
# Version control & repo config (root)
├── .git/                      # metadata Git (history, branches, remotes)
├── .gitignore                 # pengabaian file build/env/editor/log
├── docs/AGENTS.md             # workflow guide (branch m0nskyBuildAI, konvensi commit)

# Orchestrasi task & build
├── Taskfile.yml               # task automation (build frontend/backend, release, docs)
├── go.mod / go.sum            # dependency Go + testify (test)

# Linting & quality
├── .golangci.yml              # golangci-lint v2 (standard + gocritic/govet/revive)
└── frontend/
    ├── eslint.config.js       # ESLint flat config Vue + TS + Prettier
    ├── tsconfig.app.json      # typecheck config (vue-tsc)
    └── package.json           # scripts: lint, typecheck, test (vitest)

# Release & packaging automation
├── .goreleaser.yml            # goreleaser v2 (binary matrix + docker + manifests + homebrew)

# Build & test artifacts inputs
├── main.go                    # entry point
├── */*_test.go                # 33 file unit/security tests (Go)
└── frontend/src/**/__tests__/*.test.ts   # 5 file test Vitest
```

> Lokasi pipeline CI-ready (mis. `.github/workflows/*.yml`) **belum ada** — akan ditambahkan pada roadmap.

---

## 3. Version Control (Git & GitHub)

### 3.1 Remote & branch

```
origin  git@github.com:M0nskyBoozy/m0nskyFileManager.git
Branch  lokal : master, m0nskyBuildAI
Branch  remote: origin/master, origin/m0nskyBuildAI
```

- **`master`** — cabang utama/produksi. **Dilarang diubah** (aturan AGENTS.md).
- **`m0nskyBuildAI`** — cabang kerja aktif untuk semua pengembangan (seluruh docs layer dibuat di sini).
- Belum ada `git tag` (0 tag) → versi masih `(untracked)` / `(unknown)` pada `version.Version`.

### 3.2 Konvensi commit (dari `git log`)

| Prefix | Contoh | Penggunaan |
|--------|--------|------------|
| `docs:` | `docs: setup AGENTS.md workflow guide` | Perubahan dokumentasi |
| `fix:` | `fix: enforce rules on recursive operations...` | Perbaikan bug |
| `feat:` | (perlu ditambahkan) | Fitur baru |
| `chore(release):` | `chore(release): 2.63.23` | Sukses release (commit-and-tag-version) |
| `chore(deps):` | `chore(deps): update dependency ... [security]` | Update dependency |
| `chore:` | `chore: remove renovate.json` | Tugas pemeliharaan |
| `clean up:` | `clean up: remove unused docs...` | Pembersihan |

Pola ini adalah **Conventional Commits** — selaras dengan alat `commit-and-tag-version` dipakai di `Taskfile.yml` untuk update versi otomatis.

### 3.3 Aturan AI/workflow (docs/AGENTS.md)

- **Wajib baca** `PRD.md` → `ARCHITECTURE.md` → `STYLEGUIDE.md` sebelum koding.
- **Konsistensi** struktur & gaya; **validasi** hanya di branch `m0nskyBuildAI`, jangan sentuh `main`/`master`.

---

## 4. Orkestrasi Build — `Taskfile.yml`

Taskfile (Go Task) menjadi **satu pintu otomasi** build & release:

| Task | Perintah inti | Fungsi |
|------|---------------|--------|
| `build:frontend` | `pnpm install --frozen-lockfile` + `pnpm run build` (di `frontend/`) | Bangun aset Vue → `dist/` |
| `build:backend` | `go build -ldflags=...` | Kompilasi binary `filebrowser` |
| `build` | `build:frontend` lalu `build:backend` | Build lengkap frontend + backend |
| `release` | `docs:cli:generate` → commit docs → dry-run → `release:make` | Menjalankan proses rilis |
| `release:dry-run` | `commit-and-tag-version --dry-run --skip` | Simulasi rilis (internal) |
| `release:make` | `commit-and-tag-version -s` | Update versi + tag (internal, prompt) |
| `docs:cli:generate` | `go run . docs` | Generate dokumentasi CLI |

### 4.1 `build:backend` (ldflags versi)

```sh
go build -ldflags='-s -w \
  -X "github.com/filebrowser/filebrowser/v2/version.Version={{.VERSION}}" \
  -X "github.com/filebrowser/filebrowser/v2/version.CommitSHA={{.GIT_COMMIT}}"' \
  -o filebrowser .
```

- `GIT_COMMIT` = `git log -n 1 --format=%h` (short hash).
- `VERSION` = `git describe --tags --abbrev=0 --match=v* | cut -c 2-` (dari tag `v*`).
- Karena belum ada tag, build lokal menghasilkan `Version = (untracked)`; setelah tag dibuat otomatis terisi.

### 4.2 Task `release` (alur)

1. `docs:cli:generate` → memperbarui `docs/cli`.
2. `git add docs/cli` (+ commit `chore(docs): update CLI documentation` jika ada perubahan).
3. `release:dry-run` → simulasi.
4. `release:make` → `pnpm dlx commit-and-tag-version -s` (bump versi + buat tag).

> `commit-and-tag-version` membaca **Conventional Commits** dan otomatis menaikkan major/minor/patch + membuat tag → `VERSION` Taskfile terisi dari tag ini.

---

## 5. Linting Go — `.golangci.yml`

```yaml
version: "2"
linters:
  default: standard
  enable:
    - gocritic
    - govet
    - revive
  exclusions:
    presets:
      - std-error-handling
      - comments
    paths:
      - frontend/
```

- **`golangci-lint` v2** dengan **`default: standard`** (set linter bawaan standar).
- **Ditambah**: `gocritic` (reviewer gaya, mendeteksi pola suboptimal/bug-prone), `govet` (vet bawaan Go — kemungkinan error), `revive` (linter gaya robust, pengganti golint).
- **Exclusion presets**: `std-error-handling` (pola `pkg/errors`/handling error standar) dan `comments`.
- **Path exclusion**: `frontend/` (bukan target linter Go).
- Kualitas lint ini adalah *quality gate* sebelum commit/build.

---

## 6. Linting, Typecheck & Test Frontend (Vite/vitest)

`frontend/package.json` scripts (+ devDeps):

| Script | Perintah | Fungsi |
|--------|----------|--------|
| `typecheck` | `vue-tsc -p ./tsconfig.app.json --noEmit` | Cek tipe TypeScript/Vue |
| `lint` | `eslint src/` | Lint Vue + TS |
| `lint:fix` | `eslint --fix src/` | Auto-fix lint |
| `format` | `prettier --write .` | Format kode |
| `test` | `vitest run` | Jalankan unit test |
| `build` | `pnpm run typecheck && vite build` | Typecheck lalu build |

**ESLint flat config** (`eslint.config.js`):
- `plugin-vue` `flat/essential` + `@vue/eslint-config-typescript` (recommended) + Prettier.
- Rules khusus: nonaktifkan `no-explicit-any` & `vue/multi-word-component-names` (TODO konversi TS legacy), `vue/no-mutating-props` error (shallowOnly).

**Test frontend (Vitest v4.1.0)** — 5 file:

```
frontend/src/
├── components/prompts/__tests__/copy-move-conflict.test.ts
├── css/__tests__/mobile.test.ts
├── utils/__tests__/check-conflict.test.ts
├── utils/__tests__/upload.test.ts
└── views/files/__tests__/upload-conflict-resolution.test.ts
```

Fokus: konflik copy/move, upload, konflik upload, CSS mobile.

---

## 7. Testing Go

**33 file `*_test.go`** tersebar per paket:

```
auth/          hook_test.go, proxy_test.go
cmd/           cmd_test.go
diskcache/     file_cache_test.go
files/         case_test.go, file_test.go, fs_test.go
fileutils/     copy_test.go, file_test.go
http/          auth_test.go, public_symlink_test.go, public_test.go, raw_test.go,
               resource_checksum_test.go, resource_recursive_test.go, resource_test.go,
               rules_path_test.go, rules_recursive_test.go, share_test.go,
               subtitle_test.go, tus_multichunk_test.go, tus_symlink_test.go,
               tus_upload_length_test.go, upload_cache_memory_test.go, utils_test.go
img/           service_test.go
rules/         rules_test.go
runner/        commands_test.go
settings/      dir_test.go
storage/bolt/  share_test.go, users_test.go
users/         storage_test.go, users_test.go
```

- Dependency: **`github.com/stretchr/testify v1.11.1`** (assert/require).
- Banyak test berfokus **keamanan**: symlink escape (`public_symlink_test.go`, `tus_symlink_test.go`), rules recursive (`rules_recursive`, `resource_recursive`), auth hardening (`auth/proxy_test.go`, `auth/hook_test.go` — anti credential injection), TUS multichunk/upload length.
- Test ini dilindungi di `docs/layers/04-auth` & `05` • bagian security.

**Menjalankan** (anjuran):
```sh
go test ./...          # seluruh paket
go test ./http/ -v     # satu paket (verbose)
go vet ./...           # lint/vet (govet)
```

---

## 8. Release & Distribusi — `.goreleaser.yml`

Sudah dijabarkan detail di **Layer 5**. Ringkas untuk layer ini:

- **Project**: goreleaser v2, `project_name: filebrowser`.
- **Builds**: matrix OS/arch (darwin/linux/windows/freebsd/openbsd × amd64/386/arm/arm64/riscv64), ldflags injeksi `Version`/`CommitSHA`.
- **Archives**: `tar.gz` (zip utk windows).
- **Dockers**: buildx multi-arch (alpine + s6), label OCI, manifest `latest`/`vX.Y`/`s6`.
- **Homebrew cask**: publish ke `filebrowser/homebrew-tap` + hapus quarantine.

Ini adalah **langkah distribusi akhir** setelah `task release` menghasilkan tag/versi.

---

## 9. Versioning Pipeline (alur end-to-end)

```
Developer commit (Conventional Commits)
        │
        ▼
task release  →  docs:cli:generate → git add docs/cli → commit
        │            release:dry-run → release:make
        ▼
commit-and-tag-version -s   (bump versi + buat tag org.opencontainers.image/version)
        ▼
VERSION terisi (git describe --tags --match=v*)
        ▼
goreleaser  →  binary matrix + docker images + manifests + homebrew
        ▼
Version.Version / CommitSHA (ldflags) → embedded di release
```

---

## 10. Roadmap CI/CD (sesuai ARCHITECTURE & PRD)

Belum diimplementasi (`.github/workflows` kosong):

1. **Pipeline CI otomatis** — build + lint + test otomatis pada tiap push/PR (Go test, govet/gocritic/revive, vue-tsc, eslint, vitest).
2. **Integrasi Podman** — build & deploy otomatis via Podman (Layer 6/5); pipeline dapat menjalankan container build Podman.
3. **Deploy otomatis** — memanfaatkan `compose.yaml`/goreleaser ke instance (Layer 5 & 6).
4. **Continous security** — menjalankan test keamanan (symlink/recursive/TUS) sebagai gate.

Titik integrasi: folder `.github/workflows/` (belum ada); dapat memanggil task Taskfile (`task build`, `task release`) dan `go test ./...` + `pnpm test`.

---

## 11. Catatan Pengembangan (Saran)

1. **Buat STYLEGUIDE.md** — dirujuk `docs/AGENTS.md` namun belum ada di repo; perlu dibuat sesuai standar m0nsky (sintaks, penamaan, error handling, konvensi commit).
2. **Tambahkan `.github/workflows/ci.yml`** untuk otomatisasi test/lint di setiap PR (roadmap).
3. **Buat tag versi** (`vX.Y.Z`) bila akan rilis — mengaktifkan `VERSION` Taskfile & goreleaser.
4. **Terapkan `go vet` & `golangci-lint run`** dalam workflow sebelum merge.
5. Pastikan semua *quality gate* (typecheck, lint, test) dijalankan sebelum `task release`.

---

## 12. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 7: CI/CD & Version Control (navigator).
- `docs/AGENTS.md` — workflow, branch `m0nskyBuildAI`, aturan commit.
- `docs/PRD.md` — roadmap pipeline CI/CD & integrasi Podman.
- `docs/layers/05-hosting-deployment.md` — goreleaser/container/build (distribusi).
- `docs/layers/04-auth-authorization.md` — test keamanan auth/hook/proxy.
- `docs/layers/06-cloud-compute.md` — eksekusi & roadmap Podman yang terintegrasi CI/CD.
- `STYLEGUIDE.md` — (belum ada; disarankan dibuat) standar penulisan kode.
