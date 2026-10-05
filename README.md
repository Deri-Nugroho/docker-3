# LKPD 3 - Membangun Image dengan Dockerfile

## Petunjuk Awal
Instance/mesin yang digunakan adalah Ubuntu/Debian.

## Langkah Kerja

### A. Persiapkan Lingkungan Kerja Container Docker

#### 1. Instal paket yang dibutuhkan
```bash
sudo apt update && sudo apt install -y git nano curl links mc docker.io nmap
sudo usermod -aG docker $USERMU
sudo newgrp docker
```

Contoh jika usernya "deri":
```bash
sudo usermod -aG docker deri
sudo newgrp docker
```

#### 2. Pull image yang diperlukan
```bash
docker pull ubuntu:24.04
docker pull mariadb:11-jammy
```

Lihat hasilnya:
```bash
docker image ls
```

#### 3. Buat jaringan yang akan digunakan untuk komunikasi antar container
```bash
docker network create mynet
```

---

### B. Jalankan Container dbserver

#### 4. Jalankan container image mariadb:11-jammy tanpa mengekspose port ke host
Ini lebih aman karena database hanya bisa diakses oleh container lain yang berada pada network yang sama (mynet).

```bash
docker run -d --name dbserver --network mynet -e MYSQL_ROOT_PASSWORD=pass123 mariadb:11-jammy
```

---

### C. Buat Dockerfile

#### 5. Buat direktori kerja dan file Dockerfile
```bash
mkdir ~/bws
cd ~/bws
```

**Opsi 1: Clone repository ini (rekomendasi)**
```bash
git clone https://github.com/Deri-Nugroho/docker-3.git .
```

Dengan cara ini, Dockerfile dan file-file yang dibutuhkan akan otomatis terisi tanpa perlu input manual.

**Opsi 2: Buat Dockerfile manual**
```bash
nano Dockerfile
```

Isi dengan:
```dockerfile
FROM ubuntu:24.04

# Supaya apt install tidak nanya interaktif
ENV DEBIAN_FRONTEND=noninteractive

# Install Apache, PHP, dan modul yang dibutuhkan aplikasi
RUN apt update && apt install -y \
	nano \
	apache2 \
	php \
	php-mysqli \
	php-mysql \
	libapache2-mod-php \
	mysql-client \
	curl \
	&& apt clean && rm -rf /var/lib/apt/lists/*

# Expose port HTTP
EXPOSE 80

# Jalankan Apache sebagai proses utama (foreground)
CMD ["apache2ctl", "-D", "FOREGROUND"]
```

---

### D. Build Image dari Dockerfile

#### 6. Build image-nya
```bash
docker build -t ubuntu-ws:v1 ~/bws/
```

#### 7. Lihat image-nya
```bash
docker image ls
```

---

### E. Coba Run Container dari Image Baru

#### 7. Run image menjadi container
```bash
docker run -d \
 --name webserver1 \
 --network mynet \
 -p 8001:80 \
 -v /var/mywww:/var/www/html \
 --restart unless-stopped \
 ubuntu-ws:v1
```

---

### F. Verifikasi dan Akses Web Server

#### 8. Persiapkan folder aplikasi dan uploads
Jika folder `/var/mywww` belum berisi aplikasi, clone repository LKPD 2:
```bash
sudo mkdir -p /var/mywww
sudo git clone https://github.com/Deri-Nugroho/docker-2.git /var/mywww
sudo chown -R $USER:$USER /var/mywww
sudo mkdir -p /var/mywww/uploads
sudo chown -R www-data:www-data /var/mywww/uploads
sudo chmod 755 /var/mywww/uploads
```

#### 9. Buat database yang dibutuhkan aplikasi
```bash
docker exec -it dbserver mariadb -u root -ppass123 -e "CREATE DATABASE toko_db;"
```

#### 10. Verifikasi semua container berjalan
```bash
docker ps
```
Pastikan `dbserver` dan `webserver1` berstatus `Up`.

#### 11. Cek aplikasi bisa diakses
```bash
curl http://localhost:8001
```
Atau buka lewat browser: `http://<IP-server>:8001`

#### 12. Cek koneksi webserver ke dbserver (dari dalam container)
```bash
docker exec -it webserver1 bash -c "mysql -h dbserver -u root -ppass123 -e 'SHOW DATABASES;'"
```

#### 13. Verifikasi tabel database dan data dummy
```bash
# Cek tabel
docker exec -it webserver1 bash -c "mysql -h dbserver -u root -ppass123 toko_db -e 'SHOW TABLES;'"

# Cek data users
docker exec -it webserver1 bash -c "mysql -h dbserver -u root -ppass123 toko_db -e 'SELECT username, nama_lengkap, role FROM users;'"

# Cek data barang
docker exec -it webserver1 bash -c "mysql -h dbserver -u root -ppass123 toko_db -e 'SELECT * FROM barang;'"
```

---

### G. Perintah Bantuan (Troubleshooting)

| Kebutuhan | Command |
|-----------|---------|
| Lihat log container | `docker logs -f webserver1` |
| Masuk ke shell container yang jalan | `docker exec -it webserver1 bash` |
| Restart container | `docker restart webserver1` |
| Lihat network & container yang tergabung | `docker network inspect mynet` |
| Hapus semua container (reset total) | `docker rm -f webserver1 dbserver` |
| Hapus network | `docker network rm mynet` |

---

### Informasi Aplikasi

Aplikasi ini adalah Toko Sederhana berbasis web menggunakan PHP native (mysqli), Bootstrap 5, dan MySQL.

**Fitur Utama:**
- 🔐 Login multi-role: `admin`, `kasir`, `gudang`
- 📦 Manajemen Barang: tambah, edit, hapus, upload foto, pencarian
- 🏷️ Manajemen Kategori barang
- 🧾 Point of Sale (POS): keranjang belanja, hitung kembalian, cetak struk
- 📊 Laporan Penjualan: filter tanggal, total omset, barang terlaris
- 👥 Manajemen User: tambah/edit/hapus user & role (khusus admin)
- ⚙️ Auto setup database: tabel & data dummy dibuat otomatis jika belum ada

**Akun Demo (Dummy):**
| Username | Password | Role |
|----------|----------|------|
| admin | 123 | admin |
| kasir | 123 | kasir |
| gudang | 123 | gudang |

⚠️ **Penting:** Ganti password akun-akun ini sebelum digunakan di lingkungan produksi.

---

### Catatan
Password "pass123" pada dokumen ini hanya untuk keperluan pembelajaran/lokal. Untuk lingkungan produksi, gunakan password yang kuat dan pertimbangkan menyimpan kredensial melalui Docker secret atau file `.env`, bukan langsung di command line.

