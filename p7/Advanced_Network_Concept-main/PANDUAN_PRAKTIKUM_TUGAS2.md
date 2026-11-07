# Panduan Praktikum Tugas 2: Menguji S1 (iteratif) vs S2 (poll) dengan Wireshark

Semua perintah dijalankan di **terminal Ubuntu (WSL)**, kecuali bagian yang tertulis "PowerShell" atau "Windows".
Lokasi kode: `D:\PENS\SMT 1\KOMPUTER JARINGAN\p7\Advanced_Network_Concept-main\kode_lab_chat_c_1`
Di WSL, drive D: bernama `/mnt/d`.

---

## BAGIAN A. Memperbaiki dan menyiapkan WSL

### A1. Diagnosis (PowerShell biasa)
```powershell
wsl --status
wsl -l -v
wsl -d Ubuntu -e echo hi
```
Kalau `echo hi` mengeluarkan `hi`, WSL sehat, lompat ke A4.
Kalau error `E_UNEXPECTED` / "Catastrophic failure", lanjut A2.

### A2. Perbaikan berurutan (PowerShell **Run as Administrator**)
Coba satu per satu, tes `wsl -d Ubuntu -e echo hi` setelah tiap langkah.

1. Update WSL, lalu matikan semua instance:
   ```powershell
   wsl --update
   wsl --shutdown
   ```
2. Aktifkan fitur Windows yang dibutuhkan, lalu **restart PC**:
   ```powershell
   dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
   dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
   ```
3. Kalau masih gagal, pastikan virtualisasi aktif. Task Manager > Performance > CPU > "Virtualization: Enabled".
   Jika "Disabled", aktifkan Intel VT-x / AMD SVM di BIOS.
4. Reset layanan:
   ```powershell
   Restart-Service LxssManager
   ```
   (Jika layanan tidak ada, abaikan.)
5. **Pilihan terakhir:** pasang ulang distro. **Ini menghapus seluruh isi Ubuntu yang ada sekarang.**
   Kode lab ada di drive D:, jadi aman. Cek dulu bahwa tidak ada file penting di dalam Ubuntu.
   ```powershell
   wsl --unregister Ubuntu
   wsl --install -d Ubuntu
   ```
   Ikuti prompt untuk membuat username dan password Linux.

### A3. Alternatif jika WSL tidak mau jalan sama sekali
- Pakai komputer lab / laptop teman yang punya Linux atau WSL, atau
- Pakai VM (VirtualBox + Ubuntu). Langkah B dan seterusnya sama persis.

### A4. Pasang kompiler dan alat bantu (di terminal Ubuntu)
```bash
sudo apt update
sudo apt install -y build-essential libssl-dev tcpdump netcat-openbsd strace iproute2
gcc --version
```
Catatan: `libssl-dev` hanya dipakai Modul 6 (TLS). Untuk Tugas 2 cukup `build-essential` dan `tcpdump`.

### A5. Masuk ke folder kode
```bash
cd "/mnt/d/PENS/SMT 1/KOMPUTER JARINGAN/p7/Advanced_Network_Concept-main/kode_lab_chat_c_1"
ls
```
Peringatan performa: compile dan jalankan dari `/mnt/d` itu normal untuk lab ini. Kalau muncul error "required file not found" atau karakter aneh, penyebabnya biasanya file `.sh` berakhiran CRLF. Perbaiki dengan `sed -i 's/\r$//' *.sh`.

### A6. Kompilasi
Untuk Tugas 2:
```bash
make s1_server s1_klien s2_server s2_klien
```
Untuk semua modul (Modul 6 butuh libssl-dev):
```bash
make
```
Hasil: file `s1_server`, `s1_klien`, `s2_server`, `s2_klien`. Tidak boleh ada error. Warning boleh dicatat.

---

## BAGIAN B. Cara merekam paket (Wireshark + WSL)

