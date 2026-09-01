# PRD.md - Product Requirements Document

## m0nskyFileManager

Dokumen ini adalah **Product Requirements Document (PRD)** untuk proyek **m0nskyFileManager**. Dokumen menjelaskan tujuan, fitur, target pengguna, dan kriteria keberhasilan program secara menyeluruh, dan menjadi acuan utama (bersama `ARCHITECTURE.md` dan `STYLEGUIDE.md`) bagi setiap AI Agent maupun pengembang yang bekerja pada repositori ini.

---

## 1. Ringkasan Produk & Tujuan

**m0nskyFileManager** dikembangkan untuk menjadi **private file manager sebuah server** yang berjalan **di browser** melalui **SSH tunneling** pada server. Program ini memiliki tujuan penuh untuk:

- Mendukung integrasi **Podman** secara mendalam, sehingga mempermudah pengguna dalam melakukan pengelolaan **container**, **volume**, serta **images** (ataupun fitur Podman lainnya) langsung dari browser.
- **Ramah bagi pemula** untuk melakukan deploy website mereka menggunakan **wrapper Podman** agar lebih aman dan tanpa perlu memahami konfigurasi server yang kompleks.

Produk ini merupakan kelanjutan dari fork resmi proyek **File Browser** yang dipelihara secara mandiri oleh tim internal **m0nsky**, dengan fokus utama pada pengembangan fitur-fitur berbasis Podman/automation di atas fondasi file manager web yang sudah matang.

---

## 2. Fitur Produk

### 2.1 Manajemen Podman Intuitive (Podman Dashboard & Manager)

Fitur ini adalah **inti utama** program untuk mengelola Podman tanpa perlu mengetikkan perintah CLI yang rumit.

**GUI Container Management:**
- Melihat status container (`Running`, `Stopped`, `Paused`, `Exited`).
- Aksi cepat: `Start`, `Stop`, `Restart`, `Pause`, dan `Delete` container.
- **Live Resource Monitoring**: grafik real-time penggunaan **CPU**, **Memori**, **I/O Disk**, dan **Network** per container.

**Logs:**
- *Streaming logs* container (real-time log viewer).

**Image & Volume Management:**
- *Pull* / *Build* image dari **Container Registry** (Docker Hub, Quay, Quay.io, dll.) atau dari **Dockerfile/Containerfile** lokal.
- *Prune* / hapus image dan volume yang tidak terpakai (*unused resources*).
- Manajemen **persistent volume** (browse file di dalam volume, mount point, backup/restore volume).

**Pod & Network Management:**
- Dukungan fitur khas Podman: membuat dan mengelola **Pods** (grup container yang berbagi IP/namespace).
- Pemetaan port (**port forwarding**) dan konfigurasi jaringan internal.

---

### 2.2 Beginner-Friendly Website Deployer (Podman Wrapper)

Memudahkan pemula meng-online-kan website mereka tanpa harus pusing memikirkan konfigurasi server yang kompleks.

**Smart Reverse Proxy & Auto-SSL:**
- Integrasi otomatis dengan reverse proxy.
- Penerbitan sertifikat SSL otomatis (**Let's Encrypt**) hanya dengan memasukkan nama domain.

**Zero-Config Security Isolation:**
- Menjalankan container secara otomatis dalam mode **Rootless Podman** demi keamanan maksimal.
- Alokasi **resource limit (CPU & RAM max)** secara visual agar website pemula tidak "memakan" seluruh daya server.

---

### 2.3 Web File Manager & Editor

Pengelolaan berkas server berbasis browser yang kaya fitur untuk mendukung pekerjaan web development.

**Core File Operations:**
- **Upload** (dukungan *drag-and-drop* & berkas besar), **Download**, **Move**, **Copy**, **Delete**, **Rename**, **Compress/Extract** (`.zip`, `.tar.gz`).

**Code Editor Terintegrasi:**
- Text/Code editor berbasis web lengkap dengan **syntax highlighting**.
- Kemampuan mengedit **Dockerfile/Containerfile** atau berkas konfigurasi secara langsung di browser.

**File Isolation & Scope Limit:**
- Membatasi akses direktori pengguna agar tidak bisa mengubah berkas sistem OS utama secara tidak sengaja.

---

### 2.4 Keamanan & SSH Tunneling Architecture

Mengingat program ini difungsikan sebagai **private manager** yang berjalan di atas **SSH tunneling**.

**Secure SSH Tunneling Integration:**
- Program berjalan via koneksi SSH aman (**encrypted payload**), sehingga tidak perlu membuka port publik yang berisiko di server.
- Mendukung autentikasi **SSH Key (Ed25519/RSA)** dan password.

**Session Control:**
- **Auto-logout** jika sesi menganggur (*idle timeout*).

**Audit Logs & Backup:**
- Catatan riwayat aksi (*action history*) untuk memantau aktivitas yang dilakukan via web manager.
- Fitur **One-click Backup & Restore** untuk konfigurasi website dan data volume.

---

## 3. Target Pengguna

| Segment User | Karakteristik | Kebutuhan Utama |
|---|---|---|
| **Pemula** | Tidak familier dengan Linux/Podman/CLI | Deploy website dengan cepat, SSL otomatis, resource limit visual, tanpa konfigurasi kompleks |
| **Pengguna Tingkat Lanjut** | Familiar dengan container & server | Manajemen Podman penuh (Rootless, Pods, Volume, Network), monitoring real-time, log streaming |

---

## 4. Summary Indikator Keberhasilan

Program **m0nskyFileManager** dapat dikatakan **100% berhasil** jika:

- **Pemula** dapat mengunggah dan mengelola website, membuatnya online dengan SSL, serta mendeploy menggunakan Podman.
- **Pengguna tingkat lanjut** dapat mengelola Podman (Rootless, Pods, Volume) secara aman hanya dari browser melalui koneksi tunneling SSH.
