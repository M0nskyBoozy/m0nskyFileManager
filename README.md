> [!WARNING]
> 
> **File Browser is archived on 2026-09-01**. The last planned release has already shipped. There will be no further releases, bug fixes, or security fixes.   

<p align="center">
  <img src="./branding/banner.png" width="550"/>
</p>

File Browser provides a file managing interface within a specified directory and it can be used to upload, delete, preview and edit your files. It is a **create-your-own-cloud**-kind of software where you can just install it on your server, direct it to a path and access your files through a nice web interface.

**Background:** [Goodbye File Browser, for Real This Time](https://hacdias.com/2026/07/28/filebrowser/), July 2026.

## Tujuan Fork m0nsky

Repositori ini merupakan fork resmi dari project open-source **File Browser** yang dikelola oleh tim internal **m0nsky**. Fork ini dibuat dengan tujuan utama sebagai berikut:

- **Pengembangan dan penyesuaian fitur khusus untuk kebutuhan internal m0nsky.** Kami menambah, memodifikasi, dan menyesuaikan fitur agar sesuai dengan kebutuhan operasional internal, bukan hanya mengikuti rilis upstream.

- **Eksperimen integrasi fitur berbasis AI/automation.** Cabang ini menjadi lingkungan eksperimen untuk mengintegrasikan kemampuan AI dan automation ke dalam alur kerja manajemen file, sebagai langkah awal menuju produk yang lebih cerdas.

- **Menjaga independensi pemeliharaan kode (maintenance) secara mandiri.** Dengan catatan bahwa upstream File Browser sudah tidak lagi dipelihara (archived), kami mengambil alih tanggung jawab pemeliharaan, perbaikan bug, dan keamanan secara mandiri di dalam fork ini.

## Security

Published advisories are listed under [security advisories](https://github.com/filebrowser/filebrowser/security/advisories),
and reporting instructions are in [SECURITY.md](SECURITY.md). Two known issue classes
remain unaddressed and will not be fixed:

- **Command execution, runner, and hooks.** This feature is plagued with vulnerabilities across many published advisories, and would need a full rewrite to be made safe. It is disabled by default; if you re-enable it with `--disable-exec=false`, treat the ability to run commands as equivalent to shell access on the host. Background: [#5199](https://github.com/filebrowser/filebrowser/issues/5199).
- **Session and JWT handling.** Sessions are self-contained JWTs rather than server-side identifiers, so they cannot be revoked, which means that logout, password changes, and renewal leave previously issued tokens valid until they expire, and the same refresh token can be redeemed repeatedly. Assume a leaked token is valid until expiry. Background: [#5216](https://github.com/filebrowser/filebrowser/issues/5216).

If you keep running File Browser, treat it as unmaintained software:

- **Do not expose it directly to the internet.** Put it behind a reverse proxy that terminates TLS and performs its own authentication.
- **Keep the command runner disabled.** It is off by default, so leave it off. See [#5199](https://github.com/filebrowser/filebrowser/issues/5199) and [`docs/command-execution.md`](docs/command-execution.md).
- **Run it unprivileged, inside a container**, with only the directory you intend to serve mounted into it.

## Documentation

Documentation on how to install, configure, and build this project lives in [`docs`](docs) in this repository.

[CONTRIBUTING.md](CONTRIBUTING.md) documents how to build and develop the project, which remains useful to anyone forking it.

## License

[Apache License 2.0](LICENSE) © File Browser Contributors
