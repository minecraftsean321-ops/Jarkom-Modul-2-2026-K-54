
# Jarkom-Modul-2-2026-K-54  

| Nama                 | NRP        |
|----------------------|------------|
|  Viko Rizky Fauzan | 5027251017   |
| Sean Arthur Tamajaya | 5027251050 |

# Laporan Praktikum: Konfigurasi IP & Routing "The Mesh"

**Proyek GNS3:** `K-54-MODUL-2`
**Prefix IP kelompok:** `192.238.0.0`
**OS node:** Alpine Linux

## Soal 1

Sebagai pusat kesadaran *The Mesh*, `rootkit` merentangkan koneksinya ke lima gerbang utama (Switch). Tugasnya adalah menetapkan alamat IP dan *default gateway* untuk seluruh Entitas sesuai topologi pembagian switch:

| Peran | Node |
|---|---|
| Operator | `alpha`, `beta`, `gamma` |
| Penjaga *directory* | `prab`, `tedd` |
| Gerbang penyaring | `abbey`, `penny` |
| *Repository* | `obladi`, `desmond`, `oblada`, `molly` |
| Klien tambahan | `delta`, `epsilon` |
| Router pusat | `rootkit` |

## 2. Topologi

<img width="1470" height="835" alt="Screenshot 2026-09-29 at 16 33 07" src="https://github.com/user-attachments/assets/e846f3a9-bad4-4462-b645-0ea52a42cdd5" />


## 3. Rencana Pengalamatan

| Interface `rootkit` | Terhubung ke | Subnet | IP `rootkit` (gateway) | Node di subnet |
|---|---|---|---|---|
| `eth0` | NAT1 | DHCP | dari DHCP | (jalur internet) |
| `eth1` | Switch1 | `192.238.1.0/24` | `192.238.1.1` | `prab`, `tedd`, `obladi`, `desmond`, `oblada`, `molly` |
| `eth2` | Switch4 | `192.238.2.0/24` | `192.238.2.1` | `abbey` |
| `eth3` | Switch5 | `192.238.3.0/24` | `192.238.3.1` | `penny` |
| `eth4` | Switch6 | `192.238.4.0/24` | `192.238.4.1` | `alpha`, `beta`, `gamma` |
| `eth5` | Switch7 | `192.238.5.0/24` | `192.238.5.1` | `delta`, `epsilon` |

Contoh IP yang sudah diuji pada hasil pengerjaan:

| Node | IP | Gateway |
|---|---|---|
| `alpha` | `192.238.4.2` | `192.238.4.1` |
| `delta` | `192.238.5.2` | `192.238.5.1` |
| `prab` | `192.238.1.2` | `192.238.1.1` |

> Node lain mengikuti pola yang sama: ubah `address` (nomor host, mis. `.3`, `.4`, dst.) dan `gateway` sesuai subnet-nya. Nomor host node lain di atas hanyalah usulan, sesuaikan dengan konfigurasimu.

## 4. Langkah Pengerjaan

### Langkah 1 - Buka konsol node
Jalankan semua node di GNS3 (tombol ▶), lalu buka konsol masing-masing node (`rootkit`, `alpha`, `prab`, dll.).

### Langkah 2 - Konfigurasi router `rootkit`
Di konsol `rootkit`, jalankan skrip berikut. Skrip membuat `/root/setup.sh` yang menulis `/etc/network/interfaces` lalu menerapkannya.

```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash

cat << 'NET' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.238.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 192.238.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 192.238.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 192.238.4.1
    netmask 255.255.255.0

auto eth5
iface eth5 inet static
    address 192.238.5.1
    netmask 255.255.255.0
NET

# Menerapkan jaringan di Alpine
ifup -a 2>/dev/null || true
EOF

chmod +x /root/setup.sh
bash /root/setup.sh
```

### Langkah 3 - Aktifkan forwarding & NAT di `rootkit`
> **Catatan:** bagian ini tidak terlihat pada screenshot, tetapi diperlukan agar node dapat meneruskan paket antar-subnet dan mengakses internet (ping ke `8.8.8.8` berhasil pada hasil uji). Sesuaikan jika kamu sudah mengaturnya dengan cara lain.

```bash
# Aktifkan IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# NAT (masquerade) lewat eth0 yang mengarah ke NAT1
apk add iptables
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### Langkah 4 - Konfigurasi node non-router
Contoh untuk `alpha` (subnet 4). Untuk node lain, ganti nilai `address` dan `gateway` sesuai tabel pengalamatan.

```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash

cat << 'NET' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.238.4.2
    netmask 255.255.255.0
    gateway 192.238.4.1
NET

echo "nameserver 192.168.122.1" > /etc/resolv.conf

ifup -a 2>/dev/null || true
EOF

chmod +x /root/setup.sh
bash /root/setup.sh
```

Contoh nilai untuk node lain:

| Node | `address` | `gateway` |
|---|---|---|
| `beta` | `192.238.4.3` | `192.238.4.1` |
| `gamma` | `192.238.4.4` | `192.238.4.1` |
| `prab` | `192.238.1.2` | `192.238.1.1` |
| `abbey` | `192.238.2.2` | `192.238.2.1` |
| `penny` | `192.238.3.2` | `192.238.3.1` |
| `delta` | `192.238.5.2` | `192.238.5.1` |

## 5. Pengujian

### 5.1 Tes koneksi internet (dari `alpha`)

```bash
ping -c 3 8.8.8.8
ping -c 3 1.1.1.1
```

**Hasil:** kedua tujuan membalas, 3 paket terkirim, 3 diterima, **0% packet loss** (rata-rata ±21,7 ms ke `8.8.8.8` dan ±22,9 ms ke `1.1.1.1`). Artinya *default gateway* dan jalur NAT pada `rootkit` berfungsi.

### 5.2 Tes routing internal lintas subnet (dari `alpha`)

Menguji apakah `rootkit` berhasil meneruskan lalu lintas antar subnet/switch.

```bash
# Ke delta (subnet 5)
ping -c 3 192.238.5.2

# Ke prab - DNS Server (subnet 1)
ping -c 3 192.238.1.2
```

**Hasil:**

<img width="546" height="363" alt="Screenshot 2026-09-29 at 16 35 24" src="https://github.com/user-attachments/assets/1bcdcc5b-6f26-4c6d-aa80-12b7014a1eda" />
<img width="541" height="361" alt="Screenshot 2026-09-29 at 16 36 02" src="https://github.com/user-attachments/assets/521341a8-ad74-4b06-b1bc-4f35b052c808" />

## Soal 4
 
- Di **`prab`**: bangun zona `k54.com` sebagai *authoritative* dengan SOA yang menunjuk ke `prab.k54.com`, serta tambahkan catatan **NS** untuk `prab.k54.com` dan `tedd.k54.com`.
- Buat **A record** untuk `prab.k54.com` dan `tedd.k54.com` ke IP masing-masing, serta A record **apex** `k54.com` yang mengarah ke gerbang aplikasi dinamis (`penny`).
- Aktifkan **notify** dan **allow-transfer** ke `tedd`, lalu set **forwarders** ke `192.168.122.1`.
- Di **`tedd`**: tarik zona `k54.com` dari master dan pastikan server menjawab secara *authoritative*.
- Perbarui urutan resolver pada seluruh Entitas non-router menjadi: IP `prab`, IP `tedd`, lalu `192.168.122.1`.
- Verifikasi bahwa query ke domain apex maupun hostname di dalam zona dijawab dengan benar oleh `prab` atau `tedd`.
## 2. Rencana
 
| Peran | Node | IP | Keterangan |
|---|---|---|---|
| DNS Master | `prab` | `192.238.1.2` | Zona `k54.com` bertipe `master` |
| DNS Slave | `tedd` | `192.238.1.3` | Zona `k54.com` bertipe `slave`, master `192.238.1.2` |
| Gerbang aplikasi dinamis | `penny` | `192.238.3.2` | Target A record apex `k54.com` |
| Forwarder | - | `192.168.122.1` | Meneruskan query domain luar |
 
Isi zona `k54.com`:
 
| Nama | Tipe | Nilai |
|---|---|---|
| `@` | SOA | `prab.k54.com. root.k54.com.` (serial `2026092901`) |
| `@` | NS | `prab.k54.com.` |
| `@` | NS | `tedd.k54.com.` |
| `prab` | A | `192.238.1.2` |
| `tedd` | A | `192.238.1.3` |
| `@` (apex) | A | `192.238.3.2` (penny) |
 
Urutan resolver akhir di semua node non-router:
 
```
nameserver 192.238.1.2   # prab
nameserver 192.238.1.3   # tedd
nameserver 192.168.122.1
```
 
## 3. Langkah Pengerjaan
 
### Langkah 1 - Konfigurasi DNS Master (`prab`, `192.238.1.2`)
 
```bash
# 1. Set resolver internet sementara (wajib sebelum install paket)
echo "nameserver 192.168.122.1" > /etc/resolv.conf
 
# 2. Install paket BIND DNS server & tools
apk update && apk add bind bind-tools
 
# 3. Buat folder sistem BIND & PID di Alpine
mkdir -p /etc/bind /var/bind /run/named
 
# 4. Buat file konfigurasi BIND (/etc/bind/named.conf)
cat << 'NAMED' > /etc/bind/named.conf
options {
    directory "/var/bind";
    allow-query { any; };
    auth-nxdomain no;
    listen-on-v6 { none; };
    forwarders {
        192.168.122.1;
    };
    pid-file "/run/named/named.pid";
};
 
zone "k54.com" IN {
    type master;
    file "/var/bind/k54.com.zone";
    allow-transfer { 192.238.1.3; };   # izinkan transfer ke tedd
    notify yes;                        # beri tahu tedd jika ada perubahan
    also-notify { 192.238.1.3; };
};
NAMED
 
# 5. Buat file database zona k54.com (/var/bind/k54.com.zone)
cat << 'ZONE' > /var/bind/k54.com.zone
$TTL 1D
@   IN  SOA prab.k54.com. root.k54.com. (
            2026092901 ; Serial YYYYMMDDNN
            1D         ; Refresh
            1H         ; Retry
            1W         ; Expire
            1D )       ; Minimum TTL
 
; Record Name Server (prab & tedd)
@       IN  NS  prab.k54.com.
@       IN  NS  tedd.k54.com.
 
; Record A Name Server
prab    IN  A   192.238.1.2
tedd    IN  A   192.238.1.3
 
; Record A apex domain (k54.com mengarah ke IP penny)
@       IN  A   192.238.3.2
ZONE
 
# 6. Set permission & jalankan daemon BIND
chown -R named:named /etc/bind /var/bind /run/named
pkill named 2>/dev/null || true
named -u named -c /etc/bind/named.conf
 
# 7. Update resolver internal setelah BIND aktif
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
```
 
### Langkah 2 - Konfigurasi DNS Slave (`tedd`, `192.238.1.3`)
 
```bash
# 1. Set resolver internet sementara (wajib sebelum install paket)
echo "nameserver 192.168.122.1" > /etc/resolv.conf
 
# 2. Install paket BIND DNS server & tools
apk update && apk add bind bind-tools
 
# 3. Buat folder penampung zona slave & PID di Alpine
mkdir -p /etc/bind /var/bind/slaves /run/named
 
# 4. Buat file konfigurasi BIND slave (/etc/bind/named.conf)
cat << 'NAMED' > /etc/bind/named.conf
options {
    directory "/var/bind";
    allow-query { any; };
    auth-nxdomain no;
    listen-on-v6 { none; };
    forwarders {
        192.168.122.1;
    };
    pid-file "/run/named/named.pid";
};
 
zone "k54.com" IN {
    type slave;
    file "/var/bind/slaves/k54.com.zone";
    masters { 192.238.1.2; };   # menunjuk ke prab sebagai master
};
NAMED
 
