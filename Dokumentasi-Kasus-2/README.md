# Dokumentasi Praktikum Keamanan Jaringan --- Kasus 2

README ini merangkum praktikum **Kasus 2 (K2), Pertemuan 1--7**. Folder
`Pertemuan1` sampai `Pertemuan7` menyimpan screenshot/bukti visual,
sedangkan dokumen ini menjelaskan tujuan, konsep, proses, hasil, dan
kesimpulan setiap praktikum.

> **Batas penggunaan:** Seluruh pengujian keamanan hanya dilakukan pada
> container/lab milik sendiri atau sistem yang secara jelas memberikan
> izin. Jangan menguji jaringan maupun perangkat pihak lain tanpa
> persetujuan.

## Struktur folder

``` text
Dokumentasi-Kasus-2/
├── README.md
├── Pertemuan1/
├── Pertemuan2/
├── Pertemuan3/
├── Pertemuan4/
├── Pertemuan5/
├── Pertemuan6/
└── Pertemuan7/
```

------------------------------------------------------------------------

## Pertemuan 1 --- Docker, Container, dan Volume

### Tujuan dan konsep

Praktikum ini memperkenalkan Docker dan penyimpanan persisten. Image
merupakan template untuk membuat container; container adalah lingkungan
proses terisolasi; sedangkan volume adalah penyimpanan yang dikelola
Docker dan dapat digunakan oleh lebih dari satu container. Data pada
volume tetap tersedia walaupun container dihapus.

### Langkah praktikum

1.  Masuk ke direktori praktikum.
2.  Buat volume bernama `vol-k2`.
3.  Jalankan container Alpine dan pasang volume ke direktori `/data`.
4.  Buat file `laporan-k2.txt` di dalam `/data`.
5.  Hapus container pertama.
6.  Jalankan container Alpine lain dengan volume yang sama.
7.  Baca kembali file yang telah dibuat.

Contoh perintah:

``` bash
docker volume create vol-k2
docker run --name w-k2 -v vol-k2:/data alpine sh -c 'echo "DATA_UJI" > /data/laporan-k2.txt'
docker rm w-k2
docker run --rm -v vol-k2:/data alpine cat /data/laporan-k2.txt
```

### Hasil dan analisis

File yang dibuat oleh container pertama tetap dapat dibaca dari
container kedua. Ini membuktikan bahwa data tersimpan pada volume, bukan
hanya pada sistem file container. Volume bermanfaat ketika data harus
bertahan melewati siklus hidup container.

### Kesimpulan

Docker volume memisahkan penyimpanan data dari container. Dengan
demikian, container dapat dihapus atau dibuat ulang tanpa otomatis
menghilangkan data yang tersimpan pada volume.

------------------------------------------------------------------------

## Pertemuan 2 --- Kriptografi dengan OpenSSL

### Tujuan dan konsep

Praktikum ini membahas tiga fungsi keamanan yang berbeda: hash untuk
memeriksa integritas, enkripsi untuk menjaga kerahasiaan, dan sertifikat
digital untuk mengikat identitas dengan kunci publik.

-   **SHA-256:** menghasilkan ringkasan (hash) dari file. Hash bukan
    enkripsi dan tidak digunakan untuk mengembalikan data asli.
-   **AES-256-CBC:** algoritma enkripsi simetris. Proses dekripsi
    memerlukan kata sandi/kunci yang sesuai.
-   **PBKDF2 dan salt:** membantu menurunkan kunci dari kata sandi dan
    membuat proses penebakan lebih mahal.
-   **Sertifikat X.509:** memuat informasi identitas dan kunci publik.
    Private key harus dijaga rahasia.

### Langkah praktikum

1.  Siapkan file teks sebagai plaintext.
2.  Hitung hash SHA-256 file awal.
3.  Enkripsi file menggunakan AES-256-CBC dengan PBKDF2 dan salt.
4.  Dekripsi ciphertext menjadi file baru.
5.  Hitung hash file hasil dekripsi dan bandingkan dengan hash awal.
6.  Buat kunci RSA dan sertifikat X.509 untuk pengujian lokal.
7.  Periksa informasi sertifikat, seperti Subject, Issuer, dan masa
    berlaku.

Contoh pola perintah (OpenSSL dapat meminta kata sandi secara
interaktif):

``` bash
sha256sum laporan-k2.txt
openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k2.txt -out laporan-k2.enc
openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k2.enc -out hasil-k2.txt
sha256sum hasil-k2.txt
openssl req -x509 -newkey rsa:2048 -keyout key-k2.pem -out cert-k2.pem -days 30
```

