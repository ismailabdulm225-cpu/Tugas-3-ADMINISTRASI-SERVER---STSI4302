# Praktikum Administrasi Server: FTP Server pada CentOS Stream 10

Dokumentasi Tugas Tutorial Online 3 mata kuliah **Administrasi Server (STSI4302)**. Praktikum ini membahas instalasi, konfigurasi, dan pengujian FTP Server menggunakan **vsftpd** pada CentOS Stream 10 dengan Windows 11 sebagai client.

## 1. Pendahuluan

FTP (*File Transfer Protocol*) merupakan protokol jaringan yang digunakan untuk melakukan pertukaran file antara komputer client dan server. Pada praktikum ini, CentOS Stream 10 digunakan sebagai server, sedangkan Windows 11 digunakan sebagai client. Kedua perangkat dihubungkan menggunakan jaringan Host-Only pada Oracle VM VirtualBox.

Praktikum mencakup instalasi dan konfigurasi vsftpd, pengaturan firewall, akun FTP, hak akses direktori, serta SELinux. Pengujian dilakukan dengan login FTP, mengirim file TXT dari Windows ke CentOS, mengirim file foto melalui Windows File Explorer, dan membuat file TXT pada CentOS untuk dibaca melalui Windows.

## 2. Persiapan Awal

### Lingkungan praktikum

| Komponen | Keterangan |
|---|---|
| Mata kuliah | Administrasi Server (STSI4302) |
| Sistem operasi server | CentOS Stream 10 |
| Layanan FTP | vsftpd |
| Sistem operasi client | Windows 11 |
| Virtualisasi | Oracle VM VirtualBox |
| Adapter jaringan | NAT dan Host-Only |
| Interface NAT | `enp0s3` |
| Interface Host-Only | `enp0s8` |
| IP CentOS Stream 10 | `192.168.56.103` |
| IP Windows Host-Only | `192.168.56.1` |
| Akun FTP | `ftpuser` |
| Direktori utama FTP | `/var/www/html` |
| Direktori upload | `/var/www/html/upload` |

Alamat IP tersebut mengikuti konfigurasi mesin virtual pada praktikum ini. Jika diterapkan pada lingkungan lain, sesuaikan alamat IP dengan hasil konfigurasi jaringan masing-masing.

### Memeriksa sistem operasi

```bash
cat /etc/os-release
```

Perintah ini digunakan untuk mengetahui sistem operasi yang digunakan.

### Memeriksa alamat IPv4

```bash
ip -4 a
```

Perintah ini menampilkan alamat IPv4 pada perangkat jaringan. Adapter Host-Only `enp0s8` digunakan sebagai jalur komunikasi antara Windows dan CentOS.

## 3. Instalasi dan Konfigurasi FTP Server

### 3.1 Update sistem

```bash
dnf update -y
```

Perintah `dnf update` digunakan untuk memeriksa dan memperbarui paket pada CentOS Stream 10. Opsi `-y` menyetujui konfirmasi pembaruan secara otomatis. Pada CentOS Stream 10 digunakan `dnf` sebagai pengelola paket.

### 3.2 Instalasi vsftpd

```bash
dnf install vsftpd -y
```

Setelah paket tersedia, jalankan dan aktifkan layanan:

```bash
systemctl start vsftpd
systemctl enable vsftpd
systemctl status vsftpd
```

- `start` menjalankan layanan.
- `enable` membuat layanan aktif otomatis ketika sistem dinyalakan.
- `status` memeriksa kondisi layanan. Status `active (running)` menunjukkan layanan sedang berjalan.

### 3.3 Konfigurasi firewall

Izinkan port 21 yang digunakan FTP:

```bash
firewall-cmd --permanent --add-port=21/tcp
```

Izinkan layanan FTP:

```bash
firewall-cmd --permanent --add-service=ftp
```

Izinkan rentang port untuk koneksi passive mode:

```bash
firewall-cmd --permanent --add-port=10000-10100/tcp
```

Terapkan aturan firewall:

```bash
firewall-cmd --reload
```