# 5. Set permission & jalankan daemon BIND
chown -R named:named /etc/bind /var/bind /run/named
pkill named 2>/dev/null || true
named -u named -c /etc/bind/named.conf
 
# 6. Update resolver internal setelah BIND aktif
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
```
 
### Langkah 3 - Update resolver seluruh node non-router lainnya
 
Dijalankan di `alpha`, `beta`, `gamma`, `delta`, `epsilon`, `abbey`, `penny`, `obladi`, `desmond`, `oblada`, dan `molly`.
 
```bash
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
```
 
## 4. Pengujian
 
> Blok "Ekspektasi output" di bawah adalah hasil yang seharusnya muncul. Tempelkan screenshot output aktual dari terminalmu di bagian yang diberi tanda **[Screenshot]**.
 
### 4.1 Uji zone transfer di slave (`tedd`)
 
Membuktikan `tedd` berhasil menarik file zona `k54.com` dari `prab` secara otomatis.
 
```bash
ls -la /var/bind/slaves/
```
 
Ekspektasi output: file `k54.com.zone` muncul di folder tersebut.
 
```text
-rw-r--r-- 1 named named  ... k54.com.zone
```
<img width="546" height="366" alt="Screenshot 2026-09-29 at 17 34 21" src="https://github.com/user-attachments/assets/56e38a39-69a8-4618-bcfe-4f75aa137f45" />

 
### 4.2 Uji query domain apex dari klien (`alpha`)
 
Membuktikan query `k54.com` dialihkan ke IP `penny` (`192.238.3.2`).
 
```bash
nslookup k54.com
```
 
Ekspektasi output:
 
```text
Server:         192.238.1.2
Address:        192.238.1.2#53
 
Name:   k54.com
Address: 192.238.3.2
```
<img width="544" height="365" alt="Screenshot 2026-09-29 at 17 34 55" src="https://github.com/user-attachments/assets/e9659309-375c-4280-9efe-6775a6f61d04" />

 
### 4.3 Uji jawaban authoritative dari slave (`tedd`)
 
Memastikan `tedd` tidak sekadar menyimpan file, tetapi menjawab sebagai server authoritative yang sah.
 
```bash
dig @192.238.1.3 k54.com
```
 
Ekspektasi output: pada bagian header terdapat flag **`aa`** (authoritative answer), dan bagian ANSWER berisi `k54.com. ... IN A 192.238.3.2`.
 
<img width="541" height="361" alt="Screenshot 2026-09-29 at 17 35 10" src="https://github.com/user-attachments/assets/28ca5531-6bf1-4ed1-8c83-0f9ca0cf7928" />

 
### 4.4 Uji forwarders (resolusi domain luar via DNS internal)
 
Membuktikan aturan `forwarders { 192.168.122.1; };` bekerja, sehingga DNS internal dapat meneruskan query domain internet.
 
```bash
nslookup google.com
```
 
Ekspektasi output: `google.com` berhasil di-resolve (muncul alamat IP) melalui server `192.238.1.2`.
 
<img width="346" height="314" alt="Screenshot 2026-09-29 at 17 38 52" src="https://github.com/user-attachments/assets/cf67ce90-09b0-4015-9c0f-a9985e912bd1" />

 
### 4.5 Uji urutan resolver (`/etc/resolv.conf`)
 
Memastikan urutan resolver pada host non-router sesuai petunjuk soal.
 
```bash
cat /etc/resolv.conf
```
 
Ekspektasi output:
 
```text
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
```
 
<img width="226" height="58" alt="Screenshot 2026-09-29 at 17 47 00" src="https://github.com/user-attachments/assets/53763af6-3155-45cc-b3e1-8710e2b69bde" />

 
### 4.6 Uji hostname di dalam zona (tambahan)
 
Soal meminta verifikasi untuk domain apex **dan** hostname di dalam zona. Uji hostname belum ada di daftar pengujian, jadi disarankan menambahkan:
 
```bash
nslookup prab.k54.com
nslookup tedd.k54.com
```
 
Ekspektasi: `prab.k54.com` menjawab `192.238.1.2` dan `tedd.k54.com` menjawab `192.238.1.3`.

<img width="537" height="311" alt="Screenshot 2026-09-29 at 17 48 46" src="https://github.com/user-attachments/assets/0afc9902-3cbb-4b34-b33d-923986b41190" />


## Soal 5
 
"Entitas tanpa identitas adalah anomali," pesan Rootkit. Tugasnya:
 
- Menamai semua Entitas (*hostname*) sesuai glosarium: `rootkit`, `alpha`, `beta`, `gamma`, `delta`, `epsilon`, `prab`, `tedd`, `abbey`, `penny`, `obladi`, `desmond`, `oblada`, `molly`. Verifikasi bahwa setiap host mengenali hostname tersebut secara *system-wide*.
- Membuat domain untuk masing-masing node sesuai namanya (contoh: `alpha.k54.com`) dan meng-*assign* IP masing-masing.
- Pengecualian untuk node yang bertanggung jawab atas `prab` dan `tedd`: record keduanya sudah dibuat pada Soal 4, sehingga tidak perlu dibuat ulang.
## 2. Rencana
 
| Node | Hostname | Domain (FQDN) | IP |
|---|---|---|---|
| `alpha` | `alpha` | `alpha.k54.com` | `192.238.4.2` |
| `beta` | `beta` | `beta.k54.com` | `192.238.4.3` |
| `gamma` | `gamma` | `gamma.k54.com` | `192.238.4.4` |
| `delta` | `delta` | `delta.k54.com` | `192.238.5.2` |
| `epsilon` | `epsilon` | `epsilon.k54.com` | `192.238.5.3` |
| `abbey` | `abbey` | `abbey.k54.com` | `192.238.2.2` |
| `penny` | `penny` | `penny.k54.com` | `192.238.3.2` |
| `obladi` | `obladi` | `obladi.k54.com` | `192.238.1.4` |
| `desmond` | `desmond` | `desmond.k54.com` | `192.238.1.5` |
| `oblada` | `oblada` | `oblada.k54.com` | `192.238.1.6` |
| `molly` | `molly` | `molly.k54.com` | `192.238.1.7` |
| `prab` | `prab` | `prab.k54.com` (sudah ada, Soal 4) | `192.238.1.2` |
| `tedd` | `tedd` | `tedd.k54.com` (sudah ada, Soal 4) | `192.238.1.3` |
| `rootkit` | `rootkit` | - | router (banyak interface) |
 
## 3. Langkah Pengerjaan
 
### Langkah 1 - Tambah A record semua node di zona (`prab`, DNS Master)
 
Tambahkan record berikut ke `/var/bind/k54.com.zone` di `prab`. Record `prab` dan `tedd` tidak ditambahkan lagi karena sudah ada.
 
```text
; A Record Seluruh Subdomain Node (Soal 5)
alpha   IN  A   192.238.4.2
beta    IN  A   192.238.4.3
gamma   IN  A   192.238.4.4
delta   IN  A   192.238.5.2
epsilon IN  A   192.238.5.3
abbey   IN  A   192.238.2.2
penny   IN  A   192.238.3.2
obladi  IN  A   192.238.1.4
desmond IN  A   192.238.1.5
oblada  IN  A   192.238.1.6
molly   IN  A   192.238.1.7
```
 
Cara menambahkannya lewat terminal `prab`:
 
```bash
cat << 'ZONE' >> /var/bind/k54.com.zone
 
; A Record Seluruh Subdomain Node (Soal 5)
alpha   IN  A   192.238.4.2
beta    IN  A   192.238.4.3
gamma   IN  A   192.238.4.4
delta   IN  A   192.238.5.2
epsilon IN  A   192.238.5.3
abbey   IN  A   192.238.2.2
penny   IN  A   192.238.3.2
obladi  IN  A   192.238.1.4
desmond IN  A   192.238.1.5
oblada  IN  A   192.238.1.6
molly   IN  A   192.238.1.7
ZONE
```
 
Isi akhir file zona (bagian record) setelah ditambahkan:
 
```text
@        IN  A   192.238.3.2
 
prab     IN  A   192.238.1.2
tedd     IN  A   192.238.1.3
alpha    IN  A   192.238.4.2
beta     IN  A   192.238.4.3
gamma    IN  A   192.238.4.4
delta    IN  A   192.238.5.2
epsilon  IN  A   192.238.5.3
abbey    IN  A   192.238.2.2
penny    IN  A   192.238.3.2
obladi   IN  A   192.238.1.4
desmond  IN  A   192.238.1.5
oblada   IN  A   192.238.1.6
molly    IN  A   192.238.1.7
```
 
**Naikkan serial SOA dan muat ulang BIND.** Tanpa kenaikan serial, `tedd` tidak akan menarik versi zona yang baru.
 
```bash
# Ubah serial 2026092901 menjadi 2026092902
sed -i 's/2026092901/2026092902/' /var/bind/k54.com.zone
 
# Muat ulang named (cara yang sama seperti pada Soal 4)
pkill named 2>/dev/null || true
named -u named -c /etc/bind/named.conf
```
 
Karena `notify yes` aktif, `tedd` akan diberi tahu dan menarik zona terbaru secara otomatis.
 
### Langkah 2 - Atur hostname system-wide di seluruh node
 
Jalankan perintah di terminal **masing-masing node**. `hostname` mengubah nama pada sesi berjalan, sedangkan `/etc/hostname` membuatnya permanen setelah restart.
 
```bash
# Format umum (ganti NAMA sesuai node)
hostname NAMA && echo "NAMA" > /etc/hostname
```
 
| Di node | Perintah |
|---|---|
| `alpha` | `hostname alpha && echo "alpha" > /etc/hostname` |
| `beta` | `hostname beta && echo "beta" > /etc/hostname` |
| `gamma` | `hostname gamma && echo "gamma" > /etc/hostname` |
| `delta` | `hostname delta && echo "delta" > /etc/hostname` |
| `epsilon` | `hostname epsilon && echo "epsilon" > /etc/hostname` |
| `abbey` | `hostname abbey && echo "abbey" > /etc/hostname` |
| `penny` | `hostname penny && echo "penny" > /etc/hostname` |
| `obladi` | `hostname obladi && echo "obladi" > /etc/hostname` |
| `desmond` | `hostname desmond && echo "desmond" > /etc/hostname` |
| `oblada` | `hostname oblada && echo "oblada" > /etc/hostname` |
| `molly` | `hostname molly && echo "molly" > /etc/hostname` |
| `prab` | `hostname prab && echo "prab" > /etc/hostname` |
| `tedd` | `hostname tedd && echo "tedd" > /etc/hostname` |
| `rootkit` | `hostname rootkit && echo "rootkit" > /etc/hostname` |
 
## 4. Pengujian
 
> Blok output di bawah adalah hasil yang seharusnya muncul. Tempelkan screenshot output aktual dari terminalmu di bagian bertanda **[Screenshot]**.
 
### 4.1 Tes hostname system-wide (di masing-masing node)
 
Jalankan di node yang ingin diuji (mis. `alpha`, `beta`, `penny`, dst.):
 
```bash
hostname
```
 
Ekspektasi: output sesuai nama node, misalnya di `alpha` tampil `alpha`.
 
<img width="541" height="359" alt="Screenshot 2026-09-29 at 21 22 31" src="https://github.com/user-attachments/assets/6d70bc0b-c6be-4d0b-9205-4b0927a5c638" />

 
### 4.2 Tes resolusi DNS subdomain node (dari klien, mis. `alpha`)
 
Mengecek apakah DNS master (`prab`) dan slave (`tedd`) sudah mengenali domain seluruh node.
 
```bash
# Klien sayap kiri & kanan
nslookup alpha.k54.com
nslookup beta.k54.com
nslookup delta.k54.com
 