**Penting:** Wireshark versi Windows **tidak bisa** menangkap trafik `127.0.0.1` milik WSL2, karena loopback WSL berada di dalam mesin virtualnya. Contoh laporan Anda memperlihatkan "Linux cooked capture v1 / interface any", artinya rekaman dibuat di sisi Linux. Ada dua cara:

### Cara 1 (disarankan): tcpdump di WSL, buka hasilnya di Wireshark Windows
Di terminal Ubuntu khusus perekam (sebut **T-CAP**):
```bash
sudo tcpdump -i lo -nn -s0 tcp port 5000 -w /mnt/d/capture_s1.pcap
```
- `-i lo` = antarmuka loopback, `-w` = simpan ke file. Biarkan berjalan.
- Hentikan dengan `Ctrl+C` setelah percobaan selesai (akan tertulis "N packets captured").
- Buka `D:\capture_s1.pcap` dengan Wireshark Windows, filter `tcp.port == 5000`.

Catatan: pada interface `lo` header-nya Ethernet palsu (bukan "Linux cooked"). Isinya tetap sama. Jika ingin tampilan persis seperti contoh, ganti `-i lo` dengan `-i any`.

### Cara 2: teman yang memberi screenshot Wireshark
Anda bisa pakai screenshot Wireshark dari teman, asalkan **skenario dan port sama** dengan percobaan Anda. Lebih aman kalau Anda minta file `.pcap`-nya lalu buka sendiri.

### Alur kerja tiap percobaan
1. Mulai tcpdump (T-CAP).
2. Jalankan server (T-SRV) lalu klien (T-KLIEN).
3. Setelah selesai, hentikan tcpdump dengan Ctrl+C.
4. Buka pcap di Wireshark, terapkan filter, screenshot.

Gunakan **nama file pcap berbeda** untuk S1 dan S2 supaya tidak tertimpa.

### Screenshot terminal
Tekan `Win + Shift + S`, pilih area terminal, simpan. Pastikan username/path tampak jelas dan perintah yang dijalankan ikut terlihat.

---

## BAGIAN C. Percobaan S1: server iteratif (blocking)

Siapkan **4 terminal Ubuntu**: T-CAP (perekam), T-SRV (server), T-A (klien A), T-B (klien B).

### C1. Uji dasar (satu klien)
T-CAP:
```bash
sudo tcpdump -i lo -nn -s0 tcp port 5000 -w /mnt/d/capture_s1.pcap
```
T-SRV:
```bash
./s1_server 5000
```
T-A (kirim satu pesan, sesuai Gambar 1.1 di contoh):
```bash
printf 'halo\n' | ./s1_klien 5000 A
```
Lalu pesan beruntun:
```bash
printf 'halo\nnapa kabar\nbelajar jaringan komputer\n' | ./s1_klien 5000 A
```
**Yang diamati:**
- Klien: `connect() sukses`, `kirim`, `balasan` (server mengembalikan teks yang sama = echo).
- Server: urutan `MASUK` → `terima N byte` → `KELUAR`, dengan **satu klien tuntas dulu** baru klien berikutnya.

### C2. Eksperimen inti: bukti blocking (dua klien bersamaan)
Tujuan: klien A "menahan" koneksi, lalu klien B mencoba masuk.

T-SRV: `./s1_server 5000` (jika masih jalan dari C1, biarkan).

T-A (tahan koneksi 15 detik, jangan menutup):
```bash
(printf 'dari-A\n'; sleep 15) | ./s1_klien 5000 A
```
**Segera** (dalam 15 detik) di T-B:
```bash
printf 'dari-B\n' | ./s1_klien 5000 B
```
**Hasil yang seharusnya terlihat:**
| Waktu | T-A | T-B | T-SRV |
|---|---|---|---|
| t=0 | connect, kirim, balasan | | MASUK A, terima |
| t≈1 s | menunggu (sleep) | `connect() sukses` tetapi `kirim`/`balasan` **tertunda** | **tidak ada log MASUK untuk B** |
| t≈15 s | selesai | baru dilayani: `kirim`, `balasan`, `selesai` | KELUAR A, lalu MASUK B, terima, KELUAR B |

