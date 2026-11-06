# Tugas-3-ADMINISTRASI-SERVER---STSI4302

# Praktikum FTP Server Menggunakan vsftpd pada CentOS Stream 10

## 1. Pendahuluan

FTP (*File Transfer Protocol*) merupakan protokol jaringan yang digunakan untuk melakukan pertukaran file antara komputer client dan server. Dengan menggunakan FTP, pengguna dapat mengirimkan maupun mengambil file melalui jaringan menggunakan layanan FTP Server.

Praktikum ini bertujuan untuk melakukan instalasi dan konfigurasi FTP Server menggunakan layanan **vsftpd** pada sistem operasi CentOS Stream 10. Dalam praktik ini, CentOS Stream 10 digunakan sebagai server, sedangkan Windows 11 digunakan sebagai client. Keduanya dihubungkan menggunakan jaringan Host-Only pada VirtualBox.

Selain instalasi dan konfigurasi layanan FTP, praktikum ini juga mencakup pengaturan firewall, hak akses direktori, dan SELinux. Pengujian dilakukan dengan mengakses FTP Server melalui Windows, mengirimkan file TXT dari Windows ke CentOS, membuat file TXT pada CentOS untuk dibaca melalui Windows, serta mengirimkan dan membuka file gambar pada server.

## 2. Persiapan Awal

Sebelum melakukan instalasi dan konfigurasi, berikut perangkat dan pengaturan yang digunakan dalam praktikum:

| Komponen | Keterangan |
|---|---|
| Sistem operasi server | CentOS Stream 10 |
| Sistem operasi client | Windows 11 |
| Virtualisasi | Oracle VM VirtualBox |
| FTP Server | vsftpd versi 3.0.5 |
| Adapter 1 | NAT |
| Adapter 2 | Host-Only Adapter |
| IP Address CentOS | `192.168.56.102` |
| IP Address Windows | `192.168.56.1` |
| Akun FTP | `ftpuser` |
| Direktori utama FTP | `/var/www/html` |
| Direktori upload | `/var/www/html/upload` |

Adapter NAT digunakan agar mesin virtual dapat terhubung ke internet, sedangkan Host-Only Adapter digunakan agar Windows sebagai client dapat berkomunikasi langsung dengan CentOS sebagai server.

## 3. Langkah Instalasi dan Konfigurasi FTP Server

### 3.1 Update Sistem

Langkah pertama adalah memperbarui sistem operasi CentOS Stream 10 agar paket yang digunakan berada dalam kondisi terbaru.

Karena praktikum ini menggunakan CentOS Stream 10, perintah yang digunakan adalah `dnf` sebagai pengganti `yum` pada CentOS 7.

```bash
dnf update -y
```

Setelah perintah dijalankan, tunggu sampai proses pembaruan selesai.

### 3.2 Instalasi vsftpd

Setelah sistem diperbarui, langkah selanjutnya adalah memasang aplikasi vsftpd yang akan digunakan sebagai FTP Server.

```bash
dnf install vsftpd -y
```

Jika instalasi sudah selesai, jalankan layanan FTP dengan perintah:

```bash
systemctl start vsftpd
```

Agar layanan FTP otomatis aktif ketika CentOS dinyalakan, jalankan:

```bash
systemctl enable vsftpd
```

Kemudian periksa apakah layanan sudah berjalan:

```bash
systemctl status vsftpd
```

Jika muncul keterangan `active (running)`, berarti layanan vsftpd sudah berhasil dijalankan.

### 3.3 Konfigurasi Firewall

Selanjutnya, firewall diatur agar komputer Windows dapat terhubung dengan FTP Server.

Pertama, izinkan koneksi melalui port 21 yang digunakan oleh FTP:

```bash
firewall-cmd --permanent --add-port=21/tcp
```

Kemudian izinkan layanan FTP pada firewall:

```bash
firewall-cmd --permanent --add-service=ftp
```

Karena konfigurasi menggunakan passive mode untuk proses transfer data, tambahkan rentang port berikut:

```bash
firewall-cmd --permanent --add-port=10000-10100/tcp
```

Setelah semua aturan ditambahkan, terapkan perubahan dengan perintah:

```bash
firewall-cmd --reload
```

Untuk memastikan pengaturan sudah diterapkan, jalankan:

```bash
firewall-cmd --list-all
```

