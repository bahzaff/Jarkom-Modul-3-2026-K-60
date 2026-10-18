# LAPORAN RESMI PRAKTIKUM KOMUNIKASI DATA & JARINGAN KOMPUTER

## MODUL 3: CURTAIN PROTOCOL — SHIBUYA INCIDENT (SOAL 11–20)

* **Kelompok:** K-60
* **Prefix IP:** `192.241.x.x`
* **Domain Resmi:** `jujutsu.k60.com` / `jujutsu.k-60.com`

---
**Cakupan pengerjaan:** Soal 1–10  
**Prefix jaringan:** `192.241.x.x`  
**Server GNS3 kelompok C:** `10.4.89.250`  
**Platform:** GNS3, Debian 13 (`ardhptr21/debinet:latest`) dan Alpine Linux (`ardhptr21/alpinet:latest`)

## A. Pendahuluan

Praktikum Modul 3 bertujuan membangun kembali konektivitas antarsegmen jaringan dan layanan pengalamatan otomatis pada topologi *Shibuya Incident*. Jichi berperan sebagai router utama sekaligus DHCP Relay, sedangkan Tengen berperan sebagai DHCP Server. Infrastruktur dibagi menjadi empat network: Network 1 sebagai DMZ, Network 2 untuk client Divisi A, Network 3 untuk client Divisi B, dan Network 4 sebagai Data Center.

Pada laporan ini dibahas konfigurasi dasar jaringan, NAT, DHCP Server, DHCP Relay, pemisahan DHCP pool, pengaturan lease, reservasi alamat, verifikasi client, pengamatan protokol DHCP, simulasi kegagalan relay, serta pemulihan layanan secara otomatis setelah restart.

## B. Informasi Topologi dan Pengalamatan

| Node/Interface | Alamat IP | Keterangan |
|---|---|---|
| Jichi `eth0` | DHCP dari NAT; teramati `192.168.122.206/24` | Antarmuka WAN; default gateway `192.168.122.1` |
| Jichi `eth1` | `192.241.1.1/24` | Gateway Network 1 (DMZ) |
| Jichi `eth2` | `192.241.2.1/24` | Gateway Network 2 |
| Jichi `eth3` | `192.241.3.1/24` | Gateway Network 3 |
| Jichi `eth4` | `192.241.4.1/24` | Gateway Network 4 |
| Tengen `eth0` | `192.241.4.10/24` | DHCP Server |
| Utahime | `192.241.1.2/24` | DNS Master pada rancangan topologi |
| Shoko | `192.241.1.3/24` | DNS Slave pada rancangan topologi |
| Gojo | `192.241.1.4/24` | Reverse proxy pada rancangan topologi |
| DNS yang diberikan DHCP | `192.241.1.2`, `192.241.1.3` | Alamat resolver sesuai konfigurasi DHCP |
| Domain search DHCP | `K-60.com` | Opsi pencarian domain |

> **Link bukti screenshot/video Google Drive + FILE SCRIPT (soal 1–10):** [Buka folder dokumentasi](MASUKKAN_LINK_DRIVE_DI_SINI)
> https://drive.google.com/drive/folders/1f81LdfTe-upTKz5U0Qx1EvODHD1iUTMo
> Seluruh bukti visual praktikum diletakkan pada folder Google Drive tersebut agar dokumentasi tidak diulang pada setiap pembahasan soal.

---


> **Catatan kelengkapan script:** Blok `.sh` berikut adalah **rekonstruksi berdasarkan rancangan praktikum dan catatan konfigurasi K-60**, bukan ekspor persis berkas `/root` dari container GNS3. Bagian hasil pengujian dibatasi pada output yang sudah tercatat. Script nomor 9 mengubah status relay dan script instalasi dapat mengubah konfigurasi; **jangan menjalankan semuanya sekaligus di proyek yang sedang aktif**.

> **Ketergantungan:** script DHCP soal 2, 4, 5, dan 6 memerlukan `dhcp-lib.sh` pada direktori yang sama; jalankan sesuai urutan jika membangun dari awal. Autostart asli Tengen terakhir direvisi untuk memulihkan backup ketika deklarasi Network 4 hilang. Karena itu, file `soal10-tengen-init.sh` rekonstruksi di bawah diberi catatan khusus dan perlu disesuaikan sebelum dipakai sebagai autostart operasional.

## 1. Konfigurasi Routing, IP Forwarding, NAT, dan IP Statis

### Tujuan
Memulihkan konektivitas empat network melalui router Jichi dan menyediakan akses keluar melalui antarmuka NAT.

### Konfigurasi
Jichi menggunakan `eth0` sebagai jalur keluar menuju NAT dan `eth1`–`eth4` sebagai gateway untuk masing-masing jaringan internal. Alamat IP internal dikonfigurasi secara statis dengan subnet mask `/24`. IPv4 forwarding diaktifkan agar paket dapat diteruskan antarsubnet. Untuk akses keluar, diterapkan source NAT berupa MASQUERADE pada antarmuka `eth0` bagi jaringan sumber `192.241.0.0/16`.

Perintah inti yang digunakan atau relevan dengan konfigurasi akhir:

```bash
ip addr replace 192.241.1.1/24 dev eth1
ip addr replace 192.241.2.1/24 dev eth2
ip addr replace 192.241.3.1/24 dev eth3
ip addr replace 192.241.4.1/24 dev eth4
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -C POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE \
  || iptables -t nat -A POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE
```

### Pengujian dan Hasil
Pemeriksaan dengan `ip -br -4 addr` menunjukkan seluruh IP gateway Jichi sesuai rancangan. Perintah `ip route` menampilkan rute ke Network 1–4 dan rute default melalui `192.168.122.1`. Nilai `net.ipv4.ip_forward` teramati `1`, sedangkan tabel NAT menampilkan:

```text
-A POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE
```

Konektivitas gateway dan akses ke Tengen dari client Network 2 juga terbukti melalui pengujian pada soal berikutnya. Bukti hasil ping internet dari seluruh host infrastruktur tidak tersedia pada hasil terminal yang tercatat untuk penyusunan laporan ini, sehingga tidak dinyatakan sebagai pengujian menyeluruh.

### Analisis dan Kesimpulan
Pengalamatan statis pada antarmuka internal memastikan tiap network mempunyai gateway konsisten. IP forwarding memungkinkan komunikasi antarsubnet, sedangkan MASQUERADE memungkinkan paket keluar menggunakan alamat antarmuka WAN Jichi. Hasil verifikasi menunjukkan komponen routing, forwarding, dan aturan NAT Jichi terkonfigurasi.

### Lampiran script `.sh` per node — Soal 1

#### `soal01-setup-router-jichi.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

for i in eth0 eth1 eth2 eth3 eth4; do ip link set "$i" up; done
for n in 1 2 3 4; do ip addr replace "192.241.$n.1/24" dev "eth$n"; done
sysctl -w net.ipv4.ip_forward=1
mkdir -p /etc/sysctl.d
printf '%s\n' 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-jichi-forward.conf
iptables -t nat -C POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE
ip -br -4 addr; ip route; iptables -t nat -S POSTROUTING
```

#### `soal01-tengen.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.10/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-utahime.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.1.2/24 dev eth0
ip route replace default via 192.241.1.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-shoko.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.1.3/24 dev eth0
ip route replace default via 192.241.1.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-gojo.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.1.4/24 dev eth0
ip route replace default via 192.241.1.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-geto.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.11/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-maki.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.12/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-panda.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.13/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-nanami.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.14/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-todo.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.15/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-kirara.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.16/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-yaga.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.17/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

#### `soal01-inumaki.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ip link set eth0 up
ip addr replace 192.241.4.18/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
ip -br -4 addr show eth0
ip route
```

## 2. Konfigurasi DHCP Server pada Tengen

### Tujuan
Menyediakan DHCP Server terpusat untuk Network 2 dan Network 3 serta mendeklarasikan Network 4 sebagai subnet lokal tanpa pool dinamis.

### Konfigurasi
Pada Tengen (`192.241.4.10/24`), digunakan layanan `isc-dhcp-server` dengan antarmuka layanan `eth0`. Konfigurasi disimpan dalam `/etc/dhcp/dhcpd.conf`, dengan deklarasi subnet `192.241.2.0/24`, `192.241.3.0/24`, dan `192.241.4.0/24`. Network 4 tidak dialokasikan melalui pool dinamis karena merupakan subnet infrastruktur yang menggunakan alamat statis.

Contoh deklarasi subnet lokal:

```conf
subnet 192.241.4.0 netmask 255.255.255.0 {
    option routers 192.241.4.1;
    option subnet-mask 255.255.255.0;
}
```

### Pengujian dan Hasil
Validasi konfigurasi dilakukan dengan:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server status
```

Pada tahap persiapan demo, layanan sempat gagal berjalan meskipun sintaks file valid karena konfigurasi aktif kembali ke bawaan image dan tidak memiliki deklarasi subnet bagi `eth0`. Diagnostik foreground menunjukkan `No subnet declaration for eth0 (192.241.4.10)`. Setelah file konfigurasi yang telah dibackup di `/root/backup-modul3/dhcpd.conf` dipulihkan ke `/etc/dhcp/dhcpd.conf`, layanan kembali berstatus `dhcpd is running`.

### Analisis dan Kesimpulan
Validasi sintaks saja belum menjamin DHCP Server dapat beroperasi. Server harus mengenali subnet dari antarmuka yang digunakan untuk mendengarkan permintaan. Pemulihan konfigurasi yang benar berhasil mengaktifkan kembali DHCP Server Tengen.

### Lampiran script `.sh` per node — Soal 2

#### `dhcp-lib.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

