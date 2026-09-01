# Layer 10 — Cache & CDN

> Dokumen teknis mendalam untuk lapisan **Cache & CDN** dari `m0nskyFileManager`.
> Dokumen ini adalah *layer-specific* yang dirujuk dari `docs/ARCHITECTURE.md` (Layer 10).
> Ditulis dalam Bahasa Indonesia dengan label teknis berbahasa Inggris.

---

## 1. Ringkasan (Overview)

Lapisan **Cache & CDN** mempercepat penyajian data dan aset melalui **caching** (lokal/ter-distribusi) serta penyajian aset statis yang efisien.

Sesuai `ARCHITECTURE.md` Layer 10, tanggung jawab:
1. Menyimpan **cache thumbnail/preview** agar tidak dihitung ulang.
2. Mendukung **cache ter-distribusi antar-instance (Redis)**.
3. Menyajikan **aset statis frontend** secara efisien.

Komponen & teknologi nyata:
- **diskcache** (`diskcache/`) — disk/memory cache, implement `Interface`.
- **Redis** — upload cache multi-instance (`compose.yaml`, `--redisCacheUrl`).
- **static assets** — embedded ke binary Go (`frontend/assets.go`).
- **HTTP cache headers / ETag** — `Cache-Control`, `ETag`.

> **CDN**: integrasi CDN aset statis belum diimplementasi (roadmap). Namun fondasi HTTP-caching (Cache-Control, gzip) telah ada untuk mendukung CDN di depan aplikasi.

---

## 2. Peta Struktur File Cache & CDN (Lengkap)

```
# Cache primitif (disk) — abstraction Interface
├── diskcache/
│   ├── cache.go            # Interface {Store, Load, Delete}
│   ├── file_cache.go       # FileCache — cache di disk, SHA-1 key, scoped locks
│   ├── noop_cache.go       # NoOp — cache nonaktif
│   └── file_cache_test.go  # unit test FileCache

# Upload cache (tracking upload aktif) — memory / redis
├── http/
│   ├── upload_cache_memory.go  # UploadCache interface + memory impl (ttlcache)
│   ├── upload_cache_redis.go   # Redis impl (go-redis v9)
│   └── upload_cache_memory_test.go

# Static assets & HTTP cache headers
├── http/
│   ├── static.go           # index/static handler, Cache-Control public max-age, gzip .js
│   ├── headers.go          # global Cache-Control no-cache/no-store
│   ├── preview.go          # thumbnail/preview cache (FileCache) + Cache-Control private
│   ├── preview_enum.go     # PreviewSize thumb/big
│   ├── resource.go         # ETag generation (upload/save) + cache invalidation
│   ├── raw.go / subtitle.go / tus_handlers.go  # Cache-Control private/no-store
│   └── upload_cache_memory.go (lihat atas)
├── frontend/
│   └── assets.go           # //go:embed dist/* → binary
└── cmd/
    └── root.go             # wiring: FileCache (--cacheDir) + UploadCache (--redisCacheUrl)
```

---

## 3. Cache Interface — `diskcache/cache.go`

```go
type Interface interface {
	Store(ctx context.Context, key string, value []byte) error
	Load(ctx context.Context, key string) (value []byte, exist bool, err error)
	Delete(ctx context.Context, key string) error
}
```

- Kontrak untuk semua backend cache (disk/memory/redis).
- `exist bool` membedakan "key tidak ada" (`exist=false`) vs error nyata (`err != nil`).
- Dipakai sebagai tipe parameter `FileCache` di handler (`previewHandler`, `resource*`).

---

## 4. Cache Disk — `FileCache` (`diskcache/file_cache.go`)

### 4.1 Struktur & pembuatan

```go
type FileCache struct {
	fs afero.Fs
	scopedLocks struct { sync.Mutex; sync.Once; locks map[string]sync.Locker }
}
func New(fs afero.Fs, root string) *FileCache {
	return &FileCache{fs: afero.NewBasePathFs(fs, root)}
}
```

- Berbasis **afero** (abstraksi FS) → mudah diuji dengan `afero.NewMemMapFs()`.
- **`BasePathFs`** membatasi seluruh operasi cache ke bawah `root`.

### 4.2 Nama file (SHA-1)

```go
func (f *FileCache) getFileName(key string) string {
	hasher := sha1.New(); hasher.Write([]byte(key))
	hash := hex.EncodeToString(hasher.Sum(nil))
	return fmt.Sprintf("%s/%s/%s", hash[:1], hash[1:3], hash)
}
```

- Key → SHA-1 → path `a/62/a62f2...` (sharding 2 tingkat) → distribusi merata di direktori, hindari satu folder penuh.
- Contoh hasil untuk key `"key"`: `a/62/a62f2225bf70bfaccbc7f1ef2a397836717377de`.

### 4.3 Operasi