Pada hasil pemeriksaan, pastikan layanan FTP dan port yang diperlukan sudah tercantum.

### 3.4 Konfigurasi File vsftpd

Setelah firewall diatur, langkah berikutnya adalah menyesuaikan konfigurasi FTP Server.

Buka file konfigurasi menggunakan perintah:

```bash
nano /etc/vsftpd/vsftpd.conf
```

Di dalam file tersebut, sesuaikan pengaturan berikut:

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

Pengaturan tersebut digunakan agar FTP hanya dapat diakses oleh pengguna yang memiliki akun, mengizinkan pengguna melakukan upload file, dan membatasi akses pengguna pada direktori FTP yang telah ditentukan.

Passive mode juga diaktifkan dengan menentukan rentang port 10000 sampai 10100 agar proses pertukaran data dapat berjalan.

Setelah selesai mengubah konfigurasi, simpan file dengan menekan `Ctrl + O`, tekan Enter, lalu keluar menggunakan `Ctrl + X`.

Selanjutnya, siapkan file daftar pengecualian chroot:

```bash
touch /etc/vsftpd/chroot_list
```

Pada praktikum ini, akun `ftpuser` tidak dimasukkan ke dalam daftar tersebut agar tetap berada dalam pembatasan direktori FTP.

### 3.5 Membuat User FTP

Setelah konfigurasi vsftpd selesai, langkah berikutnya adalah membuat akun yang nantinya digunakan untuk login ke FTP Server.

Buat akun dengan perintah:

```bash
useradd -m -s /sbin/nologin ftpuser
```

Kemudian buat password untuk akun tersebut:

```bash
passwd ftpuser
```

Masukkan password yang akan digunakan ketika login dari Windows.

Untuk memastikan akun sudah berhasil dibuat, jalankan:

```bash
id ftpuser
```

Kemudian periksa pengaturan shell pengguna:

```bash
getent passwd ftpuser
```

Pastikan bagian akhir hasil pemeriksaan menunjukkan `/sbin/nologin`.

Dengan pengaturan tersebut, akun `ftpuser` dapat digunakan untuk mengakses layanan FTP tanpa memberikan akses login terminal secara langsung.

### 3.6 Mengatur Hak Akses Direktori

Berikutnya, kita menyiapkan direktori yang akan digunakan sebagai tempat penyimpanan file FTP.

Buat direktori utama beserta folder upload:

```bash
mkdir -p /var/www/html/upload
```

Setelah direktori dibuat, atur agar direktori utama tetap dimiliki oleh root:

```bash
chown root:root /var/www/html
chmod 755 /var/www/html
```

Kemudian berikan kepemilikan folder upload kepada akun FTP:

```bash
chown ftpuser:ftpuser /var/www/html/upload
chmod 755 /var/www/html/upload
```

Periksa hasilnya menggunakan:

```bash
ls -ld /var/www/html
ls -ld /var/www/html/upload
```

Pada pengaturan ini, direktori utama tetap dimiliki oleh root, sedangkan folder upload dimiliki oleh `ftpuser`. Tujuannya agar pengguna FTP dapat mengirimkan file ke folder upload tanpa memiliki hak untuk mengubah direktori utama.

Pengaturan ini merupakan penyesuaian dari contoh `chown -R` pada panduan praktikum agar sesuai dengan pembatasan chroot yang digunakan.

### 3.7 Penyesuaian SELinux

Pada CentOS Stream 10 terdapat SELinux yang membantu membatasi akses layanan terhadap file dan direktori. Oleh karena itu, pengaturannya perlu disesuaikan agar FTP Server dapat membaca direktori dan menerima file yang diunggah.

Pertama, periksa status SELinux:

```bash
getenforce
```

Selanjutnya, pastikan perintah `semanage` tersedia. Jika belum, pasang paket pendukung berikut:

```bash
dnf install policycoreutils-python-utils -y
```

Atur konteks SELinux pada direktori utama FTP beserta isinya:

```bash
semanage fcontext -a -t public_content_t '/var/www/html(/.*)?'
```

Kemudian atur konteks khusus untuk folder upload:

```bash
semanage fcontext -a -t public_content_rw_t '/var/www/html/upload(/.*)?'
```

Terapkan pengaturan tersebut:

```bash
restorecon -Rv /var/www/html
```

