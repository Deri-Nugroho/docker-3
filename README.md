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