### Hasil dan analisis

Hash file awal dibandingkan dengan hash file setelah dekripsi. Nilai
yang sama menunjukkan bahwa isi file hasil dekripsi identik dengan isi
awal. File terenkripsi tidak terbaca sebagai teks biasa tanpa proses
dekripsi dan kata sandi yang benar. Pemeriksaan sertifikat
memperlihatkan informasi identitas dan periode berlakunya.

### Kesimpulan

Hash, enkripsi, dan sertifikat memiliki tujuan berbeda. Hash memeriksa
integritas, enkripsi menjaga kerahasiaan, sedangkan sertifikat membantu
memverifikasi identitas/kunci publik. Jangan mengunggah private key
(`key-k2.pem`), kata sandi, atau kredensial ke repository publik.

------------------------------------------------------------------------

## Pertemuan 3 --- ARP Spoofing dan Mitigasi

### Tujuan dan konsep

ARP (Address Resolution Protocol) digunakan pada jaringan IPv4 lokal
untuk menemukan alamat MAC yang terkait dengan alamat IP. Hasil
pencarian disimpan sementara di ARP cache.

ARP spoofing terjadi ketika perangkat mengirim pesan ARP palsu agar host
lain menyimpan pemetaan IP--MAC yang keliru. Akibatnya, lalu lintas yang
seharusnya menuju gateway dapat diarahkan ke perangkat lain. Kondisi ini
dapat membuka peluang penyadapan atau manipulasi lalu lintas.

### Langkah praktikum

1.  Buat jaringan Docker khusus untuk lab.
2.  Jalankan container korban, gateway, dan penyerang pada jaringan
    tersebut.
3.  Periksa alamat IP dan MAC setiap container.
4.  Tampilkan tabel ARP korban sebagai kondisi awal.
5.  Jalankan simulasi ARP spoofing di jaringan lab.
6.  Periksa kembali tabel ARP korban untuk melihat perubahan pemetaan
    gateway.
7.  Terapkan mitigasi, misalnya memulihkan pemetaan yang benar atau
    menggunakan entri ARP statis.
8.  Periksa tabel ARP sekali lagi untuk memastikan pemetaan telah
    sesuai.

### Hasil dan analisis

Bukti praktikum menunjukkan pemetaan alamat IP ke MAC dapat berubah
selama simulasi. Setelah mitigasi, pemetaan gateway kembali mengarah ke
alamat MAC yang benar dan dapat ditandai sebagai entri permanen. Hal ini
menunjukkan bahwa ARP tidak memiliki autentikasi bawaan yang cukup untuk
memastikan keaslian setiap pesan pemetaan.

### Kesimpulan

ARP spoofing memanfaatkan lemahnya verifikasi pesan ARP. Mitigasi dapat
dilakukan dengan entri ARP statis untuk perangkat tertentu, Dynamic ARP
Inspection pada switch yang mendukung, segmentasi jaringan, dan
pemantauan perubahan pemetaan IP--MAC.

------------------------------------------------------------------------

## Pertemuan 4 --- Analisis WPA/WPA2 Handshake

### Tujuan dan konsep

Praktikum ini mempelajari autentikasi Wi-Fi WPA/WPA2-PSK. Saat client
terhubung ke access point, keduanya melakukan **four-way handshake**
untuk membuktikan kepemilikan rahasia bersama dan membentuk kunci sesi.

Capture handshake dapat digunakan untuk audit kata sandi secara offline.
Alat audit membandingkan kandidat dari wordlist dengan informasi
autentikasi dalam capture. Pengujian hanya boleh dilakukan pada access
point milik sendiri atau jaringan yang secara eksplisit mengizinkan
pengujian.

### Langkah praktikum

1.  Pastikan jaringan target merupakan jaringan lab atau jaringan yang
    diizinkan.
2.  Siapkan adapter Wi-Fi yang mendukung mode monitor.
3.  Identifikasi interface dan kanal access point yang diuji.
4.  Tangkap proses koneksi client yang sah hingga handshake tercatat.
5.  Periksa file capture untuk memastikan target dan handshake
    terdeteksi.
6.  Siapkan wordlist uji yang diizinkan.
7.  Jalankan analisis offline dengan Aircrack-ng.
8.  Catat status hasilnya tanpa menyebarkan kata sandi sebenarnya.

Contoh perintah analisis offline:

``` bash
aircrack-ng -w wordlist-k2.txt handshake-k2-01.cap
```

### Hasil dan analisis