Setelah itu, aktifkan pengaturan SELinux yang diperlukan untuk akses tulis FTP:

```bash
setsebool -P ftpd_anon_write on
```

Pengaturan ini dilakukan agar akses FTP terhadap direktori sesuai dengan kebijakan SELinux tanpa harus menonaktifkan SELinux.

### 3.8 Restart Layanan

Setelah seluruh konfigurasi selesai, jalankan ulang layanan vsftpd agar perubahan yang telah dilakukan dapat diterapkan.

```bash
systemctl restart vsftpd
```

Kemudian periksa kembali status layanan:

```bash
systemctl status vsftpd
```

Jika status menunjukkan `active (running)`, berarti layanan FTP Server sudah berjalan setelah konfigurasi dilakukan.

## 4. Pengujian FTP

### 4.1 Mengakses FTP Server dari Windows

Setelah FTP Server siap, selanjutnya dilakukan pengujian dari komputer Windows sebagai client.

Pertama, buka File Explorer pada Windows. Pada bagian address bar, masukkan alamat FTP Server menggunakan IP Host-Only CentOS.

```text
ftp://192.168.56.102
```

Setelah alamat dibuka, masukkan username `ftpuser` dan password yang telah dibuat sebelumnya.

Jika berhasil login, direktori FTP akan tampil pada File Explorer. Untuk masuk ke folder upload, buka alamat:

```text
ftp://192.168.56.102/upload/
```

### 4.2 Pengujian File TXT dari Windows ke CentOS

Untuk menguji pengiriman file dari Windows ke server, pertama buat sebuah file teks pada Windows.

Buka PowerShell dan jalankan:

```powershell
"Ini adalah file pengujian FTP yang dibuat di Windows dan dikirim ke server CentOS Stream 10." | Set-Content "$env:USERPROFILE\Documents\uji-windows.txt"
```

Setelah file dibuat, kirimkan ke folder upload menggunakan perintah:

```powershell
curl.exe --user ftpuser -T "$env:USERPROFILE\Documents\uji-windows.txt" ftp://192.168.56.102/upload/uji-windows.txt
```

Kemudian kembali ke terminal CentOS dan periksa apakah file sudah masuk:

```bash
ls -lh /var/www/html/upload/
```

Untuk melihat isi file, gunakan:

```bash
cat /var/www/html/upload/uji-windows.txt
```

Jika nama dan isi file tampil, berarti pengiriman file dari Windows ke CentOS berhasil.

### 4.3 Pengujian File TXT dari CentOS ke Windows

Selanjutnya, dilakukan pengujian dari arah sebaliknya, yaitu membuat file pada CentOS dan membacanya melalui Windows.

Buat file teks pada folder upload dengan perintah:

```bash
echo "Ini adalah file pengujian FTP yang dibuat di CentOS Stream 10 dan dibaca melalui Windows." > /var/www/html/upload/uji-centos.txt
```

Atur kepemilikan dan izin file:

```bash
chown ftpuser:ftpuser /var/www/html/upload/uji-centos.txt
chmod 644 /var/www/html/upload/uji-centos.txt
```

Setelah file dibuat, buka PowerShell pada Windows dan jalankan:

```powershell
curl.exe --user ftpuser ftp://192.168.56.102/upload/uji-centos.txt
```

Jika isi file muncul di PowerShell, berarti file yang dibuat pada CentOS berhasil dibaca melalui Windows.

### 4.4 Pengujian File Gambar dari Windows ke CentOS

Pengujian terakhir dilakukan dengan mengirimkan file gambar dari Windows ke FTP Server.

Pertama, buka kembali File Explorer dan akses folder FTP:

```text
ftp://192.168.56.102/upload/
```

Pilih salah satu gambar yang tersimpan pada Windows, kemudian salin dan tempelkan ke folder upload tersebut.

Setelah proses pengiriman selesai, kembali ke CentOS dan periksa file yang sudah masuk:

```bash
ls -lh /var/www/html/upload/
```

Untuk memastikan file dikenali sebagai gambar, jalankan:

```bash
file /var/www/html/upload/nama-gambar.jpg
```

Ganti `nama-gambar.jpg` sesuai nama file gambar yang dikirim.

Selanjutnya, buka File Manager CentOS, tekan `Ctrl + L`, lalu masukkan lokasi:

```text
/var/www/html/upload
```