Port 21 digunakan untuk koneksi kontrol FTP, sedangkan rentang 10000–10100 digunakan untuk koneksi data passive mode sesuai konfigurasi vsftpd.

### 3.4 Konfigurasi file vsftpd

Buka file konfigurasi:

```bash
nano /etc/vsftpd/vsftpd.conf
```

Parameter yang digunakan:

```ini
anonymous_enable=NO
local_enable=YES
write_enable=YES

chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list

local_root=/var/www/html

pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10100
```

Penjelasan parameter:

| Parameter | Fungsi |
|---|---|
| `anonymous_enable=NO` | Menonaktifkan login anonim. |
| `local_enable=YES` | Mengizinkan pengguna lokal Linux login ke FTP Server. |
| `write_enable=YES` | Mengizinkan perintah FTP yang berkaitan dengan penulisan atau perubahan file, sesuai hak akses Linux dan SELinux. |
| `chroot_local_user=YES` | Membatasi ruang akses pengguna lokal melalui mekanisme chroot. |
| `chroot_list_enable=YES` | Mengaktifkan penggunaan daftar pengecualian chroot. |
| `chroot_list_file=/etc/vsftpd/chroot_list` | Menentukan lokasi file daftar pengecualian chroot. |
| `local_root=/var/www/html` | Menentukan direktori awal pengguna lokal setelah login FTP. |
| `pasv_enable=YES` | Mengaktifkan passive mode. |
| `pasv_min_port=10000` | Menentukan batas bawah rentang port passive mode. |
| `pasv_max_port=10100` | Menentukan batas atas rentang port passive mode. |

**Memeriksa file chroot_list**

Pada mesin praktikum, file `/etc/vsftpd/chroot_list` sudah tersedia dari konfigurasi sebelumnya. Oleh karena itu, keberadaannya diperiksa menggunakan:

```bash
ls -l /etc/vsftpd/chroot_list
```

Jika file belum tersedia pada sistem lain, file kosong dapat dibuat dengan:

```bash
touch /etc/vsftpd/chroot_list
```

Dengan `chroot_local_user=YES`, pengguna yang tercantum dalam daftar tersebut menjadi pengecualian dari pembatasan chroot. Karena itu, akun `ftpuser` tidak dimasukkan ke dalam daftar agar tetap dibatasi.

Setelah memeriksa dan menyesuaikan parameter, simpan file Nano dengan menekan `Ctrl + O`, lalu `Enter`, kemudian `Ctrl + X`.

### 3.5 Membuat atau memeriksa user FTP

Akun FTP yang digunakan adalah `ftpuser`. Periksa keberadaan akun:

```bash
id ftpuser
```

Jika akun belum tersedia, buat akun menggunakan:

```bash
useradd -m -s /sbin/nologin ftpuser
```

Kemudian atur kata sandinya:

```bash
passwd ftpuser
```

Pada mesin virtual yang digunakan dalam praktikum ini, akun `ftpuser` sudah tersedia dari konfigurasi sebelumnya sehingga akun tersebut digunakan kembali. Pembuatan akun tidak perlu diulang apabila akun sudah ada.

### 3.6 Mengatur hak akses direktori

Buat direktori untuk menyimpan file hasil transfer:

```bash
mkdir -p /var/www/html/upload
```

Berikan kepemilikan direktori kepada akun FTP:

```bash
chown ftpuser:ftpuser /var/www/html/upload
```

Atur hak akses direktori:

```bash
chmod 755 /var/www/html/upload
```

Periksa hasil pengaturan:

```bash
ls -ld /var/www/html /var/www/html/upload
```

Direktori utama `/var/www/html` tetap dimiliki `root`, sedangkan direktori `/var/www/html/upload` dimiliki `ftpuser`. Hak akses `755` memberikan pemilik hak membaca, menulis, dan memasuki direktori, sedangkan grup dan pengguna lainnya memiliki hak membaca dan memasuki direktori tanpa hak menulis.

