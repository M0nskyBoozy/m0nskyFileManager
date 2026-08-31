# m0nskyFileManager

**m0nskyFileManager** adalah fork resmi dari project open-source **File Browser** yang dikelola oleh tim internal **m0nsky**. Aplikasi ini menyediakan antarmuka web untuk mengelola file di dalam direktori tertentu — upload, download, preview, edit, dan hapus file dengan mudah.

## Tujuan Fork

- **Pengembangan fitur khusus kebutuhan internal m0nsky.**
- **Eksperimen integrasi fitur berbasis AI/automation** dalam alur kerja manajemen file.
- **Pemeliharaan kode secara mandiri** (upstream File Browser sudah di-archive dan tidak lagi dipelihara).

Repositori ini merupakan hasil **fork dan pembersihan**: dokumentasi, kontribusi, CI, dan asset bawaan yang tidak diperlukan telah dihapus agar repositori lebih ringan dan bersih.

## Teknologi

- **Backend:** Go (`github.com/filebrowser/filebrowser/v2`)
- **Frontend:** Vue (di dalam folder `frontend/`)

## Build

```sh
go build ./...
```

## Lisensi

[Apache License 2.0](LICENSE) © File Browser Contributors