Pada bukti praktikum, Aircrack-ng membaca file capture, mengenali satu
jaringan target dengan satu handshake, lalu menguji kandidat dalam
wordlist. Terminal menunjukkan bahwa salah satu kandidat cocok. Kata
sandi yang ditemukan sengaja tidak dicantumkan dalam dokumentasi ini.

Hasil ini memperlihatkan bahwa kata sandi yang lemah atau mudah ditebak
berisiko ditemukan melalui pengujian offline jika capture handshake
diperoleh pihak yang tidak berwenang.

### Kesimpulan

Gunakan kata sandi Wi-Fi yang panjang, unik, dan sulit ditebak. Aktifkan
WPA2-AES atau WPA3 jika tersedia, perbarui firmware access point, dan
jangan membagikan capture maupun kredensial secara terbuka. Hindari
deauthentication atau pengujian terhadap pengguna/jaringan tanpa izin.

------------------------------------------------------------------------

## Pertemuan 5 --- Pengiriman dan Analisis ICMP dengan Scapy

### Tujuan dan konsep

Praktikum ini menggunakan Scapy untuk membentuk dan mengirim paket
IP/ICMP pada jaringan Docker, kemudian mengamati respons host tujuan.

-   **IP:** menyediakan pengalamatan dan pengiriman paket antarhost.
-   **ICMP:** membawa pesan kontrol dan diagnostik jaringan.
-   **Echo Request:** pesan permintaan yang digunakan oleh ping.
-   **Echo Reply:** balasan terhadap Echo Request.
-   **TTL (Time To Live):** membatasi jumlah hop yang dapat dilewati
    paket; TTL bukan waktu dalam detik.

### Langkah praktikum

1.  Siapkan jaringan dan container Docker untuk pengujian.
2.  Pastikan alamat IP sumber dan tujuan sesuai topologi lab.
3.  Jalankan skrip Python berbasis Scapy dari container alat praktikum.
4.  Kirim paket IP yang membawa ICMP Echo Request ke host tujuan.
5.  Amati alamat IP, TTL, dan tipe ICMP pada respons.
6.  Bandingkan jenis pesan yang dikirim dengan pesan balasan.

### Hasil dan analisis

Pada bukti yang dibagikan, paket dikirim dari `172.19.0.5` menuju
`172.19.0.2`. Respons menampilkan TTL `64` dan tipe ICMP `0`, yaitu Echo
Reply. Echo Request menggunakan tipe ICMP `8`. Adanya Echo Reply
menunjukkan host tujuan merespons permintaan pada saat pengujian. Nilai
TTL yang terlihat tidak boleh ditafsirkan sebagai durasi.

Praktikum ini juga berkaitan dengan pemahaman risiko penyalahgunaan
ICMP, termasuk Smurf attack. Secara konsep, serangan Smurf memanfaatkan
Echo Request dengan alamat sumber palsu yang dikirim ke alamat broadcast
sehingga banyak host dapat mengirimkan balasan ke korban. Pertahanan
mencakup menonaktifkan directed broadcast pada router, penyaringan
ingress/egress, dan pemantauan lalu lintas ICMP yang tidak wajar.

### Kesimpulan

Scapy membantu memahami struktur paket dan respons jaringan. Pemeriksaan
alamat IP, TTL, serta tipe ICMP berguna untuk menganalisis komunikasi.
Semua pengujian harus dibatasi pada jaringan lab yang diotorisasi.

------------------------------------------------------------------------

## Pertemuan 6 --- SYN Flood dan TCP Three-Way Handshake

### Tujuan dan konsep

Praktikum ini mempelajari pembentukan koneksi TCP dan mengamati status
koneksi ketika server menerima banyak permintaan SYN.

TCP membentuk koneksi melalui **three-way handshake**:

1.  **SYN:** client meminta pembentukan koneksi.
2.  **SYN-ACK:** server membalas dan menyetujui permintaan.
3.  **ACK:** client mengonfirmasi balasan sehingga koneksi dapat
    digunakan.

Status `SYN_RECV` berarti server telah menerima SYN dan mengirim
SYN-ACK, tetapi handshake belum selesai dari sisi server. SYN flood
berupaya memenuhi sumber daya server dengan banyak permintaan SYN yang
tidak diselesaikan.

### Langkah praktikum

1.  Jalankan container target dan attacker melalui Docker Compose pada
    jaringan lab.
2.  Periksa jumlah koneksi `SYN_RECV` sebelum pengujian sebagai nilai
    awal.
3.  Jalankan simulasi paket SYN dalam durasi dan batas yang ditentukan
    instruksi lab.
