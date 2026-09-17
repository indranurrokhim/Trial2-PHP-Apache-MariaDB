# Trial 2 --- PHP + Apache + MariaDB

Project ini adalah project belajar Docker untuk memahami cara
menjalankan aplikasi PHP menggunakan **Apache sebagai web server**,
berbeda dari `trial1` yang menggunakan Nginx + PHP-FPM.

Pada project ini, Apache dan PHP berada dalam **satu container**
menggunakan official PHP image:

``` text
php:8.4-apache
```

## 1. Struktur Project

``` text
trial2/
├── docker/
│   └── php/
│       └── Dockerfile
├── src/
│   ├── index.php
│   └── test.php
├── .gitignore
├── docker-compose.yml
└── README.md
```

Berdasarkan struktur project yang terlihat:

  File / Directory          Fungsi
  ------------------------- ---------------------------------------
  `docker/php/Dockerfile`   Membuat custom image PHP + Apache
  `src/`                    Source code aplikasi PHP
  `src/index.php`           File PHP utama
  `src/test.php`            File testing PHP
  `.gitignore`              File yang tidak perlu disimpan di Git
  `docker-compose.yml`      Mendefinisikan service/container
  `README.md`               Dokumentasi project

> Isi `index.php`, `test.php`, dan `.gitignore` tidak diberikan pada
> saat dokumentasi ini dibuat, sehingga README ini tidak mengasumsikan
> isi file-file tersebut.

## 2. Konsep Utama Trial2

`trial2` dibuat untuk membandingkan architecture dengan `trial1`.

### Trial1

``` text
Browser
   │
   ▼
Nginx container
   │
   │ FastCGI
   ▼
PHP-FPM container
   │
   ▼
MariaDB container
```

### Trial2

``` text
Browser
   │
   ▼
Apache + PHP container
   │
   ▼
MariaDB container
```

Pada `trial2`, **Apache dan PHP berjalan dalam container yang sama**.

Tidak diperlukan konfigurasi Nginx dan tidak diperlukan `fastcgi_pass`.

## 3. Dockerfile

Isi `docker/php/Dockerfile`:

``` dockerfile
FROM composer/composer:2-bin AS composer

FROM php:8.4-apache

COPY --from=composer /composer /usr/bin/composer

RUN docker-php-ext-install pdo_mysql

WORKDIR /var/www/html
```

### `FROM composer/composer:2-bin`

``` dockerfile
FROM composer/composer:2-bin AS composer
```

Digunakan sebagai temporary build stage untuk mengambil binary Composer.

Stage ini diberi nama:

``` text
composer
```

### `FROM php:8.4-apache`

``` dockerfile
FROM php:8.4-apache
```

Ini adalah bagian utama image.

Image sudah menyediakan:

-   PHP 8.4
-   Apache HTTP Server
-   integrasi PHP dengan Apache

Jadi berbeda dengan `trial1` yang menggunakan:

``` text
php:8.4-fpm
```

Pada `trial2`, PHP menggunakan:

``` text
php:8.4-apache
```

Artinya Apache dan PHP berada dalam satu container.

### Composer

``` dockerfile
COPY --from=composer /composer /usr/bin/composer
```

Binary Composer dari build stage sebelumnya disalin ke image utama.

Dengan demikian container `app` memiliki command:

``` bash
composer
```

### PDO MySQL

``` dockerfile
RUN docker-php-ext-install pdo_mysql
```

Extension `pdo_mysql` dipasang agar PHP dapat berkomunikasi dengan
MySQL/MariaDB melalui PDO.

### Working Directory

``` dockerfile
WORKDIR /var/www/html
```

Directory kerja aplikasi di dalam container adalah:

``` text
/var/www/html
```

## 4. Docker Compose

`docker-compose.yml` mendefinisikan tiga service:

``` text
app
db
phpmyadmin
```

### Service `app`

``` yaml
app:
  build:
    context: .
    dockerfile: docker/php/Dockerfile
  ports:
    - "8090:80"
  volumes:
    - ./src:/var/www/html
```

Image untuk `app` dibuat dari Dockerfile:

``` text
docker/php/Dockerfile
```

Port mapping:

``` text
Host :8090 → Container :80
```

Karena Apache berjalan pada port `80` di dalam container.

Source code:

``` text
./src
```

di-mount ke:

``` text
/var/www/html
```

di dalam container.

Karena menggunakan bind mount, perubahan source code di host langsung
terlihat oleh Apache/PHP container.

### Service `db`

``` yaml
db:
  image: mariadb:11
```

Database menggunakan MariaDB 11.