Jelaskan di laporan: `connect()` B berhasil karena **kernel** menyelesaikan 3-way handshake dan menaruh koneksi di antrean `listen()` (backlog 16), padahal aplikasi server belum memanggil `accept()` untuk B. Server masih terjebak di `while (recv(...) > 0)` untuk A, sesuai komentar kode: `Selama loop ini berjalan, accept() TIDAK dipanggil lagi.`

Opsional: ukur waktu tunggu B. Catat angka `+ ... ms` pada baris `balasan` klien B. Angkanya mendekati 15000 ms.

### C3. Lihat antrean backlog (opsional, T-CAP atau terminal lain)
Saat C2 berjalan:
```bash
ss -tn state established '( sport = :5000 )'
ss -ltn 'sport = :5000'
```
Kolom `Recv-Q` pada socket LISTEN menunjukkan koneksi yang menunggu di antrean.

### C4. Hentikan dan analisis di Wireshark
Ctrl+C di T-CAP. Buka `capture_s1.pcap`. Filter:
- `tcp.port == 5000` : semua trafik.
- `tcp.flags.syn == 1` : lihat SYN/SYN-ACK. Pada C2, handshake B muncul **di awal**, walau data B baru dibalas setelah A selesai.
- `tcp.flags.fin == 1` : titik penutupan A. Cek bahwa echo ke B muncul **setelah** FIN A.
- `tcp.len > 0` : hanya paket yang membawa data.

Hal yang dicatat per fase (seperti contoh laporan): nomor frame dan port ephemeral klien, 3-way handshake (SYN, SYN-ACK, ACK), data (PSH,ACK dengan Len=...), dan teardown (FIN,ACK dari kedua sisi + ACK akhir). Untuk menampilkan isi payload: klik kanan paket > **Follow > TCP Stream**.

Tip: Statistics > Conversations > tab TCP menampilkan tiap koneksi beserta waktu mulai dan durasinya. Pada C2 ini membuktikan durasi dan urutan sekuensial.

---

## BAGIAN D. Percobaan S2: server poll() (multi klien)

### D1. Siapkan
Hentikan `s1_server` (Ctrl+C), lalu:

T-CAP:
```bash
sudo tcpdump -i lo -nn -s0 tcp port 5000 -w /mnt/d/capture_s2.pcap
```
T-SRV:
```bash
./s2_server 5000
```
Log server berformat `TIMESTAMP KEJADIAN` (`LISTEN`, `MASUK`, `DAFTAR`, `PESAN`, `KELUAR`).

### D2. Dua klien interaktif (untuk screenshot seperti Gambar 2.2)
T-A:
```bash
./s2_klien 5000
```
Ketik nama (misal `Ferdiansyah`) lalu Enter. Muncul `*** selamat datang, Ferdiansyah`.

T-B (saat A masih terbuka, **tanpa menunggu A selesai**):
```bash
./s2_klien 5000
```
Ketik `budi` lalu Enter. A akan menerima `*** budi bergabung ke ruang chat`.

Coba di T-A: ketik `halo budi`. T-B menerima `[Ferdiansyah] halo budi`. Balas dari T-B. Coba juga:
- `/who` : daftar pengguna online (yang Anda ditandai "(Anda)").
- `/quit` : keluar. Pengguna lain melihat `*** ... meninggalkan ruang chat`.
- Nama ganda: ketik nama yang sama dengan pengguna lain, server menjawab `!!! nama sudah dipakai` dan menutup koneksi.

**Bandingkan dengan S1:** klien B langsung dilayani tanpa menunggu A. Ini bukti utama konkurensi.

### D3. Versi otomatis (hasil repeatable untuk tabel laporan)
```bash
# jalankan A dan B hampir bersamaan, masing-masing hidup 10 detik
(sleep 1; printf 'halo dari A\n'; sleep 8) | ./s2_klien 5000 anna 10 &
(sleep 3; printf 'halo dari B\n'; sleep 6) | ./s2_klien 5000 budi 10 &
wait
```
Argumen `s2_klien PORT NAMA BATAS_DETIK`: nama otomatis dan batas waktu hidup.

