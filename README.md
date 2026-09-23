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

**Opsional:** Jika butuh mengakses database dari luar (misal dari komputer host, atau tools seperti DBeaver/HeidiSQL), tambahkan parameter `-p`:
```bash
docker run -d --name dbserver --network mynet -p 3306:3306 -e MYSQL_ROOT_PASSWORD=pass123 mariadb:11-jammy
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

#### 8. Persiapkan folder untuk dimounting ke DocumentRoot webserver
```bash
sudo mkdir -p /var/mywww
cd /var/mywww
sudo git clone https://github.com/Deri-Nugroho/docker-3.git .
sudo chown -R $USER:$USER /var/mywww
```

#### 9. Run image menjadi container
```bash
docker run -d \
 --name webserver1 \
 --network mynet \
 -p 8001:80 \
 -v /var/mywww:/var/www/html \
 --restart unless-stopped \
 ubuntu-ws:v1
```

**Penjelasan parameter:**

| Parameter | Fungsi |
|-----------|--------|
| `--network mynet` | Menghubungkan container ke network yang sama dengan dbserver, sehingga bisa saling akses menggunakan nama container |
| `-p 8001:80` | Meneruskan port 8001 di host ke port 80 (Apache) di dalam container |
| `-v /var/mywww:/var/www/html` | Mount folder aplikasi dari host ke DocumentRoot Apache, sehingga perubahan file di host langsung terlihat tanpa build ulang image |
| `--restart unless-stopped` | Container otomatis menyala kembali jika Docker/host di-restart |

---

### F. Verifikasi

#### 1. Cek semua container sudah berjalan
```bash
docker ps
```

Pastikan `dbserver` dan `webserver1` berstatus `Up`.

#### 2. Cek aplikasi bisa diakses
```bash
curl http://localhost:8001
```

Atau buka lewat browser: `http://<IP-server>:8001`

#### 3. Cek koneksi webserver ke dbserver (dari dalam container)
```bash
docker exec -it webserver1 bash -c "mysql -h dbserver -u root -ppass123 -e 'SHOW DATABASES;'"
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

### H. Perbandingan dengan LKPD 2 (docker commit vs Dockerfile)

| Aspek | LKPD 2 (docker commit) | LKPD 3 (Dockerfile) |
|-------|------------------------|---------------------|
| **Metode** | Build image secara manual dari container yang berjalan | Build image otomatis dari file konfigurasi |
| **Reproduktif** | Tidak - tergantung langkah manual yang dilakukan | Ya - Dockerfile mendokumentasikan semua langkah |
| **Version Control** | Sulit - perlu commit container setiap perubahan | Mudah - Dockerfile bisa di-track dengan Git |
| **Transparansi** | Tidak jelas apa yang diinstall di dalam container | Jelas - semua perintah terdokumentasi |
| **Best Practice** | Tidak direkomendasikan untuk produksi | Direkomendasikan untuk produksi |

---

### I. Keunggulan Menggunakan Dockerfile

1. **Reproduktif**: Siapapun bisa membangun image yang sama persis dengan menjalankan `docker build`
2. **Version Control**: Dockerfile bisa disimpan di Git dan di-track perubahannya
3. **Transparan**: Semua langkah instalasi dan konfigurasi terdokumentasi dengan jelas
4. **Otomatis**: Build process otomatis tanpa intervensi manual
5. **Best Practice**: Cara standar industri untuk membangun Docker image

---

## Informasi Aplikasi

Aplikasi ini adalah **Toko Sederhana** berbasis web menggunakan PHP native (mysqli), Bootstrap 5, dan MySQL.

### Fitur Utama
- 🔐 Login multi-role: `admin`, `kasir`, `gudang` 
- 📦 Manajemen Barang: tambah, edit, hapus, upload foto, pencarian
- 🏷️ Manajemen Kategori barang
- 🧾 Point of Sale (POS): keranjang belanja, hitung kembalian, cetak struk
- 📊 Laporan Penjualan: filter tanggal, total omset, barang terlaris
- 👥 Manajemen User: tambah/edit/hapus user & role (khusus admin)
- ⚙️ Auto setup database: tabel & data dummy dibuat otomatis jika belum ada

### Akun Demo (Dummy)

| Username | Password | Role   |
|----------|----------|--------|
| admin    | 123      | admin  |
| kasir    | 123      | kasir  |
| gudang   | 123      | gudang  |

> ⚠️ **Penting:** Ganti password akun-akun ini sebelum digunakan di lingkungan produksi.

---

## Catatan Keamanan

Password "pass123" pada dokumen ini hanya untuk keperluan pembelajaran/lokal. Untuk lingkungan produksi:
- Gunakan password yang kuat
- Pertimbangkan menyimpan kredensial melalui Docker secret atau file `.env`
- Jangan mengekspose port database ke host jika tidak diperlukan
- Gunakan environment variable untuk menyimpan konfigurasi sensitif