Credential yang digunakan oleh project belajar:

``` text
Database : trial2
Username : trial2
Password : trial2
Root     : root
```

Data MariaDB menggunakan named volume:

``` yaml
volumes:
  db_data:
    /var/lib/mysql
```

Tujuannya agar data database tetap tersimpan ketika container database
dibuat ulang.

> Credential ini hanya cocok untuk learning project. Untuk production,
> gunakan environment variable atau secret management dan jangan
> menyimpan credential production di repository.

### Service `phpmyadmin`

``` yaml
phpmyadmin:
  image: phpmyadmin:latest
  environment:
    PMA_HOST: db
  ports:
    - "8091:80"
```

phpMyAdmin digunakan untuk mengelola MariaDB melalui browser.

Port:

``` text
Host :8091 → Container :80
```

phpMyAdmin diarahkan ke service database:

``` text
db
```

## 5. Arsitektur

Secara sederhana:

``` text
                 Docker Host
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
  :8090                       :8091
        │                         │
        ▼                         ▼
┌─────────────────┐       ┌─────────────────┐
│      app        │       │   phpMyAdmin    │
│                 │       │                 │
│ Apache + PHP    │       └────────┬────────┘
└────────┬────────┘                │
         │                         │
         └───────────┬─────────────┘
                     ▼
              ┌─────────────┐
              │   MariaDB   │
              │     db      │
              └─────────────┘
```

Perbedaan penting dari `trial1`:

``` text
trial1:
Nginx → PHP-FPM
       2 container

trial2:
Apache + PHP
       1 container
```

## 6. Mengapa Apache + PHP Bisa Dalam Satu Container?

Image:

``` text
php:8.4-apache
```

memang disediakan untuk menjalankan PHP melalui Apache.

Apache menangani HTTP request:

``` text
Browser
   ↓
Apache
   ↓
PHP
```

Tidak diperlukan FastCGI antara Apache dan PHP seperti pada
architecture:

``` text
Nginx
   ↓ FastCGI
PHP-FPM
```

Karena itu konfigurasi `trial2` lebih sederhana dibanding `trial1`.

## 7. Menjalankan Project

Masuk ke directory project:

``` bash
cd trial2
```

Build image dan menjalankan container:

``` bash
docker compose up -d --build
```

Periksa status:

``` bash
docker compose ps
```

Jika berhasil, terdapat service:

``` text
app
db
phpmyadmin
```

## 8. Mengakses Web

Aplikasi PHP:

``` text
http://localhost:8090
```

phpMyAdmin:

``` text
http://localhost:8091
```

Untuk koneksi phpMyAdmin ke MariaDB:

``` text
Server   : db
Username : trial2
Password : trial2
Database : trial2
```

`db` digunakan sebagai hostname karena Docker Compose menyediakan
internal DNS berdasarkan service name.

## 9. Mengecek PHP dan Apache

Masuk ke container:

``` bash
docker compose exec app bash
```

Cek PHP:

``` bash
php -v
```

Cek Composer:

``` bash
composer --version
```

Cek extension:

``` bash
php -m
```

Keluar:

``` bash
exit
```

## 10. Mengecek Log

Semua service:

``` bash
docker compose logs
```

Service Apache/PHP:

``` bash
docker compose logs app
```

Database:

``` bash
docker compose logs db
```

phpMyAdmin:

``` bash
docker compose logs phpmyadmin
```

Untuk mengikuti log secara realtime:

``` bash
docker compose logs -f app
```

## 11. Stop dan Start

Menghentikan container:

``` bash
docker compose stop
```

Menjalankan kembali:

``` bash
docker compose start
```

Menghapus container dan network:

``` bash
docker compose down
```

Menghapus container sekaligus named volume database:

``` bash
docker compose down -v
```

> `docker compose down -v` akan menghapus volume `db_data`, sehingga
> data MariaDB ikut hilang. Gunakan hanya jika memang ingin reset
> database.

## 12. Bind Mount

Project menggunakan:

``` yaml
volumes:
  - ./src:/var/www/html
```

Artinya:

``` text
Host
trial2/src
    │
    │ bind mount
    ▼
Container
/var/www/html
```

Keuntungannya untuk development:

``` text
edit source code
      ↓
file langsung berubah di container
      ↓
Apache menjalankan source terbaru
```

Jadi perubahan file PHP biasanya tidak membutuhkan rebuild image.

## 13. Docker Network

Docker Compose membuat network internal untuk service.

Karena itu service dapat menggunakan nama service sebagai hostname.

Contoh:

``` text
app → db
phpmyadmin → db
```