### D4. Uji banyak klien (kapasitas MAKS_KLIEN = 32)
```bash
for i in $(seq 1 34); do
  (sleep 8) | ./s2_klien 5000 "u$i" 10 >/dev/null &
done; wait
```
Di log server, klien ke-33 dan ke-34 harus muncul sebagai `TOLAK ... alasan=penuh` dan menerima `!!! server penuh`. Ini membuktikan batas 32 adalah batas aplikasi (konstanta), bukan batas OS.

### D5. Analisis Wireshark S2
Ctrl+C di T-CAP, buka `capture_s2.pcap`, filter `tcp.port == 5000`.
- Beri warna/identifikasi tiap klien dengan filter per port ephemeral: `tcp.port == 33650` (ganti dengan port klien A), dst. Port ephemeral terlihat di log server (`MASUK 127.0.0.1:PORT`).
- Buktikan **interleaving**: handshake klien 2 terjadi **di tengah** sesi klien 1 (bandingkan nomor frame SYN klien 2 dengan frame data/FIN klien 1).
- Cari payload sambutan: Follow TCP Stream pada klien A menampilkan `*** ketik nama Anda lalu ENTER: ` dan `*** selamat datang, ...`.
- Statistics > Conversations > TCP: dua+ koneksi dengan rentang waktu **tumpang tindih** (berbeda dengan S1 yang berurutan).
- Statistics > Flow Graph (pilih "TCP flows") untuk diagram visual.

### D6. Bukti single-thread (opsional tapi bagus untuk argumen)
Saat `s2_server` berjalan dengan beberapa klien:
```bash
ps -o pid,nlwp,cmd -C s2_server
```
Kolom `NLWP` = jumlah thread. Harus `1`.

Lihat pemanggilan sistem server:
```bash
sudo strace -f -e trace=poll,accept,recvfrom,sendto -p $(pgrep s2_server)
```
Tampak `poll(...)` memblok sampai ada aktivitas, lalu `accept`/`recvfrom` dipanggil hanya untuk fd yang siap.

Untuk S1 dengan cara yang sama (`pgrep s1_server`): server diam di `recvfrom` untuk klien A dan **tidak** memanggil `accept`.

---

## BAGIAN E. Membaca source code: poin pembanding

Gunakan tabel ini sebagai kerangka bagian "Analisis Source Code" (nomor baris merujuk berkas di folder kode).

| Aspek | S1 `s1_server_iteratif.c` | S2 `s2_server_poll.c` |
|---|---|---|
| Struktur utama | `for(;;){ accept(); while(recv()){send();} close(); }` | `for(;;){ poll(); proses klien siap; accept bila listener siap }` |
| Titik blocking | `recv()` pada satu klien menahan seluruh server | `poll()` menunggu **banyak fd** sekaligus; `recv()` hanya dipanggil bila fd siap (POLLIN), jadi tidak memblok |
| Data per klien | Tidak ada, variabel lokal `cs` | Array `Klien klien[MAKS_KLIEN]` berisi fd, nama, buffer rakit baris, alamat asal |
| Batas klien | 1 aktif, sisanya antre di backlog `listen(ls,16)` | `MAKS_KLIEN = 32`, lebih dari itu `TOLAK ... penuh` |
| Protokol aplikasi | Echo mentah, tidak peduli batas pesan | Chat berbasis baris: daftar nama, `/who`, `/quit`, siaran ke semua klien (`siar`) |
| Perakitan pesan | Tidak ada. TCP adalah byte stream, jadi satu `recv()` bisa memuat potongan pesan | `rakit_baris()` mengumpulkan byte di `in[]` sampai ada `\n` |
| Penanganan putus | `recv()` mengembalikan 0 lalu `KELUAR` | `tutup_klien()`, array dipadatkan, siar "meninggalkan ruang chat" |
| Sinyal SIGPIPE | Tidak ditangani | `signal(SIGPIPE, SIG_IGN)` agar server tidak mati saat menulis ke socket tertutup |
| Thread/proses | 1 proses, 1 thread | 1 proses, 1 thread (konkurensi tanpa thread) |
| Kelemahan | Satu klien lambat/diam memblok semua | `send()` di `kirim_ke()` masih blocking (klien yang tidak membaca bisa membekukan server, dibahas Modul 4); pencarian array O(N); `poll()` menyalin array fd tiap iterasi |

