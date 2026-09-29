# Jarkom-Modul-2-2026-K-54

| Nama                 | NRP        |
|----------------------|------------|
|  Viko Rizky Fauzan | 5027251017   |
| Sean Arthur Tamajaya | 5027251050 |

# Laporan Praktikum: Konfigurasi IP & Routing "The Mesh"

**Proyek GNS3:** `K-54-MODUL-2`
**Prefix IP kelompok:** `192.238.0.0`
**OS node:** Alpine Linux

## 1. Tujuan

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

## 6. Kesimpulan

- `rootkit` berhasil menjadi router pusat yang menghubungkan lima gerbang (Switch) ke lima subnet `192.238.X.0/24` dan satu jalur keluar ke NAT.
- Seluruh Entitas memperoleh IP statis dan *default gateway* menuju `rootkit` pada subnet masing-masing.
- Pengujian membuktikan konektivitas ke internet (`8.8.8.8`, `1.1.1.1`) dan routing antar subnet (`alpha` → `delta`, `alpha` → `prab`) berhasil dengan 0% packet loss.

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



      





   

   
