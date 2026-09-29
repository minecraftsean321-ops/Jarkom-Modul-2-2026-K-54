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




      





   

   