Pengaturan ini memungkinkan `ftpuser` mengirim file ke direktori `upload` tanpa memiliki hak untuk menulis langsung pada direktori utama `/var/www/html`. Perintah menggunakan alamat direktori lengkap sehingga tidak perlu berpindah direktori dengan `cd` terlebih dahulu.

### 3.7 Penyesuaian SELinux

Periksa status SELinux:

```bash
getenforce
```

Pada praktikum ini, SELinux dipertahankan dalam mode `Enforcing`.

Periksa konteks keamanan direktori:

```bash
ls -Zd /var/www/html /var/www/html/upload
```

Konteks yang digunakan:

- `/var/www/html` menggunakan `public_content_t`.
- `/var/www/html/upload` menggunakan `public_content_rw_t`.

Aturan konteks kedua direktori sudah tersedia dari konfigurasi sebelumnya. Oleh karena itu, pada pelaksanaan praktikum ini aturan tidak dibuat ulang menggunakan `semanage fcontext`.

**Perintah alternatif jika aturan konteks belum tersedia pada sistem:**

Jika perintah `semanage` belum tersedia, pasang paket pendukung:

```bash
dnf install policycoreutils-python-utils -y
```

Atur konteks direktori utama:

```bash
semanage fcontext -a -t public_content_t '/var/www/html'
```

Atur konteks direktori upload:

```bash
semanage fcontext -a -t public_content_rw_t '/var/www/html/upload(/.*)?'
```

Setelah direktori upload tersedia, terapkan aturan konteks yang telah terdaftar:

```bash
restorecon -Rv /var/www/html
```

Periksa kembali konteks direktori:

```bash
ls -Zd /var/www/html /var/www/html/upload
```

Periksa Boolean SELinux:

```bash
getsebool ftpd_anon_write
```

Pada sistem yang digunakan, Boolean tersebut telah bernilai `on`. SELinux tetap dipertahankan aktif dan tidak dinonaktifkan. Hak akses Linux dan konteks SELinux perlu sesuai agar layanan FTP dapat mengakses direktori menurut kebijakan keamanan.

### 3.8 Restart layanan

Setelah konfigurasi selesai, jalankan ulang layanan:

```bash
systemctl restart vsftpd
```

Periksa status layanan:

```bash
systemctl status vsftpd
```

Status `active (running)` menunjukkan layanan vsftpd sedang berjalan.

## 4. Pengujian FTP Server

Pengujian dilakukan dari Windows 11 sebagai client dengan alamat server `192.168.56.103`.

### 4.1 Menguji koneksi dari Windows ke CentOS

Buka Windows PowerShell dan jalankan:

```powershell
Test-NetConnection 192.168.56.103 -Port 21
```

Jika hasil menunjukkan:

```text
TcpTestSucceeded : True
```

berarti koneksi TCP dari Windows menuju port 21 pada server berhasil dijangkau. Pengujian ini memeriksa keterjangkauan port, belum membuktikan login FTP berhasil.

### 4.2 Menguji login ke FTP Server

Jalankan melalui PowerShell:

```powershell
curl.exe -v --user ftpuser ftp://192.168.56.103/
```

Masukkan kata sandi akun `ftpuser` ketika diminta. Opsi `-v` menampilkan informasi proses komunikasi. Respons `230 Login successful` menunjukkan autentikasi berhasil.

### 4.3 Membuat dan mengirim file teks dari Windows ke CentOS

Buat file teks melalui PowerShell:

```powershell
"Ini adalah file pengujian FTP tugas 3 Administrasi server." | Set-Content "$env:USERPROFILE\Documents\uji-coba.txt"
```

Kirim file ke direktori upload server:

```powershell
curl.exe --user ftpuser -T "$env:USERPROFILE\Documents\uji-coba.txt" ftp://192.168.56.103/upload/uji-coba.txt
```

Periksa file pada CentOS:

```bash
ls -lh /var/www/html/upload/
```

Baca isi file:

```bash
cat /var/www/html/upload/uji-coba.txt
```

### 4.4 Mengirim file foto dari Windows ke CentOS melalui File Explorer