4.  Periksa kembali jumlah koneksi `SYN_RECV`.
5.  Jalankan `tcpdump` pada interface container yang sesuai.
6.  Hasilkan lalu lintas HTTP uji ke layanan target.
7.  Identifikasi paket SYN, SYN-ACK, ACK, HTTP GET, dan respons HTTP
    pada hasil capture.
8.  Hentikan layanan dan hapus container serta network setelah
    pengujian.

### Hasil dan analisis

Bukti menunjukkan jumlah koneksi `SYN_RECV` berubah dari `0` sebelum
pengujian menjadi `1` pada pengamatan setelah simulasi. Angka tersebut
adalah hasil pada waktu perintah dijalankan, bukan ukuran keseluruhan
dampak atau bukti bahwa layanan mengalami gangguan.

Hasil `tcpdump` memperlihatkan pertukaran SYN, SYN-ACK, ACK, permintaan
HTTP GET, dan respons `HTTP/1.1 200 OK`. Rangkaian tersebut
memperlihatkan koneksi TCP yang berhasil dibentuk dan digunakan untuk
bertukar data.

### Mitigasi dan kesimpulan

Mitigasi SYN flood dapat mencakup SYN cookies, pengaturan backlog
koneksi, pembatasan laju koneksi, firewall, pemantauan lonjakan SYN, dan
perlindungan anti-DDoS. `SYN_RECV` dapat menjadi salah satu indikator
koneksi setengah terbuka, sedangkan `tcpdump` membantu memeriksa urutan
paket secara langsung.

------------------------------------------------------------------------

## Pertemuan 7 --- DNS Exfiltration dan Analisis Log DNS

### Tujuan dan konsep

Praktikum ini menunjukkan bagaimana data dapat dikodekan menjadi label
subdomain dan dikirim melalui query DNS. Log DNS kemudian dianalisis
untuk mencari pola query yang mengandung data terenkode.

Dalam DNS exfiltration, potongan data disisipkan ke dalam nama
subdomain. Data sering dikodekan sebagai hexadecimal agar dapat
dimasukkan ke label domain. Server DNS penerima mencatat query tersebut.
Subdomain yang panjang, acak, atau berulang dapat menjadi indikator yang
perlu diperiksa.

### Langkah praktikum

1.  Jalankan container attacker dan DNS melalui Docker Compose pada
    jaringan lab.
2.  Ubah data uji menjadi hexadecimal menggunakan `xxd -p`.
3.  Masukkan label hexadecimal sebagai subdomain di domain lab
    `k2.exfil.lab`.
4.  Kirim query menggunakan `dig` dan amati jawaban DNS.
5.  Simpan log DNS ke `dns-k2.log`.
6.  Jalankan skrip `analisis_dns.py` untuk menghitung query dan
    mendeteksi label berpola hexadecimal.
7.  Ubah label hexadecimal kembali menjadi teks menggunakan `xxd -r -p`.
8.  Hentikan container dan network setelah pengujian.

Contoh pola perintah:

``` bash
echo -n "DATA_UJI" | xxd -p
dig +short LABEL_HEX.k2.exfil.lab @dns
docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k2.log
python3 kasus/k9-dns-detect/analisis_dns.py dns-k2.log
echo "LABEL_HEX" | xxd -r -p
```

Ganti `DATA_UJI` dan `LABEL_HEX` dengan data uji dan label hasil
konversi dari praktikum. Jangan menggunakan data pribadi atau rahasia.

### Hasil dan analisis

Bukti praktikum menunjukkan query dengan label hexadecimal berhasil
dikirim ke domain lab dan memperoleh jawaban `10.0.0.99`. Log DNS
disimpan dan dianalisis oleh skrip. Hasil analisis menampilkan jumlah
query untuk domain `exfil.lab` dan label panjang yang berpola
hexadecimal. Label tersebut dapat dikonversi kembali menjadi teks,
sehingga alur pengiriman data melalui query DNS dapat dipahami.

### Deteksi, mitigasi, dan kesimpulan

Pemantauan volume query, pemeriksaan subdomain panjang/acak, DNS
filtering, logging terpusat, pembatasan resolver, serta kebijakan egress
dapat membantu mendeteksi dan mengurangi DNS exfiltration. DNS memang
diperlukan untuk komunikasi normal, tetapi juga dapat disalahgunakan
sebagai saluran pengiriman data.

------------------------------------------------------------------------