- **`Store`**: `getScopedLocks(key)` (lock per key) → `MkdirAll(dir,0700)` → `WriteFile(path, value, 0700)`.
- **`Load`**: buka file → `io.ReadAll` → kembalikan bytes; tidak ada → `(nil,false,nil)`.
- **`Delete`**: buka kunci → `Remove`; `os.ErrNotExist` dianggap sukses.
- **`getScopedLocks`**: map lock lazy-init via `sync.Once`; lock per-key (bukan global) → kecepatan akses bersamaan tinggi.

---

## 5. Cache Non-Aktif — `NoOp` (`diskcache/noop_cache.go`)

- `NewNoOp()` → object tanpa operasi (Store/Load/Delete mengembalikan no-op).
- Default bila `--cacheDir` kosong (dipakai `cmd/root.go`).
- Berguna saat disable cache tanpa mengubah alur kode (Null Object pattern).

---

## 6. Upload Cache — tracker upload aktif

### 6.1 Interface `UploadCache` (`http/upload_cache_memory.go`)

```go
type UploadCache interface {
	Register(filePath string, fileSize int64, remove func() error)
	Complete(filePath string)
	GetLength(filePath string) (int64, error)
	Touch(filePath string)
	Close()
}
```

- **Register**: catat upload + ukuran; `remove` callback untuk hapus file parsial bila upload expired (harus via filesystem scoped user → anti symlink escape).
- **Complete**: hapus dari cache saat upload selesai.
- **GetLength**: ukuran file aktif (untuk TUS balasan length).
- **Touch**: refresh TTL agar upload panjang tetap hidup.
- Dibuat via `NewUploadCache(redisURL)`: kosong → memory; terisi → redis.

### 6.2 Memory (`memoryUploadCache`)

- Basis **`jellydator/ttlcache/v3`**, TTL `uploadCacheTTL = 3 * time.Minute`.
- `OnEviction` dengan reason `Expired` → panggil `remove()` untuk hapus file parsial (scoped).
- Contoh desain keamanan: komentar eksplisit bahwa penghapusan leviat `remove` (bukan raw `os.Remove` path), sehingga directory ancestor yang diswap jadi symlink **tidak bisa** melewatkan delete keluar scope.

### 6.3 Redis (`redisUploadCache`) — multi-instance

- Basis **`github.com/redis/go-redis/v9`**.
- `newRedisUploadCache(url)`: `redis.ParseURL` → `redis.NewClient` → `Ping` (uji koneksi).
- Key: `"filebrowser:upload:" + filePath`.
- **Register**: `SET key size EX uploadCacheTTL`.
- **Complete**: `DEL`.
- **GetLength**: `GET`; `redis.Nil` → "no active upload"; parse int; panggil `Touch`.
- **Touch**: `EXPIRE key uploadCacheTTL`.
- `remove` callback **tidak dipakai** pada backend redis (file parsial tidak dihapus di eviction — dicatat komentar).
- Dipilih saat `--redisCacheUrl` set (multi-instance berbagi state upload TUS).

---

## 7. Static Assets & HTTP Caching

### 7.1 Embed aset (`frontend/assets.go`)

```go
//go:build !dev
package frontend
//go:embed dist/*
var assets embed.FS
func Assets() embed.FS { return assets }
```

- Build produksi meng-embed `dist/` (hasil Vite) ke binary Go → satu biner; `fs.Sub(..., "dist")` di `cmd/root.go` → `assetsFs` → `getStaticHandlers`.

### 7.2 Static handler (`http/static.go`)

- **`index` handler** (`/`):
  - Method non-GET → `404`.
  - `x-xss-protection: 1; mode=block`, `X-Content-Type-Options: nosniff`.
  - Render `public/index.html` dengan `Delims "[{[", "]}]"` + data template (`StaticURL`, `Version`, `BaseURL`, dsb.) — lihat Layer 1.
- **`static` handler** (`/static/`):
  - `Cache-Control: public, max-age=86400` (**1 hari**) — cache publik untuk aset statis.
  - Trailing `/` → 404; dukungan branding override (`img/`, `custom.css`).
  - **`.js` khusus**: cari `{path}.gz`; jika client `Accept-Encoding: gzip` → kirim sebagai gzip (`Content-Encoding: gzip`), else dekompres dan kirim. Ini adalah **kompresi gzip pre-built** (diproduksi Vite `compression` plugin, Layer 7).

### 7.3 Global headers (`http/headers.go`)

```go
var globalHeaders = map[string]string{
	"Cache-Control": "no-cache, no-store, must-revalidate",
}
```
- Diterapkan ke *semua* respons (default) → API tidak di-cache; hanya aset `/static/` yang memakai `public, max-age`.

### 7.4 Headers per-handler

| Handler | Header | Makna |
|---------|--------|-------|
| `previewHandler` | `Cache-Control: private` | Preview disimpan hanya di cache client (private user) |
| `rawHandler` | `Cache-Control: private` | Raw file private |
| `subtitleHandler` | `Cache-Control: private` | Subtitle private |
| `tus POST` | `Cache-Control: no-store` | Upload tidak di-cache |

---

## 8. ETag (Validasi HTTP caching)

`http/resource.go` (upload & save):

