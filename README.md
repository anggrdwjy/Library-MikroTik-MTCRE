# MTCRE — Panduan Implementasi dan Operasi

## Daftar Isi

A. [Module 1: Static Routing](#1-module-1-static-routing)
B. [Module 2: Point to Point Addressing](#2-module-2-point-to-point-addressing)
C. [Module 3: VPN, VLAN, dan QinQ](#3-module-3-vpn-vlan-dan-qinq)
D. [Module 4: OSPF](#4-module-4-ospf)

## 1. Module 1: Static Routing

Static routing adalah routing yang ditulis secara manual. Kelebihannya: bersifat deterministik, mudah dibaca, dan mudah diaudit. Kekurangannya: routing statis tidak adaptif secara otomatis apabila topologi berubah. Karena itu di operasi nyata, static digunakan untuk: default gateway, link khusus, failover sederhana, dan policy routing.

### 1.0 Kolom Perbandingan dan Konfigurasi RouterOS v6 dan v7 - Module 1

Gunakan tabel berikut sebagai referensi cepat sebelum membaca detail modul. Kolom konfigurasi menampilkan perintah inti yang berbeda pada kedua versi.

| Aspek | Konfigurasi RouterOS 6.49 | Konfigurasi RouterOS 7.2 | Perberbedaan Utama |
|---|---|---|---|
| Policy routing | `/ip route add gateway=192.0.2.1 routing-mark=via-WAN2` | `/routing table add name=via-WAN2 fib` + `/ip route add gateway=192.0.2.1 routing-table=via-WAN2` | v7 wajib membuat tabel terlebih dahulu |
| Routing rule | `/ip route rule add table=via-WAN2 action=lookup` | `/routing rule add table=via-WAN2 action=lookup` | Lokasi menu berubah |
| Mangle mark-routing | `/ip firewall mangle add action=mark-routing new-routing-mark=via-WAN2` | Sama, tetapi mark harus merujuk ke tabel yang sudah dibuat | Prinsip sama, prasyarat tabel berbeda |
| Recursive gateway | `/ip route add gateway=8.8.8.8 scope=30 target-scope=10` | Sama, tetapi objek gateway terisolasi per route | v6 satu objek gateway per alamat, v7 banyak objek |
| Pemantauan route | `/ip route print` | `/ip route print` + `/routing route print` | v7 menambah menu `/routing route` |

### 1.1 Konsep yang harus dikuasai

- **More specific routes:** Router selalu memilih prefix yang paling panjang (longest prefix match). Route 192.168.20.128/25 akan menang atas 192.168.20.0/24 untuk IP 192.168.20.130, walau distance-nya sama.
- **ECMP:** Jika ada dua route dengan dst-address dan distance yang sama, RouterOS akan membagi beban ke keduanya. Ini bukan load balancing per-paket yang sempurna, tetapi per-koneksi.
- **Gateway reachability dan distance:** Distance adalah prioritas. Semakin kecil, semakin diprioritaskan. Check-gateway dengan ping membuat router tidak lagi menganggap gateway aktif jika tidak menjawab ping.
- **Routing mark dan route policy:** Cara untuk memaksa kelompok trafik tertentu melalui jalur tertentu, tanpa mengubah route utama untuk semua orang.
- **Recursive next-hop dan scope/target-scope:** Gateway boleh berupa IP yang tidak langsung terhubung, asal router dapat menyelesaikan gateway tersebut melalui route lain. Scope mengontrol seberapa jauh pencarian itu boleh dilakukan.

### 1.2 Implementasi: Route dasar dan more specific routes

Langkah 1: Buat route ke LAN cabang melalui R2. Langkah 2: Buat route yang lebih spesifik untuk satu subnet kecil di cabang melalui jalur lain, untuk membuktikan longest prefix match.

```routeros
# [HQ-R1] Route umum ke LAN BR melalui R2
/ip route add dst-address=192.168.20.0/24 gateway=10.0.12.2

# [HQ-R1] Route lebih spesifik ke server di BR (192.168.20.128/25)
# Route ini akan menang untuk IP 192.168.20.130 - 192.168.20.254
/ip route add dst-address=192.168.20.128/25 gateway=10.0.23.2

# Verifikasi: lihat route mana yang aktif (tanda A) dan mana yang dipilih
/ip route print where dst-address in 192.168.20.0/24
```

Operasi: setelah menambahkan route, selalu uji dengan ping ke IP di dalam subnet umum dan subnet spesifik. Jika ping ke subnet spesifik gagal tetapi ping ke subnet umum berhasil, kemungkinan gateway untuk route spesifik salah atau tidak reachable.

### 1.3 Implementasi: ECMP untuk load sharing

ECMP berguna saat terdapat dua ISP dengan kualitas mirip dan keduanya ingin digunakan, bukan hanya satu sebagai cadangan.

```routeros
# [HQ-R1] Dua gateway dengan distance sama = ECMP
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1,192.0.2.1 check-gateway=ping distance=1

# Periksa apakah kedua gateway aktif
/ip route print where dst-address=0.0.0.0/0
```

> **Batasan ECMP:** ECMP di RouterOS bekerja per-koneksi. Satu koneksi besar tidak akan terpecah sehingga dua. Jika pembagian berdasarkan jenis trafik dibutuhkan, gabungkan ECMP dengan routing-mark di sub-bab 1.6.

### 1.4 Implementasi: Force gateway melalui antarmuka tertentu

Terterkadang ada dua gateway dengan IP yang sama atau router harus dipaksa keluar melalui antarmuka tertentu, misalnya karena ada dua ISP dengan gateway yang sama. Gunakan sintaks `gateway%interface`.

```routeros
# [HQ-R1] Paksa gateway melalui WAN1 secara eksplisit
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1%ether1-WAN1 distance=1

# Contoh kasus IP gateway sama di dua ISP (misal sama-sama 192.168.1.1)
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1%ether1-WAN1 distance=1
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1%ether2-WAN2 distance=2
```

### 1.5 Implementasi: Failover dengan check-gateway dan distance

Pola ini paling sering digunakan dalam operasional: ISP 1 utama, ISP 2 cadangan. Jika ISP 1 tidak menjawab ping, route-nya dianggap tidak aktif dan route cadangan mengambil alih.

```routeros
# [HQ-R1] Failover sederhana
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 check-gateway=ping distance=1 comment="WAN1-Utama"
/ip route add dst-address=0.0.0.0/0 gateway=192.0.2.1 check-gateway=ping distance=2 comment="WAN2-Cadangan"

# Periksa status: route aktif ditandai A, route tidak aktif tetap ada tetapi tidak dipakai
/ip route print
```

Operasi: jangan hanya mengandalkan check-gateway ke IP gateway ISP. Jika gateway ISP tetap menjawab ping tetapi internet di belakangnya putus, failover tidak akan terjadi. Untuk kasus itu, gunakan Netwatch atau script ping ke 8.8.8.8 dan ubah distance secara otomatis, atau gunakan recursive gateway di sub-bab 1.7.

### 1.6 Implementasi: Routing mark dan route policy

Contoh kebutuhan: semua user di LAN HQ harus melalui WAN1, kecuali Divisi Finance yang harus melalui WAN2 karena alasan compliance. Route utama tidak boleh diubah untuk semua orang, sehingga gunakan routing-mark.

```routeros
# [HQ-R1] v7: Buat tabel routing baru
/routing table add name=via-WAN2 fib

# [HQ-R1] Tandai trafik Finance (192.168.10.50-192.168.10.60) di mangle
/ip firewall mangle add chain=prerouting src-address=192.168.10.50-192.168.10.60 action=mark-routing new-routing-mark=via-WAN2 passthrough=yes comment="Finance via WAN2"

# [HQ-R1] Buat rule agar mark tersebut menggunakan tabel via-WAN2
/ip route rule add src-address=192.168.10.50-192.168.10.60 table=via-WAN2 action=lookup

# [HQ-R1] Isi tabel via-WAN2 dengan default melalui WAN2
/ip route add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=via-WAN2

# [HQ-R1] Route utama tetap melalui WAN1 untuk yang lain
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1
```

**Catatan v6:** di v6, sintaksnya lebih sederhana, `/ip route add gateway=... routing-mark=via-WAN2`. Di v7, tabel wajib dibuat terlebih dahulu. Jangan mencampurkan kedua cara tersebut.

### 1.7 Implementasi: Recursive next-hop dan scope/target-scope

Recursive gateway digunakan untuk failover yang lebih cerdas: gateway default tidak langsung ke IP ISP, tetapi ke IP virtual yang hanya dianggap reachable jika IP pantau di internet dapat di-ping melalui ISP tersebut.

```routeros
# [HQ-R1] Contoh recursive: 8.8.8.8 hanya dapat dicapai melalui WAN1
/ip route add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10 comment="Pemantau WAN1"
/ip route add dst-address=8.8.4.4/32 gateway=192.0.2.1 scope=10 target-scope=10 comment="Pemantau WAN2"

# Default melalui IP pantau, bukan langsung ke gateway ISP
/ip route add dst-address=0.0.0.0/0 gateway=8.8.8.8 check-gateway=ping distance=1 scope=30 target-scope=10 comment="Default via WAN1 recursive"
/ip route add dst-address=0.0.0.0/0 gateway=8.8.4.4 check-gateway=ping distance=2 scope=30 target-scope=10 comment="Default via WAN2 recursive"
```

Penjelasan scope: scope adalah seberapa jauh route ini boleh digunakan untuk menyelesaikan gateway lain. Target-scope adalah scope yang dicari. Aturan praktis: route ke IP pantau harus memiliki scope 10, route default yang menggunakan IP pantau harus memiliki scope 30 dan target-scope 10. Jika terjadi kesalahan penempatan, recursive tidak akan berjalan dan route akan terlihat tidak aktif.

### 1.8 Studi kasus Module 1: Dual ISP dengan pemisahan trafik

**Situasi:** HQ memiliki dua ISP. ISP 1 cepat tetapi sering putus di siang hari. ISP 2 lambat tetapi stabil. Manajemen minta: (1) jika ISP 1 putus, internet pindah otomatis ke ISP 2, (2) trafik Finance selalu melalui ISP 2, (3) trafik server cadangan (192.168.10.100) selalu melalui ISP 1 saat ISP 1 hidaktif, tetapi boleh pindah ke ISP 2 jika ISP 1 mati.

**Desain:** Gunakan recursive gateway untuk failover yang benar-benar mengeperiksa internet, bukan hanya gateway ISP. Gunakan routing-mark untuk Finance. Gunakan route lebih spesifik untuk server cadangan.

```routeros
# [HQ-R1] Langkah 1: Pemantau recursive
/ip route add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10
/ip route add dst-address=8.8.4.4/32 gateway=192.0.2.1 scope=10 target-scope=10
/ip route add dst-address=0.0.0.0/0 gateway=8.8.8.8 check-gateway=ping distance=1 scope=30 target-scope=10
/ip route add dst-address=0.0.0.0/0 gateway=8.8.4.4 check-gateway=ping distance=2 scope=30 target-scope=10

# Langkah 2: Finance selalu via WAN2
/routing table add name=via-WAN2 fib
/ip firewall mangle add chain=prerouting src-address=192.168.10.50-192.168.10.60 action=mark-routing new-routing-mark=via-WAN2 passthrough=yes
/ip route rule add src-address=192.168.10.50-192.168.10.60 table=via-WAN2 action=lookup
/ip route add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=via-WAN2

# Langkah 3: Uji failover
# Cabut kabel WAN1 atau matikan gateway 203.0.113.1, lalu:
/ip route print where dst-address=0.0.0.0/0
/ping 8.8.8.8
/tool traceroute 8.8.8.8
```

### 1.9 Verifikasi dan operasi Module 1

```routeros
# Periksa route aktif dan distance
/ip route print
/ip route print where active

# Periksa route untuk IP tujuan tertentu (simulasi longest prefix)
/ip route print where dst-address in 192.168.20.130

# Periksa apakah mark berjalan (counter harus naik)
/ip firewall mangle print where comment~"Finance"

# Periksa failover: matikan WAN1, lalu periksa route berubah dalam 10-20 detik
/log print where message~"route"
```

- **Operasi harian:** periksa `/ip route print` setiap pagi, pastikan route utama berstatus A. Jika ada route cadangan yang tiba-tiba aktif, itu tanda ISP utama bermasalah.
- **Troubleshooting:** jika ping ke gateway berhasil tetapi ping ke internet gagal, periksa apakah recursive gateway-nya masih reachable dan apakah ada NAT yang hilang. Jika routing-mark tidak berjalan, periksa urutan mangle dan pastikan passthrough tidak memotong mark berikutnya.
- **Dokumentasi:** setiap route harus memiliki comment yang menjelaskan pemilik dan tujuannya. Route tanpa comment adalah hutang teknis.


### 1.10 Perbandingan RouterOS 6.49 dan 7.2 untuk Module 1

| Aspek | RouterOS 6.49 | RouterOS 7.2 |
|---|---|---|
| Routing-mark | Dapat langsung ditulis pada `/ip route` dengan parameter `routing-mark`. | Harus membuat tabel terlebih dahulu di `/routing table` dengan parameter `fib`, kemudian merujuk tabel tersebut pada `/ip route`. |
| Route rule | Menggunakan `/ip route rule`. | Menggunakan `/routing rule` dengan sintaks yang lebih fleksibel. |
| Objek gateway | Hanya satu objek gateway per alamat, sehingga perubahan `target-scope` pada satu route dapat memengaruhi route lain. | Satu objek gateway per kombinasi scope dan pemeriksaan gateway, sehingga perubahan pada satu route tidak memengaruhi route lain. |
| ECMP dan check-gateway | Tersedia dan bekerja dengan cara yang sama. | Tersedia dan bekerja dengan cara yang sama, tetapi pemrosesan print route lebih cepat karena ditangani proses routing. |
| Recursive gateway | Menggunakan `scope` dan `target-scope` dengan cara yang sama. | Menggunakan konsep yang sama, tetapi penyelesaian gateway lebih terisolasi antar route. |
| Pemantauan route | Menggunakan `/ip route print`. | Menambah `/routing route` yang menampilkan seluruh route, route terfilter, dan seluruh atribut route. |

**Implikasi:** Apabila dilakukan migrasi konfigurasi policy routing dari 6.49, tabel routing di v7 wajib dibuat ulang. Konfigurasi lama yang hanya menulis `routing-mark` pada route tidak akan berfungsi tanpa tabel yang didefinisikan terlebih dahulu.

#### Perberbedaan Konfigurasi Module 1: Policy Routing

**RouterOS 6.49 - Policy routing Finance via WAN2:**
```routeros
# v6: routing-mark langsung pada route, tidak perlu membuat tabel
/ip firewall mangle add chain=prerouting src-address=192.168.10.50-192.168.10.60 action=mark-routing new-routing-mark=via-WAN2 passthrough=yes
/ip route add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-mark=via-WAN2
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1
/ip route rule add src-address=192.168.10.50-192.168.10.60 table=via-WAN2 action=lookup
```

**RouterOS 7.2 - Policy routing Finance via WAN2:**
```routeros
# v7: wajib membuat tabel terlebih dahulu
/routing table add name=via-WAN2 fib
/ip firewall mangle add chain=prerouting src-address=192.168.10.50-192.168.10.60 action=mark-routing new-routing-mark=via-WAN2 passthrough=yes
/routing rule add src-address=192.168.10.50-192.168.10.60 table=via-WAN2 action=lookup
/ip route add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=via-WAN2
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1
```

#### Perberbedaan Konfigurasi Module 1: Recursive Gateway dan Scope

**RouterOS 6.49:**
```routeros
/ip route add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10
/ip route add dst-address=0.0.0.0/0 gateway=8.8.8.8 check-gateway=ping distance=1 scope=30 target-scope=10
# Perhatian: satu objek gateway per alamat, perubahan target-scope pada satu route dapat memengaruhi route lain
```

**RouterOS 7.2:**
```routeros
/ip route add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10
/ip route add dst-address=0.0.0.0/0 gateway=8.8.8.8 check-gateway=ping distance=1 scope=30 target-scope=10
# Perilaku: setiap kombinasi scope + gateway check memiliki objek tersendiri, perubahan pada satu route tidak memengaruhi route lain
# Pemantauan baru:
/routing route print where dst-address in 8.8.8.8
```


## 2. Module 2: Point to Point Addressing

Point to Point (PtP) adalah cara memberi alamat IP untuk link yang hanya menghubungkan dua router. Pada link seperti ini, network address dan broadcast address tidak dibutuhkan, karena hanya terdapat dua ujung. Dengan PtP, Dapat menghemat IP dan membuat konfigurasi lebih bersih.

### 2.0 Kolom Perbandingan dan Konfigurasi RouterOS v6 dan v7 - Module 2

| Aspek | Konfigurasi RouterOS 6.49 | Konfigurasi RouterOS 7.2 | Perberbedaan Utama |
|---|---|---|---|
| PtP /30 | `/ip address add address=10.0.12.1/30 interface=ether4-to-R2 network=10.0.12.0` | Sama persis | Tidak ada perberbedaan |
| PtP /32 | `/ip address add address=10.0.12.1/32 interface=ether4-to-R2 network=10.0.12.2` | Sama persis | Field `network` tetap berisi IP lawan |
| Route terhubunged | Otomatis dari `/ip address` | Otomatis, pemrosesan lebih cepat | Tidak ada perubahan sintaks |
| Verifikasi | `/ip route print` | `/ip route print` + `/routing route print` | v7 menambah pemantauan terpadu |

### 2.1 Konsep

- **/30 biasa:** 10.0.12.0/30 memiliki 4 alamat: network, 2 host, broadcast. Hanya 2 yang dapat dipakai. Cukaktif, tetapi boros jika digunakan di banyak link.
- **/32 PtP:** 10.0.12.1/32 dengan network=10.0.12.2 artinya: IP pada router pertama adalah 10.0.12.1, IP pada router lawan adalah 10.0.12.2. Tidak ada broadcast yang terbuang. Ini yang digunakan di banyak tunnel dan link antar router.
- **Kapan gunakan PtP:** link antar router, tunnel, dan koneksi yang memang hanya dua titik. Jangan gunakan PtP untuk LAN yang berisi banyak host.

### 2.2 Implementasi

```routeros
# Cara 1: /30 konvensional (mudah dibaca, cocok untuk laboratorium awal)
# [HQ-R1]
/ip address add address=10.0.12.1/30 interface=ether4-to-R2 network=10.0.12.0
# [R2]
/ip address add address=10.0.12.2/30 interface=ether1-to-HQ network=10.0.12.0

# Cara 2: /32 Point to Point (hemat IP, digunakan di banyak tunnel)
# [HQ-R1] network diisi IP lawan
/ip address add address=10.0.12.1/32 interface=ether4-to-R2 network=10.0.12.2
# [R2]
/ip address add address=10.0.12.2/32 interface=ether1-to-HQ network=10.0.12.1

# Verifikasi
/ip address print
/ping 10.0.12.2
/ip route print where dst-address in 10.0.12.2
```

Perhatikan field network di cara 2. Di RouterOS, untuk PtP, field network harus diisi IP lawan, bukan network address. Jika diisi 10.0.12.0, ping akan gagal walau IP terlihat benar. Kesalahan tersebut paling sering muncul dalam ujian MTCRE.

### 2.3 Studi kasus Module 2: Hemat IP untuk 20 link antar site

**Situasi:** ISP memiliki 20 link antar POP dan hanya memiliki satu blok /28 (16 alamat) untuk semua link. Jika gunakan /30, satu link membutuhkan 4 alamat, sehingga 20 link membutuhkan 80 alamat, tidak cukup.

**Solusi:** Gunakan /32 PtP. Setiap link hanya membutuhkan 2 alamat IP host, tanpa network dan broadcast. Dengan /28, ISP dapat membuat 8 link, dan dengan blok /27 dapat membuat 16 link, jauh lebih efisien. Untuk 20 link, cukup gunakan blok /26 dengan PtP, dibanding /24 jika gunakan /30.

```routeros
# Contoh 3 link PtP dari satu blok 10.10.0.0/26
# Link 1: HQ-R2
# [HQ] 10.10.0.1/32 network=10.10.0.2
# [R2] 10.10.0.2/32 network=10.10.0.1

# Link 2: R2-BR
# [R2] 10.10.0.3/32 network=10.10.0.4
# [BR] 10.10.0.4/32 network=10.10.0.3

# Link 3: HQ-BR cadangan
# [HQ] 10.10.0.5/32 network=10.10.0.6
# [BR] 10.10.0.6/32 network=10.10.0.5
```

### 2.4 Verifikasi dan operasi Module 2

```routeros
# Periksa IP dan status interface
/ip address print
/interface print where running

# Periksa route terhubunged otomatis muncul
/ip route print where type=terhubunged

# Jika ping gagal, periksa network di kedua sisi harus saling menunjuk
/ip address print detail where interface=ether4-to-R2
```

- **Operasi:** dokumentasikan pasangan IP PtP di spreadsheet. Kesalahan satu digit di field network akan membuat link terlihat aktif tetapi tidak dapat ping. Selalu periksa dari kedua sisi.
- **Troubleshooting:** jika interface aktif tetapi route terhubunged tidak muncul, periksa apakah ada IP yang duplikat atau interface salah. Jika ping berhasil dari satu arah saja, periksa firewall dan pastikan tidak ada drop di chain input.


### 2.5 Perbandingan RouterOS 6.49 dan 7.2 untuk Module 2

| Aspek | RouterOS 6.49 | RouterOS 7.2 |
|---|---|---|
| Sintaks `/ip address` | Sama, termasuk penggunaan `/32` dengan field `network` berisi IP lawan untuk PtP. | Sama, tidak terdapat perubahan sintaks yang signifikan. |
| Route terhubunged | Dibuat otomatis dari alamat IP. | Dibuat otomatis dari alamat IP, dengan pemrosesan yang lebih cepat pada proses routing. |
| Penggunaan IP | Efisiensi yang sama untuk `/30` dan `/32`. | Efisiensi yang sama untuk `/30` dan `/32`. |
| Perilaku PtP | Field `network` harus berisi IP lawan agar ping berhasil. | Perilaku yang sama, field `network` tetap harus berisi IP lawan. |

**Implikasi:** Modul ini merupakan modul yang paling stabil antar versi. Konfigurasi yang sama dapat digunakan pada 6.49 dan 7.2 tanpa penyesuaian.

#### Perberbedaan Konfigurasi Module 2: Point to Point

**RouterOS 6.49 dan 7.2 - Konfigurasi identik:**
```routeros
# Cara /30 konvensional - sama pada v6 dan v7
# [HQ-R1]
/ip address add address=10.0.12.1/30 interface=ether4-to-R2 network=10.0.12.0
# [R2]
/ip address add address=10.0.12.2/30 interface=ether1-to-HQ network=10.0.12.0

# Cara /32 PtP - sama pada v6 dan v7, field network berisi IP lawan
# [HQ-R1]
/ip address add address=10.0.12.1/32 interface=ether4-to-R2 network=10.0.12.2
# [R2]
/ip address add address=10.0.12.2/32 interface=ether1-to-HQ network=10.0.12.1

# Verifikasi - v6
/ip route print where dst-address in 10.0.12.2
# Verifikasi - v7 (tambahan)
/routing route print where dst-address in 10.0.12.2
```


## 3. Module 3: VPN, VLAN, dan QinQ

VPN menghubungkan dua jaringan melalui jaringan publik seolah-olah mereka satu jaringan privat. VLAN memisahkan jaringan di dalam satu infrastruktur fisik. Keduanya sering digunakan bersamaan: VPN untuk antar site, VLAN untuk antar divisi di dalam site.

### 3.0 Kolom Perbandingan dan Konfigurasi RouterOS v6 dan v7 - Module 3

| Aspek | Konfigurasi RouterOS 6.49 | Konfigurasi RouterOS 7.2 | Perberbedaan Utama |
|---|---|---|---|
| WireGuard | Tidak tersedia, gunakan L2TP/IPsec | `/interface wireguard add name=wg-HQ-BR listen-port=13231` + `/interface wireguard peer add ...` | Fitur baru hanya di v7 |
| OpenVPN | Hanya TCP, `/interface ovpn-server server set enabled=yes` | TCP dan UDP, `/interface ovpn-server server set enabled=yes` + `udp` | v7 mendukung UDP |
| EoIP | `/interface eoip add ... arp=enabled` | `/interface eoip add ...` tanpa `arp` | Parameter `arp` dihapus di v7 |
| PPPoE | Mendukung PAP | PAP dihapus, gunakan CHAP/MSCHAPv2 | Metode autentikasi berubah |
| VLAN | Switch chip: `/interface ethernet switch vlan add ...` | Bridge VLAN filtering: `/interface bridge vlan add ...` + `vlan-filtering=yes` | Model konfigurasi berubah, performa lebih baik di v7 |
| QinQ | `/interface vlan add vlan-id=1000 interface=cvlan-100` | Sama, tetap perlu penyesuaian MTU | Tidak ada perberbedaan sintaks |
| Container | Tidak tersedia | `/interface veth add ...` + container | Fitur baru hanya di v7 |

### 3.1 Perbandingan jenis VPN

| Tunnel | Layer | Enkripsi bawaan | Dapat membawa Layer 2? | Kapan digunakan |
|---|---|---|---|---|
| IPIP | L3 | Tidak | Tidak | Tunnel paling sederhana, cepat, untuk routing antar site tanpa enkripsi |
| EoIP | L2 | Tidak | Ya | Membutuhkan bridge antar site, VLAN sama di dua site |
| PPTP | L2 | MPPE, lemah | Ya, PPP | Legacy, hindari untuk data sensitif |
| SSTP | L2 | TLS / SSL | Ya, PPP | Tembus firewall karena gunakan port 443 |
| L2TP + IPsec | L2 | IPsec | Ya, PPP | Standar aman untuk remote dan site to site |
| PPPoE | L2 | Dapat + IPsec | Ya, PPP | Untuk ISP ke pelanggan, autentikasi per user |

Aturan praktis: jika hanya membutuhkan routing dan kecepatan, gunakan IPIP. Jika membutuhkan satu subnet yang sama di dua site, gunakan EoIP. Jika membutuhkan keamanan, gunakan L2TP + IPsec atau SSTP, jangan PPTP untuk data penting.

### 3.2 Implementasi: IPIP untuk routing antar site

IPIP membungkus paket IP di dalam paket IP lain. Sederhana dan cepat, tetapi tidak terenkripsi. Cocok untuk laboratorium dan untuk jaringan yang sudah aman di bawahnya.

```routeros
# [HQ-R1] IPIP ke BR
/interface ipip add name=ipip-BR local-address=203.0.113.2 remote-address=198.51.100.2
/ip address add address=10.255.1.1/30 interface=ipip-BR network=10.255.1.0
/ip route add dst-address=192.168.20.0/24 gateway=10.255.1.2

# [BR-R3] IPIP ke HQ (kebalikan)
/interface ipip add name=ipip-HQ local-address=198.51.100.2 remote-address=203.0.113.2
/ip address add address=10.255.1.2/30 interface=ipip-BR network=10.255.1.0
/ip route add dst-address=192.168.10.0/24 gateway=10.255.1.1

# Verifikasi
/interface ipip print
/ping 10.255.1.2
/ping 192.168.20.1 src-address=192.168.10.1
```

### 3.3 Implementasi: EoIP untuk bridge antar site

EoIP membungkus frame Ethernet, sehingga dua site dapat merasa satu LAN. Perlu kehati-hatian: broadcast akan ikut terkirim melalui tunnel, sehingga jangan digunakan untuk jaringan besar tanpa filter.

```routeros
# [HQ-R1] EoIP ke BR
/interface eoip add name=eoip-BR tunnel-id=10 local-address=203.0.113.2 remote-address=198.51.100.2
/interface bridge add name=bridge-EoIP
/interface bridge port add bridge=bridge-EoIP interface=eoip-BR
/interface bridge port add bridge=bridge-EoIP interface=ether5-LAN-BR-Extended

# [BR-R3] EoIP ke HQ
/interface eoip add name=eoip-HQ tunnel-id=10 local-address=198.51.100.2 remote-address=203.0.113.2
/interface bridge add name=bridge-EoIP
/interface bridge port add bridge=bridge-EoIP interface=eoip-HQ
/interface bridge port add bridge=bridge-EoIP interface=ether5-LAN-Extended

# Verifikasi: MAC dari BR harus terlihat di bridge HQ
/interface bridge host print where bridge=bridge-EoIP
```

> **Peringatan EoIP:** Jangan bridge EoIP dengan bridge LAN utama tanpa rencana. Loop bridging dapat menyebabkan jaringan tidak berfungsi dalam hitungan detik. Selalu aktifkan **STP / RSTP** di bridge dan batasi broadcast dengan bridge filter jika perlu.

### 3.4 Implementasi: PPTP, SSTP, dan L2TP

Tiga tunnel ini berbasis PPP, sehingga ada konsep server, client, secret, dan profile. Polanya sama, berbeda di keamanan dan port.

```routeros
# --- PPTP Server di HQ (legacy, untuk laboratorium saja) ---
/ppp profile add name=pptp-profile local-address=10.255.2.1 remote-address=10.255.2.10
/ppp secret add name=branch1 password=ChangeMe123 service=pptp profile=pptp-profile
/interface pptp-server server set enabled=yes

# --- SSTP Server di HQ (lebih aman, port 443) ---
/ppp profile add name=sstp-profile local-address=10.255.3.1 remote-address=10.255.3.10
/ppp secret add name=branch1 password=ChangeMe123 service=sstp profile=sstp-profile
/interface sstp-server server set enabled=yes port=443

# --- L2TP + IPsec (rekomendasi untuk produksi) ---
# Di HQ: aktifkan L2TP server dan IPsec
/interface l2tp-server server set enabled=yes use-ipsec=yes ipsec-secret=IPsecSecretKuat123
/ppp profile add name=l2tp-profile local-address=10.255.4.1 remote-address=10.255.4.10
/ppp secret add name=branch1 password=ChangeMe123 service=l2tp profile=l2tp-profile

# Di BR: L2TP Client ke HQ
/interface l2tp-client add name=l2tp-HQ terhubung-to=203.0.113.2 user=branch1 password=ChangeMe123 use-ipsec=yes ipsec-secret=IPsecSecretKuat123 disabled=no
```

Operasi: untuk L2TP + IPsec, pastikan IPsec secret di kedua sisi sama persis, dan firewall mengizinkan UDP 500, 4500, dan protokol IPsec. Jika L2TP terhubung tetapi tidak dapat ping, biasanya masalahnya di IPsec atau di profile yang tidak memiliki IP.

### 3.5 Implementasi: PPPoE Server untuk ISP atau hotspot internal

PPPoE digunakan ISP untuk memberi IP dan autentikasi per pelanggan. Setiap pelanggan dapat sesi PPP sendiri.

```routeros
# [HQ-R1] PPPoE Server
/interface pppoe-server server add service-name=Internet interface=ether3-LAN disabled=no
/ppp profile add name=pppoe-10Mbps local-address=192.168.99.1 remote-address=pppoe-pool rate-limit=10M/10M
/ip pool add name=pppoe-pool ranges=192.168.99.10-192.168.99.100
/ppp secret add name=pelanggan01 password=Pelanggan123 service=pppoe profile=pppoe-10Mbps

# Verifikasi sesi aktif
/ppp active print
/interface pppoe-server print
```

### 3.6 Implementasi: VLAN

VLAN memisahkan broadcast domain. Contoh: Staff, Guest, dan IoT berada di switch yang sama, tetapi tidak dapat saling melihat tanpa melalui router.

```routeros
# [HQ-R1] VLAN di atas bridge LAN
/interface bridge add name=bridge-LAN
/interface bridge port add bridge=bridge-LAN interface=ether3-LAN

# Buat VLAN 10 Staff, 20 Guest, 30 IoT
/interface vlan add name=vlan-10-Staff vlan-id=10 interface=bridge-LAN
/interface vlan add name=vlan-20-Guest vlan-id=20 interface=bridge-LAN
/interface vlan add name=vlan-30-IoT vlan-id=30 interface=bridge-LAN

# Beri IP gateway per VLAN
/ip address add address=192.168.10.1/24 interface=vlan-10-Staff
/ip address add address=192.168.20.1/24 interface=vlan-20-Guest
/ip address add address=192.168.30.1/24 interface=vlan-30-IoT

# Atur VLAN di bridge (v7, penting untuk hardware offload)
/interface bridge vlan add bridge=bridge-LAN tagged=bridge-LAN,ether3-LAN vlan-ids=10,20,30
/interface bridge set bridge-LAN vlan-filtering=yes
```

Jika vlan-filtering diaktifkan tetapi VLAN tidak berjalan, periksa apakah port trunk sudah ditandai tagged dan port access sudah ditandai untagged dengan pvid yang benar. Kesalahan tersebut paling sering di VLAN.

### 3.7 Implementasi: QinQ

QinQ adalah VLAN di dalam VLAN (double tag). ISP menggunakan QinQ untuk membawa VLAN pelanggan (C-VLAN) di dalam VLAN backbone ISP (S-VLAN), supaya VLAN pelanggan tidak bertabrakan dengan VLAN ISP lain.

```routeros
# [ISP Edge] C-VLAN 100 pelanggan dibungkus S-VLAN 1000
# C-VLAN dulu di atas interface fisik
/interface vlan add name=cvlan-100 vlan-id=100 interface=ether1-Customer
# S-VLAN di atas C-VLAN
/interface vlan add name=svlan-1000 vlan-id=1000 interface=cvlan-100

# Di sisi lain, buka bungkusnya dengan urutan terbalik
# S-VLAN 1000 dulu, lalu C-VLAN 100
/interface vlan add name=svlan-1000-rx vlan-id=1000 interface=ether2-Uplink
/interface vlan add name=cvlan-100-rx vlan-id=100 interface=svlan-1000-rx
```

Operasi QinQ: pastikan MTU dinaikkan, karena double tag menambah 8 byte. Set MTU minimal 1526 di interface yang membawa QinQ, atau paket besar akan terpotong.

### 3.8 Implementasi: VLAN dan switch chip di RouterBOARD

RouterBOARD memiliki switch chip yang dapat memproses VLAN di perangkat keras, jauh lebih cepat daripada CPU. Jika VLAN hanya dikonfigurasi di bridge tanpa memanfaatkan switch chip, CPU akan cepat penuh saat trafik besar.

```routeros
# [RouterBOARD] Contoh switch chip VLAN (periksa model, menu dapat berbeda)
# Aktifkan VLAN di switch
/interface ethernet switch set switch1 vlan-mode=secure

# VLAN 10 untuk ether2-ether3, VLAN 20 untuk ether4
/interface ethernet switch vlan add switch=switch1 vlan-id=10 ports=ether2,ether3,switch1-cpu
/interface ethernet switch vlan add switch=switch1 vlan-id=20 ports=ether4,switch1-cpu

# Atur PVID untuk port access
/interface ethernet switch port set ether2 vlan-mode=secure default-vlan-id=10
/interface ethernet switch port set ether4 vlan-mode=secure default-vlan-id=20
```

Catatan: di RouterOS v7 dan perangkat baru, cara yang disarankan adalah gunakan bridge VLAN filtering (sub-bab 3.6) karena sudah terintegrasi dengan perangkat keras offload. Cara switch chip lama masih ada untuk model tertentu, tetapi jangan campur keduanya di perangkat yang sama.

### 3.9 Studi kasus Module 3: Kantor cabang terhubung aman dengan VLAN yang sama

**Situasi:** HQ dan BR harus merasa satu jaringan untuk aplikasi lama yang hanya berjalan di satu subnet (192.168.10.0/24). Tetapi koneksi harus terenkripsi, dan di HQ ada pemisahan Staff dan Guest.

**Desain:** Gunakan L2TP + IPsec untuk enkripsi, EoIP di atasnya untuk bridge Layer 2, dan VLAN untuk pemisahan Staff dan Guest. Hal tersebut merupakan pola bertingkat: IPsec mengamankan, L2TP membawa PPP, EoIP membawa Ethernet, VLAN memisahkan.

```routeros
# Langkah 1: L2TP + IPsec HQ-BR (lihat 3.4)
# Langkah 2: EoIP di atas IP L2TP (local-address = IP tunnel L2TP, bukan IP publik)
# [HQ] local 10.255.4.1, remote 10.255.4.2
/interface eoip add name=eoip-BR-Secure tunnel-id=20 local-address=10.255.4.1 remote-address=10.255.4.2

# Langkah 3: Bridge EoIP dengan VLAN Staff di HQ
/interface bridge add name=bridge-BR
/interface bridge port add bridge=bridge-BR interface=eoip-BR-Secure
/interface bridge port add bridge=bridge-BR interface=vlan-10-Staff

# Langkah 4: Verifikasi dari BR
# Ping dari PC BR ke gateway Staff HQ harus berhasil
/ping 192.168.10.1 src-address=192.168.10.50
```

**Hasil yang diharapkan:** PC di BR dapat ping gateway HQ, aplikasi lama berjalan, dan trafik Guest di BR tidak dapat masuk ke VLAN Staff tanpa melalui firewall. Jika ping berhasil tetapi aplikasi lambat, periksa MTU tunnel: EoIP + IPsec menambah overhead, turunkan MTU atau aktifkan MSS clamping.

```routeros
# MSS clamping untuk tunnel (menghindari paket terpotong)
/ip firewall mangle add chain=forward protocol=tcp tcp-flags=syn action=change-mss new-mss=1380 passthrough=yes comment="MSS for VPN"
```

### 3.10 Verifikasi dan operasi Module 3

```routeros
# Periksa tunnel aktif
/interface print where type~"ipip|eoip|l2tp|sstp|pptp|pppoe"
/ppp active print

# Periksa VLAN
/interface vlan print
/interface bridge vlan print
/interface bridge host print

# Periksa throughput tunnel (perlu diperhatikan bahwa jika lebih rendah dari link fisik)
/tool bandwidth-test 10.255.1.2 direction=both duration=10s
```

- **Operasi:** pantau jumlah sesi PPP aktif. Lonjakan sesi yang tidak wajar dapat berarti ada brute force. Aktifkan logging untuk PPP dan batasi masuk gagal.
- **Troubleshooting:** tunnel aktif tetapi tidak dapat ping: periksa IP tunnel, route, dan firewall di chain input dan forward. Tunnel tidak mau aktif: periksa local-address dan remote-address tertukar atau tidak, dan periksa apakah IP publik berubah (DDNS). VLAN tidak berjalan: periksa vlan-filtering dan tagged/untagged.
- **Keamanan:** jangan gunakan PPTP untuk data sensitif. Ganti kata sandi PPP secara berkala dan jangan gunakan kata sandi yang sama untuk IPsec secret dan PPP secret.


### 3.11 Perbandingan RouterOS 6.49 dan 7.2 untuk Module 3

| Aspek | RouterOS 6.49 | RouterOS 7.2 |
|---|---|---|
| WireGuard | Tidak tersedia. | Tersedia sebagai interface bawaan dengan konfigurasi sederhana dan performa tinggi. |
| OpenVPN | Hanya TCP, UDP tidak didukung secara stabil. | Mendukung TCP dan UDP. |
| L2TP/IPsec | Tersedia, konfigurasi standar. | Tersedia, tetapi perlu pengujian ulang setelah migrasi karena terdapat laporan masalah kompatibilitas dengan klien tertentu pada rilis awal 7.2. |
| EoIP dan IPIP | Tersedia, parameter `arp` masih ada. | Tersedia, parameter `arp` dihapus dan terdapat opsi baru. |
| PPPoE | Mendukung PAP. | PAP dihapus, hanya metode autentikasi yang lebih aman yang didukung. |
| VLAN dan bridge | VLAN filtering tersedia, tetapi perangkat keras offload lebih terbatas. | VLAN filtering dengan FastPath/FastTrack pada bridge meningkatkan performa inter-VLAN, terutama pada RB5009. |
| QinQ | Tersedia dengan penambahan MTU manual. | Tersedia dengan perilaku yang sama, tetap perlu penyesuaian MTU. |
| Container dan veth | Tidak tersedia. | Tersedia untuk menjalankan container dan interface virtual. |
| DNS | DNS statis dan cache standar. | Menambah DNS over HTTPS (DoH) dan dukungan regexp. |

**Implikasi:** Apabila WireGuard atau container dibutuhkan, wajib menggunakan 7.2. Apabila hanya menggunakan EoIP/IPIP dan VLAN sederhana, konfigurasi 6.49 dapat dipindahkan dengan penyesuaian minor pada parameter yang dihapus.

#### Perberbedaan Konfigurasi Module 3: WireGuard

**RouterOS 6.49 - Tidak tersedia:**
```routeros
# WireGuard tidak tersedia pada v6. Gunakan L2TP/IPsec sebagai alternatif
/interface l2tp-client add name=l2tp-HQ terhubung-to=203.0.113.2 user=branch1 password=ChangeMe123 use-ipsec=yes ipsec-secret=IPsecSecretKuat123 disabled=no
```

**RouterOS 7.2 - WireGuard tersedia:**
```routeros
# v7: WireGuard bawaan
/interface wireguard add name=wg-HQ-BR listen-port=13231 private-key="PRIVATE-KEY-HQ" mtu=1420
/interface wireguard peer add interface=wg-HQ-BR public-key="PUBLIC-KEY-BR" endpoint-address=198.51.100.2 endpoint-port=13231 allowed-address=10.255.1.0/30,192.168.20.0/24
/ip address add address=10.255.1.1/30 interface=wg-HQ-BR network=10.255.1.0
/ip route add dst-address=192.168.20.0/24 gateway=10.255.1.2
```

#### Perberbedaan Konfigurasi Module 3: VLAN - Switch Chip vs Bridge VLAN Filtering

**RouterOS 6.49 - Switch chip (model tertentu):**
```routeros
# v6: konfigurasi switch chip lama
/interface ethernet switch set switch1 vlan-mode=secure
/interface ethernet switch vlan add switch=switch1 vlan-id=10 ports=ether2,ether3,switch1-cpu
/interface ethernet switch port set ether2 vlan-mode=secure default-vlan-id=10
```

**RouterOS 7.2 - Bridge VLAN filtering (disarankan):**
```routeros
# v7: gunakan bridge VLAN filtering agar hardware offload tetap jalan
/interface bridge add name=bridge-LAN vlan-filtering=yes
/interface bridge port add bridge=bridge-LAN interface=ether3-LAN
/interface vlan add name=vlan-10-Staff vlan-id=10 interface=bridge-LAN
/interface bridge vlan add bridge=bridge-LAN tagged=bridge-LAN,ether3-LAN vlan-ids=10
# Performa inter-VLAN lebih baik karena FastPath/FastTrack pada bridge vlan-filtering
```

#### Perberbedaan Konfigurasi Module 3: EoIP dan PPPoE

**RouterOS 6.49 - EoIP:**
```routeros
/interface eoip add name=eoip-BR tunnel-id=10 local-address=203.0.113.2 remote-address=198.51.100.2 arp=enabled
```

**RouterOS 7.2 - EoIP:**
```routeros
# Parameter arp dihapus pada v7
/interface eoip add name=eoip-BR tunnel-id=10 local-address=203.0.113.2 remote-address=198.51.100.2
```

**RouterOS 6.49 - PPPoE (PAP masih didukung):**
```routeros
/ppp secret add name=pelanggan01 password=Pelanggan123 service=pppoe profile=pppoe-10Mbps
# PAP dapat digunakan
```

**RouterOS 7.2 - PPPoE (PAP dihapus):**
```routeros
# PAP dihapus, gunakan CHAP/MSCHAPv2
/ppp secret add name=pelanggan01 password=Pelanggan123 service=pppoe profile=pppoe-10Mbps
# Pastikan klien tidak lagi menggunakan PAP, jika tidak autentikasi akan gagal
```


## 4. Module 4: OSPF

OSPF adalah routing dinamis yang membuat router saling bertukar informasi topologi, lalu masing-masing menghitung jalur terbaik sendiri. Kelebihannya: jika ada link putus, router otomatis mencari jalur lain tanpa harus diubah manual. Kekurangannya: lebih kompleks dan harus dirancang area-nya dengan benar.

### 4.0 Kolom Perbandingan dan Konfigurasi RouterOS v6 dan v7 - Module 4

| Aspek | Konfigurasi RouterOS 6.49 | Konfigurasi RouterOS 7.2 | Perberbedaan Utama |
|---|---|---|---|
| OSPF instance | `/routing ospf instance set router-id=1.1.1.1` | `/routing ospf instance add name=ospf-1 version=2 router-id=1.1.1.1` | v7 wajib membuat instance, tidak ada default |
| OSPF area | Area tersirat dari `network` | `/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf-1` | Area eksplisit di v7 |
| OSPF interface | `/routing ospf network add network=10.0.12.0/30 area=backbone` | `/routing ospf interface-template add networks=10.0.12.0/30 area=backbone` | Konsep template baru di v7 |
| OSPF filter | `/routing filter add chain=ospf-out prefix=... action=reject` | `/routing filter rule add chain=ospf-out rule="if (dst == ...) { reject; }"` | Sintaks filter berbasis skrip |
| BGP peer | `/routing bgp peer add remote-address=... remote-as=...` | `/routing bgp terhubungion add remote.address=... remote.as=... local.role=ebgp` | Menu `terhubungion` + `template` + `session` |
| BGP filter | Filter lama berbasis firewall | `output.filter-chain` + `input.filter` + `accept-nlri` | Filter lebih granular, hemat memori |
| Pemantauing | `/routing ospf neighbor print` | Sama + `/routing stats process print` | v7 menambah pemantauan proses routing |

### 4.1 Konsep yang harus dikuasai

- **Hello protocol:** Router mengirim hello setiap 10 detik (di broadcast) untuk menemukan neighbor dan memastikan neighbor masih hidup. Jika hello tidak diterima dalam dead interval (40 detik), neighbor dianggap mati.
- **Database dan LSA:** Setiap router memiliki Link State Database (LSDB) yang berisi LSA. LSA tipe 1 untuk router, tipe 2 untuk network, tipe 3 untuk summary antar area, tipe 5 untuk eksternal. Semua router di satu area harus memiliki LSDB yang sama.
- **Areas:** Area 0 adalah backbone, wajib ada dan semua area lain harus terhubung ke Area 0, langsung atau melalui virtual link. Pembagian area membuat LSDB lebih kecil dan jaringan lebih stabil.
- **Router types:** Internal Router (di dalam satu area), Area Border Router (ABR, menghubungkan dua area), Backbone Router, AS Boundary Router (ASBR, menghubungkan OSPF ke routing lain).
- **DR dan BDR:** Di jaringan broadcast, router memilih satu DR dan satu BDR untuk mengurangi jumlah adjacency. Router lain hanya bertukar database dengan DR/BDR, bukan dengan semua router.
- **SPT / Dijkstra:** Setiap router menjalankan algoritma Shortest Path First untuk menghitung jalur terpendek dari dirinya ke semua network, berdasarkan cost. Cost dihitung dari bandwidth: semakin besar bandwidth, semakin kecil cost.

### 4.2 Implementasi dasar: OSPF single area

Mulailah dari konfigurasi yang paling sederhana: semua router di Area 0. Langkah tersebut bertujuan untuk memastikan hello, neighbor, dan distribusi route berjalan.

```routeros
# [Semua router] v7: Buat instance OSPF
/routing ospf instance add name=ospf-1 version=2 router-id=1.1.1.1
# Ganti router-id per router: R2=2.2.2.2, BR=3.3.3.3

# [Semua router] Buat Area 0
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf-1

# [HQ-R1] Daftarkan network yang ikut OSPF
/routing ospf interface-template add networks=10.0.12.0/30 area=backbone type=broadcast
/routing ospf interface-template add networks=192.168.10.0/24 area=backbone type=broadcast

# [R2] Daftarkan network-nya
/routing ospf interface-template add networks=10.0.12.0/30 area=backbone
/routing ospf interface-template add networks=10.0.23.0/30 area=backbone

# [BR-R3] Daftarkan network-nya
/routing ospf interface-template add networks=10.0.23.0/30 area=backbone
/routing ospf interface-template add networks=192.168.20.0/24 area=backbone

# Verifikasi
/routing ospf neighbor print
/routing ospf lsa print
/ip route print where ospf
```

**Catatan v6:** di v6, sintaksnya `/routing ospf network add network=... area=backbone` dan `/routing ospf instance set router-id=...`. Jangan campur sintaks v6 dan v7.

### 4.3 Implementasi: Multi-area

Setelah single area berjalan, pecah menjadi Area 0 dan Area 1. HQ dan R2 di Area 0, R2 dan BR di Area 1. R2 menjadi ABR karena berada di dua area.

```routeros
# [R2] Tambah Area 1
/routing ospf area add name=area-1 area-id=0.0.0.1 instance=ospf-1

# [R2] Pindahkan link ke BR ke Area 1
/routing ospf interface-template add networks=10.0.23.0/30 area=area-1

# [BR-R3] Pindahkan network-nya ke Area 1
/routing ospf area add name=area-1 area-id=0.0.0.1 instance=ospf-1
/routing ospf interface-template add networks=10.0.23.0/30 area=area-1
/routing ospf interface-template add networks=192.168.20.0/24 area=area-1

# Verifikasi: HQ harus tetap dapat ping BR melalui summary LSA
/routing ospf neighbor print
/ip route print where dst-address in 192.168.20.0/24
```

### 4.4 Implementasi: Interface cost dan tipe interface

Cost menentukan jalur mana yang dipilih. Jika ada dua jalur ke BR, jalur dengan total cost lebih kecil akan menang, walau hop-nya lebih banyak.

```routeros
# [HQ-R1] Ubah cost interface ke R2 menjadi 10 (default 10 untuk 100Mbps, 1 untuk 1Gbps)
# Di v7, cost diatur di interface-template
/routing ospf interface-template set [find networks=10.0.12.0/30] cost=10

# Tipe interface: broadcast untuk Ethernet, ptp untuk link point to point
/routing ospf interface-template add networks=10.0.12.0/30 area=backbone type=ptp

# NBMA: untuk jaringan seperti Frame Relay, harus tentukan neighbor manual
# Contoh NBMA (jarang di laboratorium modern, tetapi keluar di ujian)
/routing ospf nbma-neighbor add address=10.0.12.2 instance=ospf-1
```

Operasi: jangan ubah cost secara acak. Dokumentasikan mengapa cost diubah, misalnya karena link utama 1 Gbps dan link cadangan 10 Mbps. Cost yang salah akan membuat trafik melalui jalur lambat.

### 4.5 Implementasi: Neighbor states dan DR/BDR

Neighbor OSPF memelaluii tahapan: Down, Init, 2-Way, ExStart, Exchange, Loading, Full. Hanya neighbor yang Full yang benar-benar bertukar database.

```routeros
# Periksa neighbor dan state-nya
/routing ospf neighbor print detail

# Contoh output yang sehat:
# neighbor 2.2.2.2 state=Full, DR dan BDR terpilih
# Jika state stuck di ExStart atau Exchange, periksa MTU kedua sisi harus sama
/interface print where name=ether4-to-R2

# Periksa siapa DR dan BDR
/routing ospf neighbor print where dr-address!=""
```

Untuk mengontrol siapa yang sehingga DR, ubah priority. Priority 0 artinya tidak akan pernah sehingga DR. Priority lebih tinggi lebih diprioritaskan.

```routeros
# [HQ-R1] Jadikan HQ sebagai DR dengan priority tinggi
/routing ospf interface-template set [find networks=10.0.12.0/30] priority=100
# [R2] Jadikan R2 tidak sehingga DR
/routing ospf interface-template set [find networks=10.0.12.0/30] priority=0
```

### 4.6 Implementasi: External route distribution Type 1 dan Type 2

Jika HQ memiliki route static ke internet atau ke jaringan lain yang ingin disebarkan ke OSPF, route tersebut harus di-redistribute sebagai external. Type 1 dan Type 2 berbeda cara menghitung cost-nya.

```routeros
# [HQ-R1] Sebarkan route static sebagai external
# Buat filter untuk memilih route yang boleh disebar
/routing filter rule add chain=ospf-out rule="if (dst == 0.0.0.0/0) { set ospf-metric-type=type-1; accept; }"

# Atau di v6, gunakan redistribute di instance
# /routing ospf instance set redistribute-static=as-type-1

# Verifikasi di BR: route 0.0.0.0/0 harus muncul sebagai OSPF external
/ip route print where dst-address=0.0.0.0/0
```

- **Type 1:** Cost = cost external + cost internal untuk mencapai ASBR. Lebih akurat, karena memperhitungkan jarak ke ASBR.
- **Type 2:** Cost = hanya cost external, cost internal diabaikan. Lebih sederhana, dan ini adalah default. Jika ada dua ASBR yang menyebarkan route yang sama, Type 2 akan memilih yang cost external-nya lebih kecil, walau jaraknya lebih jauh.

### 4.7 Implementasi: Stub, NSSA, dan agregasi area

Stub area tidak menerima LSA Type 5 (external). NSSA mirip stub, tetapi masih boleh menerima external dari dalam area itu sendiri melalui Type 7 yang diubah sehingga Type 5 oleh ABR. Agregasi (area range) membuat ABR menyebarkan satu summary, bukan banyak route kecil.

```routeros
# [BR-R3] Jadikan Area 1 sebagai Stub (tidak membutuhkan route external lengkap)
/routing ospf area set [find name=area-1] type=stub

# [BR-R3] Atau sebagai NSSA jika BR memiliki route external sendiri
/routing ospf area set [find name=area-1] type=nssa

# [R2 sebagai ABR] Agregasi: sebarkan 192.168.20.0/24 sebagai satu summary
# Di ABR, buat area range untuk Area 1
/routing ospf area-range add area=area-1 address=192.168.20.0/24 advertise=yes

# Verifikasi: di HQ, route ke BR harus muncul sebagai satu summary, bukan banyak /32
/ip route print where dst-address in 192.168.20.0/24
```

Operasi: stub dan NSSA mengecilkan tabel routing dan membuat router di area kecil lebih ringan. Tetapi jika route external lengkap dibutuhkan di area tersebut, stub tidak dapat dijadikan pilihan. Selalu periksa setelah mengubah tipe area, karena neighbor dapat turun dan naik lagi.

### 4.8 Implementasi: Virtual link

Virtual link digunakan saat ada area yang tidak terhubung langsung ke Area 0. Misalnya Area 2 hanya terhubung ke Area 1, tidak ke Area 0. Virtual link membuat Area 1 menjadi transit untuk menghubungkan Area 2 ke Area 0.

```routeros
# [R2 ABR] Virtual link ke BR yang berada di Area 1, untuk menghubungkan Area 2
# Virtual link selalu dibuat di dua ABR yang mengapit area transit
/routing ospf virtual-link add neighbor-id=3.3.3.3 transit-area=area-1 instance=ospf-1

# [BR-R3] Kebalikannya
/routing ospf virtual-link add neighbor-id=2.2.2.2 transit-area=area-1 instance=ospf-1

# Verifikasi
/routing ospf virtual-link print
/routing ospf neighbor print where virtual-link
```

Batasan: virtual link tidak boleh memelaluii stub area, dan sebaiknya hanya digunakan untuk sementara. Solusi permanen adalah memperbaiki topologi agar semua area terhubung langsung ke Area 0.

### 4.9 Implementasi: OSPF routing filters

Filter OSPF mengontrol route mana yang boleh masuk atau keluar. Filter yang salah dapat membuat jaringan terputus, sehingga selalu uji di laboratorium terlebih dahulu.

```routeros
# Contoh: Tolak route 192.168.99.0/24 agar tidak disebar ke OSPF
/routing filter rule add chain=ospf-out rule="if (dst == 192.168.99.0/24) { reject; }"

# Contoh: Hanya terima route dari Area 0 yang prefix-nya /24
/routing filter rule add chain=ospf-in rule="if (dst-len == 24) { accept; } else { reject; }"

# Terapkan filter di instance
/routing ospf instance set ospf-1 out-filter-chain=ospf-out in-filter-chain=ospf-in
```

### 4.10 Studi kasus Module 4: OSPF multi-area dengan failover otomatis

**Situasi:** HQ, R2, dan BR terhubung dengan dua jalur: jalur utama HQ-R2-BR (cepat, 1 Gbps) dan jalur cadangan HQ-BR langsung (lambat, 10 Mbps, via tunnel). Manajemen minta: jika jalur utama putus, trafik pindah otomatis ke jalur cadangan dalam hitungan detik, tanpa konfigurasi manual.

**Desain:** Gunakan OSPF multi-area. HQ dan R2 di Area 0, R2 dan BR di Area 1, plus link cadangan HQ-BR juga di Area 0 tetapi dengan cost tinggi. OSPF akan memilih jalur utama karena cost-nya lebih kecil, dan otomatis pindah saat jalur utama mati.

```routeros
# Langkah 1: Buat OSPF seperti di 4.2 dan 4.3 (Area 0 dan Area 1)

# Langkah 2: Tambah link cadangan HQ-BR langsung
# [HQ] Link cadangan 10.0.13.0/30, cost 100 (mahal, sehingga hanya digunakan saat utama mati)
/ip address add address=10.0.13.1/30 interface=ether5-to-BR-Backup network=10.0.13.0
/routing ospf interface-template add networks=10.0.13.0/30 area=backbone cost=100

# [BR] Link cadangan
/ip address add address=10.0.13.2/30 interface=ether3-to-HQ-Backup network=10.0.13.0
/routing ospf interface-template add networks=10.0.13.0/30 area=backbone cost=100

# Langkah 3: Verifikasi jalur utama dipakai
/ip route print where dst-address in 192.168.20.0/24
# Gateway harus melalui 10.0.12.2 (R2), bukan 10.0.13.2

# Langkah 4: Uji failover
# Matikan ether4-to-R2 di HQ, lalu periksa dalam 10-40 detik
/interface disable ether4-to-R2
/ip route print where dst-address in 192.168.20.0/24
/ping 192.168.20.1
# Gateway harus pindah ke 10.0.13.2

# Langkah 5: Hidupkan lagi dan pastikan balik ke utama
/interface enable ether4-to-R2
```

**Hasil yang diharapkan:** Failover terjadi tanpa intervensi manual. Jika failover lebih dari 40 detik, periksa hello dan dead interval. Di link yang kritis, Hello dapat diturunkan ke 2 detik dan dead ke 8 detik agar failover lebih cepat, tetapi jangan terlalu kecil karena dapat menyebabkan flapping.

```routeros
# Contoh tuning hello untuk failover lebih cepat (di link kritis saja)
/routing ospf interface-template set [find networks=10.0.12.0/30] hello-interval=2s dead-interval=8s
```

### 4.11 Verifikasi dan operasi Module 4

```routeros
# Periksa neighbor dan state
/routing ospf neighbor print
/routing ospf neighbor print detail

# Periksa database dan LSA
/routing ospf lsa print
/routing ospf lsa print where type=router

# Periksa route OSPF
/ip route print where ospf

# Periksa log OSPF
/log print where topics~"ospf"
```

- **Operasi harian:** periksa neighbor Full untuk semua tetangga yang seharusnya ada. Jika ada neighbor yang hilang, itu tanda link atau hello bermasalah.
- **Troubleshooting:** neighbor stuck di Init: periksa apakah hello diterima, periksa firewall dan MTU. Stuck di ExStart: periksa MTU harus sama di kedua sisi. Stuck di 2-Way: periksa apakah salah satu router tidak menganggap yang lain sebagai neighbor karena priority atau network type. Database tidak sama: periksa apakah ada LSA yang tertahan karena filter.
- **Perubahan yang berbahaya:** mengubah router-id, area-id, atau tipe area akan membuat OSPF mulai ulang. Lakukan di jam sepi dan beri tahu tim. Jangan mengubah router-id tanpa mulai ulang OSPF.
- **Pemantauan:** pantau jumlah LSA dan jumlah route OSPF. Lonjakan LSA yang tiba-tiba dapat berarti ada loop atau router yang flapping.


### 4.12 Perbandingan RouterOS 6.49 dan 7.2 untuk Module 4

| Aspek | RouterOS 6.49 | RouterOS 7.2 |
|---|---|---|
| Menu OSPF | OSPFv2 dan OSPFv3 terpisah, menggunakan `/routing ospf network` untuk mendaftarkan network. | OSPFv2 dan OSPFv3 digabung dalam satu menu `/routing ospf`, menggunakan `instance`, `area`, dan `interface-template`. |
| Instance dan area | Instance dibuat dengan router-id, area ditambahkan dengan cara yang sama tetapi tanpa template. | Wajib membuat instance terlebih dahulu, kemudian area, kemudian interface-template. Tidak terdapat instance dan area default. |
| Neighbor dan interface | Status pada menu `interface` dan `neighbor` yang dapat dikonfigurasi. | Menu `interface` dan `neighbor` hanya bersifat read-only untuk pemantauan, konfigurasi dilakukan melalui `interface-template`. |
| Filter routing | Menggunakan filter lama berbasis firewall. | Menggunakan `/routing filter rule` dengan sintaks berbasis skrip yang berlaku untuk seluruh protokol routing. |
| Redistribusi | Menggunakan knob `redistribute-*` pada instance. | Menggunakan `redistribute` pada instance ditambah filter chain untuk mengatur cost dan tipe, dengan aksi default `reject`. |
| BGP | Menggunakan `instance` dan `peer`, single-thread, lambat untuk tabel penuh. | Desain ulang total dengan `terhubungion`, `template`, dan `session`, multi-thread, mampu menangani tabel penuh dengan lebih cepat. |
| Kinerja | Konvergensi lebih lambat pada tabel besar. | Konvergensi lebih cepat dan proses routing terpisah per protokol pada `/routing stats`. |

**Implikasi:** Migrasi OSPF dari 6.49 ke 7.2 bukan sekadar salin-tempel. Konfigurasi OSPF wajib ditulis ulang menggunakan instance, area, dan interface-template. Filter dan redistribusi juga harus ditulis ulang menggunakan sintaks filter baru.

#### Perberbedaan Konfigurasi Module 4: OSPF Single Area

**RouterOS 6.49 - OSPF:**
```routeros
# v6: OSPFv2 terpisah, network langsung didaftarkan
/routing ospf instance set router-id=1.1.1.1
/routing ospf network add network=10.0.12.0/30 area=backbone
/routing ospf network add network=192.168.10.0/24 area=backbone
# Periksa neighbor
/routing ospf neighbor print
```

**RouterOS 7.2 - OSPF:**
```routeros
# v7: wajib instance + area + interface-template, tidak ada instance default
/routing ospf instance add name=ospf-1 version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf-1
/routing ospf interface-template add networks=10.0.12.0/30 area=backbone type=broadcast
/routing ospf interface-template add networks=192.168.10.0/24 area=backbone type=broadcast
# Periksa neighbor (interface dan neighbor kini read-only untuk pemantauan)
/routing ospf neighbor print
/routing ospf interface print
```

#### Perberbedaan Konfigurasi Module 4: OSPF Filter dan Redistribusi

**RouterOS 6.49 - Filter lama:**
```routeros
# v6: filter berbasis firewall, redistribute via knob
/routing ospf instance set redistribute-static=as-type-1
/routing filter add chain=ospf-out prefix=192.168.99.0/24 action=reject
```

**RouterOS 7.2 - Filter baru:**
```routeros
# v7: filter berbasis skrip, aksi default reject
/routing ospf instance set ospf-1 out-filter-chain=ospf-out redistribute=static
/routing filter rule add chain=ospf-out rule="if (dst == 192.168.99.0/24) { reject; }"
/routing filter rule add chain=ospf-out rule="if (dst in 192.168.0.0/16) { set ospf-metric-type=type-1; accept; }"
```

#### Perberbedaan Konfigurasi Module 4: BGP

**RouterOS 6.49 - BGP:**
```routeros
# v6: instance + peer
/routing bgp instance set as=65500 router-id=1.1.1.1
/routing bgp peer add remote-address=10.0.12.2 remote-as=65501 name=peer-BR
# Filter lama
/routing filter add chain=bgp-out prefix=192.168.10.0/24 action=accept
```

**RouterOS 7.2 - BGP:**
```routeros
# v7: terhubungion + template + session, multi-thread
/routing bgp template set default as=65500
/routing bgp terhubungion add remote.address=10.0.12.2 remote.as=65501 local.role=ebgp template=default
# Filter baru dengan input.accept dan output.filter-chain
/routing bgp template set default output.filter-chain=bgp-out input.filter=bgp-in
/routing filter rule add chain=bgp-out rule="if (dst in 192.168.10.0/24) { accept; }"
# Pemantauan proses routing
/routing stats process print
```


## 5. Appendix A: Cheat Sheet Perintah Operasi

| Kebutuhan | Perintah | Apa yang dilihat |
|---|---|---|
| Periksa route aktif | `/ip route print` | Route dengan tanda A, gateway, distance |
| Periksa route ke IP tertentu | `/ip route print where dst-address in 192.168.20.10` | Route mana yang dipilih untuk IP itu |
| Periksa neighbor OSPF | `/routing ospf neighbor print` | State harus Full, DR/BDR |
| Periksa LSA OSPF | `/routing ospf lsa print` | Database harus sama di satu area |
| Periksa tunnel | `/interface print where type~"eoip\|ipip\|l2tp"` | Running, MTU, traffic |
| Periksa PPP aktif | `/ppp active print` | Siapa yang masuk, sejak kapan |
| Periksa VLAN | `/interface vlan print` | VLAN ID dan interface induk |
| Periksa mangle | `/ip firewall mangle print` | Counter naik atau tidak |
| Periksa log routing | `/log print where topics~"route\|ospf"` | Perubahan route dan neighbor |
| Export cadangan | `/export file=backup-hari-ini` | berkas cadangan di Files |


## 6. Appendix B: Matriks Troubleshooting Cepat

| Gejala | Kemungkinan penyebab | Periksa pertama |
|---|---|---|
| Ping ke gateway gagal | IP salah, kabel, interface tidak aktif | `/ip address print`, `/interface print` |
| Ping gateway OK, ping internet gagal | Route default hilang, NAT hilang, recursive gateway mati | `/ip route print`, `/ip firewall nat print` |
| Failover tidak pindah | Check-gateway tidak berjalan, distance sama, scope salah | `/ip route print`, periksa scope/target-scope |
| Routing-mark tidak berjalan | Mangle salah, tabel tidak ada, rule tidak ada | `/ip firewall mangle print`, `/routing table print` |
| Tunnel aktif tetapi tidak dapat ping | IP tunnel salah, route hilang, firewall blokir | `/interface print`, `/ip route print` |
| L2TP tidak terhubung | IPsec secret salah, port UDP blokir | `/log print`, periksa firewall UDP 500/4500 |
| VLAN tidak berjalan | VLAN filtering salah, tagged/untagged tertukar | `/interface bridge vlan print` |
| OSPF neighbor tidak muncul | Hello tidak diterima, MTU berbeda, firewall blokir | `/routing ospf neighbor print`, `/log print` |
| OSPF stuck ExStart | MTU tidak sama | `/interface print`, samakan MTU |
| OSPF route hilang setelah ubah area | Filter salah, area type salah, ABR tidak ada | `/routing filter rule print`, `/routing ospf area print` |

> **Penutup:** Panduan ini sengaja disusun agar tidak hanya menghafal perintah, tetapi memahami kapan menggunakan static vs OSPF, kapan menggunakan IPIP vs EoIP vs L2TP, dan bagaimana mengoperasikannya setelah go-live. Apabila terdapat satu hal yang perlu diingat: **verifikasi setelah setiap perubahan**. Jaringan yang baik bukan jaringan yang konfigurasinya panjang, tetapi yang dapat dibuktikan berjalan dengan ping, traceroute, dan print route.


## 7. Appendix C: Ringkasan Perberbedaan Konfigurasi RouterOS 6.49 dan 7.2

| Modul | Konfigurasi v6 | Konfigurasi v7 | Tindakan Migrasi |
|---|---|---|---|
| Module 1 - Static Routing | `routing-mark` langsung pada route, `/ip route rule` | Buat `/routing table` dahulu, `/routing rule`, `/routing route` untuk pemantauan | Tulis ulang policy routing, buat tabel baru |
| Module 2 - PtP | `/ip address` sama | `/ip address` sama | Tidak ada perubahan, dapat disalin langsung |
| Module 3 - VPN | Tanpa WireGuard, EoIP dengan `arp`, PPPoE dengan PAP | WireGuard bawaan, EoIP tanpa `arp`, PPPoE tanpa PAP | Ganti VPN ke WireGuard bila perlu, periksa parameter yang dihapus |
| Module 3 - VLAN | Switch chip lama pada model tertentu | Bridge VLAN filtering disarankan, FastPath/FastTrack | Pindahkan ke bridge VLAN filtering |
| Module 4 - OSPF | `/routing ospf network`, filter lama | `instance` + `area` + `interface-template`, filter baru | Tulis ulang OSPF dan filter |
| Module 4 - BGP | `instance` + `peer`, single-thread | `terhubungion` + `template` + `session`, multi-thread | Tulis ulang BGP dan filter |