Jika aplikasi PHP nanti membutuhkan database, konfigurasi koneksi dapat
menggunakan:

``` text
Host     : db
Port     : 3306
Database : trial2
Username : trial2
Password : trial2
```

**Jangan menggunakan `localhost` untuk koneksi dari `app` ke MariaDB.**

Di dalam container `app`:

``` text
localhost
```

mengacu ke container `app` itu sendiri, bukan container `db`.

## 14. Build vs Run

Jika hanya mengubah:

``` text
src/index.php
src/test.php
```

tidak perlu:

``` bash
docker compose build
```

karena `src` menggunakan bind mount.

Cukup refresh browser.

Tetapi jika mengubah:

``` text
docker/php/Dockerfile
```

misalnya menambahkan PHP extension:

``` dockerfile
RUN docker-php-ext-install pdo_mysql mysqli
```

maka image harus dibuat ulang:

``` bash
docker compose up -d --build
```

Secara sederhana:

``` text
ubah source code
→ tidak perlu rebuild

ubah Dockerfile
→ perlu rebuild
```

## 15. Trial2 sebagai Pembelajaran

Project ini membantu memahami bahwa Docker tidak mengharuskan:

``` text
1 application = 1 container
```

Yang lebih penting adalah memahami service dan responsibility.

Pada `trial2`:

``` text
app
├── Apache
└── PHP

db
└── MariaDB

phpmyadmin
└── Database management UI
```

Sedangkan pada `trial1`:

``` text
web
└── Nginx

app
└── PHP-FPM

db
└── MariaDB

phpmyadmin
└── Database management UI
```

Keduanya valid untuk pembelajaran, tetapi architecture dan cara request
diproses berbeda.

## 16. Perbandingan Trial1 dan Trial2

  Bagian              Trial1          Trial2
  ------------------- --------------- ------------------
  Web Server          Nginx           Apache
  PHP Runtime         PHP-FPM         PHP + Apache
  PHP Image           `php:8.4-fpm`   `php:8.4-apache`
  Web/PHP Container   Terpisah        Satu container
  FastCGI             Ya              Tidak
  Nginx Config        Ada             Tidak
  Apache Config       Default image   Default image
  MariaDB             11              11
  phpMyAdmin          Ada             Ada
  App Port            `8080`          `8090`
  phpMyAdmin Port     `8081`          `8091`

## 17. Kapan Architecture Apache + PHP Berguna?

Model seperti `trial2` cukup praktis untuk:

-   learning project
-   aplikasi PHP sederhana
-   development environment
-   aplikasi yang memang membutuhkan Apache
-   setup yang ingin lebih sederhana

Model ini mengurangi jumlah container karena web server dan PHP berada
dalam image yang sama.

## 18. Kapan Menggunakan Nginx + PHP-FPM?

Model seperti `trial1` memisahkan:

``` text
Web Server
   ↓
Application Runtime
```

Hal ini memberikan separation yang lebih jelas dan lebih fleksibel untuk
architecture yang lebih kompleks.

Contohnya:

``` text
Internet
   ↓
Reverse Proxy / Load Balancer
   ↓
Nginx
   ↓
PHP-FPM
```

Namun untuk belajar Docker, kedua model penting dipahami karena membantu
memahami bahwa Docker adalah cara **mengemas dan menjalankan service**,
bukan sekadar mengganti XAMPP/Laragon.

## 19. Next Step

Setelah `trial1` dan `trial2`, pembelajaran dapat dilanjutkan dengan:

``` text
Trial1
Nginx + PHP-FPM
       ↓
Trial2
Apache + PHP
       ↓
Trial3
Production-oriented Docker setup
       ↓
Laravel
       ↓
Git + GitHub
       ↓
Development / Production
       ↓
Deploy ke Ubuntu Server
```

Tujuan akhirnya adalah memahami workflow:

``` text
Developer
   ↓
Git
   ↓
Docker Image
   ↓
Docker Compose
   ↓
Ubuntu Server
   ↓
Production Application
```

## Catatan

`trial2` adalah **learning project**, bukan production configuration.

Konfigurasi yang sengaja dibuat sederhana antara lain:

-   Database credential ditulis langsung di `docker-compose.yml`.
-   `phpmyadmin:latest` digunakan.
-   Source code menggunakan bind mount.
-   Apache menggunakan konfigurasi default dari image.
-   Belum menggunakan HTTPS.
-   Belum menggunakan reverse proxy.
-   Belum menggunakan secrets.
-   Belum menggunakan production-specific configuration.
-   Belum ada healthcheck.
