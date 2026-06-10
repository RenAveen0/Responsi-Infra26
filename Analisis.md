# Identitas Diri
Nama: Hana Nur Fathiyyah\
NIM: H1H024017\
Shift KRS: C

# Analisis Perbaikan

## 1. Kesalahan Sintaksis Utama pada Orchestration (Docker Compose)

### A. Root Services Error
* **Gejala:** Berkas `docker-compose.yml` sama sekali tidak dapat dieksekusi oleh Docker daemon. Muncul pesan kesalahan berupa `yaml: line X: ...` atau `services must be a mapping` saat menjalankan perintah `docker compose up -d`.
* **Penyebab:** Terjadi kesalahan penulisan (*syntax error*) pada baris paling atas berkas konfigurasi. Kata kunci root `services` tidak diakhiri dengan tanda titik dua (`:`), sehingga parser YAML gagal mengidentifikasi blok di bawahnya sebagai kumpulan sub-service infrastruktur.
* **Solusi:** Menambahkan tanda titik dua (`:`) tepat setelah kata `services` menjadi `services:`.

### B. Ketidakcocokan Deklarasi Volume Global (Data Persistence)
* **Gejala:** Kontainer database `mysql-db` langsung mati sesaat setelah dinyalakan, memunculkan status `Exited (1)` pada pemeriksaan `docker ps -a`. 
* **Penyebab:** Terjadi ketidaksinkronan penamaan volume antara blok internal service database dengan blok deklarasi volume global di bagian paling bawah file. Service `db` memanggil nama volume `- db-data:/var/lib/mysql`, tetapi deklarasi root terbawah justru menuliskan `database-data:`.
* **Solusi:** Mengubah penamaan deklarasi volume global di baris paling bawah berkas `docker-compose.yml` dari `database-data:` menjadi `db-data:` agar sinkron dengan volume eksternal yang dialokasikan oleh kontainer database.

---

## 2. Kesalahan Konfigurasi Jaringan & Variabel Lingkungan (Networking & Environment)

### A. Kesalahan Target Host Database pada Web-1
* **Gejala:** Aplikasi Web-1 berhasil berjalan, namun memunculkan error *Database Connection Refused* saat mencoba memproses data mahasiswa dari database.
* **Penyebab:** Pada konfigurasi environment `web1`, variabel lingkungan `DB_HOST` diisi dengan nilai `mysql`. Nama service database yang tertera pada berkas konfigurasi adalah `db`, bukan `mysql`.
* **Solusi:** Mengubah baris konfigurasi dari `DB_HOST: mysql` menjadi `DB_HOST: db`.

### B. Kesalahan Kredensial Database pada Web-2
* **Gejala:** Aplikasi Web-2 memunculkan error *Access Denied for User* saat mencoba melakukan autentikasi ke database MySQL.
* **Penyebab:** Variabel lingkungan `DB_PASS` pada konfigurasi service `web2` diisi dengan string `wrongpassword`, yang mana tidak cocok dengan password inisialisasi database utama pada service `db` (`student123`).
* **Solusi:** Mengubah parameter `DB_PASS: wrongpassword` pada service `web2` menjadi `DB_PASS: student123`.

### C. Isolasi Jaringan pada Web-3
* **Gejala:** Kontainer `web3` berstatus `Up`, namun load balancer Nginx gagal meneruskan request ke server Web-3 (menyebabkan error 502 Bad Gateway di sisi Nginx) atau Web-3 tidak dapat dijangkau oleh load balancer.
* **Penyebab:** Kontainer `web3` hanya dimasukkan ke dalam jaringan `backend`. Padahal, agar dapat berkomunikasi langsung dengan Nginx yang berada di gardu depan, kontainer tersebut juga harus menjangkau jaringan `frontend`.
* **Solusi:** Menambahkan deklarasi network `- frontend` di bawah sub-bagian `networks:` pada konfigurasi service `web3` di berkas `docker-compose.yml`.

---

## 3. Kesalahan Target Build Context & Dockerfile (Image Building)

