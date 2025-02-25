Berikut adalah langkah-langkah lengkap untuk mengonfigurasi WireGuard VPN antara AlmaLinux 9 dan dua router MikroTik:

---

### **1. Instal dan Konfigurasi WireGuard di AlmaLinux 9**
#### **1.1 Instal WireGuard**
Jalankan perintah berikut untuk menginstal WireGuard di AlmaLinux 9:
```bash
dnf install -y epel-release
dnf install -y wireguard-tools
```

#### **1.2 Generate Key Pair**
Buat kunci private dan public untuk server:
```bash
umask 077
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
```
Cek hasilnya:
```bash
cat /etc/wireguard/privatekey
cat /etc/wireguard/publickey
```

#### **1.3 Konfigurasi WireGuard Server**
Buat file konfigurasi WireGuard:
```bash
nano /etc/wireguard/wg0.conf
```
Tambahkan konfigurasi berikut:
```
[Interface]
Address = 10.9.9.1/24
ListenPort = 51820
PrivateKey = [ISI_PRIVATE_KEY_SERVER]
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = [ISI_PUBLIC_KEY_MIKROTIK_1]
AllowedIPs = 10.9.9.2/32

[Peer]
PublicKey = [ISI_PUBLIC_KEY_MIKROTIK_2]
AllowedIPs = 10.9.9.3/32
```
Gantilah `[ISI_PRIVATE_KEY_SERVER]`, `[ISI_PUBLIC_KEY_MIKROTIK_1]`, dan `[ISI_PUBLIC_KEY_MIKROTIK_2]` dengan key yang sesuai.

#### **1.4 Mengaktifkan dan Menjalankan WireGuard**
```bash
systemctl enable --now wg-quick@wg0
systemctl status wg-quick@wg0
```

#### **1.5 Konfigurasi Firewall**
Izinkan port WireGuard dan aktifkan IP forwarding:
```bash
firewall-cmd --permanent --add-port=51820/udp
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

---

### **2. Konfigurasi MikroTik (Client)**
#### **2.1 Generate Key Pair di MikroTik**
Di MikroTik, masuk ke terminal dan jalankan:
```shell
/interface wireguard key
```
Simpan `private-key` dan `public-key` untuk digunakan nanti.

#### **2.2 Konfigurasi Interface WireGuard di MikroTik 1**
```shell
/interface wireguard add name=wg0 listen-port=51820 private-key=[ISI_PRIVATE_KEY_MIKROTIK_1]
/interface wireguard peers add interface=wg0 public-key=[ISI_PUBLIC_KEY_SERVER] endpoint-address=103.77.107.254 endpoint-port=51820 allowed-address=10.9.9.0/24 persistent-keepalive=25
/ip address add address=10.9.9.2/24 interface=wg0
```
Gantilah `[ISI_PRIVATE_KEY_MIKROTIK_1]` dan `[ISI_PUBLIC_KEY_SERVER]` sesuai dengan yang dihasilkan sebelumnya.

#### **2.3 Konfigurasi Interface WireGuard di MikroTik 2**
```shell
/interface wireguard add name=wg0 listen-port=51820 private-key=[ISI_PRIVATE_KEY_MIKROTIK_2]
/interface wireguard peers add interface=wg0 public-key=[ISI_PUBLIC_KEY_SERVER] endpoint-address=103.77.107.254 endpoint-port=51820 allowed-address=10.9.9.0/24 persistent-keepalive=25
/ip address add address=10.9.9.3/24 interface=wg0
```

#### **2.4 Konfigurasi Firewall di MikroTik**
Tambahkan rule untuk mengizinkan trafik melalui WireGuard:
```shell
/ip firewall filter add chain=input action=accept protocol=udp dst-port=51820
/ip firewall filter add chain=forward action=accept in-interface=wg0
```

#### **2.5 Konfigurasi Routing**
Jika ingin semua trafik MikroTik diarahkan melalui VPN:
```shell
/ip route add dst-address=0.0.0.0/0 gateway=10.9.9.1
```
Jika hanya ingin rute ke subnet tertentu:
```shell
/ip route add dst-address=10.9.9.0/24 gateway=wg0
```

---

### **3. Verifikasi Koneksi**
Di server, jalankan:
```bash
wg show
```
Pastikan MikroTik muncul sebagai peer.

Di MikroTik, lakukan ping:
```shell
ping 10.9.9.1
```
Jika ping berhasil, konfigurasi sudah berjalan dengan baik.

---

Dengan konfigurasi ini, MikroTik 1 dan MikroTik 2 bisa terhubung ke server WireGuard di AlmaLinux 9 dan berkomunikasi satu sama lain melalui VPN. 🚀