Cari gambar yang sudah dikirim dan buka untuk memastikan file dapat dibaca.

## 5. Hasil Praktikum

Berdasarkan pengujian yang dilakukan, hasil praktikum yang diperoleh meliputi:

1. FTP Server dapat dijalankan menggunakan layanan vsftpd pada CentOS Stream 10.
2. Windows dapat mengakses FTP Server menggunakan akun `ftpuser`.
3. File TXT dari Windows dapat dikirim dan dibaca pada CentOS.
4. File TXT yang dibuat pada CentOS dapat dibaca melalui Windows.
5. File gambar dari Windows dapat dikirim dan dibuka pada CentOS.

## 6. Kesimpulan

Berdasarkan praktikum yang telah dilakukan, instalasi dan konfigurasi FTP Server menggunakan vsftpd pada CentOS Stream 10 berhasil dilaksanakan. Layanan FTP dapat dijalankan dan diakses melalui Windows 11 menggunakan alamat IP server `192.168.56.102`.

Proses konfigurasi meliputi instalasi layanan, pembuatan akun FTP, pengaturan file konfigurasi vsftpd, pengaturan firewall, pemberian hak akses direktori, serta penyesuaian kebijakan SELinux. Penyesuaian SELinux diperlukan karena sistem menggunakan mode `Enforcing`, sehingga akses layanan terhadap direktori FTP harus disesuaikan dengan kebijakan keamanan yang berlaku.

Hasil pengujian menunjukkan bahwa FTP Server dapat digunakan untuk melakukan pertukaran file. File TXT yang dibuat di Windows dapat dikirim dan dibaca melalui CentOS, sedangkan file TXT yang dibuat pada CentOS dapat dibaca melalui Windows. Pengiriman dan pembukaan file gambar juga dilakukan sebagai bagian dari pengujian.

Dengan demikian, praktikum ini memberikan pemahaman mengenai proses instalasi, konfigurasi, pengamanan, dan pengujian FTP Server pada lingkungan virtualisasi menggunakan CentOS Stream 10.

## 7. Penyesuaian Perintah dari Panduan Praktikum

Dalam pelaksanaan praktikum terdapat beberapa penyesuaian karena sistem operasi yang digunakan adalah CentOS Stream 10, sedangkan panduan awal menggunakan CentOS 7.

Penyesuaian yang dilakukan meliputi:

1. **Pengelolaan paket sistem.** Perintah `dnf` digunakan untuk memperbarui sistem dan menginstal layanan vsftpd karena CentOS Stream 10 menggunakan DNF sebagai pengelola paket.
2. **Penyesuaian jaringan.** Pemeriksaan dan pengaturan jaringan dilakukan menggunakan NetworkManager. Interface yang digunakan pada VirtualBox adalah `enp0s3` untuk NAT dan `enp0s8` untuk Host-Only.
3. **Pengaturan firewall.** Selain mengizinkan layanan FTP dan port 21, rentang port 10000–10100 juga diizinkan karena konfigurasi vsftpd menggunakan passive FTP.
4. **Pengaturan direktori FTP.** Direktori utama `/var/www/html` tetap dimiliki oleh root, sedangkan direktori `/var/www/html/upload` diberikan kepada pengguna `ftpuser` agar dapat digunakan untuk mengirimkan file.
5. **Penyesuaian SELinux.** Karena SELinux berada dalam mode `Enforcing`, dilakukan penyesuaian konteks keamanan pada direktori utama dan direktori upload agar layanan FTP dapat mengakses direktori sesuai kebijakan keamanan.
6. **Pengujian transfer file.** Pengujian dilakukan melalui Windows PowerShell dan File Explorer. File TXT diuji dalam dua arah, sedangkan file gambar dikirim melalui File Explorer dan dibuka melalui File Manager CentOS.

Penyesuaian tersebut dilakukan agar konfigurasi FTP dapat berjalan pada CentOS Stream 10 dan tujuan praktikum tetap dapat tercapai.

## Daftar Pustaka

Arifin, Z. (2026). *Administrasi Server (MSIM4308, Edisi 1, Cetakan ke-12).* Tangerang Selatan: Universitas Terbuka.

Red Hat. (2025). *Securing networks: Securing the FTP service*. Red Hat Enterprise Linux 10 Documentation.  
https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/securing_networks/securing-network-services
