# PATCHES — perbedaan dari upstream

Fork ini berangkat dari `risqinf/autoscript` v5.6.3 (Apache-2.0).
Semua perubahan di bawah ini adalah **perbaikan bug**, bukan fitur baru.
Tujuannya satu: supaya SlowDNS benar-benar bisa dipakai client nyata
(HTTP Custom / SlowDNS fork yang menyebar paket ke banyak QTYPE).

---

## 1. Bundled `dnstt-server` di-patch

**Gejala.** SSH connect, `begin session` + `begin stream` muncul di log
server, tapi tidak ada data yang lewat — app bilang "connected" lalu diam.
Di sisi client, tunnel kelihatan hidup tapi halaman tidak terbuka.

**Sebab.** Dua hal di `dnstt-server` upstream:

1. Hanya menerima `QTYPE == TXT`; selain itu dijawab `NXDOMAIN`.
   Client nyata mengirim data sebagai TXT/CNAME/NULL/MX/NS/SRV/A
   (dipakai bergiliran untuk lolos filter resolver), jadi ~6 dari 7
   paket data hilang.
2. Rdata balasan non-TXT tetap di-encode gaya TXT (ada 1 byte panjang di
   depan). Client membaca rdata non-TXT sebagai raw bytes — jadi dapat
   byte sampah `0x00` di awal setiap paket dan KCP-nya rusak.

**Fix.** Dua hunk di `dnstt-server/main.go`:

```diff
-	if question.Type != dns.RRTypeTXT {
-		// We only support QTYPE == TXT.
-		resp.Flags |= dns.RcodeNameError
-		// ... (komentar upstream)
-		return resp, nil
-	}
+	// Accept any QTYPE: real-world clients multiplex packets across query
+	// types (TXT/CNAME/NULL/MX/NS/SRV/A) to evade resolver filtering. The
+	// answer RR echoes the question's Type and Class.
```

```diff
-			rec.Resp.Answer[0].Data = dns.EncodeRDataTXT(payload.Bytes())
+			if rec.Resp.Question[0].Type == dns.RRTypeTXT {
+				// TXT rdata is a sequence of length-prefixed
+				// character-strings.
+				rec.Resp.Answer[0].Data = dns.EncodeRDataTXT(payload.Bytes())
+			} else {
+				// Other qtypes carry the payload as raw rdata (no
+				// length prefixes).
+				rec.Resp.Answer[0].Data = payload.Bytes()
+			}
```

Jangan menyentuh `-mtu`: default upstream `1232` (1280 − 48) adalah batas
aman. Menaikkannya (mis. 4096) membuat respons >1232 byte; di jalur
resolver operator yang MTU-nya <1500 paket ke-fragmentasi dan di-drop →
gejala "konek tapi tanpa data" balik lagi. Sudah diuji: resolver yang
mengiklankan 1472 byte + server `-mtu 4096` = data mati total.

**Verifikasi.**

```bash
# NULL = NOERROR dan rdata kosong tanpa prefix: '\# 0'
dig +noall +comments +answer -p 5300 NULL \
  mtuclg6lqo47x2gzqzcvlattc2xa.<NS> @127.0.0.1
```

Hasil benar: `status: NOERROR` + `IN NULL \# 0`.
Build stock: `status: NXDOMAIN` (atau `NULL \# 1 00`).

`scripts/system/slowdns.sh` menjalankan cek ini otomatis setelah install
(`slowdns_verify_binary`) dan berteriak kalau binary-nya masih stock.

**Catatan `-mtu` (hasil uji):**

| `-mtu` | KCP MTU | Throughput (jalur bersih, 8.8.8.8) |
|---|---|---|
| 1232 (default) | 932 | ~2,1 Mbps |
| 512 | 215 | ~1,0 Mbps |
| 4096 | 932 | data mati total di jalur yang MTU-nya < 1500 |