# Gerbang penyaring (reverse proxy)
nslookup abbey.k54.com
nslookup penny.k54.com
 
# Web server (repository)
nslookup obladi.k54.com
nslookup desmond.k54.com
nslookup oblada.k54.com
nslookup molly.k54.com
```
 
Ekspektasi: setiap query menjawab IP sesuai tabel rencana, misalnya `alpha.k54.com` menjawab `192.238.4.2` dan `molly.k54.com` menjawab `192.238.1.7`.
 
<img width="538" height="360" alt="Screenshot 2026-09-29 at 21 23 15" src="https://github.com/user-attachments/assets/4fdee3de-fc81-455c-97e0-e4de7c27f3af" />
<img width="539" height="351" alt="Screenshot 2026-09-29 at 21 22 58" src="https://github.com/user-attachments/assets/e0b0acf8-33d6-40c4-be7d-c99c1c6cbfe0" />

 
### 4.3 Tes ping menggunakan nama domain (dari `alpha`)
 
```bash
ping -c 2 delta.k54.com
ping -c 2 penny.k54.com
```
 
Ekspektasi: nama di-resolve ke `192.238.5.2` (delta) dan `192.238.3.2` (penny), lalu paket ICMP dibalas tanpa *packet loss*. Ini sekaligus membuktikan resolver dan routing internal via `rootkit` berjalan.
 
<img width="542" height="360" alt="Screenshot 2026-09-29 at 21 28 27" src="https://github.com/user-attachments/assets/e710302a-f8e0-4365-82bb-ba227c0f4f39" />

 
### 4.4 Tes zona di slave (tambahan)
 
Memastikan `tedd` sudah menarik record baru dari `prab`:
 
```bash
dig @192.238.1.3 alpha.k54.com
```
 
Ekspektasi: jawaban `192.238.4.2` dengan flag `aa`.

<img width="538" height="345" alt="Screenshot 2026-09-29 at 21 30 13" src="https://github.com/user-attachments/assets/80d6d84a-7a8b-453e-84f6-c9eef36d25ac" />

 
# 6. Memastikan zona transfer dari DNS master (prab) ke DNS slave (tedd) berjalan dengan serial SOA di kedua node identik.

### Langkah 1   
1. Memastikan konfigurasi Zone transfer di Master (prab) dan Slave (tedd).
   - Pastikan di dalam /etc/bind/named.conf blok zone "k54.com" ada parameter dibawah ini:

   `
     allow-transfer { 192.238.1.3; };
     notify yes;
     also-notify { 192.238.1.3; };
   `
   <img width="680" alt="image" src="https://github.com/user-attachments/assets/135a21a1-b33b-4f01-a3e4-78d9cc48d661" />

2. Di tedd pastikan direktori penampug Zona slave sudah ada dan memiliki hak akses ke user named.

   `
      mkdir -p /var/bind/slaves 
      chown -R named:named /var/bind
      chmod 777 /var/bind/slaves
   `

### Langkah 2
1. Cek Serial SOA di DNS Mater (prab)
   
   ``` dig SOA k54.com @192.238.1.2 ```

   <img width="680" alt="image" src="https://github.com/user-attachments/assets/f2d8f884-2b2d-4645-b6df-1ba9c2a6689d" />

2. Cek Serial SOA di DNS Slave (tedd)

   ``` dig SOA k54.com @192.238.1.3 ```

   <img width="680" alt="image" src="https://github.com/user-attachments/assets/0fb9748c-33e4-474e-83b0-cf97ffd998cf" />

Disitu bisa kita lihat Serias SOA di DNS Mater dengan DNS Slave itu sama yaitu: 2026092901

3. Cek File Salinan Zona Fisik di Node di Slave (tedd)

   ` ls -la /var/bind/slaves/ `

   <img width="820" height="142" alt="image" src="https://github.com/user-attachments/assets/02f428c0-9ae7-4c85-8b3c-74155ece0aff" />

   Disitu bisa terlihat bahwa salinan zona terbaru dari prab telah diterima oleh tedd, melalui munculnya file k54.com.zone.


### Langkah 3 : Pembuktian
1. Buka terminal prab, coba edit nomor serial di /var/bind/k54.com.zone dari 2026092901 menjadi 2026092902.
2. Restard/reload BIND di prab
   ` pkill named && named -u named `
3. Langsung cek ulang di node klien.

   <img width="680" alt="image" src="https://github.com/user-attachments/assets/0130bb77-3ec5-4020-8f54-fbe6aa548bf5" />

   <img width="680" alt="image" src="https://github.com/user-attachments/assets/e85cb722-4a00-4f4b-beb4-94862827ee29" />

Bisa terlihat bahwa SOA di node beta telah berubah juga menjadi 2026092902.

# 7. Menambahkan beberapa Record DNS baru di zona k54.com

## Langkah 1
Memperbarui script setup yang ada di node prab:

```
cat << 'EOF' > /root/setup.sh
#!/bin/bash

# 1. Hostname
hostname prab
echo "prab" > /etc/hostname

# 2. Network Interface
cat << 'NET' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.238.1.2
    netmask 255.255.255.0
    gateway 192.238.1.1
NET

ifup -a 2>/dev/null || true

# 3. Resolver
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV

# 4. Install BIND DNS
apk update && apk add bind bind-tools

# 5. Konfigurasi BIND Master
cat << 'NAMED' > /etc/bind/named.conf
options {
    directory "/var/bind";
    allow-query { any; };
    auth-nxdomain no;
    listen-on-v6 { none; };
    forwarders {
        192.168.122.1;
    };
};

zone "k54.com" IN {
    type master;
    file "/var/bind/k54.com.zone";
    allow-transfer { 192.238.1.3; };
    notify yes;
    also-notify { 192.238.1.3; };
};
NAMED

# 6. File Zona k54.com (DENGAN TAMBAHAN SOAL 7)
cat << 'ZONE' > /var/bind/k54.com.zone
$TTL 1D
@   IN  SOA prab.k54.com. root.k54.com. (
            2026092902 ; Serial dinaikkan ke 02
            1D         ; Refresh
            1H         ; Retry
            1W         ; Expire
            1D )       ; Minimum TTL

; Record Name Server
@       IN  NS  prab.k54.com.
@       IN  NS  tedd.k54.com.

; Record A Apex
@       IN  A   192.238.3.2

; Record A Name Server
prab    IN  A   192.238.1.2
tedd    IN  A   192.238.1.3

; Record A Node Utama
alpha   IN  A   192.238.4.2
beta    IN  A   192.238.4.3
gamma   IN  A   192.238.4.4
delta   IN  A   192.238.5.2
epsilon IN  A   192.238.5.3
abbey   IN  A   192.238.2.2
penny   IN  A   192.238.3.2
obladi  IN  A   192.238.1.4
desmond IN  A   192.238.1.5
oblada  IN  A   192.238.1.6
molly   IN  A   192.238.1.7

; --- SOAL 7: A Record Round-Robin ---
vault   IN  A   192.238.1.4
vault   IN  A   192.238.1.5

core    IN  A   192.238.1.6
core    IN  A   192.238.1.7

; --- SOAL 7: CNAME Record ---
www     IN  CNAME penny.k54.com.
static  IN  CNAME abbey.k54.com.
ZONE

chown -R named:named /var/bind
pkill named 2>/dev/null || true
named -u named

# 7. AUTOSTART GNS3 DOCKER
if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
    echo "pgrep named >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
fi
if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
    echo "pgrep named >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
fi
EOF