HERE=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
# File snippets tersimpan di /root agar persistensi konfigurasi tidak bergantung pada /etc.
BASE=/root/k60-dhcp
mkdir -p "$BASE" /etc/dhcp /var/lib/dhcp
create_base() {
  cat > "$BASE/00-global.conf" <<'CONF'
authoritative;
default-lease-time 3600;
max-lease-time 7200;
option domain-name-servers 192.241.1.2, 192.241.1.3;
option domain-search "K-60.com";
subnet 192.241.4.0 netmask 255.255.255.0 {
  option routers 192.241.4.1;
  option broadcast-address 192.241.4.255;
}
CONF
  [ -e "$BASE/10-net2.conf" ] || cat > "$BASE/10-net2.conf" <<'CONF'
subnet 192.241.2.0 netmask 255.255.255.0 {
  option routers 192.241.2.1;
  option broadcast-address 192.241.2.255;
}
CONF
  [ -e "$BASE/20-net3.conf" ] || cat > "$BASE/20-net3.conf" <<'CONF'
subnet 192.241.3.0 netmask 255.255.255.0 {
  option routers 192.241.3.1;
  option broadcast-address 192.241.3.255;
}
CONF
}
apply_dhcp() {
  cat "$BASE/00-global.conf" "$BASE/10-net2.conf" "$BASE/20-net3.conf" > /etc/dhcp/dhcpd.conf
  [ ! -f "$BASE/30-reservation.conf" ] || cat "$BASE/30-reservation.conf" >> /etc/dhcp/dhcpd.conf
  printf 'INTERFACESv4="eth0"\nINTERFACESv6=""\n' > /etc/default/isc-dhcp-server
  touch /var/lib/dhcp/dhcpd.leases
  dhcpd -t -cf /etc/dhcp/dhcpd.conf
  service isc-dhcp-server restart
  service isc-dhcp-server status
}
```

#### `soal02-dhcp-server-tengen.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
. "$DIR/dhcp-lib.sh"
ip link set eth0 up
ip addr replace 192.241.4.10/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
if [ ! -x /usr/sbin/dhcpd ]; then
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-server
fi
create_base
apply_dhcp
```

## 3. Konfigurasi DHCP Relay pada Jichi

### Tujuan
Meneruskan permintaan DHCP dari Network 2 dan Network 3 kepada DHCP Server Tengen di Network 4.

### Konfigurasi
Jichi menjalankan `isc-dhcp-relay` dengan alamat tujuan `192.241.4.10`. Dalam konfigurasi yang teruji, relay mendengarkan antarmuka `eth2`, `eth3`, dan `eth4`, dengan dua antarmuka pertama melayani subnet client.

Konfigurasi `/etc/default/isc-dhcp-relay`:

```bash
SERVERS="192.241.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
```

### Pengujian dan Hasil
Proses aktif diverifikasi menggunakan:

```bash
ps -eo pid,stat,args | grep '[d]hcrelay'
```

Hasil pemeriksaan menampilkan proses seperti:

```text
/usr/sbin/dhcrelay -q -i eth2 -i eth3 -i eth4 192.241.4.10
```

Client Itadori pada Network 2 berhasil memperoleh `192.241.2.10/24` dari server `192.241.4.10` dengan waktu sewa 3600 detik. Hal ini membuktikan jalur relay Network 2 berfungsi. Output hasil uji client Network 3 tidak dilampirkan sebagai data numerik karena belum tersedia dalam catatan terminal yang digunakan menyusun laporan.

### Analisis dan Kesimpulan
DHCP Discover mula-mula dikirim secara broadcast dalam subnet client. Karena router tidak meneruskan broadcast lokal tersebut secara default, Jichi bertindak sebagai DHCP Relay. Informasi `giaddr` membantu server menentukan subnet asal permintaan dan memilih scope DHCP yang sesuai.

### Lampiran script `.sh` per node — Soal 3

#### `soal03-dhcp-relay-jichi.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# Soal 3: eth2 + eth3 adalah subnet klien yang diminta soal.
if [ ! -x /usr/sbin/dhcrelay ]; then
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-relay
fi
cat > /etc/default/isc-dhcp-relay <<'CONF'
SERVERS="192.241.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
CONF
service isc-dhcp-relay restart
ps -eo pid,stat,args | grep '[d]hcrelay'
```

#### `soal07-itadori-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

#### `soal07-ino-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

## 4. Pembagian DHCP Pool pada Network 2

### Tujuan
Memisahkan alokasi IP client Divisi A menjadi dua pool berdasarkan identitas client.

### Konfigurasi
Network 2 menggunakan dua kelompok DHCP berdasarkan MAC Address atau Client ID melalui DHCP class.

| Pool | Client | Rentang IP |
|---|---|---|
| Pool 1 | Itadori, Fushiguro | `192.241.2.10`–`192.241.2.35` |
| Pool 2 | Zenin, Okkotsu | `192.241.2.70`–`192.241.2.95` |

Rentang `192.241.2.36`–`192.241.2.69` tidak termasuk pool yang boleh dibagikan.

### Pengujian dan Hasil
Pada pengujian yang tercatat, Itadori berhasil memperoleh `192.241.2.10/24`, yaitu alamat dalam rentang Pool 1. Konfigurasi kelas dan pool tersimpan dalam backup akhir DHCP Tengen. Belum tersedia output perolehan lease Zenin, Okkotsu, dan Fushiguro dalam catatan terakhir untuk memastikan seluruh anggota pool satu per satu.

### Analisis dan Kesimpulan
DHCP class memungkinkan client dalam subnet yang sama memperoleh alamat dari rentang berbeda berdasarkan identitasnya. Pengujian Itadori mendukung bahwa Pool 1 berfungsi, sedangkan pengujian lengkap seluruh anggota kedua pool perlu dicocokkan dengan dokumentasi demonstrasi.

### Lampiran script `.sh` per node — Soal 4

#### `soal04-pool-network2.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
. "$DIR/dhcp-lib.sh"
create_base
cat > "$BASE/10-net2.conf" <<'CONF'
class "n2_pool1" {
  match if substring(hardware, 1, 6) = 02:42:05:8c:a7:00 or
           substring(hardware, 1, 6) = 02:42:c6:f1:1a:00;
}
class "n2_pool2" {
  match if substring(hardware, 1, 6) = 02:42:a5:3c:c9:00 or
           substring(hardware, 1, 6) = 02:42:a9:52:86:00;
}
subnet 192.241.2.0 netmask 255.255.255.0 {
  option routers 192.241.2.1;
  option broadcast-address 192.241.2.255;
  pool { allow members of "n2_pool1"; range 192.241.2.10 192.241.2.35; }
  pool { allow members of "n2_pool2"; range 192.241.2.70 192.241.2.95; }
}
CONF
apply_dhcp
```

#### `soal07-fushigoro-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

#### `soal07-zenin-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

#### `soal07-okkotsu-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

## 5. Konfigurasi Short Lease dan Long Lease pada Network 3

### Tujuan
Menerapkan durasi sewa alamat yang berbeda pada client Network 3 sesuai kebutuhan operasional.

### Konfigurasi

| Jenis alokasi | Client | Rentang IP | Lease time |
|---|---|---|---|
| Short lease | Ino, Kusakabe | `192.241.3.10`–`192.241.3.89` | 1800 detik (30 menit) |
| Long lease | Miwa | `192.241.3.100`–`192.241.3.199` | 43200 detik (12 jam) |

Konfigurasi Network 3 juga menyediakan gateway `192.241.3.1`, DNS `192.241.1.2` dan `192.241.1.3`, serta domain-search `K-60.com`.

### Pengujian dan Hasil
Aturan short lease dan long lease telah menjadi bagian dari konfigurasi final DHCP yang disimpan sebagai backup pada Tengen. Bukti numerik lease hasil renewal Ino, Kusakabe, dan Miwa tidak terdapat dalam output terbaru yang tersedia pada penyusunan laporan; karena itu durasi tersebut dicatat sebagai kebijakan konfigurasi dan bukan hasil verifikasi ulang semua client.

### Analisis dan Kesimpulan
Short lease memungkinkan server lebih cepat mengelola ulang alokasi pada perangkat yang sering berganti koneksi, sedangkan long lease mengurangi frekuensi pembaruan untuk perangkat yang lebih stabil. Pengujian lease masing-masing client perlu dipastikan sesuai konfigurasi ketika demonstrasi.

### Lampiran script `.sh` per node — Soal 5

#### `soal05-lease-network3.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
. "$DIR/dhcp-lib.sh"
create_base
cat > "$BASE/20-net3.conf" <<'CONF'
class "n3_short" {
  match if substring(hardware, 1, 6) = 02:42:f7:02:50:00 or
           substring(hardware, 1, 6) = 02:42:05:13:af:00;
}
class "n3_long" {
  match if substring(hardware, 1, 6) = 02:42:48:95:26:00;
}
subnet 192.241.3.0 netmask 255.255.255.0 {
  option routers 192.241.3.1;
  option broadcast-address 192.241.3.255;
  pool {
    allow members of "n3_short";
    range 192.241.3.10 192.241.3.89;
    default-lease-time 1800;
    max-lease-time 1800;
  }
  pool {
    allow members of "n3_long";
    range 192.241.3.100 192.241.3.199;
    default-lease-time 43200;
    max-lease-time 43200;
  }
}
CONF
apply_dhcp
```

#### `soal07-rusakabe-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

#### `soal07-miwa-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

## 6. DHCP Fixed Address untuk Kugisaki

### Tujuan
Memberikan alamat IP yang konsisten kepada client Kugisaki tanpa mengatur IP secara manual di sisi client.

### Konfigurasi
Kugisaki dipetakan ke IP `192.241.3.99` melalui reservasi berdasarkan MAC Address.

```conf
host kugisaki {
    hardware ethernet 02:42:47:41:db:00;
    fixed-address 192.241.3.99;
}
```

Alamat tersebut tidak masuk ke dalam kedua pool dinamis Network 3.

### Pengujian dan Hasil
Deklarasi reservasi Kugisaki telah terlihat pada konfigurasi backup DHCP Tengen. Tidak terdapat output DHCP renewal Kugisaki yang dapat dikutip dari log terakhir untuk memastikan hasil aktual `192.241.3.99` setelah persiapan demo; pemeriksaan tersebut perlu dicocokkan dengan dokumentasi atau direkam saat demonstrasi.