**Koreksi untuk laporan contoh Anda:** pada tabel perbandingan di contoh tertulis system call S2 "`fcntl/pollfd, non-blocking flow`". Di kode `s2_server_poll.c` **tidak ada** `fcntl()` dan socket **tidak** dibuat non-blocking. S2 hanya memakai `poll()`, `accept`, `recv`, `send`. Tulis "poll()" dan "I/O multiplexing berbasis readiness", bukan "non-blocking".
Juga cek konsistensi angka: pada contoh S1, log server mencatat `terima 5 byte` untuk `halo\n`, sedangkan teks analisis Wireshark menyebut `Len=7`. Pastikan angka di analisis berasal dari **capture yang sama** dengan screenshot (pesan `halo\n` = 5 byte, `pesan-dua\n` = 10, dst).

---

## BAGIAN F. Kerangka laporan Tugas 2

1. **Pendahuluan / tujuan**: menguji S1 dan S2, tangkap trafik TCP dengan Wireshark, bandingkan arsitekturnya.
2. **Lingkungan uji**: Windows 11, WSL2 Ubuntu (tulis versi: `lsb_release -a`), gcc (`gcc --version`), Wireshark/tcpdump (versi), alamat `127.0.0.1:5000`.
3. **Tahap 1 (S1)**: kode, penjelasan, screenshot klien (Gambar 1.1), server (1.2), eksperimen blocking dua klien (tabel waktu C2), Wireshark (1.3) dengan analisis tiga fase.
4. **Tahap 2 (S2)**: kode, penjelasan poll(), screenshot server (2.1), klien (2.2), Wireshark (2.3) dengan analisis interleaving dan isolasi sesi.
5. **Perbandingan** (tabel Bagian E), plus kesimpulan: kapan memilih iteratif vs poll.
6. **Kesimpulan**: S1 sederhana tapi konkurensi 1; S2 melayani banyak klien dalam satu thread dengan `poll()`, tetapi `send()` blocking jadi risiko, motivasi Modul 4.
7. **Lampiran**: perintah yang dijalankan, nama file pcap.

Tips: tiap gambar diberi caption "Gambar X.Y ..." dan dirujuk di teks. Sebut nomor frame yang benar-benar tampak di screenshot **Anda**.

---

## BAGIAN G. Modul 3 sampai 6 (jika dosen juga meminta)

Port contoh di bawah bebas dipilih. Catatan hasil pembacaan kode:
- `s3_server_mentah.c` memakai socket TCP biasa (bukan `SOCK_RAW`), jadi **tidak butuh sudo**. Ia membuktikan bahwa TCP adalah byte stream: lihat jumlah dan ukuran `recv#N`.

### Modul 3: byte stream
```bash
make s3_server s3_pengirim
./s3_server 5000          # terminal 1
./s3_pengirim 5000        # terminal 2
```
Amati: 3 `send()` pendek bisa muncul sebagai satu `recv()`, dan 200 KB terpecah menjadi banyak `recv()` (buffer 4096). Wireshark: `tcp.port == 5000`, lihat segmentasi dan ukuran Len.