## Ringkasan Pertemuan 1--7

  -----------------------------------------------------------------------
  Pertemuan               Topik                   Fokus hasil
  ----------------------- ----------------------- -----------------------
  1                       Docker dan volume       Data tetap tersedia
                                                  setelah container
                                                  penulis dihapus dan
                                                  dibaca dari container
                                                  lain.

  2                       Kriptografi             Hash memeriksa
                                                  integritas;
                                                  enkripsi/dekripsi
                                                  menjaga kerahasiaan;
                                                  sertifikat diperiksa
                                                  dengan OpenSSL.

  3                       ARP spoofing            Pemetaan IP--MAC
                                                  diamati sebelum,
                                                  selama, dan setelah
                                                  simulasi serta
                                                  mitigasi.

  4                       WPA/WPA2 handshake      Capture handshake
                                                  dianalisis secara
                                                  offline menggunakan
                                                  wordlist yang
                                                  diizinkan.

  5                       ICMP dengan Scapy       Echo Request/Reply
                                                  diamati beserta TTL dan
                                                  tipe ICMP.

  6                       SYN flood dan TCP       Status `SYN_RECV` dan
                                                  urutan three-way
                                                  handshake diamati
                                                  dengan `tcpdump`.

  7                       DNS exfiltration        Label hexadecimal
                                                  dikirim melalui query
                                                  DNS, dicatat,
                                                  dianalisis, dan
                                                  didekode.
  -----------------------------------------------------------------------

## Pemeriksaan keamanan sebelum publikasi

Sebelum mengunggah dokumentasi ke repository publik, periksa seluruh
screenshot dan file. Samarkan atau jangan unggah:

-   Kata sandi Wi-Fi atau baris `KEY FOUND` dari Aircrack-ng.
-   Private key seperti `key-k2.pem`.
-   Password, token, API key, dan kredensial lain.
-   NIM atau informasi identitas pribadi yang tidak diperlukan.
-   Capture jaringan atau log mentah yang memuat data pihak lain.
-   File hasil praktikum yang tidak diminta untuk dipublikasikan.

README ini sengaja tidak mencantumkan kata sandi atau private key.
Screenshot yang dipublikasikan harus sudah disamarkan. Ingat bahwa
menghapus file dari versi terbaru Git tidak otomatis menghapusnya dari
riwayat commit sebelumnya.

------------------------------------------------------------------------

## Langkah menambahkan README dan push ke GitHub

Perintah berikut dijalankan dari direktori utama repository lokal.

### 1. Masuk ke repository

``` bash
cd ~/Network-Security-Course-Bank
pwd
```

Pastikan direktori yang ditampilkan adalah
`Network-Security-Course-Bank`.

### 2. Pastikan folder dokumentasi tersedia

``` bash
mkdir -p Dokumentasi-Kasus-2/{Pertemuan1,Pertemuan2,Pertemuan3,Pertemuan4,Pertemuan5,Pertemuan6,Pertemuan7}
```

Simpan file ini dengan nama `README.md` di dalam `Dokumentasi-Kasus-2/`,
bukan di root repository.

### 3. Periksa file

``` bash
ls -lah Dokumentasi-Kasus-2/
```

Pastikan `README.md` dan folder Pertemuan1 sampai Pertemuan7 terlihat.

### 4. Periksa perubahan Git

``` bash
git status --short
```

### 5. Stage hanya README

``` bash
git add Dokumentasi-Kasus-2/README.md
git diff --cached --name-only
git diff --cached --check
```

Pastikan file yang masuk staging hanya README jika memang hanya README
yang akan di-commit. Perintah `git diff --cached --check` memeriksa
masalah whitespace, bukan kebocoran rahasia.

### 6. Commit

``` bash
git commit -m "Add README for Praktikum Kasus 2 Pertemuan 1-7"
```

### 7. Push ke repository pribadi

``` bash
git remote -v
git push origin main
```

Pastikan `origin` mengarah ke repository pribadi
`Hikmal30/Network-Security-Course-Bank` dan `upstream` mengarah ke
repository dosen. Jika autentikasi HTTPS diminta, gunakan username
GitHub dan Personal Access Token (PAT) sebagai pengganti password akun.
Jangan pernah memasukkan PAT ke README atau screenshot.

### 8. Verifikasi

Buka repository pribadi di GitHub, masuk ke folder
`Dokumentasi-Kasus-2`, lalu buka `README.md`. Periksa apakah judul,
penjelasan setiap pertemuan, tabel ringkasan, dan catatan keamanan
tampil dengan benar.

------------------------------------------------------------------------

**Catatan cakupan:** Penjelasan ini disusun berdasarkan perintah dan
screenshot praktikum yang telah dibagikan. Jika modul dosen atau bukti
pada folder tertentu memuat tahapan tambahan, sesuaikan bagian terkait
agar README tetap sama dengan pelaksanaan sebenarnya.