### A. Kesalahan Path Folder Context Web-3
* **Gejala:** Proses `docker compose up -d --build` langsung gagal di awal karena Docker tidak dapat menemukan lokasi Dockerfile untuk service `web3`.
* **Penyebab:** Parameter `context` untuk service `web3` diarahkan ke folder `./web33`. Folder tersebut tidak eksis di dalam direktori proyek, karena nama folder yang benar adalah `./web3`.
* **Solusi:** Mengubah baris instruksi `context: ./web33` pada service `web3` menjadi `context: ./web3`.

### B. Kesalahan Typo Base Image pada Dockerfile (Web-1 & Web-3)
* **Gejala:** Proses pembuatan image (*building image*) gagal total pada tahap pembacaan instruksi `FROM` di dalam berkas Dockerfile milik `web1` dan `web3`.
* **Sebab:** Terjadi kesalahan pengetikan nama base image resmi PHP-Apache. Pada `web1/Dockerfile` tertulis `FROM php:8.2-apach` (kurang huruf 'e') dan pada `web3/Dockerfile` tertulis `FROM php:8.2-apche` (kurang huruf 'a').
* **Solusi:** Memperbaiki baris pertama pada berkas `web1/Dockerfile` dan `web3/Dockerfile` menjadi `FROM php:8.2-apache`.

---

## 4. Kesalahan Konfigurasi Reverse Proxy & Load Balancing (Nginx)

### A. Kesalahan Alamat Anggota Upstream Web-1
* **Gejala:** Kontainer `nginx-lb` langsung mati dengan status `Exited (1)`. Berdasarkan log, Nginx mengeluhkan *host not found* atau gagal melakukan resolve DNS backend.
* **Penyebab:** Di dalam berkas `nginx/nginx.conf`, pada bagian blok `upstream backend`, baris pertama berisi instruksi `server web11:80;`. Nama host `web11` tidak dikenali oleh DNS internal Docker karena nama service yang benar di `docker-compose.yml` adalah `web1`.
* **Solusi:** Mengubah entri `server web11:80;` di dalam `nginx/nginx.conf` menjadi `server web1:80;`.

### B. Kesalahan Port Target Upstream Web-3
* **Gejala:** Saat load balancer mengarahkan lalu lintas ke Web-3, koneksi mengalami kegagalan (*Connection Refused* atau *502 Bad Gateway*).
* **Penyebab:** Pada blok `upstream backend` di `nginx/nginx.conf`, Web-3 diarahkan ke port 8080 (`server web3:8080;`). Padahal, base image `php:8.2-apache` yang digunakan oleh Web-3 secara internal menjalankan web servernya pada port default `80`. Port `8080` hanya digunakan sebagai gerbang luar publik milik Nginx.
* **Solusi:** Mengubah entri `server web3:8080;` di dalam `nginx/nginx.conf` menjadi `server web3:80;`.

---

## 5. Kesalahan Sinkronisasi Konten Tampilan & Inisialisasi Data

### A. Typo Identitas Nama Kontainer (Web-2 & Web-3)
* **Gejala:** Aplikasi berhasil diakses via `curl localhost:8080` secara bergantian. Namun, teks yang tercetak di layar tidak konsisten (menampilkan nama kontainer yang salah, seperti `WEB-WEB` atau `WEB-WOB`).
* **Penyebab:** Terdapat kesalahan ketik (*hardcoded typo*) pada elemen HTML `<strong>` di dalam berkas `web2/index.php` dan `web3/index.php`.
* **Solusi:** * Mengubah elemen `<strong>WEB-WEB</strong>` menjadi `<strong>WEB-2</strong>` pada file `web2/index.php`.
    * Mengubah elemen `<strong>WEB-WOB</strong>` menjadi `<strong>WEB-3</strong>` pada file `web3/index.php`.

### B. Kegagalan Inisialisasi Data Default Riwayat Database
* **Gejala:** Tabel database berhasil dibuat, tetapi saat aplikasi web melakukan query data praktikan, records data yang muncul kosong atau masih berupa teks placeholder mentah.
* **Penyebab:** File inisialisasi query database `db/init.sql` masih memuat nilai placeholder mentah berupa `'REPLACE_NIM'` dan `'REPLACE_NAME'`.
* **Solusi:** Mengganti string penampung `'REPLACE_NIM'` dan `'REPLACE_NAME'`.