### Analisis dan Kesimpulan
DHCP reservation berbeda dari IP statis manual: pada reservation, client tetap menggunakan DHCP, tetapi server menentukan alamat tetap berdasarkan identitas client. Cara ini memungkinkan alamat tetap konsisten sekaligus mempertahankan pengelolaan konfigurasi jaringan secara terpusat.

### Lampiran script `.sh` per node — Soal 6

#### `soal06-reservasi-kugisaki.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
. "$DIR/dhcp-lib.sh"
create_base
cat > "$BASE/30-reservation.conf" <<'CONF'
host kugisaki {
  hardware ethernet 02:42:47:41:db:00;
  fixed-address 192.241.3.99;
}
CONF
apply_dhcp
```

#### `soal07-kugisaki-dhcp.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# DHCP pada Alpine; pastikan paket iproute2 ada jika ingin memakai ip -br
udhcpc -i eth0 -n -q -O 119
ip -4 addr show dev eth0
ip route
cat /etc/resolv.conf
```

## 7. Verifikasi IP, Gateway, DNS, dan Lease Client

### Tujuan
Memastikan konfigurasi yang diterima client sesuai subnet, pool, gateway, dan opsi DHCP yang ditentukan.

### Metode Pengujian
Pemeriksaan menggunakan perintah berikut pada client, serta pencocokan file lease pada Tengen:

```sh
ip -br -4 addr
ip route
cat /etc/resolv.conf
ping -c 3 192.241.2.1
ping -c 3 192.241.4.10
```

### Hasil Pengujian
Pengujian pada Itadori setelah pemulihan DHCP Tengen menghasilkan:

| Parameter | Hasil teramati |
|---|---|
| IP address | `192.241.2.10/24` |
| DHCP Server | `192.241.4.10` |
| Lease time | 3600 detik |
| Gateway | `192.241.2.1` |
| DNS | `192.241.1.2`, `192.241.1.3` |
| Domain search | `K-60.com` |
| Ping gateway | 3 dari 3 paket diterima |
| Ping DHCP Server | 3 dari 3 paket diterima |

### Analisis dan Kesimpulan
Hasil Itadori menunjukkan konfigurasi IP, gateway, DNS, dan domain-search berhasil diterima melalui DHCP serta konektivitas ke router dan server DHCP berjalan. Pemeriksaan serupa diperlukan untuk mendokumentasikan seluruh client Divisi A dan B, termasuk memastikan tidak ada IP di luar kebijakan pool.

### Lampiran script `.sh` per node — Soal 7

#### `soal07-verifikasi-lease-tengen.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

[ -f /var/lib/dhcp/dhcpd.leases ] || { echo 'File leases tidak ditemukan'; exit 1; }
cat /var/lib/dhcp/dhcpd.leases
```

## 8. Pengamatan DHCP DORA menggunakan Packet Capture

### Tujuan
Mengamati proses pertukaran pesan DHCP Discover, Offer, Request, dan Acknowledge, serta mengidentifikasi alamat DHCP Server dan relay.

### Metode Pengujian
Capture dapat dilakukan pada interface Tengen yang menerima paket relay menggunakan:

```bash
tcpdump -ni eth0 -vvv -s 0 'udp port 67 or udp port 68'
```

Pada client, permintaan alamat baru dijalankan menggunakan `udhcpc -i eth0 -n -q -O 119`. Tahapan protokol yang harus diamati mencakup:

1. **DHCPDISCOVER:** client mencari server.
2. **DHCPOFFER:** server menawarkan alamat IP.
3. **DHCPREQUEST:** client meminta alamat yang ditawarkan.
4. **DHCPACK:** server menyetujui lease.

Karena client berada pada subnet berbeda dengan server, field `giaddr` untuk permintaan dari Network 2 diharapkan menunjukkan `192.241.2.1`, yaitu alamat interface relay Jichi pada subnet asal.

### Pengujian dan Hasil
Perangkat `tcpdump` telah diverifikasi tersedia pada Tengen. Namun, output paket DORA lengkap dari `tcpdump` belum terdapat dalam log terminal yang tersedia pada penyusunan laporan. Oleh sebab itu bagian ini menjelaskan metode dan parameter analisis, bukan mengklaim bahwa keempat paket telah teramati dalam satu capture.

### Analisis dan Kesimpulan
Packet capture merupakan bukti alur protokol di tingkat paket dan dapat menunjukkan kontribusi relay melalui `giaddr`. Untuk menyimpulkan DORA lengkap berhasil ditangkap, seluruh empat tipe pesan perlu benar-benar terlihat pada hasil capture.

### Lampiran script `.sh` per node — Soal 8

#### `soal08-tcpdump-tengen.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

if ! command -v tcpdump >/dev/null 2>&1; then
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y tcpdump
fi
echo 'Jalankan ulang DHCP di client TERPISAH selama capture berlangsung.'
# Stop capture dengan Ctrl+C; simpan pcap untuk dianalisis.
exec tcpdump -i eth0 -nn -vvv -s 0 -w /root/k60-dhcp-dora.pcap 'udp port 67 or udp port 68'
```

## 9. Simulasi Gangguan DHCP Relay dan Pemulihannya

### Tujuan
Menganalisis dampak penghentian DHCP Relay pada permintaan lease baru serta memastikan layanan pulih ketika relay kembali dinyalakan.

### Metode Pengujian
Simulasi dilakukan dengan menghentikan relay pada Jichi, meminta lease baru dari client Network 2 atau 3, kemudian menyalakan relay kembali dan mengulangi permintaan.

```bash
# Jichi: simulasi gangguan
service isc-dhcp-relay stop

# Jichi: pulihkan layanan
service isc-dhcp-relay start
```

### Pengujian dan Hasil
Dalam pengujian kondisi normal yang telah dicatat, Jichi memiliki proses `dhcrelay` aktif dan Itadori memperoleh alamat dari Tengen. Output kegagalan permintaan baru ketika relay dimatikan beserta output pemulihannya tidak disertakan pada catatan terakhir. Karena itu, langkah di atas dicatat sebagai prosedur uji dan hasil detailnya perlu dicocokkan dengan rekaman demo.

### Analisis dan Kesimpulan
Ketika relay tidak aktif, broadcast DHCP client dari subnet berbeda tidak dapat mencapai server Tengen melalui mekanisme relay. Client yang masih memiliki lease valid tidak otomatis kehilangan IP, sehingga evaluasi kegagalan harus menggunakan permintaan DHCP baru, bukan hanya melihat IP yang sudah terpasang. Setelah relay dipulihkan, client seharusnya kembali dapat mengakses server DHCP.

### Lampiran script `.sh` per node — Soal 9

#### `soal09a-relay-off-jichi.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

service isc-dhcp-relay stop
ps -eo pid,stat,args | grep '[d]hcrelay' || true
echo 'Sekarang uji DHCP client tanpa relay, lalu jalankan soal09b.'
```

#### `soal09b-relay-on-jichi.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

service isc-dhcp-relay start
ps -eo pid,stat,args | grep '[d]hcrelay'
echo 'Sekarang renew DHCP client dan buktikan IP/gateway.'
```

## 10. Autostart dan Pemulihan Setelah Restart

### Tujuan
Memastikan konfigurasi jaringan, DHCP Server Tengen, dan DHCP Relay Jichi dapat dipulihkan secara otomatis setelah node di-*restart*.

### Konfigurasi
File `/root/init.sh` dipakai pada Jichi dan Tengen karena mekanisme startup container memanggil file tersebut saat node dimulai. Pada Jichi, script menjalankan setup alamat IP, forwarding, NAT, dan DHCP Relay. Pada Tengen, script memulihkan alamat IP, memastikan paket DHCP Server tersedia, memvalidasi konfigurasi, dan menjalankan service.

Log yang dihasilkan masing-masing script disimpan pada:

```text
/root/autostart-jichi.log
/root/autostart-tengen.log
```

### Pengujian dan Hasil
Pada pengujian restart yang terdokumentasi sebelumnya:

| Node | Container dimulai (UTC) | Autostart tercatat (UTC) | Hasil |
|---|---|---|---|
| Tengen | 14 Oktober 2026, 16:54:00 | 14 Oktober 2026, 16:54:01 | DHCP Server berjalan |
| Jichi | 14 Oktober 2026, 17:36:18 | 14 Oktober 2026, 17:36:20 | IP, forwarding, NAT, dan relay pulih |

Setelah kedua node tersebut menyala kembali, Itadori berhasil memperoleh `192.241.2.10/24` dari Tengen dengan lease 3600 detik, gateway `192.241.2.1`, DNS `192.241.1.2` dan `192.241.1.3`, serta ping 3/3 ke gateway dan Tengen.

Pada persiapan rekaman berikutnya, ditemukan bahwa file DHCP aktif sempat kembali ke konfigurasi bawaan image meskipun `/etc/dhcp/dhcpd.conf` tersedia. Script autostart Tengen lalu direvisi agar memulihkan backup ketika deklarasi subnet lokal Network 4 tidak ditemukan. **Revisi terakhir tersebut sudah lolos pemeriksaan sintaks, tetapi belum memiliki hasil restart ulang yang terdokumentasi dalam catatan ini.**

### Analisis dan Kesimpulan
Autostart terbukti berhasil pada pengujian sebelumnya melalui kesesuaian waktu startup container dan eksekusi script serta hasil DHCP client sesudah restart. Temuan konfigurasi bawaan pada persiapan rekaman menunjukkan pentingnya memastikan isi konfigurasi tetap sesuai kebijakan praktikum, bukan hanya memeriksa keberadaan file dan status service. Revisi terbaru masih membutuhkan pengujian restart agar dapat dinyatakan terverifikasi.

---

## C. Kesimpulan Umum

Praktikum nomor 1–10 berfokus pada penyediaan konektivitas jaringan dan manajemen IP otomatis. Jichi berfungsi sebagai router utama yang melakukan routing, IPv4 forwarding, NAT, dan DHCP Relay; sementara Tengen menyediakan DHCP Server untuk client Network 2 dan Network 3.

Dari pengujian yang tercatat, konfigurasi antarmuka Jichi, NAT, forwarding, proses relay, status DHCP Server setelah pemulihan, serta pemberian lease DHCP pada Itadori berhasil diverifikasi. Itadori memperoleh IP `192.241.2.10/24`, gateway `192.241.2.1`, DNS yang sesuai, dan dapat menjangkau Jichi maupun Tengen. Mekanisme autostart juga pernah berhasil pada kedua node dalam pengujian restart sebelumnya.

Verifikasi lengkap untuk seluruh client, capture DORA, simulasi gangguan relay, dan pengujian restart untuk revisi autostart terakhir harus diselaraskan dengan rekaman demonstrasi sebelum laporan dikumpulkan. Dengan begitu, kesimpulan pada laporan tetap sesuai bukti pengujian yang benar-benar tersedia.

## D. Referensi Konfigurasi dan Berkas Pendukung

- Soal Praktikum Modul 3 Komunikasi Data dan Jaringan Komputer 2026 — *Curtain Protocol: Shibuya Incident*.
- Script node pada direktori `/root` di masing-masing container GNS3.
- Backup konfigurasi router dan DHCP Server: `K60-JICHI-FINAL.tar.gz` dan `K60-TENGEN-FINAL.tar.gz`.
- Bukti screenshot dan rekaman demonstrasi pada tautan Google Drive di awal pembahasan.

### Lampiran script `.sh` per node — Soal 10

#### `soal10-jichi-init.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# Disalin ke /root/init.sh agar dipanggil /etc/debinet-init.sh saat container start.
exec >> /root/autostart-jichi.log 2>&1
echo "AUTOSTART JICHI: $(date -u)"
for i in eth0 eth1 eth2 eth3 eth4; do ip link set "$i" up; done
for n in 1 2 3 4; do ip addr replace "192.241.$n.1/24" dev "eth$n"; done
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -C POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -s 192.241.0.0/16 -o eth0 -j MASQUERADE
if [ ! -x /usr/sbin/dhcrelay ]; then
  # Tunggu konektivitas WAN tersedia.
  i=0; until ping -c 1 -W 2 1.1.1.1 >/dev/null 2>&1; do
    i=$((i+1)); [ "$i" -lt 15 ] || exit 1; sleep 3
  done
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-relay
fi
cat > /etc/default/isc-dhcp-relay <<'CONF'
SERVERS="192.241.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
CONF
service isc-dhcp-relay restart
ip -br -4 addr
ps -eo pid,stat,args | grep '[d]hcrelay'
echo 'AUTOSTART JICHI SELESAI'
```

#### `soal10-tengen-init.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

