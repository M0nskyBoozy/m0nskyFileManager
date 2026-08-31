# AGENTS.md - System Instructions & Workflow for AI

Dokumen ini adalah panduan utama (Single Source of Truth) untuk setiap AI Agent yang bekerja pada repositori `m0nskyFileManager`. 

## MANDATORY RULE FOR AI AGENT

1. **WAJIB BACA DOCS:** Sebelum mengeksekusi perintah koding, perbaikan bug, atau penambahan fitur dari user, AI WAJIB membaca dan memahami 3 dokumen utama berikut secara berurutan:
   - `PRD.md` (Untuk memahami *tujuan*, *logika bisnis*, dan *spesifikasi fitur*).
   - `ARCHITECTURE.md` (Untuk memahami *struktur folder*, *aliran data*, dan *teknologi* yang digunakan).
   - `STYLEGUIDE.md` (Untuk memastikan *gaya penulisan kode*, *penamaan*, dan *standar* m0nsky terpenuhi).
2. **KONSISTENSI:** Dilarang keras membuat struktur folder baru atau gaya penulisan kode yang menyimpang dari `ARCHITECTURE.md` dan `STYLEGUIDE.md`.
3. **VALIDASI:** Jangan pernah mengubah branch `main`. Seluruh pekerjaan hanya boleh dilakukan di branch `m0nskyBuildAI`.

---

## Hierarki Dokumentasi Referensi

Saat menjalankan tugas, gunakan urutan prioritas pemahaman dokumentasi berikut:

1. **`PRD.md`** -> "Apa yang harus dibuat?"
   - Berisi detail fitur, kebutuhan fungsional, dan goal akhir proyek m0nskyFileManager.
2. **`ARCHITECTURE.md`** -> "Di mana & Bagaimana arsitekturnya?"
   - Berisi struktur direktori, pola modul, integrasi backend/frontend, dan pengelolaan state.
3. **`STYLEGUIDE.md`** -> "Bagaimana cara menulis kodenya?"
   - Berisi standar sintaks, penamaan variabel/fungsi, penanganan error (error handling), dan konvensi commit.

---

## Alur Kerja AI (Execution Workflow)

Setiap kali AI menerima prompt dari user untuk menambah/mengubah fitur:
1. Periksa `PRD.md` untuk mencocokkan scope fitur.
2. Cek `ARCHITECTURE.md` untuk menentukan lokasi file yang harus ditambahkan/diubah.
3. Terapkan aturan `STYLEGUIDE.md` dalam penulisan kodenya.
4. Lakukan pengujian/cek sintaks sederhana sebelum melaporkan ke user.