Kesimpulan: biarkan `1232`. Turun ke 512 hanya masuk akal kalau jalur klien
membuang paket besar (di uji kami justru separuh). Naik ke 4096 bikin respons
ke-fragmentasi dan di-drop — gejala "konek tapi tanpa data".

**Rebuild binary yang di-vendor** (`bin/dnstt-server-linux-{amd64,arm64}.tar.gz`):

```bash
git clone https://www.bamsoftware.com/git/dnstt.git
cd dnstt
# apply dua hunk di atas ke dnstt-server/main.go
for arch in amd64 arm64; do
  CGO_ENABLED=0 GOOS=linux GOARCH=$arch \
    go build -trimpath -ldflags="-s -w" -o /tmp/dnstt-$arch ./dnstt-server
done
# paket: tar.gz berisi `dnstt-server` + `SHA256SUMS`
# lalu perbarui bin/*.sha256 dan bin/checksums.txt
```

Build diambil dari HEAD upstream (≥ v1.20260501.0) supaya ikut fix
`sendLoop`: di versi lama satu error `sendto` mematikan pengiriman
respons padahal proses tetap hidup ("running but not working").

---

## 2. Dropbear tidak lagi memblokir root

`install.sh` dan `scripts/menu/menu-dropbear.sh` memakai
`dropbear -F -w -g ...`:

- `-w` → root dilarang login
- `-g` → password login untuk root dilarang

Akibatnya login lewat SlowDNS (yang di-forward ke dropbear `127.0.0.1:109`)
selalu ditolak `root login rejected`. Sekarang: `dropbear -F -K 60 ...`.

Catatan keamanan: `sshd` di port 22 memang sudah menerima root + password,
jadi ini tidak menambah kelas eksposur baru — hanya port 109/143.

---

## 3. Akun SSH dibuat dengan home + shell nyata

Sebelumnya semua jalur provisioning memakai:

```bash
useradd -e <exp> -M -N -s "$nologin" "$user"
```

- `-M` → tidak ada home directory. Dropbear keluar dengan
  `Exit (user): Error changing directory` sebelum sempat memberi shell.
- `-s nologin` → walau home ada, shell-nya menolak login.

Sekarang: `useradd -e <exp> -m -N -s /bin/bash "$user"`, dan `usermod -s`
untuk akun yang di-unban/recover juga memakai `/bin/bash`.

File yang diubah:

| File | Peran |
|---|---|
| `scripts/lib/account.sh` | provisioning utama (CLI/menu) |
| `scripts/api/recovery-ssh.sh` | recovery via API |
| `scripts/api/trial-ssh.sh` | trial via API |
| `scripts/ssh/recovery-ssh.sh` | recovery via menu |
| `scripts/ssh/trial-ssh.sh` | trial via menu |
| `scripts/system/add-bulk.sh` | bulk create |
| `scripts/system/autounban.sh` | unban otomatis |
| `files/internal/service/account.go` | provisioning API (bulk + single) |
| `files/internal/service/shadow_lock.go` | batch provisioning under flock |

---

## 4. Guard regresi

`slowdns_verify_binary()` di `scripts/system/slowdns.sh` dipanggil setelah
install. Kalau binary yang terpasang masih stock (NXDOMAIN untuk non-TXT
atau rdata ber-prefix), installer memberi warning eksplisit alih-alih
diam-diam memasang tunnel yang tidak jalan.

---

## Rebase ke upstream baru

1. `git remote add upstream https://github.com/risqinf/autoscript.git`
2. `git fetch upstream && git rebase upstream/main`
3. Konflik yang biasanya muncul di: `install.sh` (baris dropbear),
   `scripts/lib/account.sh` + 5 file provisioning, `bin/*`.
4. Setelah rebase, **wajib** jalankan ulang build dnstt dan ganti
   `bin/dnstt-server-linux-*.tar.gz` + `checksums.txt`.
5. Cek ulang dengan `dig NULL ... @127.0.0.1 -p 5300` → harus `\# 0`.