### Modul 4: ketahanan
```bash
make s4_server s4_lambat s4_probe
./s4_server 5000                  # terminal 1
./s4_lambat 5000 lambat 20        # terminal 2: daftar lalu diam 20 detik tanpa recv()
./s4_probe 5000 probe 8           # terminal 3: ukur responsivitas selama serangan
```
Amati: probe harus tetap mendapat balasan walau si lambat tidak membaca. Server menutup klien lambat bila antreannya melewati `MAKS_OUT`. Untuk pembanding, bandingkan dengan S2 (tanpa antrean keluar) yang dapat membeku.

### Modul 5: thread
```bash
make s5_server
./s5_server 5000
# beberapa klien seperti D3, mis. ./s2_klien 5000 anna 10
```
Bandingkan memori dengan S2 (poll), keduanya dengan 32 klien aktif:
```bash
ps -o pid,nlwp,rss,vsz,cmd -C s2_server
ps -o pid,nlwp,rss,vsz,cmd -C s5_server
```
`NLWP` (thread): S2 = 1, S5 = 1 + jumlah klien. `VSZ` S5 membesar karena tiap thread mendapat stack (biasanya 8 MB virtual), sedangkan `RSS` (memori fisik) naik jauh lebih kecil. Jelaskan perbedaan VSZ vs RSS di laporan. Protokolnya sama dengan S2, jadi klien `s2_klien` bisa dipakai.

Demo race condition (opsional): `make tsan` lalu jalankan `./s5_race 5000` dengan beberapa klien. ThreadSanitizer melaporkan `WARNING: ThreadSanitizer: data race`.

### Modul 6: TLS
```bash
make pki                          # membuat pki/ca.crt, server.crt, server.key
make s6_server s6_klien s6_probe
./s6_server 5443                  # terminal 1
./s6_klien 5443 pki/ca.crt anna 10   # terminal 2, nama + batas waktu opsional
./s6_probe 5443 pki/ca.crt probe 8
```
Uji penolakan plaintext (server TLS menerima klien biasa):
```bash
printf 'halo\n' | nc 127.0.0.1 5443
printf 'halo\n' | ./s2_klien 5443 anna 3     # klien plaintext
```
Amati log server: handshake TLS gagal dan koneksi ditutup, pesan plaintext tidak diproses. Di Wireshark: `tcp.port == 5443` memperlihatkan Client Hello / Server Hello lalu `Application Data` yang terenkripsi, sementara S2/S4 menampilkan teks terbaca. Folder `kode_lab_chat_c_1/contoh_capture` berisi contoh pcap plaintext vs TLS dan cara membukanya (`BACA_INI.md`).
Jika ingin mendekripsi di Wireshark: jalankan dengan `SSLKEYLOGFILE=$PWD/kunci.log ./s6_server 5443` (server dan klien membaca variabel ini), lalu Wireshark > Preferences > Protocols > TLS > (Pre)-Master-Secret log filename.

Peringatan: sertifikat dan kunci dari `make pki` hanya untuk lab. Jangan dipasang ke trust store Windows/browser.

---

## BAGIAN H. Troubleshooting cepat

| Gejala | Penyebab / solusi |
|---|---|
| `bind: Address already in use` | Server lama masih jalan. `pkill s1_server` atau ganti port. |
| `connect: Connection refused` | Server belum jalan atau beda port. |
| tcpdump: `Permission denied` | Pakai `sudo`. |
| Wireshark Windows tidak melihat paket | Normal untuk WSL2. Rekam dengan tcpdump (Bagian B). |
| File `/mnt/d/capture.pcap` kosong | tcpdump belum dihentikan dengan Ctrl+C, atau filter port salah. |
| `make: command not found` | `sudo apt install build-essential` |
| `fatal error: openssl/ssl.h` | `sudo apt install libssl-dev` (hanya Modul 6) |
| `\r: command not found` pada `.sh` | `sed -i 's/\r$//' buat_sertifikat.sh` |
| Klien B pada C2 langsung dilayani | Anda menutup A terlalu cepat. Pastikan `sleep 15` masih berjalan. |
| Output server tidak muncul langsung | Program S1 sudah `fflush`. Pastikan Anda melihat terminal yang benar. |