```go
etag := fmt.Sprintf(`"%x%x"`, info.ModTime().UnixNano(), info.Size())
w.Header().Set("ETag", etag)
```

- ETag = hash dari **ModTime (nanodetik) + ukuran file**.
- Dipakai client/proxy untuk **subsequent conditional request** (`If-None-Match`) → menghemat bandwidth.

---

## 9. Preview/Thumbnail Cache (inti layer)

### 9.1 Alur (`http/preview.go`)

```
 previewHandler
   key = previewCacheKey(file, size)   // RealPath + ModTime.Unix + size
   resizedImage, ok = fileCache.Load(key)
   │                               
   ├─ ok=true  → pakai cache
   └─ ok=false → createPreview (resize via imgSvc)
                    → simpan ke fileCache (async goroutine)
   lalu: Cache-Control: private + http.ServeContent
```

### 9.2 Key cache

```go
func previewCacheKey(f *files.FileInfo, previewSize PreviewSize) string {
	return fmt.Sprintf("%x%x%x", f.RealPath(), f.ModTime.Unix(), previewSize)
}
```
- Berbasis path, mtime, dan ukuran preview → **invalid otomatis** saat file berubah (mtime beda → key beda).

### 9.3 Ukuran preview (`preview_enum.go`)

- `PreviewSizeThumb` (0) — 256×256, `ResizeModeFill`, `QualityLow`, `FormatJpeg`.
- `PreviewSizeBig` (1) — 1080×1080, `ResizeModeFit`, `QualityMedium`.
- Menghindari resize berulang → hemat CPU (work pool img, Layer 6).

### 9.4 Cache invalidation (`http/resource.go:382`)

- Saat file dihapus/berubah, `fileCache.Delete(ctx, previewCacheKey(file,size))` membersihkan preview lama.

---

## 10. Wiring di `cmd/root.go`

```go
var fileCache diskcache.Interface = diskcache.NewNoOp()
if cacheDir != "" {
	os.MkdirAll(cacheDir, 0700)
	fileCache = diskcache.New(afero.NewOsFs(), cacheDir)   // FileCache disk
}

uploadCache, err := fbhttp.NewUploadCache(redisCacheURL)  // memory | redis
```

- `--cacheDir` → `FileCache` (lokal); kosong → `NoOp`.
- `--redisCacheUrl` → `UploadCache` Redis; kosong → memory.
- Konsisten dengan `compose.yaml` (FB_REDIS_CACHE_URL) untuk multi-instance.

---

## 11. Cache & CDN di Deployment (Layer 5)

- Image / volume: `/database` (Bolt), `/srv` (data), `/config` — cache disk dapat diletakkan di volume jika perlu persistent, namun umumnya **ephemeral**.
- `compose.yaml`: `filebrowser` + `redis` — `filebrowser` memakai Redis via `FB_REDIS_CACHE_URL` → semua instance berbagi upload cache.
- **CDN**: static `/static/` punya `Cache-Control: public, max-age=86400` + gzip → siap di-belakang **CDN/reverse-proxy** (Nginx/Cloudflare). Integrasi CDN penuh (aset statis via URL CDN eksternal) = roadmap.

---

## 12. Keamanan Cache

1. **Upload cache eviction via `remove` scoped** (memory) — anti symlink escape (komentar eksplisit).
2. **Redis** dilindungi ACL di `compose.yaml` (password) — hanya app yang boleh mengakses.
3. **`private` cache-control** untuk preview/raw/subtitle — tidak boleh dibagikan antar user/instance publik.
4. **`no-store`** pada TUS upload — mencegah penyimpanan data upload oleh cache antara.
5. **SHA-1 key sharding** — mencegah path collision/injection (key di-hash, bukan raw).
6. **Preview key berbasis mtime+path** — tidak melayani data basi.

---

## 13. Unit Tests & Catatan Roadmap

- `diskcache/file_cache_test.go` — Store/Load/Update/Delete + path hash `a/62/...`.
- `http/upload_cache_memory_test.go` — register/complete/touch/expire.
- (Roadmap) integrasi **CDN** aset statis, caching data Podman, rate-limit berbasis cache (Layer 9).

---

## 14. Referensi Silang (Related Docs)

- `docs/ARCHITECTURE.md` — Layer 10: Cache & CDN (navigator & roadmap).
- `docs/layers/09-rate-limiting.md` — cache (Redis/ttlcache) sebagai basis rate limiter ter-distribusi.
- `docs/layers/13-backend-static.md` / Layer 5 — static serving, gzip, deployment compose+redis.
- `docs/layers/02-database-storage.md` — Redis/upload cache, TUS handling.
- `docs/layers/06-cloud-compute.md` — cache disk (`--cacheDir`), worker image.
- `docs/layers/07-cicd-vcs.md` — build gzip plugin (Vite) untuk `.js.gz`.
- `docs/PRD.md` — roadmap CDN & caching Podman data.
- `docs/AGENTS.md` — workflow branch `m0nskyBuildAI`.