chmod +x /root/setup.sh && bash /root/setup.sh
```

## Langkah 2: Verifikasi dari Client

1. Uji `vault.k54.com`:      
   ``` nslookup vault.k54.com```    
   <img width="458" height="238" alt="image" src="https://github.com/user-attachments/assets/5474bfd6-6a71-4c0e-b7aa-1fe7d8b7c1e0" />

2. Uji `core.k54.com`:  
   ``` nslookup core.k54.com ```  
   <img width="430" height="230" alt="image" src="https://github.com/user-attachments/assets/47bad88d-25bb-4af2-a2ec-410dbd74af0a" />

3. Uji `[www.k54.com](https://www.k54.com)`:  
   ```nslookup www.k54.com```  
   <img width="690" height="217" alt="image" src="https://github.com/user-attachments/assets/6dfd118e-1a06-4f27-9704-eec7eb1243f2" />

4. Uji `static.k54.com`:  
   ```nslookup static.k54.com```  
   <img width="676" height="196" alt="image" src="https://github.com/user-attachments/assets/a80b190e-39d7-4727-a405-48b1968b8909" />

## Langkah 3: Tes di node lain untuk menguji konsistensi jawaban DNS:

<img width="680" alt="image" src="https://github.com/user-attachments/assets/2600bc49-ef2c-4089-aa87-d4e65c850d3e" />
<img width="680" alt="image" src="https://github.com/user-attachments/assets/482a0762-05ec-46be-a20b-e0d428982440" />

Dari Langkah 2 dan 3 kita bisa melihat bahwa mekanisme Round-Robin Load Balancing nya bekerja. 

- Permintaan Pertama (alpha): BIND memberikan urutan 192.238.1.5 terlebih dahulu, lalu 192.238.1.4. 
- Permintaan Kedua (beta): BIND memutar urutannya menjadi 192.238.1.4 terlebih dahulu, lalu 192.238.1.5.

Tujuan dari pergantian urutan ini adalah agar beban trafik jaringan terbagi secara merata dan seimbang antara server obladi (192.238.1.4) dan desmond (192.238.1.5).


# 8. Membuat Reverse DNS Zone (PTR Record) pada DNS Master (prab) dan menariknya di DNS Slave (tedd).

## Langkah 1
1. Perbarui script setup di prab yang sudah ditambahkan Reverse Zone
   ```
   cat << 'EOF' > /root/setup.sh
   #!/bin/bash

   hostname prab
   echo "prab" > /etc/hostname

    cat << 'NET' > /etc/network/interfaces
    auto lo
    iface lo inet loopback
    
    auto eth0
    iface eth0 inet static
        address 192.238.1.2
        netmask 255.255.255.0
        gateway 192.238.1.1
    NET
    
    ifup -a 2>/dev/null || true
    
    cat << 'RESOLV' > /etc/resolv.conf
    nameserver 192.238.1.2
    nameserver 192.238.1.3
    nameserver 192.168.122.1
    RESOLV
    
    apk update && apk add bind bind-tools
    
    # --- KONFIGURASI BIND MASTER ---
    cat << 'NAMED' > /etc/bind/named.conf
    options {
        directory "/var/bind";
        allow-query { any; };
        auth-nxdomain no;
        listen-on-v6 { none; };
        forwarders {
            192.168.122.1;
        };
    };
    
    // Forward Zone k54.com
    zone "k54.com" IN {
        type master;
        file "/var/bind/k54.com.zone";
        allow-transfer { 192.238.1.3; };
        notify yes;
        also-notify { 192.238.1.3; };
    };
    
    // Reverse Zone Subnet 1 (Prab, Tedd, Vault, Core)
    zone "1.238.192.in-addr.arpa" IN {
        type master;
        file "/var/bind/1.238.192.zone";
        allow-transfer { 192.238.1.3; };
        notify yes;
        also-notify { 192.238.1.3; };
    };
    
    // Reverse Zone Subnet 2 (Abbey)
    zone "2.238.192.in-addr.arpa" IN {
        type master;
        file "/var/bind/2.238.192.zone";
        allow-transfer { 192.238.1.3; };
        notify yes;
        also-notify { 192.238.1.3; };
    };
    
    // Reverse Zone Subnet 3 (Penny)
    zone "3.238.192.in-addr.arpa" IN {
        type master;
        file "/var/bind/3.238.192.zone";
        allow-transfer { 192.238.1.3; };
        notify yes;
        also-notify { 192.238.1.3; };
    };
    NAMED
    
    # --- FORWARD ZONE FILE ---
    cat << 'ZONE' > /var/bind/k54.com.zone
    $TTL 1D
    @   IN  SOA prab.k54.com. root.k54.com. (
                2026092903
                1D 1H 1W 1D )
    
    @       IN  NS  prab.k54.com.
    @       IN  NS  tedd.k54.com.
    @       IN  A   192.238.3.2
    
    prab    IN  A   192.238.1.2
    tedd    IN  A   192.238.1.3
    alpha   IN  A   192.238.4.2
    beta    IN  A   192.238.4.3
    gamma   IN  A   192.238.4.4
    delta   IN  A   192.238.5.2
    epsilon IN  A   192.238.5.3
    abbey   IN  A   192.238.2.2
    penny   IN  A   192.238.3.2
    obladi  IN  A   192.238.1.4
    desmond IN  A   192.238.1.5
    oblada  IN  A   192.238.1.6
    molly   IN  A   192.238.1.7
    
    vault   IN  A   192.238.1.4
    vault   IN  A   192.238.1.5
    core    IN  A   192.238.1.6
    core    IN  A   192.238.1.7
    
    www     IN  CNAME penny.k54.com.
    static  IN  CNAME abbey.k54.com.
    ZONE
    
    # --- REVERSE ZONE FILE SUBNET 1 ---
    cat << 'REV1' > /var/bind/1.238.192.zone
    $TTL 1D
    @   IN  SOA prab.k54.com. root.k54.com. (
                2026092903
                1D 1H 1W 1D )
    
    @       IN  NS  prab.k54.com.
    @       IN  NS  tedd.k54.com.
    
    2       IN  PTR prab.k54.com.
    3       IN  PTR tedd.k54.com.
    4       IN  PTR obladi.k54.com.
    5       IN  PTR desmond.k54.com.
    6       IN  PTR oblada.k54.com.
    7       IN  PTR molly.k54.com.
    REV1
    
    # --- REVERSE ZONE FILE SUBNET 2 ---
    cat << 'REV2' > /var/bind/2.238.192.zone
    $TTL 1D
    @   IN  SOA prab.k54.com. root.k54.com. (
                2026092903
                1D 1H 1W 1D )
    
    @       IN  NS  prab.k54.com.
    @       IN  NS  tedd.k54.com.
    
    2       IN  PTR abbey.k54.com.
    REV2
    
    # --- REVERSE ZONE FILE SUBNET 3 ---
    cat << 'REV3' > /var/bind/3.238.192.zone
    $TTL 1D
    @   IN  SOA prab.k54.com. root.k54.com. (
                2026092903
                1D 1H 1W 1D )
    
    @       IN  NS  prab.k54.com.
    @       IN  NS  tedd.k54.com.
    
    2       IN  PTR penny.k54.com.
    REV3
    
    chown -R named:named /var/bind
    pkill named 2>/dev/null || true
    named -u named
    
    if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
        echo "pgrep named >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
    fi
    if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
        echo "pgrep named >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
    fi
    EOF
    
    chmod +x /root/setup.sh && bash /root/setup.sh
   ```

## Langkah 2
1. Perbarui script node di tedd agar bisa menarik reverse zone dari prab:

   ```
   cat << 'EOF' > /root/setup.sh
    #!/bin/bash
    
    hostname tedd
    echo "tedd" > /etc/hostname
    
    cat << 'NET' > /etc/network/interfaces
    auto lo
    iface lo inet loopback
    
    auto eth0
    iface eth0 inet static
        address 192.238.1.3
        netmask 255.255.255.0
        gateway 192.238.1.1
    NET
    
    ifup -a 2>/dev/null || true
    
    cat << 'RESOLV' > /etc/resolv.conf
    nameserver 192.238.1.2
    nameserver 192.238.1.3
    nameserver 192.168.122.1
    RESOLV
    
    apk update && apk add bind bind-tools
    
    mkdir -p /var/bind/slaves
    chown -R named:named /var/bind
    chmod 777 /var/bind/slaves
    
    # --- KONFIGURASI SLAVE REVERSE ZONES ---
    cat << 'NAMED' > /etc/bind/named.conf
    options {
        directory "/var/bind";
        allow-query { any; };
        auth-nxdomain no;
        listen-on-v6 { none; };
        forwarders {
            192.168.122.1;
        };
    };
    
    zone "k54.com" IN {
        type slave;
        file "/var/bind/slaves/k54.com.zone";
        masters { 192.238.1.2; };
    };
    
    zone "1.238.192.in-addr.arpa" IN {
        type slave;
        file "/var/bind/slaves/1.238.192.zone";
        masters { 192.238.1.2; };
    };
    
    zone "2.238.192.in-addr.arpa" IN {
        type slave;
        file "/var/bind/slaves/2.238.192.zone";
        masters { 192.238.1.2; };
    };
    
    zone "3.238.192.in-addr.arpa" IN {
        type slave;
        file "/var/bind/slaves/3.238.192.zone";
        masters { 192.238.1.2; };
    };
    NAMED
    
    pkill named 2>/dev/null || true
    named -u named
    
    if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
        echo "pgrep named >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
    fi
    if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
        echo "pgrep named >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
    fi
    EOF
    
    chmod +x /root/setup.sh && bash /root/setup.sh
   ```

   ## Langkah 3: Pengujian Verifikasi Reverse Lookup
   Lakukan perintah pengujian ini di node client:

   1. Uji Reverse Lookup Abbey
      `host 192.238.2.2`  
      <img width="799" height="164" alt="image" src="https://github.com/user-attachments/assets/3c9fac57-1f2d-4740-90a9-55b4fa8c448c" />

   2. Uji Reverse Lookup Penny
      `host 192.238.3.2`
      <img width="804" height="127" alt="image" src="https://github.com/user-attachments/assets/26608c89-00e1-4e74-ae8c-db4377d178e9" />

   3. Uji Reverse Lookup Area Vault ( obladi & desmond )
      ```
        host 192.238.1.4
        host 192.238.1.5
      ```
      <img width="873" height="175" alt="image" src="https://github.com/user-attachments/assets/c047fae9-95e2-47d0-9143-de6bc4216034" />

   4. Uji reverse Lookup Area Core ( oblada & molly)
      ```
       host 192.238.1.6
       host 192.238.1.7
      ```
      <img width="887" height="159" alt="image" src="https://github.com/user-attachments/assets/539119bf-5550-4d8d-9d4f-8e78fa9cea59" />

   5. Uji Respon Authoritative dari Slave
      ```dig -x 192.238.1.4 @192.238.1.3```  
      <img width="1005" height="649" alt="image" src="https://github.com/user-attachments/assets/96bfc97c-b23c-4728-bb60-e4f529adf15b" />

# 9. Mengonfigurasi Web Server Statis Apache pada node-node di Area Vault (obladi & desmond).

Fitur utama yang diminta adalah Autoindex (directory listing) pada folder /arsip/. Saat folder /arsip/ diakses melalui browser atau HTTP tanpa adanya file index.html, Apache akan secara otomatis menampilkan daftar seluruh file dan sub-folder yang tersimpan di dalamnya.

## Langka 1: Jalankan scripth /root/setup.sh berikut di node obladi

```
    cat << 'EOF' > /root/setup.sh
    #!/bin/bash
    
    # 1. Hostname & Network Interface
    hostname obladi
    echo "obladi" > /etc/hostname
    
    cat << 'NET' > /etc/network/interfaces
    auto lo
    iface lo inet loopback
    
    auto eth0
    iface eth0 inet static
        address 192.238.1.4
        netmask 255.255.255.0
        gateway 192.238.1.1
    NET
    
    ifup -a 2>/dev/null || true
    
    # 2. Resolver DNS
    cat << 'RESOLV' > /etc/resolv.conf
    nameserver 192.238.1.2
    nameserver 192.238.1.3
    nameserver 192.168.122.1
    RESOLV
    
    # 3. Install Apache2
    apk update && apk add apache2 curl
    
    # 4. Buat Direktori /arsip/ dan File Sampel
    mkdir -p /var/www/localhost/htdocs/arsip
    echo "Dokumen Rahasia Vault - Server Obladi 01" > /var/www/localhost/htdocs/arsip/vault_data1.txt
    echo "Laporan Keuangan K54 - Server Obladi 02" > /var/www/localhost/htdocs/arsip/laporan_obladi.pdf
    
    # Pastikan TIDAK ADA index.html agar autoindex berjalan
    rm -f /var/www/localhost/htdocs/arsip/index.html 2>/dev/null || true
    
    # 5. Konfigurasi Autoindex (Directory Listing) pada Apache
    cat << 'APACHE' > /etc/apache2/conf.d/arsip.conf
    <Directory "/var/www/localhost/htdocs/arsip">
        Options +Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    APACHE
    
    # 6. Jalankan Service Apache
    pkill httpd 2>/dev/null || true
    httpd -k start
    
    # 7. Autostart GNS3 Docker
    if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
        echo "pgrep httpd >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
    fi
    if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
        echo "pgrep httpd >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
    fi
    EOF
    
    chmod +x /root/setup.sh && bash /root/setup.sh
```

## Langkah 2: Konfigurasi di node Desmond
Jalankan perintah scripth /root/setup.sh berikut di terminal node Desmond:

```
cat << 'EOF' > /root/setup.sh
#!/bin/bash

# 1. Hostname & Network Interface
hostname desmond
echo "desmond" > /etc/hostname

cat << 'NET' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.238.1.5
    netmask 255.255.255.0
    gateway 192.238.1.1
NET

ifup -a 2>/dev/null || true

# 2. Resolver DNS
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV

# 3. Install Apache2
apk update && apk add apache2 curl

# 4. Buat Direktori /arsip/ dan File Sampel
mkdir -p /var/www/localhost/htdocs/arsip
echo "Dokumen Rahasia Vault - Server Desmond 01" > /var/www/localhost/htdocs/arsip/vault_data2.txt
echo "Laporan Keuangan K54 - Server Desmond 02" > /var/www/localhost/htdocs/arsip/laporan_desmond.pdf

# Pastikan TIDAK ADA index.html agar autoindex berjalan
rm -f /var/www/localhost/htdocs/arsip/index.html 2>/dev/null || true

# 5. Konfigurasi Autoindex (Directory Listing) pada Apache
cat << 'APACHE' > /etc/apache2/conf.d/arsip.conf
<Directory "/var/www/localhost/htdocs/arsip">
    Options +Indexes +FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
APACHE

# 6. Jalankan Service Apache
pkill httpd 2>/dev/null || true
httpd -k start

# 7. Autostart GNS3 Docker
if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
    echo "pgrep httpd >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
fi
if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
    echo "pgrep httpd >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
fi
EOF

chmod +x /root/setup.sh && bash /root/setup.sh
```

## Langkah 3
Melakukan pengujian menggunakan nama domain/hostname, bukan IP address. Buka terminal node klien, dan pastikan paket curl terpasang (`apk add curl`) lalu jalankan pengujian ini:

1. Uji akses via Hostname Round-Robin vault (`[vault.k54.com/arsip/](https://vault.k54.com/arsip/)`):

   `curl -s http://vault.k54.com/arsip/`

   <img width="1061" height="436" alt="image" src="https://github.com/user-attachments/assets/1dadfdb2-9150-4ec5-8a59-2d23957cd50f" />

2. Uji Akses via Hostname Masing-Masing Node Server:

   ```
    curl -s http://obladi.k54.com/arsip/
    curl -s http://desmond.k54.com/arsip/
   ```

   <img width="1060" height="626" alt="image" src="https://github.com/user-attachments/assets/93dd696c-2442-44be-a11a-98e24ab2f491" />  
   <img width="1063" height="205" alt="image" src="https://github.com/user-attachments/assets/b8bb6916-4a2c-4000-a35d-fd78cd53f8de" />


# soal 10. Mengonfigurasi Web Server Dinamis (Nginx + PHP-FPM) pada node-node Area Core (oblada & molly). 

Spesifikasi utama yang harus dipenuhi:
1. Menggunakan Nginx dan PHP-FPM.
2. Membuat halaman Beranda (index.php) dan Profil (profil.php).
3. Menerapkan aturan URL Rewrite (Clean URL) pada Nginx, sehingga halaman profil dapat diakses melalui URL /profil tanpa ekstensi .php.
4. Pengujian wajib diakses menggunakan hostname core.k54.com (serta oblada.k54.com dan molly.k54.com).

## Langkah 1
1. Melakukan konfigurasi di node oblada menggunakan scripth di bawah ini:

   ```
    cat << 'EOF' > /root/setup.sh
    #!/bin/bash
    
    # 1. Hostname & Network Interface
    hostname oblada
    echo "oblada" > /etc/hostname
    
    cat << 'NET' > /etc/network/interfaces
    auto lo
    iface lo inet loopback
    
    auto eth0
    iface eth0 inet static
        address 192.238.1.6
        netmask 255.255.255.0
        gateway 192.238.1.1
    NET
    
    ifup -a 2>/dev/null || true
    
    # 2. Resolver DNS
    cat << 'RESOLV' > /etc/resolv.conf
    nameserver 192.238.1.2
    nameserver 192.238.1.3
    nameserver 192.168.122.1
    RESOLV
    
    # 3. Install Nginx & PHP-FPM (Paket Generik Alpine)
    apk update && apk add nginx php-fpm php curl
    
    # 4. Buat Direktori Web Root
    mkdir -p /var/www/html /etc/nginx/http.d
    
    # 5. Buat File Application (Beranda & Profil)
    cat << 'PHP' > /var/www/html/index.php
    <?php
    echo "<h1>Selamat Datang di Beranda Core (Oblada)</h1>";
    echo "<p>Server Hostname: " . gethostname() . "</p>";
    ?>
    PHP
    
    cat << 'PHP' > /var/www/html/profil.php
    <?php
    echo "<h1>Halaman Profil Server Core (Oblada)</h1>";
    echo "<p>Ini adalah halaman profil dengan URL Bersih (Clean URL)!</p>";
    echo "<p>Node: " . gethostname() . " (" . $_SERVER['SERVER_ADDR'] . ")</p>";
    ?>
    PHP
    
    chown -R nginx:nginx /var/www/html 2>/dev/null || true
    
    # 6. Konfigurasi Nginx Server Block + Clean URL Rewrite
    cat << 'NGINX' > /etc/nginx/http.d/default.conf
    server {
        listen 80;
        server_name core.k54.com oblada.k54.com;
    
        root /var/www/html;
        index index.php index.html;
    
        # URL Rewrite (Clean URL)
        location / {
            try_files $uri $uri/ $uri.php?$args;
        }
    
        # Handler PHP-FPM
        location ~ \.php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
    NGINX
    
    # 7. Jalankan Service PHP-FPM & Nginx
    pkill php-fpm 2>/dev/null || true
    pkill nginx 2>/dev/null || true
    
    php-fpm 2>/dev/null || php-fpm83 2>/dev/null || php-fpm82 2>/dev/null || true
    nginx
    
    # 8. Autostart GNS3 Docker
    if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
        echo "pgrep nginx >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
    fi
    if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
        echo "pgrep nginx >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
    fi
    EOF
    
    chmod +x /root/setup.sh && bash /root/setup.sh
   ```

   Scripth dibawah ini digunakan jika port 80 telah digunakan sebelumnya:

   ```
    pkill -9 nginx 2>/dev/null
    pkill -9 httpd 2>/dev/null
    sleep 1
    nginx
   ```


## Langkah 2
Konfigurasi scripth di node Molly:


```
cat << 'EOF' > /root/setup.sh
#!/bin/bash

# 1. Hostname & Network Interface
hostname molly
echo "molly" > /etc/hostname

cat << 'NET' > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.238.1.7
    netmask 255.255.255.0
    gateway 192.238.1.1
NET

ifup -a 2>/dev/null || true

# 2. Resolver DNS
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV

# 3. Install Nginx & PHP-FPM (Paket Generik Alpine)
apk update && apk add nginx php-fpm php curl

# 4. Buat Direktori Web Root
mkdir -p /var/www/html /etc/nginx/http.d

# 5. Buat File Application (Beranda & Profil)
cat << 'PHP' > /var/www/html/index.php
<?php
echo "<h1>Selamat Datang di Beranda Core (Molly)</h1>";
echo "<p>Server Hostname: " . gethostname() . "</p>";
?>
PHP

cat << 'PHP' > /var/www/html/profil.php
<?php
echo "<h1>Halaman Profil Server Core (Molly)</h1>";
echo "<p>Ini adalah halaman profil dengan URL Bersih (Clean URL)!</p>";
echo "<p>Node: " . gethostname() . " (" . $_SERVER['SERVER_ADDR'] . ")</p>";
?>
PHP

chown -R nginx:nginx /var/www/html 2>/dev/null || true

# 6. Konfigurasi Nginx Server Block + Clean URL Rewrite
cat << 'NGINX' > /etc/nginx/http.d/default.conf
server {
    listen 80;
    server_name core.k54.com molly.k54.com;

    root /var/www/html;
    index index.php index.html;

    # URL Rewrite (Clean URL)
    location / {
        try_files $uri $uri/ $uri.php?$args;
    }

    # Handler PHP-FPM
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
NGINX

# 7. Jalankan Service PHP-FPM & Nginx
pkill php-fpm 2>/dev/null || true
pkill nginx 2>/dev/null || true

php-fpm 2>/dev/null || php-fpm83 2>/dev/null || php-fpm82 2>/dev/null || true
nginx

# 8. Autostart GNS3 Docker
if ! grep -q "setup.sh" /root/.bashrc 2>/dev/null; then
    echo "pgrep nginx >/dev/null || /bin/bash /root/setup.sh" >> /root/.bashrc
fi
if ! grep -q "setup.sh" /etc/profile 2>/dev/null; then
    echo "pgrep nginx >/dev/null || /bin/bash /root/setup.sh" >> /etc/profile
fi
EOF

chmod +x /root/setup.sh && bash /root/setup.sh
```


## Langkah pengujian dari client

1. Daftarkan Mapping Hostname Core ke /etc/hosts di node yang ingin diuji

   ```
   # 1. Tambahkan Mapping Hostname Core ke /etc/hosts
    echo "192.238.1.6 core.k54.com" >> /etc/hosts
    echo "192.238.1.7 core.k54.com" >> /etc/hosts
    
    # 2. Cek apakah hostname terdaftar
    ping -c 2 core.k54.com
   ```

2. Uji halaman Beranda via Hostname Round-Robin Core
   ```wget -qO- http://core.k54.com/```
   <img width="1060" height="88" alt="image" src="https://github.com/user-attachments/assets/ba395d0d-187c-4752-bc2c-9ae892edacac" />

3. Uji Clean URL Halaman Profil
   ```wget -qO- http://core.k54.com/profil```
   <img width="1060" height="103" alt="image" src="https://github.com/user-attachments/assets/24e7dbcd-a4a9-466d-99a6-b82532f2d64b" />

4. Uji Akses vis Hostname Masing-Masing server
   ```
      wget -qO- http://oblada.k54.com/profil
      wget -qO- http://molly.k54.com/profil
   ```
   <img width="1062" height="190" alt="image" src="https://github.com/user-attachments/assets/ad9015a6-53df-45aa-86c4-ad0562a2ec5c" />

# Soal 11: Reverse Proxy Penny (Apache) & Abbey (Nginx)
 
**Proyek GNS3:** `K-54-MODUL-2`
**Domain:** `k54.com`
**OS node:** Alpine Linux (OpenRC)
 
Dokumen ini melanjutkan Soal 1-5 (IP, NAT, routing, DNS master-slave, hostname & domain node).
 
---
 
## 1. Deskripsi Soal
 
- Konfigurasikan **`penny`** (Apache) sebagai *reverse proxy* yang mengarah ke semua node di **area vault** (`obladi` & `desmond`).
- Konfigurasikan **`abbey`** (Nginx) sebagai *reverse proxy* menuju **area core** (`oblada` & `molly`).
- Kedua gerbang harus meneruskan identitas asli pengunjung ke server backend dengan *forwarding header* **`Host`** dan **`X-Real-IP`**.
- Buktikan bahwa `penny` dan `abbey` berhasil mendistribusikan lalu lintas dengan tepat.
## 2. Rencana & Arsitektur
 
| Peran | Node | IP | Software | Keterangan |
|---|---|---|---|---|
| Gerbang vault | `penny` | `192.238.3.2` | Apache (`apache2`, `apache2-proxy`) | Balancer `vaultcluster`, metode `byrequests` |
| Backend vault 1 | `obladi` | `192.238.1.4` | Nginx | Halaman statis "Vault Area 1" |
| Backend vault 2 | `desmond` | `192.238.1.5` | Nginx | Halaman statis "Vault Area 2" |
| Gerbang core | `abbey` | `192.238.2.2` | Nginx | Upstream `corecluster` (round robin) |
| Backend core 1 | `oblada` | `192.238.1.6` | Nginx + PHP-FPM | Aplikasi PHP + clean URL |
| Backend core 2 | `molly` | `192.238.1.7` | Nginx + PHP-FPM | Aplikasi PHP + clean URL |
 
 
Header yang diteruskan gerbang ke backend:
 
| Header | Penny (Apache) | Abbey (Nginx) |
|---|---|---|
| `Host` | `ProxyPreserveHost On` | `proxy_set_header Host $host;` |
| `X-Real-IP` | `RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"` | `proxy_set_header X-Real-IP $remote_addr;` |
| `X-Forwarded-For` | otomatis oleh `mod_proxy` | `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` |
 
## 3. Langkah Pengerjaan
 
Urutan: siapkan **backend** dulu (vault dan core), baru **gerbang** (`penny` dan `abbey`). Setiap skrip berdiri sendiri (mengatur hostname, IP, resolver, service, dan autostart), lalu dijalankan di terminal node terkait.
 
### Langkah 1 - Backend area vault (`obladi` & `desmond`)
 
Skrip untuk **`obladi`** (`192.238.1.4`):
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. HOSTNAME & JARINGAN ---
NAME="obladi"
IP="192.238.1.4"
GW="192.238.1.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
# --- 2. RESOLVER DNS ---
cat << RESOLV > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 3. INSTALL WEB SERVER NGINX ---
apk update && apk add nginx
 
# --- 4. BUAT DIREKTORI RUNTIME WAJIB ---
mkdir -p /run/nginx /var/www/html /etc/nginx/http.d
 
# --- 5. KONTEN HALAMAN WEB ---
cat << 'HTML' > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>Vault Area 1</title></head>
<body>
<h1>Response dari OBLADI (Vault 1)</h1>
<p>IP Address: 192.238.1.4</p>
</body>
</html>
HTML
 
# --- 6. KONFIGURASI NGINX SERVER BLOCK ---
cat << 'NGINX' > /etc/nginx/http.d/default.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;
 
    root /var/www/html;
    index index.html index.htm;
 
    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX
 
# --- 7. JALANKAN SERVICE NGINX ---
pkill -9 nginx 2>/dev/null || true
sleep 1
nginx
 
# --- 8. AUTOSTART PERSISTENSI OPENRC ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
Skrip **`desmond`** identik. Hanya empat nilai yang berbeda:
 
| Bagian | `obladi` | `desmond` |
|---|---|---|
| `NAME` | `obladi` | `desmond` |
| `IP` | `192.238.1.4` | `192.238.1.5` |
| `<title>` | `Vault Area 1` | `Vault Area 2` |
| `<h1>` / `<p>` | `Response dari OBLADI (Vault 1)` / `IP Address: 192.238.1.4` | `Response dari DESMOND (Vault 2)` / `IP Address: 192.238.1.5` |
 
### Langkah 2 - Backend area core (`oblada` & `molly`)
 
Skrip untuk **`oblada`** (`192.238.1.6`):
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. HOSTNAME & NETWORK INTERFACE ---
NAME="oblada"
IP="192.238.1.6"
GW="192.238.1.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
# --- 2. RESOLVER DNS ---
cat << RESOLV > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 3. INSTALL NGINX & PHP-FPM ---
apk update && apk add nginx php-fpm php curl
 
# --- 4. BUAT DIREKTORI WEB ROOT & RUNTIME WAJIB ---
mkdir -p /var/www/html /etc/nginx/http.d /run/nginx /run/php
 
# --- 5. BUAT FILE APLIKASI (BERANDA & PROFIL) ---
cat << 'PHP' > /var/www/html/index.php
<?php
echo "<h1>Selamat Datang di Beranda Core (Oblada)</h1>";
echo "<p>Server Hostname: " . gethostname() . "</p>";
?>
PHP
 
cat << 'PHP' > /var/www/html/profil.php
<?php
echo "<h1>Halaman Profil Server Core (Oblada)</h1>";
echo "<p>Ini adalah halaman profil dengan URL Bersih (Clean URL)!</p>";
echo "<p>Node: " . gethostname() . " (" . $_SERVER['SERVER_ADDR'] . ")</p>";
?>
PHP
 
chown -R nginx:nginx /var/www/html 2>/dev/null || true
 
# --- 6. KONFIGURASI NGINX SERVER BLOCK + CLEAN URL REWRITE ---
cat << 'NGINX' > /etc/nginx/http.d/default.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name core.k54.com oblada.k54.com;
 
    root /var/www/html;
    index index.php index.html;
 
    # URL Rewrite (Clean URL)
    location / {
        try_files $uri $uri/ $uri.php?$args;
    }
 
    # Handler PHP-FPM
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
NGINX
 
# --- 7. JALANKAN SERVICE PHP-FPM & NGINX ---
pkill -9 php-fpm 2>/dev/null || true
pkill -9 nginx 2>/dev/null || true
sleep 1
 
# Start PHP-FPM sebagai daemon (-D)
php-fpm -D 2>/dev/null || php-fpm84 -D 2>/dev/null || php-fpm83 -D 2>/dev/null
# (baris ini terpotong di screenshot; tambahkan versi php-fpm lain sesuai paket yang terpasang)
nginx
 
# --- 8. AUTOSTART PERSISTENSI OPENRC (AGAR OTOMATIS SAAT BOOT) ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
Skrip **`molly`** identik dengan `oblada`. Nilai yang berbeda:
 
| Bagian | `oblada` | `molly` |
|---|---|---|
| `NAME` | `oblada` | `molly` |
| `IP` | `192.238.1.6` | `192.238.1.7` |
| `server_name` | `core.k54.com oblada.k54.com` | `core.k54.com molly.k54.com` |
| Judul di `index.php` | `... Beranda Core (Oblada)` | `... Beranda Core (Molly)` |
| Judul di `profil.php` | `... Profil Server Core (Oblada)` | `... Profil Server Core (Molly)` |
 
### Langkah 3 - Gerbang vault: `penny` (Apache reverse proxy)
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. JARINGAN & HOSTNAME ---
NAME="penny"
IP="192.238.3.2"
GW="192.238.3.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 2. INSTALL & KONFIGURASI REVERSE PROXY APACHE ---
apk update && apk add apache2 apache2-proxy apache2-utils
 
# Set ServerName
# (path file terpotong di screenshot, diasumsikan /etc/apache2/httpd.conf)
sed -i 's/#ServerName www.example.com:80/ServerName penny.k54.com:80/' /etc/apache2/httpd.conf
 
# File konfigurasi proxy balancer ke area vault (obladi & desmond)
cat << 'PROXY' > /etc/apache2/conf.d/vault-proxy.conf
<Proxy balancer://vaultcluster>
    BalancerMember http://192.238.1.4:80
    BalancerMember http://192.238.1.5:80
    ProxySet lbmethod=byrequests
</Proxy>
 
ProxyPreserveHost On
RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
 
ProxyPass / balancer://vaultcluster/
ProxyPassReverse / balancer://vaultcluster/
PROXY
 
# --- 3. JALANKAN SERVICE APACHE ---
mkdir -p /run/apache2
pkill -9 httpd 2>/dev/null || true
sleep 1
httpd -k start
 
# --- 4. AUTOSTART PERSISTENSI OPENRC ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
### Langkah 4 - Gerbang core: `abbey` (Nginx reverse proxy)
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. KONFIGURASI DASAR JARINGAN & HOSTNAME ---
NAME="abbey"
IP="192.238.2.2"
GW="192.238.2.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 2. INSTALL & KONFIGURASI NGINX REVERSE PROXY (SOAL 11) ---
apk update && apk add nginx
 
# Folder runtime wajib Nginx di Alpine Linux
mkdir -p /run/nginx
 
# Reverse proxy ke area core (oblada & molly)
cat << 'NGINX' > /etc/nginx/http.d/default.conf
upstream corecluster {
    server 192.238.1.6:80;
    server 192.238.1.7:80;
}
 
server {
    listen 80 default_server;
    listen [::]:80 default_server;
 
    location / {
        proxy_pass http://corecluster;
 
        # Forwarding header Host & X-Real-IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
NGINX
 
# Paksa matikan proses Nginx lama agar port 80 lepas sempurna
pkill -9 nginx 2>/dev/null || true
sleep 1
nginx
 
# --- 3. AUTOSTART PERSISTENSI ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
### Prasyarat DNS untuk pengujian
 
Pengujian memakai `penny.k54.com`, `abbey.k54.com` (sudah dibuat pada Soal 5), serta CNAME **`www`** dan **`static`**. Record CNAME diasumsikan sudah dibuat pada soal sebelumnya. Jika belum, tambahkan di zona `prab`:
 
```text
www     IN  CNAME  penny.k54.com.
static  IN  CNAME  abbey.k54.com.
```
 
Lalu naikkan serial SOA dan muat ulang `named` seperti pada Soal 5.
 
## 4. Pengujian
 
Semua pengujian dijalankan dari terminal **`alpha`**. Blok "ekspektasi" adalah hasil yang seharusnya muncul; tempelkan screenshot output aktual di bagian bertanda **[Screenshot]**.
 
### 4.1 Reverse proxy Apache `penny` -> area vault
 
**Uji respons HTTP (single request):**
 
```bash
curl -i http://penny.k54.com
# atau lewat CNAME www:
curl -i http://www.k54.com
```
 
Ekspektasi: status `HTTP/1.1 200 OK` dan isi halaman dari salah satu backend vault, yaitu `Response dari OBLADI (Vault 1)` atau `Response dari DESMOND (Vault 2)`.

<img width="299" height="248" alt="Screenshot 2026-09-30 at 04 32 13" src="https://github.com/user-attachments/assets/d3a6157e-f100-4e09-804d-75dbdc62f1ca" />

 
**Uji load balancing (pergiliran backend):**
 
```bash
for i in {1..4}; do curl -s http://penny.k54.com; echo "-------------------"; done
```
 
Ekspektasi: jawaban bergantian antara `OBLADI (Vault 1)` dan `DESMOND (Vault 2)` (metode `byrequests`), sehingga dari 4 request muncul masing-masing backend dua kali.
 
<img width="541" height="360" alt="Screenshot 2026-09-30 at 04 31 36" src="https://github.com/user-attachments/assets/c259c360-0200-4f33-bede-57b4c7688624" />

### 4.2 Reverse proxy Nginx `abbey` -> area core
 
**Uji respons HTTP (single request):**
 
```bash
curl -i http://abbey.k54.com
# atau lewat CNAME static:
curl -i http://static.k54.com
```
 
Ekspektasi: status `HTTP/1.1 200 OK` dan halaman `Selamat Datang di Beranda Core (...)` dengan `Server Hostname` `oblada` atau `molly`.
 
<img width="512" height="133" alt="Screenshot 2026-09-30 at 04 32 37" src="https://github.com/user-attachments/assets/175ce6ca-c15c-4d6a-b0ee-60937f5eb0cd" />

**Uji load balancing (pergiliran backend):**
 
```bash
for i in {1..4}; do curl -s http://abbey.k54.com; echo "-------------------"; done
```
 
Ekspektasi: `Server Hostname` bergantian antara `oblada` dan `molly` (round robin bawaan Nginx).
 
<img width="545" height="147" alt="Screenshot 2026-09-30 at 04 33 16" src="https://github.com/user-attachments/assets/dbf3fdee-71e8-4180-9b90-e44cab623bbb" />


# Soal 12: Basic Authentication `/admin` di Penny
 
---
 
## 1. Deskripsi Soal
 
Terdapat ruang khusus di `penny` yang menyimpan dokumen rahasia sindikat. Terapkan perlindungan **basic authentication** untuk path **`/admin`**. Akses ke jalur tersebut harus:
 
- menolak pengunjung **tanpa kredensial**, dan
- hanya mengizinkan masuk jika memakai kredensial berikut:
| Username | Password |
|---|---|
| `prabs` | `pakar_pinter_jadi_gob***` |
 
## 2. Rencana
 
| Komponen | Nilai |
|---|---|
| Path yang dilindungi | `/admin` |
| Jenis autentikasi | `AuthType Basic` |
| Realm | `Restricted Admin Area` |
| File password | `/etc/apache2/.htpasswd` |
| File konfigurasi | `/etc/apache2/conf.d/vault-proxy.conf` |
| Tool pembuat password | `htpasswd` (paket `apache2-utils`) |
| Aturan akses | `Require valid-user` |
 
 
## 3. Langkah Pengerjaan
 
### Langkah 1 - Buat file password (`.htpasswd`)
 
Simpan username `prabs` beserta password-nya di `/etc/apache2/.htpasswd`:
 
```bash
htpasswd -c -b /etc/apache2/.htpasswd prabs 'pakar_pinter_jadi_gob***'
```
 
| Opsi | Fungsi |
|---|---|
| `-c` | Membuat file baru (menimpa jika sudah ada) |
| `-b` | Mengambil password dari argumen perintah (non-interaktif) |
 
Password ditulis dalam tanda kutip tunggal agar karakter `***` tidak diperluas oleh shell.
 
### Langkah 2 - Tambah blok `<Location /admin>`
 
Tambahkan pada `/etc/apache2/conf.d/vault-proxy.conf`:
 
```apache
<Location /admin>
    AuthType Basic
    AuthName "Restricted Admin Area"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Location>
```
 
### Langkah 3 - Skrip lengkap `penny` (Soal 11 + Soal 12)
 
Skrip di bawah menggabungkan konfigurasi reverse proxy Soal 11 dengan basic auth Soal 12, lengkap dengan autostart. Bagian khusus Soal 12 ada di langkah **3** (`htpasswd`) dan **4** (blok `<Location /admin>`).
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. KONFIGURASI DASAR JARINGAN & HOSTNAME ---
NAME="penny"
IP="192.238.3.2"
GW="192.238.3.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 2. INSTALL APACHE & UTILITIES ---
apk update && apk add apache2 apache2-proxy apache2-utils
 
# Set ServerName agar Apache tidak menampilkan warning
# (path file terpotong di screenshot, diasumsikan /etc/apache2/httpd.conf)
sed -i 's/#ServerName www.example.com:80/ServerName penny.k54.com:80/' /etc/apache2/httpd.conf
 
# --- 3. BUAT FILE BASIC AUTHENTICATION (SOAL 12) ---
# Membuat file .htpasswd untuk user 'prabs'
htpasswd -c -b /etc/apache2/.htpasswd prabs 'pakar_pinter_jadi_gob***'
 
# --- 4. KONFIGURASI PROXY & BASIC AUTH UNTUK /admin ---
cat << 'PROXY' > /etc/apache2/conf.d/vault-proxy.conf
# Protection Basic Authentication untuk path /admin (Soal 12)
<Location /admin>
    AuthType Basic
    AuthName "Restricted Admin Area"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Location>
 
# Load Balancer ke Area Vault (obladi & desmond) (Soal 11)
<Proxy balancer://vaultcluster>
    BalancerMember http://192.238.1.4:80
    BalancerMember http://192.238.1.5:80
    ProxySet lbmethod=byrequests
</Proxy>
 
ProxyPreserveHost On
RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
 
ProxyPass / balancer://vaultcluster/
ProxyPassReverse / balancer://vaultcluster/
PROXY
 
# --- 5. JALANKAN SERVICE APACHE ---
mkdir -p /run/apache2
pkill -9 httpd 2>/dev/null || true
sleep 1
httpd -k start
 
# --- 6. AUTOSTART PERSISTENSI OPENRC ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
### Langkah 4 - Siapkan halaman `/admin` di backend (opsional, agar hasil `200 OK`)
 
Tanpa langkah ini, kredensial yang benar tetap lolos autentikasi, tetapi backend membalas `404` karena direktori `/admin` belum ada. Jalankan di **`obladi`** dan **`desmond`** (keduanya, karena balancer bergantian antar backend):
 
```bash
mkdir -p /var/www/html/admin
echo "<h1>Dokumen Rahasia Sindikat</h1>" > /var/www/html/admin/index.html
```
 
## 4. Pengujian
 
Semua pengujian dijalankan dari terminal **`alpha`**. Output di bawah diambil dari hasil pengerjaan.
 
### 4.1 Tanpa kredensial (harus ditolak, 401)
 
```bash
curl -i http://penny.k54.com/admin
```
 
Hasil:
 
```text
HTTP/1.1 401 Unauthorized
Date: Wed, 30 Sep 2026 14:43:34 GMT
Server: Apache/2.4.68 (Unix)
WWW-Authenticate: Basic realm="Restricted Admin Area"
Content-Length: 498
Content-Type: text/html; charset=iso-8859-1
```

Permintaan ditolak dan server meminta kredensial lewat header `WWW-Authenticate` dengan realm `Restricted Admin Area`.

<img width="540" height="157" alt="Screenshot 2026-09-30 at 22 25 31" src="https://github.com/user-attachments/assets/3fc8ee3c-c11e-47ba-b56c-b0eea85b3f7b" />

 
### 4.2 Password salah (harus ditolak, 401)
 
```bash
curl -i -u prabs:salah123 http://penny.k54.com/admin
```
 
Hasil:
 
```text
HTTP/1.1 401 Unauthorized
Date: Wed, 30 Sep 2026 14:44:51 GMT
Server: Apache/2.4.68 (Unix)
WWW-Authenticate: Basic realm="Restricted Admin Area"
Content-Length: 498
Content-Type: text/html; charset=iso-8859-1
```
 
Username yang benar dengan password yang salah tetap ditolak.

<img width="539" height="141" alt="Screenshot 2026-09-30 at 22 26 40" src="https://github.com/user-attachments/assets/3900ef99-2865-4ee1-aaf9-f1270ea96995" />

 
### 4.3 Kredensial benar (harus diizinkan, 200 OK)
 
```bash
curl -i -L -u 'prabs:pakar_pinter_jadi_gob***' http://penny.k54.com/admin
```
 
Hasil:
 
```text
HTTP/1.1 301 Moved Permanently
Date: Wed, 30 Sep 2026 14:53:48 GMT
Server: nginx
Content-Type: text/html
Content-Length: 162
Location: http://penny.k54.com/admin/
 
HTTP/1.1 200 OK
Date: Wed, 30 Sep 2026 14:53:48 GMT
Server: nginx
Content-Type: text/html
Content-Length: 34
Last-Modified: Wed, 30 Sep 2026 14:51:59 GMT
ETag: "6abd220f-22"
Accept-Ranges: bytes
 
<h1>Dokumen Rahasia Sindikat</h1>
```
<img width="557" height="297" alt="Screenshot 2026-09-30 at 22 38 40" src="https://github.com/user-attachments/assets/e1c56c66-00f1-424e-8847-408067680298" />

 
Penjelasan:
 
- Kredensial yang benar lolos dari Apache dan diteruskan ke backend (`Server: nginx`).
- Status `301` muncul karena Nginx di backend mengalihkan `/admin` menjadi `/admin/`. Opsi `-L` membuat `curl` mengikuti pengalihan itu, sehingga akhirnya mendapat `200 OK` dan isi halaman `Dokumen Rahasia Sindikat`.
- Header `Location` bernilai `http://penny.k54.com/admin/` (bukan IP backend), menandakan `ProxyPreserveHost` dan `ProxyPassReverse` bekerja.

# Soal 13: Canonical Name & Redirect 301/302
 
---
 
## 1. Deskripsi Soal
 
Setiap entitas dari luar harus memanggil gerbang dengan **nama kanoniknya**:
 
- Jika ada yang mengakses **IP `penny`** atau domain **`penny.k54.com`**, sistem memaksa **redirect permanen (301)** menuju **`www.k54.com`**.
- Sebaliknya, jika ada yang mengakses **IP `abbey`** atau domain **`abbey.k54.com`**, lakukan **redirect sementara (302)** menuju **`static.k54.com`**.
## 2. Rencana
 
| Diakses lewat | Gerbang | Aksi | Tujuan |
|---|---|---|---|
| `192.238.3.2` (IP penny) | `penny` | Redirect **301** | `http://www.k54.com/` |
| `penny.k54.com` | `penny` | Redirect **301** | `http://www.k54.com/` |
| `www.k54.com` (kanonis) | `penny` | Tanpa redirect, `200 OK` | Konten Vault Area (`obladi`/`desmond`) |
| `192.238.2.2` (IP abbey) | `abbey` | Redirect **302** | `http://static.k54.com/` |
| `abbey.k54.com` | `abbey` | Redirect **302** | `http://static.k54.com/` |
| `static.k54.com` (kanonis) | `abbey` | Tanpa redirect, `200 OK` | Konten Core Area (`oblada`/`molly`) |
 
 
Teknik yang dipakai:
 
| Gerbang | Teknik | Penjelasan |
|---|---|---|
| `penny` (Apache) | `VirtualHost` tunggal dengan `ServerName www.k54.com` + `RewriteCond`/`RewriteRule [R=301,L]` | Semua `Host` selain `www.k54.com` (termasuk IP dan `penny.k54.com`) dialihkan permanen. |
| `abbey` (Nginx) | Dua `server` block: satu untuk `abbey.k54.com` + IP (`return 302`), satu `default_server` untuk `static.k54.com` (proxy) | Host `abbey` dan IP dialihkan sementara, `static.k54.com` dilayani langsung. |
 
## 3. Langkah Pengerjaan
 
### Langkah 1 - Prasyarat DNS
 
Pengujian memakai `www.k54.com` dan `static.k54.com`. Kedua nama ini harus bisa di-resolve klien (record CNAME diasumsikan sudah dibuat pada soal sebelumnya, lihat README Soal 11). Jika belum, tambahkan di zona `prab`, lalu naikkan serial SOA dan muat ulang `named`:
 
```text
www     IN  CNAME  penny.k54.com.
static  IN  CNAME  abbey.k54.com.
```
 
### Langkah 2 - Konfigurasi `penny` (redirect 301)
 
Skrip lengkap `/root/setup.sh` di `penny`. Bagian khusus Soal 13 adalah aktivasi modul `rewrite` dan blok `RewriteCond`/`RewriteRule` di dalam `VirtualHost`.
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. JARINGAN & HOSTNAME ---
NAME="penny"
IP="192.238.3.2"
GW="192.238.3.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 2. INSTALL APACHE & UTILITIES ---
apk update && apk add apache2 apache2-proxy apache2-utils
 
# Aktifkan modul rewrite & proxy
# (path file terpotong di screenshot, diasumsikan /etc/apache2/httpd.conf)
sed -i 's/#LoadModule rewrite_module/LoadModule rewrite_module/' /etc/apache2/httpd.conf
sed -i 's/#LoadModule proxy_module/LoadModule proxy_module/' /etc/apache2/httpd.conf
sed -i 's/#LoadModule proxy_http_module/LoadModule proxy_http_module/' /etc/apache2/httpd.conf
sed -i 's/#LoadModule proxy_balancer_module/LoadModule proxy_balancer_module/' /etc/apache2/httpd.conf
sed -i 's/#LoadModule lbmethod_byrequests_module/LoadModule lbmethod_byrequests_module/' /etc/apache2/httpd.conf
sed -i 's/#LoadModule slotmem_shm_module/LoadModule slotmem_shm_module/' /etc/apache2/httpd.conf
 
# --- 3. BASIC AUTHENTICATION (SOAL 12) ---
htpasswd -c -b /etc/apache2/.htpasswd prabs 'pakar_pinter_jadi_gob***'
 
# --- 4. VIRTUALHOST DENGAN LOGIKA REWRITE (SOAL 11, 12, 13) ---
rm -f /etc/apache2/conf.d/default.conf /etc/apache2/conf.d/vault-proxy.conf
 
cat << 'PROXY' > /etc/apache2/conf.d/vault-proxy.conf
<VirtualHost *:80>
    ServerName www.k54.com
    ServerAlias penny.k54.com 192.238.3.2
 
    # --- SOAL 13: jika Host bukan www.k54.com (mis. IP / subdomain), paksa redirect 301 ---
    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^www\.k54\.com$ [NC]
    RewriteRule ^/(.*)$ http://www.k54.com/$1 [R=301,L]
 
    # --- SOAL 12: proteksi basic auth /admin ---
    <Location /admin>
        AuthType Basic
        AuthName "Restricted Admin Area"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Location>
 
    # --- SOAL 11: load balancer vault area ---
    <Proxy balancer://vaultcluster>
        BalancerMember http://192.238.1.4:80 retry=0
        BalancerMember http://192.238.1.5:80 retry=0
        ProxySet lbmethod=byrequests
    </Proxy>
 
    ProxyPreserveHost On
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
 
    ProxyPass / balancer://vaultcluster/
    ProxyPassReverse / balancer://vaultcluster/
</VirtualHost>
PROXY
 
# --- 5. RESTART SERVICE APACHE ---
mkdir -p /run/apache2
pkill -9 httpd 2>/dev/null || true
sleep 1
httpd -k start
 
# --- 6. AUTOSTART PERSISTENSI OPENRC ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
 
### Langkah 3 - Konfigurasi `abbey` (redirect 302)
 
Skrip lengkap `/root/setup.sh` di `abbey` (jaringan, reverse proxy Soal 11, dan redirect Soal 13):
 
```bash
cat << 'EOF' > /root/setup.sh
#!/bin/bash
 
# --- 1. KONFIGURASI JARINGAN & HOSTNAME ---
NAME="abbey"
IP="192.238.2.2"
GW="192.238.2.1"
 
hostname $NAME && echo "$NAME" > /etc/hostname
 
cat << NET > /etc/network/interfaces
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet static
    address $IP
    netmask 255.255.255.0
    gateway $GW
NET
 
ifup -a 2>/dev/null || true
 
cat << 'RESOLV' > /etc/resolv.conf
nameserver 192.238.1.2
nameserver 192.238.1.3
nameserver 192.168.122.1
RESOLV
 
# --- 2. INSTALL & KONFIGURASI NGINX ---
apk update && apk add nginx
mkdir -p /run/nginx
 
# --- 3. KONFIGURASI PROXY & REDIRECT 302 (SOAL 11 & 13) ---
cat << 'NGINX' > /etc/nginx/http.d/default.conf
upstream corecluster {
    server 192.238.1.6:80;
    server 192.238.1.7:80;
}
 
# Server block redirect 302 (IP & abbey.k54.com -> static.k54.com) (Soal 13)
server {
    listen 80;
    listen [::]:80;
    server_name abbey.k54.com 192.238.2.2;
 
    return 302 http://static.k54.com$request_uri;
}
 
# Server block utama kanonis (static.k54.com) (Soal 11)
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name static.k54.com;
 
    location / {
        proxy_pass http://corecluster;
 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
NGINX
 
# --- 4. JALANKAN SERVICE NGINX ---
pkill -9 nginx 2>/dev/null || true
sleep 1
nginx
 
# --- 5. AUTOSTART PERSISTENSI OPENRC ---
mkdir -p /etc/local.d
echo -e "#!/bin/sh\n/bin/bash /root/setup.sh" > /etc/local.d/setup.start
chmod +x /etc/local.d/setup.start
rc-update add local default 2>/dev/null || true
EOF
 
chmod +x /root/setup.sh && bash /root/setup.sh
```
 
Perbedaan dibanding skrip Soal 11: `server` block lama yang menjadi `default_server` kini hanya melayani `static.k54.com`, dan ditambah satu `server` block baru untuk `abbey.k54.com` dan `192.238.2.2` yang berisi `return 302 http://static.k54.com$request_uri;`. Variabel `$request_uri` membuat path dan query string ikut terbawa ke domain tujuan.
 
## 4. Pengujian
 
Semua pengujian dijalankan dari terminal **`alpha`**. Blok "ekspektasi" adalah hasil yang seharusnya muncul; tempelkan screenshot output aktual di bagian bertanda **[Screenshot]**.
 
### 4.1 Redirect permanen 301 di `penny`
 
**a. Akses via IP penny:**
 
```bash
curl -i http://192.238.3.2
```
 
Ekspektasi:
 
```text
HTTP/1.1 301 Moved Permanently
Location: http://www.k54.com/
```
 
<img width="542" height="101" alt="Screenshot 2026-09-30 at 23 35 43" src="https://github.com/user-attachments/assets/1a9952f0-c274-4f0d-b82b-71413e741973" />

 
**b. Akses via domain subdomain penny:**
 
```bash
curl -i http://penny.k54.com
```
 
Ekspektasi:
 
```text
HTTP/1.1 301 Moved Permanently
Location: http://www.k54.com/
```
 
<img width="422" height="102" alt="Screenshot 2026-09-30 at 23 36 22" src="https://github.com/user-attachments/assets/cd4b26e9-21b7-49bd-a0b6-7369523704bc" />

 
**c. Akses via domain kanonis:**
 
```bash
curl -i http://www.k54.com
```
 
Ekspektasi: `HTTP/1.1 200 OK` dan langsung memuat konten Vault Area (`Response dari OBLADI (Vault 1)` atau `DESMOND (Vault 2)`) tanpa redirect.
 
<img width="347" height="130" alt="Screenshot 2026-09-30 at 23 37 33" src="https://github.com/user-attachments/assets/c9d37d34-25ad-4232-96fd-b4554693a2f4" />

 
### 4.2 Redirect sementara 302 di `abbey`
 
**a. Akses via IP abbey:**
 
```bash
curl -i http://192.238.2.2
```
 
Ekspektasi:
 
```text
HTTP/1.1 302 (Found / Moved Temporarily)
Location: http://static.k54.com/
```
 
<img width="306" height="113" alt="Screenshot 2026-09-30 at 23 38 48" src="https://github.com/user-attachments/assets/50dbe875-a657-4108-ad86-f6ddd8305b28" />

 
**b. Akses via domain subdomain abbey:**
 
```bash
curl -i http://abbey.k54.com
```
 
Ekspektasi:
 
```text
HTTP/1.1 302 (Found / Moved Temporarily)
Location: http://static.k54.com/
```
 
<img width="338" height="113" alt="Screenshot 2026-09-30 at 23 39 55" src="https://github.com/user-attachments/assets/5450ffbf-d580-4e85-8db5-020fd2376837" />

 
**c. Akses via domain kanonis:**
 
```bash
curl -i http://static.k54.com
```
 
Ekspektasi: `HTTP/1.1 200 OK` dan langsung memuat konten Core Area (`Selamat Datang di Beranda Core (Oblada/Molly)`) tanpa redirect.
 
<img width="311" height="111" alt="Screenshot 2026-09-30 at 23 40 32" src="https://github.com/user-attachments/assets/52f943f9-00e9-4126-a1ed-7aadc6365bd6" />

# 16. Melakukan stress testing (benchmark) menggunakan tools ApacheBench (ab) untuk menguji ketahanan dua gerbang reverse proxy

## Langkah 1: Install Package `apache-utils` di Alpine Linux. 

```apk update && apk add apache2-utils```

<img width="1079" height="720" alt="image" src="https://github.com/user-attachments/assets/c7d9918d-4bf2-41e6-b35a-ad1bdc60be9e" />

## Langkah 2: Jalankan stress test ke `[www.k54.com](https://www.k54.com)`

```ab -n 250 -c 10 http://www.k54.com/``` 

- ab: Perintah utama Apache HTTP server benchmarking tool.

- -n 250: Jumlah total request (Number of requests) yang dikirimkan ke server, yaitu sebanyak 250 permintaan.

- -c 10: Tingkat concurrency (jumlah permintaan simultan/serentak dalam satu waktu), yaitu 10 koneksi bersamaan.

- [http://www.k54.com/](http://www.k54.com/): Endpoint target server yang diuji ketahanannya.

<img width="1059" height="530" alt="image" src="https://github.com/user-attachments/assets/98813eda-5bc9-4ce1-b9e3-15932ddc21d5" />  
<img width="1063" height="583" alt="image" src="https://github.com/user-attachments/assets/5c48bd51-ac9c-4ae6-af9f-5828c2b8e188" />  
<img width="1061" height="289" alt="image" src="https://github.com/user-attachments/assets/c0868410-79f9-4f8d-94da-b2023a2855a1" />  

## Langkah 3: Jalankan stress test ke 'static.k54.com`

```ab -n 250 -c 10 http://static.k54.com/```

<img width="1025" height="506" alt="image" src="https://github.com/user-attachments/assets/08966c7a-ea20-40de-8b0f-57d1698f4dcf" />
<img width="1028" height="588" alt="image" src="https://github.com/user-attachments/assets/11ede348-ba1e-4a35-b79f-97ed93cf05d3" />
<img width="1027" height="183" alt="image" src="https://github.com/user-attachments/assets/8027079e-f9b3-4265-bb73-05262709480b" />

# 17. Menambahkan TXT Records pada DNS Master (prab) untuk 5 node client (alphat, beta, gamma, delta dan epsilon)
Jadi ketika melakukan query DNS tipe TXT ke domain klien (misalnya alpha.k54.com), DNS server harus mengembalikan isi teks string berupa nama hostname-nya ("alpha").

## Langkah 1 Perbarui Konfigurasi DNS Zone di node prab
Buka terminal node prab, lalui perbarui scripth di /root/setup.sh menggunakan script dibawah ini.

```
    cat << 'EOF' > /root/setup.sh
    #!/bin/bash
    
    # 1. Network & Hostname
    hostname prab
    echo "prab" > /etc/hostname
    
    # 2. Install BIND9
    apk update && apk add bind bind-tools
    
    mkdir -p /var/bind /etc/bind
    
    # 3. Konfigurasi named.conf
    cat << 'NAMED' > /etc/bind/named.conf
    options {
        directory "/var/bind";
        allow-query { any; };
        auth-nxdomain no;
        listen-on-v6 { none; };
    };
    
    zone "k54.com" {
        type master;
        file "/var/bind/k54.com.zone";
    };
    NAMED
    
    # 4. File Zone DNS dengan TXT Record Klien (Soal 17)
    cat << 'ZONE' > /var/bind/k54.com.zone
    $TTL 86400
    @   IN  SOA prab.k54.com. admin.k54.com. (
            2026092901 ; Serial
            3600       ; Refresh
            1800       ; Retry
            604800     ; Expire
            86400 )    ; Minimum TTL
    
    @       IN  NS  prab.k54.com.
    @       IN  NS  tedd.k54.com.
    
    ; Server Records
    prab    IN  A   192.238.1.2
    tedd    IN  A   192.238.1.3
    obladi  IN  A   192.238.1.4
    desmond IN  A   192.238.1.5
    oblada  IN  A   192.238.1.6
    molly   IN  A   192.238.1.7
    penny   IN  A   192.238.3.2
    abbey   IN  A   192.238.2.2
    
    ; CNAME Alias
    www     IN  CNAME penny.k54.com.
    static  IN  CNAME abbey.k54.com.
    
    ; Client A Records & TXT Records (Soal 17)
    alpha   IN  A   192.238.1.8
    alpha   IN  TXT "alpha"
    
    beta    IN  A   192.238.1.9
    beta    IN  TXT "beta"
    
    gamma   IN  A   192.238.1.10
    gamma   IN  TXT "gamma"
    
    delta   IN  A   192.238.2.3
    delta   IN  TXT "delta"
    
    epsilon IN  A   192.238.3.3
    epsilon IN  TXT "epsilon"
    ZONE
    
    # 5. Restart BIND Service
    pkill -9 named 2>/dev/null || true
    sleep 1
    named -4 -u named
    
    echo "DNS Master Prab berhasil dikonfigurasi dengan TXT Record Soal 17!"
    EOF
    
    chmod +x /root/setup.sh && bash /root/setup.sh
```

<img width="1058" height="649" alt="image" src="https://github.com/user-attachments/assets/e08d0be8-a322-4c19-90ed-eb1a3bc2bcc4" />

## Langkah 2: Pengujian Query TXT record dari Node alpha

Buka terminal client, lalu jalankan perintah query DNS khusus tipe TXT:

Opsi A : Menggunakan nslookup

```nslookup -type=TXT alpha.k54.com```

<img width="592" height="145" alt="image" src="https://github.com/user-attachments/assets/6eebdaa3-7d36-47f2-861d-adff7e53fe4f" />

Opsi B : Menggunakan dig 

```dig TXT alpha.k54.com +short```

<img width="518" height="56" alt="image" src="https://github.com/user-attachments/assets/1491f6e3-0dcb-4576-bf57-214dcdd9e768" />

Pengujian ke node yang lain: 

<img width="582" height="141" alt="image" src="https://github.com/user-attachments/assets/2842964a-e8fa-4142-ae8c-dd8784306f0e" />

<img width="554" height="146" alt="image" src="https://github.com/user-attachments/assets/9e47d20e-9af4-46b0-8983-62b70e421eac" />

<img width="649" height="159" alt="image" src="https://github.com/user-attachments/assets/4b33f87c-6bbe-475d-87f9-d709baecd9a5" />

<img width="571" height="145" alt="image" src="https://github.com/user-attachments/assets/bc53b997-6ea4-421e-a4b4-23628d6ff1ee" />








 
 