Pada Windows File Explorer, buka alamat:

```text
ftp://192.168.56.103/upload/
```

Pilih file foto dari komputer Windows, kemudian lakukan drag and drop ke direktori `upload` yang terbuka pada File Explorer.

Setelah proses pengiriman, periksa file pada CentOS:

```bash
ls -lh /var/www/html/upload/
```

Untuk memeriksa jenis file gambar, gunakan perintah berikut dengan menyesuaikan nama file:

```bash
file /var/www/html/upload/nama-file-foto.jpg
```

Ganti `nama-file-foto.jpg` dengan nama file yang benar-benar dikirim.

### 4.5 Membuat file teks di CentOS dan membacanya melalui Windows

Buat file teks pada CentOS:

```bash
echo "Ini pengujian dari CentOS untuk dibaca diWindows saya." > /var/www/html/upload/uji-centos.txt
```

Sesuaikan kepemilikan file:

```bash
chown ftpuser:ftpuser /var/www/html/upload/uji-centos.txt
```

Pada Windows File Explorer, buka:

```text
ftp://192.168.56.103/upload/
```

Salin file `uji-centos.txt` dari direktori FTP ke komputer Windows, misalnya ke folder Downloads, kemudian buka file tersebut untuk membaca isinya.

## 5. Hasil Praktikum

Hasil praktikum diperoleh melalui pemeriksaan layanan vsftpd, pengujian koneksi jaringan, proses login, serta kegiatan pengiriman dan pengambilan file antara Windows dan CentOS Stream 10.

Pengujian meliputi pengiriman file teks dari Windows ke CentOS, pengiriman foto melalui File Explorer, serta pembuatan file teks pada CentOS yang kemudian dibaca melalui Windows. Bukti pengujian dapat ditampilkan melalui keluaran perintah terminal dan tampilan File Explorer.

## 6. Kesimpulan

Praktikum ini membahas instalasi dan konfigurasi FTP Server menggunakan vsftpd pada CentOS Stream 10. Agar layanan dapat digunakan, diperlukan konfigurasi vsftpd, akun pengguna, direktori penyimpanan, firewall, hak akses Linux, dan SELinux.

Pengujian menggunakan Windows 11 dilakukan untuk melihat proses koneksi, login, pengiriman file dari client ke server, serta pembacaan file yang dibuat pada server melalui client.

## 7. Penyesuaian Perintah dari Panduan Praktikum

Panduan awal menggunakan CentOS 7, sedangkan pelaksanaan praktikum menggunakan CentOS Stream 10. Penyesuaian yang dilakukan meliputi:

1. **Pengelolaan paket sistem:** menggunakan `dnf` untuk memperbarui sistem dan menginstal vsftpd.
2. **Penyesuaian jaringan:** pemeriksaan jaringan dilakukan dengan melihat interface `enp0s3` untuk NAT dan `enp0s8` untuk Host-Only.
3. **Pengaturan firewall:** mengizinkan layanan FTP, port 21, dan rentang port 10000–10100 untuk passive mode.
4. **Pengaturan direktori FTP:** direktori utama `/var/www/html` tetap dimiliki root, sedangkan direktori `/var/www/html/upload` diberikan kepada `ftpuser`.
5. **Penyesuaian SELinux:** mempertahankan SELinux dalam mode `Enforcing` dan menggunakan konteks keamanan yang sesuai. Aturan `semanage fcontext` tidak dibuat ulang karena telah tersedia dari konfigurasi sebelumnya.
6. **Pengujian transfer file:** menggunakan Windows PowerShell dan File Explorer. File TXT diuji dalam dua arah, sedangkan file gambar dikirim melalui File Explorer dan diperiksa pada CentOS.

## Daftar Pustaka

Arifin, Z. (2026). *Administrasi Server (MSIM4308, Edisi 1, Cetakan ke-12).* Tangerang Selatan: Universitas Terbuka.

Red Hat. (2025). *Securing networks: Securing the FTP service.* Red Hat Enterprise Linux 10 Documentation. https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/securing_networks/securing-network-services
