# Autoscript VPN — Fork

AutoScript VPN & Tunneling Management System untuk **Enterprise Linux 9 & 10**
(Rocky / RHEL / AlmaLinux / Oracle Linux), `x86_64` & `aarch64`.

Repo ini adalah **fork** dari [risqinf/autoscript](https://github.com/risqinf/autoscript).
Fokus fork ini bukan menambah fitur, tapi **memperbaiki jalur SSH/SlowDNS yang
sebelumnya tidak bisa dipakai client nyata** (HTTP Custom, SlowDNS fork, dan
sejenisnya).

---

## Sumber & atribusi

| | |
|---|---|
| **Proyek asli** | [risqinf/autoscript](https://github.com/risqinf/autoscript) oleh **risqinf** |
| **Lisensi** | [Apache License 2.0](LICENSE) — tidak berubah |
| **Versi dasar** | `v5.6.3` |
| **Status** | Fork independen. **Bukan** produk resmi dan **tidak** didukung/di-endorse oleh penulis asli. |

Semua kode asli tetap milik penulisnya. Daftar perubahan fork ini terdokumentasi
di [`PATCHES.md`](PATCHES.md) dan [`CHANGELOG.md`](CHANGELOG.md), sesuai
persyaratan Apache-2.0 §4 (retensi lisensi + penandaan perubahan).

---

## Perbedaan dari upstream

| # | Masalah di upstream | Perbaikan |
|---|---|---|
| 1 | Bundled `dnstt-server` hanya menerima `QTYPE == TXT`; qtype lain dijawab `NXDOMAIN`, dan rdata non-TXT memakai prefix panjang gaya TXT. Client yang menyebar paket ke TXT/CNAME/NULL/MX/NS/SRV jadi **konek tapi tanpa data**. | `dnstt-server` di-patch (semua QTYPE diterima, rdata non-TXT = raw), di-build untuk `amd64` + `arm64`, di-vendor di `bin/` dengan checksum. |
| 2 | `dropbear -F -w -g` → root selalu ditolak (`root login rejected`), padahal SlowDNS forward ke `127.0.0.1:109`. | Flag `-w -g` dihapus di `install.sh` dan `scripts/menu/menu-dropbear.sh`. |
| 3 | Akun SSH dibuat `useradd -M -s nologin` → tanpa home dir (dropbear: `Error changing directory`) dan shell menolak login. | `useradd -m -s /bin/bash` di semua jalur provisioning (CLI, menu, API, bulk, autounban, recovery, trial). |
| 4 | Tidak ada deteksi kalau binary SlowDNS masih build stock. | `slowdns_verify_binary()` dicek otomatis setelah install, warning eksplisit kalau belum dipatch. |

Detail teknis, patch diff, dan cara rebuild binary ada di [`PATCHES.md`](PATCHES.md).

---

## Verifikasi cepat setelah install

```bash
NS=$(cat /etc/slowdns/nameserver)
dig +noall +comments +answer -p 5300 NULL "mtuclg6lqo47x2gzqzcvlattc2xa.${NS}" @127.0.0.1
```

Harus keluar **`status: NOERROR`** dan **`IN NULL \# 0`**.
Kalau `NXDOMAIN` atau `NULL \# 1 00`, binary-nya masih stock.

---

## Instalasi

| Distribution | Versi | Arsitektur |
|---|---|---|
| Rocky Linux | `9.x`, `10.x` | `x86_64`, `aarch64` |
| RHEL | `9.x`, `10.x` | `x86_64`, `aarch64` |
| AlmaLinux | `9.x`, `10.x` | `x86_64`, `aarch64` |
| Oracle Linux | `9.x`, `10.x` | `x86_64`, `aarch64` |

- Minimum: 1 vCPU / 1 GB RAM, akses `root`
- Domain/subdomain dengan A record mengarah ke IP server

```bash
dnf install -y wget curl openssl \
  && curl -fsSL --proto '=https' --tlsv1.2 -o install.sh \
     https://raw.githubusercontent.com/Matsumiko/autoscript-vpn-el9/main/install.sh \
  && chmod +x install.sh && ./install.sh
```

Update:

```bash
curl -fsSL --proto '=https' --tlsv1.2 \
  https://raw.githubusercontent.com/Matsumiko/autoscript-vpn-el9/main/update.sh | bash
```

Installer menarik source-nya dari repo ini (`REPO_OWNER`/`REPO_NAME` di
`install.sh` dan `update.sh`), jadi update tidak akan balik ke upstream dan
menimpa patch.

---

## Fitur (dari upstream, tidak diubah)

- **SSH**: Dropbear (109/143), SSH-WS, SlowDNS/dnstt, OpenVPN, NoobzVPN, UDP relay
- **Xray-core**: VLESS / VMess / Trojan dengan transport WS, gRPC, HTTPUpgrade, XHTTP
- **Front-end**: HAProxy (TLS/ALPN split) + Nginx (`/vless`, `/vmess`, `/trojan`, `/ssh`)
- **Manajemen**: menu interaktif, REST API lokal (`127.0.0.1`), quota/IP-limit,
  auto-unban, Telegram notifier, backup/restore
- **Tuning**: profil `ulimit`/ZRAM/conntrack/BBR (`optimize-rocky`)

Dokumentasi API: [`docs/API.md`](docs/API.md).

---

## Lisensi

[Apache License 2.0](LICENSE).

Copyright penulis asli: **risqinf** — https://github.com/risqinf/autoscript
Perubahan fork: lihat [`PATCHES.md`](PATCHES.md).

Nama "Autoscript" pada repo ini merujuk pada proyek asal; tidak ada klaim
merek, afiliasi, maupun dukungan dari penulis asli.