# Disalin ke /root/init.sh agar dipanggil /etc/debinet-init.sh saat container start.
exec >> /root/autostart-tengen.log 2>&1
echo "AUTOSTART TENGEN: $(date -u)"
ip link set eth0 up
ip addr replace 192.241.4.10/24 dev eth0
ip route replace default via 192.241.4.1 dev eth0
printf '%s\n' 'nameserver 192.168.122.1' > /etc/resolv.conf
if [ ! -x /usr/sbin/dhcpd ]; then
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y isc-dhcp-server
fi
mkdir -p /etc/dhcp /var/lib/dhcp
touch /var/lib/dhcp/dhcpd.leases
# File final TIDAK diberikan pada paket ini; ekspor dari backup Tengen asli jika dibutuhkan.
if [ ! -f /etc/dhcp/dhcpd.conf ]; then
  if [ -f /root/backup-modul3/dhcpd.conf ]; then
    cp /root/backup-modul3/dhcpd.conf /etc/dhcp/dhcpd.conf
  else
    echo 'ERROR: backup DHCP final tidak ditemukan di /root/backup-modul3/dhcpd.conf'
    exit 1
  fi
fi
printf 'INTERFACESv4="eth0"\nINTERFACESv6=""\n' > /etc/default/isc-dhcp-server
dhcpd -t -cf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
service isc-dhcp-server status
echo 'AUTOSTART TENGEN SELESAI'
```

**Koreksi penting dari sesi demo:** Setelah restart, image Debian dapat menyediakan `/etc/dhcp/dhcpd.conf` bawaan sehingga tes `-f` di rekonstruksi ini tidak cukup. File autostart Tengen yang benar-benar dipakai pada sesi terakhir menggunakan pengecekan deklarasi `subnet 192.241.4.0`, dan memulihkan `/root/backup-modul3/dhcpd.conf` jika tidak ditemukan. Jika ingin menyalin untuk menjalankan ulang, gunakan versi final asli di `/root/init.sh` Tengen, bukan contoh rekonstruksi ini.

#### `soal10-verifikasi-jichi.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ps -p 1 -o pid,lstart,args
cat /root/autostart-jichi.log | tail -25
ip -br -4 addr
sysctl net.ipv4.ip_forward
iptables -t nat -S POSTROUTING
ps -eo pid,stat,args | grep '[d]hcrelay' || true
```

#### `soal10-verifikasi-tengen.sh`

```bash
#!/bin/sh
# K-60 | Jarkom Modul 3 2026 | REKONSTRUKSI dari pekerjaan dan soal praktikum
set -eu

ps -p 1 -o pid,lstart,args
tail -25 /root/autostart-tengen.log
ip -br -4 addr
ip route
service isc-dhcp-server status
```


## Penutup

Konfigurasi Jichi dan Tengen serta uji DHCP Itadori sudah diperiksa dari output terminal. Pengujian lain yang tidak memiliki keluaran lengkap pada percakapan perlu disandingkan dengan rekaman dan bukti Google Drive sebelum laporan dianggap telah terverifikasi penuh. Seluruh shell script rekonstruksi disertakan untuk dokumentasi dan pengumpulan, sementara versi asli dalam `/root` node merupakan sumber utama apabila asisten meminta script yang benar-benar dijalankan.

## Daftar Alamat IP Infrastruktur (Soal 11–20)

| Nama Node | Jaringan | Alamat IP | Peran Layanan | Image |
| ----- | ----- | ----- | ----- | ----- |
| **Utahime** | Network 1 (DMZ) | `192.241.1.2/24` | DNS Master (BIND9) | `debinet` |
| **Shoko** | Network 1 (DMZ) | `192.241.1.3/24` | DNS Slave (BIND9) | `debinet` |
| **Gojo** | Network 1 (DMZ) | `192.241.1.4/24` | Public Reverse Proxy (Nginx) | `debinet` |
| **Geto** | Network 4 (Data Center) | `192.241.4.11/24` | Internal Load Balancer (Nginx) | `debinet` |
| **Maki** | Network 4 (Data Center) | `192.241.4.12/24` | Worker Frontend 1 (Nginx + PHP-FPM) | `debinet` |
| **Panda** | Network 4 (Data Center) | `192.241.4.13/24` | Worker Frontend 2 (Nginx + PHP-FPM) | `debinet` |
| **Nanami** | Network 4 (Data Center) | `192.241.4.14/24` | Worker API Utama 1 (Weight 3) | `debinet` |
| **Todo** | Network 4 (Data Center) | `192.241.4.15/24` | Worker API Utama 2 (Weight 1) | `debinet` |
| **Kirara** | Network 4 (Data Center) | `192.241.4.16/24` | Worker API Cadangan (`backup`) | `debinet` |
| **Yaga (Yoga)** | Network 4 (Data Center) | `192.241.4.17/24` | Database Master (MariaDB) | `debinet` |
| **Inumaki** | Network 4 (Data Center) | `192.241.4.18/24` | Database Slave (MariaDB) | `debinet` |
| **Itadori** | Network 2 (Divisi A) | `192.241.2.10/24` | Client DHCP Pool 1 | `alpinet` |
| **Kugisaki** | Network 3 (Divisi B) | `192.241.3.99/24` | Client Fixed Address | `alpinet` |
| **Rusakabe (Kusakabe)** | Network 3 (Divisi B) | `192.241.3.10/24` | Client Short Lease (1800s) & Benchmark | `alpinet` |

---

## Soal 11: Konfigurasi DNS Master (Utahime) & DNS Slave (Shoko)

### Deskripsi
Membangun layanan penamaan domain `jujutsu.k60.com` menggunakan BIND9 dengan **Utahime** (`192.241.1.2`) sebagai DNS Master dan **Shoko** (`192.241.1.3`) sebagai DNS Slave. Mengatur SOA record, NS delegasi untuk kedua server, serta menerapkan `allow-transfer` dari Master ke Slave dan menguji sinkronisasi otomatis saat nomor serial dinaikkan.

### 1. Script `/root` pada Server (Utahime & Shoko)

#### A. Node **Utahime** (`/root/soal11.sh` & `/root/soal11_naik_serial.sh`)
```bash
cat << 'EOF' > /root/soal11.sh
#!/bin/bash
ip addr add 192.241.1.2/24 dev eth0 2>/dev/null || true
ip link set eth0 up
ip route replace default via 192.241.1.1

if ! command -v named >/dev/null 2>&1; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y bind9 bind9utils dnsutils
fi

mkdir -p /etc/bind/jarkom

cat << 'EOT' > /etc/bind/named.conf.local
zone "jujutsu.k60.com" {
    type master;
    file "/etc/bind/jarkom/jujutsu.k60.com";
    allow-transfer { 192.241.1.3; };
    also-notify { 192.241.1.3; };
};

zone "jujutsu.k-60.com" {
    type master;
    file "/etc/bind/jarkom/jujutsu.k-60.com";
    allow-transfer { 192.241.1.3; };
    also-notify { 192.241.1.3; };
};
EOT

cat << 'EOT' > /etc/bind/jarkom/jujutsu.k60.com
$TTL    604800
@       IN      SOA     jujutsu.k60.com. root.jujutsu.k60.com. (
                        2026101501      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
@       IN      NS      ns1.jujutsu.k60.com.
@       IN      NS      ns2.jujutsu.k60.com.
ns1     IN      A       192.241.1.2
ns2     IN      A       192.241.1.3
@       IN      A       192.241.1.4
EOT

cat << 'EOT' > /etc/bind/jarkom/jujutsu.k-60.com
$TTL    604800
@       IN      SOA     jujutsu.k-60.com. root.jujutsu.k-60.com. (
                        2026101501      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
@       IN      NS      ns1.jujutsu.k-60.com.
@       IN      NS      ns2.jujutsu.k-60.com.
ns1     IN      A       192.241.1.2
ns2     IN      A       192.241.1.3
@       IN      A       192.241.1.4
EOT

named-checkconf
service named restart || service bind9 restart
EOF

chmod +x /root/soal11.sh
/root/soal11.sh
```

Script menaikkan nomor serial (`/root/soal11_naik_serial.sh`):
```bash
cat << 'EOF' > /root/soal11_naik_serial.sh
#!/bin/bash
sed -i 's/2026101501/2026101502/g' /etc/bind/jarkom/jujutsu.k60.com
sed -i 's/2026101501/2026101502/g' /etc/bind/jarkom/jujutsu.k-60.com
rndc reload || service named reload || service bind9 reload
EOF

chmod +x /root/soal11_naik_serial.sh
/root/soal11_naik_serial.sh
```

#### B. Node **Shoko** (`/root/soal11.sh`)
```bash
cat << 'EOF' > /root/soal11.sh
#!/bin/bash
ip addr add 192.241.1.3/24 dev eth0 2>/dev/null || true
ip link set eth0 up
ip route replace default via 192.241.1.1

if ! command -v named >/dev/null 2>&1; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y bind9 bind9utils dnsutils
fi

cat << 'EOT' > /etc/bind/named.conf.local
zone "jujutsu.k60.com" {
    type slave;
    masters { 192.241.1.2; };
    file "/var/lib/bind/jujutsu.k60.com";
};

zone "jujutsu.k-60.com" {
    type slave;
    masters { 192.241.1.2; };
    file "/var/lib/bind/jujutsu.k-60.com";
};
EOT

named-checkconf
service named restart || service bind9 restart
EOF

chmod +x /root/soal11.sh
/root/soal11.sh
```

### 2. Konfigurasi File Non-Root (`/etc/bind/*`)
File konfigurasi sistem BIND9 diletakkan di luar `/root`, yaitu pada:
* `/etc/bind/named.conf.local` (di Utahime dan Shoko)
* `/etc/bind/jarkom/jujutsu.k60.com` (di Utahime)
* `/var/lib/bind/jujutsu.k60.com` (direktori sinkronisasi otomatis milik user `bind` di Shoko)

### 3. Cara Pengujian & Hasil Verifikasi

1. **Di Shoko (Cek File Zona Hasil Transfer):**
   ```bash
   ls -l /var/lib/bind/
   ```
   **Hasil:**
   ```text
   total 8
   -rw-r--r-- 1 bind bind 351 Oct 15 04:57 jujutsu.k-60.com
   -rw-r--r-- 1 bind bind 342 Oct 15 04:57 jujutsu.k60.com
   ```
   ![Output Zone Transfer di Shoko](images/soal11_shoko_zonetransfer.png)

2. **Di Client Itadori (Query SOA ke Master & Slave):**
   ```sh
   dig @192.241.1.2 jujutsu.k60.com SOA +short
   dig @192.241.1.3 jujutsu.k60.com SOA +short
   ```
   **Hasil:**
   ```text
   jujutsu.k60.com. root.jujutsu.k60.com. 2026101502 604800 86400 2419200 604800
   jujutsu.k60.com. root.jujutsu.k60.com. 2026101502 604800 86400 2419200 604800
   ```
   *(Nomor serial pada Utahime dan Shoko terbukti sama-sama naik ke `2026101502`).*
   ![Output Dig SOA Master dan Slave di Itadori](images/soal11_itadori_dig_soa.png)

---

## Soal 12: A Record, CNAME (`www`), Reverse DNS (`PTR`), & Forwarder

### Deskripsi
Memetakan A record `jujutsu.k60.com` ke **Gojo** (`192.241.1.4`), menambahkan CNAME `www` yang merujuk ke domain utama, membuat zona Reverse DNS (`1.241.192.in-addr.arpa`) untuk melacak IP Gojo kembali ke domain resminya, serta mengaktifkan DNS Forwarder ke `192.168.122.1`.

### 1. Script `/root` pada Server (Utahime & Shoko)

#### A. Node **Utahime** (`/root/soal12.sh`)
```bash
cat << 'EOF' > /root/soal12.sh
#!/bin/bash
cat << 'EOT' > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    forwarders {
        192.168.122.1;
    };
    dnssec-validation no;
    allow-query { any; };
    auth-nxdomain no;
    listen-on-v6 { any; };
};
EOT

cat << 'EOT' > /etc/bind/named.conf.local
zone "jujutsu.k60.com" {
    type master;
    file "/etc/bind/jarkom/jujutsu.k60.com";
    allow-transfer { 192.241.1.3; };
    also-notify { 192.241.1.3; };
};

zone "jujutsu.k-60.com" {
    type master;
    file "/etc/bind/jarkom/jujutsu.k-60.com";
    allow-transfer { 192.241.1.3; };
    also-notify { 192.241.1.3; };
};

zone "1.241.192.in-addr.arpa" {
    type master;
    file "/etc/bind/jarkom/1.241.192.in-addr.arpa";
    allow-transfer { 192.241.1.3; };
    also-notify { 192.241.1.3; };
};
EOT

cat << 'EOT' > /etc/bind/jarkom/jujutsu.k60.com
$TTL    604800
@       IN      SOA     jujutsu.k60.com. root.jujutsu.k60.com. (
                        2026101503      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
@       IN      NS      ns1.jujutsu.k60.com.
@       IN      NS      ns2.jujutsu.k60.com.
ns1     IN      A       192.241.1.2
ns2     IN      A       192.241.1.3
@       IN      A       192.241.1.4
www     IN      CNAME   jujutsu.k60.com.
EOT

cat << 'EOT' > /etc/bind/jarkom/1.241.192.in-addr.arpa
$TTL    604800
@       IN      SOA     jujutsu.k60.com. root.jujutsu.k60.com. (
                        2026101503      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
@       IN      NS      ns1.jujutsu.k60.com.
@       IN      NS      ns2.jujutsu.k60.com.
4       IN      PTR     jujutsu.k60.com.
EOT

named-checkconf
named-checkzone jujutsu.k60.com /etc/bind/jarkom/jujutsu.k60.com
named-checkzone 1.241.192.in-addr.arpa /etc/bind/jarkom/1.241.192.in-addr.arpa
service named restart || service bind9 restart
EOF

chmod +x /root/soal12.sh
/root/soal12.sh
```

#### B. Node **Shoko** (`/root/soal12.sh`)
```bash
cat << 'EOF' > /root/soal12.sh
#!/bin/bash
cat << 'EOT' > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    forwarders {
        192.168.122.1;
    };
    dnssec-validation no;
    allow-query { any; };
    auth-nxdomain no;
    listen-on-v6 { any; };
};
EOT

cat << 'EOT' > /etc/bind/named.conf.local
zone "jujutsu.k60.com" {
    type slave;
    masters { 192.241.1.2; };
    file "/var/lib/bind/jujutsu.k60.com";
};

zone "jujutsu.k-60.com" {
    type slave;
    masters { 192.241.1.2; };
    file "/var/lib/bind/jujutsu.k-60.com";
};

zone "1.241.192.in-addr.arpa" {
    type slave;
    masters { 192.241.1.2; };
    file "/var/lib/bind/1.241.192.in-addr.arpa";
};
EOT

named-checkconf
service named restart || service bind9 restart
EOF

chmod +x /root/soal12.sh
/root/soal12.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/bind/named.conf.options` (konfigurasi `forwarders { 192.168.122.1; };`)
* `/etc/bind/jarkom/1.241.192.in-addr.arpa` (zona PTR untuk `192.241.1.4`)

### 3. Cara Pengujian & Hasil Verifikasi
Diuji dari client **Itadori**:
```sh
dig @192.241.1.2 jujutsu.k60.com +short
dig @192.241.1.2 www.jujutsu.k60.com
dig @192.241.1.2 -x 192.241.1.4 +short
ping -c 3 google.com
```
**Hasil Output:**
* `dig @192.241.1.2 jujutsu.k60.com +short` $\rightarrow$ `192.241.1.4`
* `dig @192.241.1.2 www.jujutsu.k60.com` $\rightarrow$ Menampilkan `www.jujutsu.k60.com. IN CNAME jujutsu.k60.com.` dan `jujutsu.k60.com. IN A 192.241.1.4`
* `dig @192.241.1.2 -x 192.241.1.4 +short` $\rightarrow$ `jujutsu.k60.com.`
* `ping -c 3 google.com` $\rightarrow$ `3 packets transmitted, 3 received, 0% packet loss`

![Output Verifikasi A Record, CNAME, PTR, dan Forwarder di Itadori](images/soal12_itadori_dns_test.png)

---

## Soal 13: Replikasi Database MariaDB Master–Slave (Yaga/Yoga & Inumaki)

### Deskripsi
Mengonfigurasi MariaDB pada **Yaga/Yoga** (`192.241.4.17`) sebagai DB-Master dan **Inumaki** (`192.241.4.18`) sebagai DB-Slave untuk database `db_shibuya_k60` dan tabel `request_logs`. Mengaktifkan `log-bin` dan `server-id`, memberikan hak akses remote untuk seluruh node di Network 4 (`192.241.4.%`), serta membuktikan replikasi secara *realtime*.

### 1. Script `/root` pada Server (Yaga/Yoga & Inumaki)

#### A. Node **Yaga / Yoga** (`/root/soal13.sh`)
```bash
cat << 'EOF' > /root/soal13.sh
#!/bin/bash
ip addr add 192.241.4.17/24 dev eth0 2>/dev/null || true
ip link set eth0 up
ip route replace default via 192.241.4.1

if ! command -v mariadbd >/dev/null 2>&1 && ! command -v mysqld >/dev/null 2>&1; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y mariadb-server mariadb-client
fi

mkdir -p /var/log/mysql /run/mysqld
chown -R mysql:mysql /var/log/mysql /run/mysqld /var/lib/mysql
chmod 755 /var/log/mysql /run/mysqld

cat << 'EOT' > /etc/mysql/mariadb.conf.d/99-master.cnf
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = db_shibuya_k60
EOT

service mariadb restart

mariadb -u root -e "STOP SLAVE;" 2>/dev/null || true
mariadb -u root -e "RESET MASTER;"

mariadb -u root << 'EOT'
CREATE DATABASE IF NOT EXISTS db_shibuya_k60;
USE db_shibuya_k60;

CREATE TABLE IF NOT EXISTS request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    worker_name VARCHAR(50),
    client_ip VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';

CREATE USER IF NOT EXISTS 'jujutsu'@'192.241.4.%' IDENTIFIED BY 'jujutsupass';
GRANT ALL PRIVILEGES ON db_shibuya_k60.* TO 'jujutsu'@'192.241.4.%';
FLUSH PRIVILEGES;
EOT

mariadb -u root -e "SHOW MASTER STATUS\G"
EOF

chmod +x /root/soal13.sh
/root/soal13.sh
```

#### B. Node **Inumaki** (`/root/soal13.sh`)
```bash
cat << 'EOF' > /root/soal13.sh
#!/bin/bash
ip addr add 192.241.4.18/24 dev eth0 2>/dev/null || true
ip link set eth0 up
ip route replace default via 192.241.4.1

if ! command -v mariadbd >/dev/null 2>&1 && ! command -v mysqld >/dev/null 2>&1; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y mariadb-server mariadb-client
fi

mkdir -p /var/log/mysql /run/mysqld
chown -R mysql:mysql /var/log/mysql /run/mysqld /var/lib/mysql
chmod 755 /var/log/mysql /run/mysqld

cat << 'EOT' > /etc/mysql/mariadb.conf.d/99-slave.cnf
[mysqld]
bind-address = 0.0.0.0
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin.log
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = db_shibuya_k60
read_only = 1
EOT

service mariadb restart

mariadb -u root << 'EOT'
STOP REPLICA;
RESET REPLICA ALL;
CREATE DATABASE IF NOT EXISTS db_shibuya_k60;
USE db_shibuya_k60;
CREATE TABLE IF NOT EXISTS request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    worker_name VARCHAR(50),
    client_ip VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOT

MASTER_STATUS=$(mariadb -h 192.241.4.17 -u repl -preplpass -e "SHOW MASTER STATUS\G" 2>/dev/null)
LOG_FILE=$(echo "$MASTER_STATUS" | awk '/File:/ {print $2}')
LOG_POS=$(echo "$MASTER_STATUS" | awk '/Position:/ {print $2}')

if [ -z "$LOG_FILE" ]; then
    LOG_FILE=${1:-mysql-bin.000001}
    LOG_POS=${2:-328}
fi

mariadb -u root << EOT
CHANGE MASTER TO
    MASTER_HOST='192.241.4.17',
    MASTER_USER='repl',
    MASTER_PASSWORD='replpass',
    MASTER_LOG_FILE='$LOG_FILE',
    MASTER_LOG_POS=$LOG_POS;
START REPLICA;
EOT

sleep 1
mariadb -u root -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Last_Error"
EOF

chmod +x /root/soal13.sh
/root/soal13.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/mysql/mariadb.conf.d/99-master.cnf` (pada Yaga/Yoga)
* `/etc/mysql/mariadb.conf.d/99-slave.cnf` (pada Inumaki)
* `/var/log/mysql/mysql-bin.log` (binary log milik user `mysql:mysql`)

### 3. Cara Pengujian & Hasil Verifikasi
1. **Insert data baru pada Master (Yaga/Yoga):**
   ```bash
   mariadb -u root -D db_shibuya_k60 -e "INSERT INTO request_logs (worker_name, client_ip) VALUES ('tes-yaga', '192.241.4.17');"
   ```
   ![Output Insert Data di Master Yoga](images/soal13_yoga_insert.png)

2. **Periksa hasil replikasi pada Slave (Inumaki):**
   ```bash
   mariadb -u root -D db_shibuya_k60 -e "SELECT * FROM request_logs;"
   ```
   **Hasil Output di Inumaki:**
   ```text
   +----+-------------+---------------+---------------------+
   | id | worker_name | client_ip     | created_at          |
   +----+-------------+---------------+---------------------+
   |  1 | tes-yaga    | 192.241.4.17  | 2026-10-15 05:11:15 |
   +----+-------------+---------------+---------------------+
   ```
   ![Output Hasil Replikasi di Slave Inumaki](images/soal13_inumaki_select.png)

---

## Soal 14: Konfigurasi Worker Frontend (Maki, Panda) & Worker API (Nanami, Todo, Kirara)

### Deskripsi
Memasang Nginx dan PHP-FPM pada **Maki** (`192.241.4.12`) dan **Panda** (`192.241.4.13`) untuk menyajikan halaman web frontend yang menampilkan hostname masing-masing, serta menyiapkan layanan API pencatat log ke database Yaga pada **Nanami** (`.4.14`), **Todo** (`.4.15`), dan **Kirara** (`.4.16`). Membatasi akses langsung dari klien: worker frontend hanya menerima trafik dari **Gojo** (`192.241.1.4`), dan worker API hanya menerima trafik dari **Geto** (`192.241.4.11`).

### 1. Script `/root` pada Server

#### A. Node **Maki** & **Panda** (`/root/soal14.sh` — Worker Frontend)
```bash
cat << 'EOF' > /root/soal14.sh
#!/bin/bash
HOST=$(hostname)
if [ "$HOST" = "maki" ]; then
    ip addr add 192.241.4.12/24 dev eth0 2>/dev/null || true
elif [ "$HOST" = "panda" ]; then
    ip addr add 192.241.4.13/24 dev eth0 2>/dev/null || true
fi
ip link set eth0 up
ip route replace default via 192.241.4.1

if ! command -v nginx >/dev/null 2>&1 || ! dpkg -l | grep -q php-fpm; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y nginx php-fpm curl
fi

for svc in $(ls /etc/init.d/php*-fpm 2>/dev/null); do $svc start; done
PHP_SOCK=$(find /run/php/ -name "php*-fpm.sock" | head -n 1)

mkdir -p /var/www/jujutsu
cat << 'EOT' > /var/www/jujutsu/index.php
<?php
echo "Halo dari Frontend Worker: " . gethostname() . "\n";
?>
EOT

cat << EOT > /etc/nginx/sites-available/jujutsu
server {
    listen 80;
    server_name jujutsu.k60.com www.jujutsu.k60.com jujutsu.k-60.com www.jujutsu.k-60.com;

    root /var/www/jujutsu;
    index index.php index.html;

    allow 192.241.1.4;
    deny all;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:$PHP_SOCK;
    }
}
EOT

ln -sf /etc/nginx/sites-available/jujutsu /etc/nginx/sites-enabled/jujutsu
rm -f /etc/nginx/sites-enabled/default
nginx -t && service nginx restart
EOF

chmod +x /root/soal14.sh
/root/soal14.sh
```

#### B. Node **Nanami**, **Todo**, & **Kirara** (`/root/soal14.sh` — Worker API)
```bash
cat << 'EOF' > /root/soal14.sh
#!/bin/bash
HOST=$(hostname)
if [ "$HOST" = "nanami" ]; then
    ip addr add 192.241.4.14/24 dev eth0 2>/dev/null || true
elif [ "$HOST" = "todo" ]; then
    ip addr add 192.241.4.15/24 dev eth0 2>/dev/null || true
elif [ "$HOST" = "kirara" ]; then
    ip addr add 192.241.4.16/24 dev eth0 2>/dev/null || true
fi
ip link set eth0 up
ip route replace default via 192.241.4.1

if ! command -v nginx >/dev/null 2>&1 || ! dpkg -l | grep -q php-mysql; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y nginx php-fpm php-mysql mariadb-client curl
fi

for svc in $(ls /etc/init.d/php*-fpm 2>/dev/null); do $svc start; done
PHP_SOCK=$(find /run/php/ -name "php*-fpm.sock" | head -n 1)

mkdir -p /var/www/api
cat << 'EOT' > /var/www/api/index.php
<?php
header('Content-Type: application/json');
$worker = gethostname();
$client_ip = $_SERVER['HTTP_X_REAL_IP'] ?? $_SERVER['REMOTE_ADDR'];

$mysqli = @new mysqli("192.241.4.17", "jujutsu", "jujutsupass", "db_shibuya_k60");
$db_status = "connected";
if ($mysqli->connect_error) {
    $db_status = "error: " . $mysqli->connect_error;
} else {
    $stmt = $mysqli->prepare("INSERT INTO request_logs (worker_name, client_ip) VALUES (?, ?)");
    if ($stmt) {
        $stmt->bind_param("ss", $worker, $client_ip);
        $stmt->execute();
        $stmt->close();
    }
    $mysqli->close();
}

echo json_encode([
    "status" => "success",
    "worker" => $worker,
    "client_ip" => $client_ip,
    "db_log" => $db_status
]) . "\n";
?>
EOT

cat << EOT > /etc/nginx/sites-available/api
server {
    listen 80;
    server_name jujutsu.k60.com www.jujutsu.k60.com jujutsu.k-60.com www.jujutsu.k-60.com _;

    root /var/www/api;
    index index.php;

    allow 192.241.4.11;
    deny all;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:$PHP_SOCK;
    }
}
EOT

ln -sf /etc/nginx/sites-available/api /etc/nginx/sites-enabled/api
rm -f /etc/nginx/sites-enabled/default
nginx -t && service nginx restart
EOF

chmod +x /root/soal14.sh
/root/soal14.sh
```

### 2. Script/File Non-Root Aplikasi Web & Virtual Host
* `/var/www/jujutsu/index.php` dan `/etc/nginx/sites-available/jujutsu` (pada Maki & Panda)
* `/var/www/api/index.php` dan `/etc/nginx/sites-available/api` (pada Nanami, Todo, & Kirara)

### 3. Cara Pengujian & Hasil Verifikasi
Diuji dari client **Itadori** dengan mengakses IP worker secara langsung:
```sh
curl -i http://192.241.4.12
curl -i http://192.241.4.14
```
**Hasil Output:**
Kedua perintah mengembalikan status **`HTTP/1.1 403 Forbidden`** karena akses langsung dari subnet client ditolak oleh aturan `allow`/`deny` pada Nginx worker.

![Output Penolakan 403 Forbidden Akses Langsung ke Worker](images/soal14_itadori_403_forbidden.png)

---

## Soal 15: Internal Load Balancer pada Geto (Weighted Round Robin 3:1)

### Deskripsi
Mengonfigurasi Nginx pada **Geto** (`192.241.4.11`) sebagai Internal Load Balancer dengan upstream `api_cluster` yang membagi beban trafik ke **Nanami** (`192.241.4.14`) dan **Todo** (`192.241.4.15`) menggunakan algoritma Weighted Round Robin dengan bobot `3:1`.

### 1. Script `/root` pada Server (**Geto** — `/root/soal15.sh`)
```bash
cat << 'EOF' > /root/soal15.sh
#!/bin/bash
ip addr add 192.241.4.11/24 dev eth0 2>/dev/null || true
ip link set eth0 up
ip route replace default via 192.241.4.1

if ! command -v nginx >/dev/null 2>&1; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y nginx curl
fi

cat << 'EOT' > /etc/nginx/sites-available/geto_lb
upstream api_cluster {
    server 192.241.4.14:80 weight=3; # nanami
    server 192.241.4.15:80 weight=1; # todo
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOT

ln -sf /etc/nginx/sites-available/geto_lb /etc/nginx/sites-enabled/geto_lb
rm -f /etc/nginx/sites-enabled/default
nginx -t && service nginx restart
EOF

chmod +x /root/soal15.sh
/root/soal15.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/nginx/sites-available/geto_lb`

### 3. Cara Pengujian & Hasil Verifikasi
Diuji dari jaringan lokal Data Center dengan mengirimkan loop request ke **Geto**:
```bash
for i in $(seq 1 8); do curl -s http://192.241.4.11/ | grep -o '"worker":"[^"]*"'; done
```
**Hasil Output:**
Dari 8 request beruntun, `"worker":"nanami"` merespons sebanyak **6 kali** dan `"worker":"todo"` merespons sebanyak **2 kali** (rasio distribusi beban tepat **3:1**).

![Output Distribusi Beban Weighted Round Robin 3:1 pada Geto](images/soal15_geto_wrr_3_1.png)

---

## Soal 16: Public Reverse Proxy pada Gojo (Path-Based Routing)

### Deskripsi
Mengonfigurasi Nginx pada **Gojo** (`192.241.1.4`) untuk melayani domain `jujutsu.k60.com` dan `www.jujutsu.k60.com` menggunakan metode *Path-Based Routing*:
* Request ke `/` diteruskan secara Round Robin ke worker frontend (**Maki** dan **Panda**).
* Request ke `/api` diteruskan ke internal load balancer **Geto** (`192.241.4.11`).

### 1. Script `/root` pada Server (**Gojo** — `/root/soal16.sh`)
```bash
cat << 'EOF' > /root/soal16.sh
#!/bin/bash
ip addr add 192.241.1.4/24 dev eth0 2>/dev/null || true
ip link set eth0 up
ip route replace default via 192.241.1.1

if ! command -v nginx >/dev/null 2>&1; then
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    apt-get update && apt-get install -y nginx curl
fi

cat << 'EOT' > /etc/nginx/sites-available/gojo_proxy
upstream frontend_cluster {
    server 192.241.4.12:80; # maki
    server 192.241.4.13:80; # panda
}

server {
    listen 80;
    server_name jujutsu.k60.com www.jujutsu.k60.com jujutsu.k-60.com www.jujutsu.k-60.com;

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api {
        proxy_pass http://192.241.4.11/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOT

ln -sf /etc/nginx/sites-available/gojo_proxy /etc/nginx/sites-enabled/gojo_proxy
rm -f /etc/nginx/sites-enabled/default
nginx -t && service nginx restart
EOF

chmod +x /root/soal16.sh
/root/soal16.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/nginx/sites-available/gojo_proxy`

### 3. Cara Pengujian & Hasil Verifikasi
Diuji dari client **Itadori** (`192.241.2.10`):
```sh
for i in 1 2 3 4; do curl -s http://jujutsu.k60.com/; done
for i in 1 2 3 4; do curl -s http://jujutsu.k60.com/api; done
```
**Hasil Output:**
```text
Halo dari Frontend Worker: maki
Halo dari Frontend Worker: panda
Halo dari Frontend Worker: maki
Halo dari Frontend Worker: panda
{"status":"success","worker":"nanami","client_ip":"192.241.1.4","db_log":"connected"}
{"status":"success","worker":"todo","client_ip":"192.241.1.4","db_log":"connected"}
{"status":"success","worker":"nanami","client_ip":"192.241.1.4","db_log":"connected"}
{"status":"success","worker":"nanami","client_ip":"192.241.1.4","db_log":"connected"}
```

![Output Path-Based Routing Web Utama dan API dari Itadori](images/soal16_itadori_path_routing.png)

---

## Soal 17: Pelacakan IP Asli Klien (`X-Real-IP`, `X-Forwarded-For`, & Trusted Proxy)

### Deskripsi
Mengonfigurasi **Gojo** dan **Geto** agar meneruskan informasi IP asli klien melalui header `X-Real-IP` dan `X-Forwarded-For`. Memastikan **Geto** hanya mempercayai informasi alamat klien dari proxy terpercaya (**Gojo** `192.241.1.4` menggunakan `set_real_ip_from`), sehingga alamat yang tersimpan di kolom `client_ip` pada tabel `request_logs` di database **Yaga** adalah IP asli klien.

### 1. Script `/root` pada Server (**Geto** — `/root/soal17.sh`)
```bash
cat << 'EOF' > /root/soal17.sh
#!/bin/bash
cat << 'EOT' > /etc/nginx/sites-available/geto_lb
upstream api_cluster {
    server 192.241.4.14:80 weight=3; # nanami
    server 192.241.4.15:80 weight=1; # todo
}

server {
    listen 80;
    server_name _;

    set_real_ip_from 192.241.1.4;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOT

nginx -t && service nginx restart
EOF

chmod +x /root/soal17.sh
/root/soal17.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/nginx/sites-available/geto_lb` (penambahan direktif `set_real_ip_from 192.241.1.4;` dan `real_ip_header X-Forwarded-For;`)
* `/var/www/api/index.php` pada Worker API (`$_SERVER['HTTP_X_REAL_IP'] ?? $_SERVER['REMOTE_ADDR']`)

### 3. Cara Pengujian & Hasil Verifikasi
1. Kirim request dari **Itadori** (Network 2 — `192.241.2.10`) dan **Kugisaki** (Network 3 — `192.241.3.99`):
   ```sh
   curl http://jujutsu.k60.com/api
   ```
2. Periksa catatan log di database **Yaga/Yoga** (`192.241.4.17`):
   ```bash
   mysql -u root -D db_shibuya_k60 -e "SELECT * FROM request_logs ORDER BY id DESC LIMIT 5;"
   ```
   **Hasil Output di Yaga/Yoga:**
   ```text
   +----+-------------+---------------+---------------------+
   | id | worker_name | client_ip     | created_at          |
   +----+-------------+---------------+---------------------+
   | 27 | nanami      | 192.241.3.99  | 2026-10-15 06:14:30 |
   | 26 | nanami      | 192.241.2.10  | 2026-10-15 06:14:20 |
   | 25 | nanami      | 192.241.1.4   | 2026-10-15 06:12:59 |
   | 24 | nanami      | 192.241.1.4   | 2026-10-15 06:12:59 |
   | 23 | todo        | 192.241.1.4   | 2026-10-15 06:12:59 |
   +----+-------------+---------------+---------------------+
   ```
   *(Terlihat jelas perbandingan sebelum Soal 17 [masih tercatat IP Gojo `192.241.1.4`] dan sesudah Soal 17 [tercatat IP asli klien `192.241.2.10` dan `192.241.3.99`]).*

   ![Output Verifikasi Client IP Asli pada Tabel Database Yoga](images/soal17_yoga_client_ip_logs.png)

---

## Soal 18: Failover Worker API Cadangan (Kirara) & Passive Health Check

### Deskripsi
Mengonfigurasi upstream `api_cluster` pada **Geto** agar **Kirara** (`192.241.4.16`) menjadi server `backup`, serta menerapkan *passive health check* dengan parameter `max_fails=2` dan `fail_timeout=5s` pada worker API utama (**Nanami** dan **Todo**).

### 1. Script `/root` pada Server (**Geto** — `/root/soal18.sh`)
```bash
cat << 'EOF' > /root/soal18.sh
#!/bin/bash
cat << 'EOT' > /etc/nginx/sites-available/geto_lb
upstream api_cluster {
    server 192.241.4.14:80 weight=3 max_fails=2 fail_timeout=5s; # nanami (utama)
    server 192.241.4.15:80 weight=1 max_fails=2 fail_timeout=5s; # todo (utama)
    server 192.241.4.16:80 backup;                               # kirara (cadangan)
}

server {
    listen 80;
    server_name _;

    set_real_ip_from 192.241.1.4;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_502 http_503 http_504;
    }
}
EOT

nginx -t && service nginx restart
EOF

chmod +x /root/soal18.sh
/root/soal18.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/nginx/sites-available/geto_lb`

### 3. Cara Pengujian & Hasil Verifikasi
1. Hentikan layanan Nginx pada kedua worker utama (**Nanami** dan **Todo**):
   ```bash
   # Di Nanami & Todo:
   service nginx stop
   ```
2. Uji akses endpoint `/api` dari client **Itadori**:
   ```sh
   curl -i http://jujutsu.k60.com/api
   ```
   **Hasil Output di Itadori:**
   ```text
   HTTP/1.1 200 OK
   Server: nginx
   Date: Thu, 15 Oct 2026 06:19:06 GMT
   Content-Type: application/json
   Transfer-Encoding: chunked
   Connection: keep-alive

   {"status":"success","worker":"kirara","client_ip":"192.241.2.10","db_log":"connected"}
   ```
   *(Request berhasil dialihkan secara otomatis ke `kirara` dengan status `200 OK` tanpa mengalami error `502 Bad Gateway`. Setelah pengujian selesai, layanan Nginx pada Nanami dan Todo dinyalakan kembali dengan `service nginx start`).*

   ![Output Failover ke Backup Worker Kirara saat Worker Utama Mati](images/soal18_itadori_failover_kirara.png)

---

## Soal 19: Proxy Caching (`10m`) & Rate Limiting (`10r/s`) pada Gojo

### Deskripsi
Mengonfigurasi *Nginx Proxy Caching* pada **Gojo** untuk halaman web frontend (`/`) dengan masa berlaku cache 10 menit dan menambahkan header `X-Cache-Status`. Menerapkan *rate limiting* pada path `/api` dengan batas 10 request per detik per IP klien (`rate=10r/s`).

### 1. Script `/root` pada Server (**Gojo** — `/root/soal19.sh`)
```bash
cat << 'EOF' > /root/soal19.sh
#!/bin/bash
mkdir -p /var/cache/nginx/frontend_cache
chown -R www-data:www-data /var/cache/nginx/frontend_cache

cat << 'EOT' > /etc/nginx/sites-available/gojo_proxy
proxy_cache_path /var/cache/nginx/frontend_cache levels=1:2 keys_zone=FRONTEND_CACHE:10m max_size=100m inactive=10m use_temp_path=off;

limit_req_zone $binary_remote_addr zone=API_LIMIT:10m rate=10r/s;

upstream frontend_cluster {
    server 192.241.4.12:80; # maki
    server 192.241.4.13:80; # panda
}

server {
    listen 80;
    server_name jujutsu.k60.com www.jujutsu.k60.com jujutsu.k-60.com www.jujutsu.k-60.com;

    location / {
        proxy_cache FRONTEND_CACHE;
        proxy_cache_valid 200 10m;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status always;

        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api {
        limit_req zone=API_LIMIT;
        limit_req_status 429;

        proxy_pass http://192.241.4.11/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOT

nginx -t && service nginx restart
EOF

chmod +x /root/soal19.sh
/root/soal19.sh
```

### 2. Konfigurasi File Non-Root
* `/etc/nginx/sites-available/gojo_proxy`
* Direktori cache `/var/cache/nginx/frontend_cache` (milik user `www-data:www-data`)

### 3. Cara Pengujian & Hasil Verifikasi
1. **Pengujian Proxy Caching dari Itadori:**
   ```sh
   curl -I http://jujutsu.k60.com/
   curl -I http://jujutsu.k60.com/
   ```
   **Hasil Output:**
   ```text
   HTTP/1.1 200 OK
   Server: nginx
   Date: Thu, 15 Oct 2026 06:22:13 GMT
   Content-Type: text/html; charset=UTF-8
   Connection: keep-alive
   X-Cache-Status: MISS

   HTTP/1.1 200 OK
   Server: nginx
   Date: Thu, 15 Oct 2026 06:22:13 GMT
   Content-Type: text/html; charset=UTF-8
   Connection: keep-alive
   X-Cache-Status: HIT
   ```
   ![Output Header X-Cache-Status MISS dan HIT di Itadori](images/soal19_itadori_cache_miss_hit.png)

2. **Pengujian Rate Limiting `/api` dari Itadori:**
   ```sh
   for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code}\n" http://jujutsu.k60.com/api & done; wait
   ```
   **Hasil Output:**
   Request awal yang masuk dalam kuota mendapatkan respons **`200`**, sedangkan 19 request paralel yang melampaui batas `10r/s` langsung ditolak dengan kode status **`429`** (*Too Many Requests*).
   ![Output Pengujian Rate Limit 200 dan 429 di Itadori](images/soal19_itadori_ratelimit_429.png)

---

## Soal 20: Audit Kapasitas Layanan dengan Apache Benchmark (`ab`) & Analisis Log

### Deskripsi
Melakukan pengujian kapasitas dari terminal **Rusakabe (Kusakabe)** (`192.241.3.10`) menggunakan Apache Benchmark (`ab`) dengan parameter `-n 600 -c 30` menuju `http://jujutsu.k60.com/` dan `http://jujutsu.k60.com/api`, serta menganalisis `access.log` dan `error.log` pada **Gojo** untuk membedakan request sukses, kegagalan aplikasi/koneksi, dan request yang ditolak oleh kebijakan *rate limit*.

### 1. Script pada Client & Server

#### A. Node Client **Rusakabe / Kusakabe** (`/root/soal20.sh`)
*(Catatan: MAC Address `eth0` aktual pada container Rusakabe `f2:dd:ec:51:c9:cb` telah disinkronkan pada `/etc/dhcp/dhcpd.conf` di Tengen sehingga Rusakabe memperoleh lease `192.241.3.10` dengan lease time `1800` detik).*
```sh
cat << 'EOF' > /root/soal20.sh
#!/bin/sh
udhcpc -i eth0 -n -q -O 119

if ! command -v ab >/dev/null 2>&1; then
    apk update && apk add apache2-utils curl
fi

echo "=========================================="
echo "1. UJI BEBAN HALAMAN WEB UTAMA (/)"
echo "=========================================="
ab -n 600 -c 30 http://jujutsu.k60.com/

echo "=========================================="
echo "2. UJI BEBAN ENDPOINT API (/api)"
echo "=========================================="
ab -n 600 -c 30 http://jujutsu.k60.com/api
EOF

chmod +x /root/soal20.sh
/root/soal20.sh
```

#### B. Node Server **Gojo** (`/root/soal20_log.sh`)
```bash
cat << 'EOF' > /root/soal20_log.sh
#!/bin/bash
echo "=== RINGKASAN HTTP STATUS CODE DI ACCESS.LOG GOJO ==="
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr

echo ""
echo "=== JUMLAH REQUEST DITOLAK OLEH RATE LIMIT DI ERROR.LOG ==="
grep -c "limiting requests" /var/log/nginx/error.log || true

echo ""
echo "=== 5 LOG TERAKHIR RATE LIMIT DI ERROR.LOG ==="
grep "limiting requests" /var/log/nginx/error.log | tail -n 5 || true
EOF

chmod +x /root/soal20_log.sh
/root/soal20_log.sh
```

### 2. File Log Non-Root yang Dianalisis
* `/var/log/nginx/access.log` (pada Gojo)
* `/var/log/nginx/error.log` (pada Gojo)

### 3. Hasil Pengujian & Analisis Metrik

#### A. Ringkasan Perbandingan Metrik Apache Benchmark (`-n 600 -c 30`)

| Parameter Metrik | Halaman Utama (`/`) — *Proxy Cache Aktif* | Endpoint API (`/api`) — *Rate Limit `10r/s` Aktif* |
| ----- | ----- | ----- |
| **Concurrency Level** | `30` | `30` |
| **Time taken for tests** | `0.100 seconds` | `0.110 seconds` |
| **Complete requests** | `600` | `600` |
| **Requests per second (RPS)** | **`6010.04 [#/sec]`** | **`5440.94 [#/sec]`** |
| **Time per request (Mean)** | **`4.992 [ms]`** | **`5.514 [ms]`** |
| **Time per request (Concurrent)** | `0.166 [ms]` | `0.184 [ms]` |
| **Failed requests (Connect/Receive)** | `0` | `0` (Tidak ada kegagalan koneksi) |
| **Non-2xx responses (Rate Limited)** | **`0`** (100% `200 OK`) | **`598`** (598 ditolak `429`, 2 lolos `200 OK`) |

1. **Screenshot Hasil Benchmark Halaman Web Utama (`/`):**
   ![Output Apache Benchmark Halaman Utama di Rusakabe](images/soal20_rusakabe_ab_web.png)

2. **Screenshot Hasil Benchmark Endpoint API (`/api`):**
   ![Output Apache Benchmark Endpoint API di Rusakabe](images/soal20_rusakabe_ab_api.png)

#### B. Hasil Analisis Access Log & Error Log pada Gojo
```text
=== RINGKASAN HTTP STATUS CODE DI ACCESS.LOG GOJO ===
    628 200
    617 429

=== JUMLAH REQUEST DITOLAK OLEH RATE LIMIT DI ERROR.LOG ===
617

=== 5 LOG TERAKHIR RATE LIMIT DI ERROR.LOG ===
2026/10/15 06:31:40 [error] 517#517: *1243 limiting requests, excess: 0.940 by zone "API_LIMIT", client: 192.241.3.10, server: jujutsu.k60.com, request: "GET /api HTTP/1.0", host: "jujutsu.k60.com"
2026/10/15 06:31:40 [error] 515#515: *1244 limiting requests, excess: 0.930 by zone "API_LIMIT", client: 192.241.3.10, server: jujutsu.k60.com, request: "GET /api HTTP/1.0", host: "jujutsu.k60.com"
```

3. **Screenshot Hasil Analisis Access Log & Error Log di Gojo:**
   ![Output Analisis Log Access dan Error pada Gojo](images/soal20_gojo_log_analysis.png)

