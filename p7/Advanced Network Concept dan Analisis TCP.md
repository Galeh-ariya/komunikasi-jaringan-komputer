# Advanced Network Concept dan Analisis TCP (Socket Programming C)

> **Matakuliah:** [ISI MATAKULIAH]
> **Dosen Pengampu:** [ISI NAMA DOSEN]
> **Disusun oleh:** [ISI NAMA MAHASISWA]
> **NIM/NRP:** [ISI NIM/NRP]
> **Program Studi:** [ISI PROGRAM STUDI]
> **Institusi:** [ISI NAMA INSTITUSI]
> **Tahun:** [ISI TAHUN]

---

## Deskripsi

Repository ini berisi pengujian dan analisis dua model arsitektur server TCP menggunakan socket programming di bahasa C, yaitu server iteratif yang bersifat blocking dan server berbasis `poll()` yang mampu melayani banyak klien sekaligus. Pengujian dijalankan langsung di lingkungan Linux (Ubuntu) pada server pribadi, dengan bukti berupa pencatatan waktu presisi milidetik di level aplikasi untuk setiap tahap koneksi.

Catatan metodologi: pengujian ini tidak menggunakan tangkapan paket Wireshark, melainkan timestamp presisi milidetik pada setiap event (`connect()`, kirim pesan, terima balasan, tutup koneksi). Pendekatan ini tetap sahih untuk membuktikan perbedaan karakteristik blocking dan non blocking, bahkan menunjukkan detail yang tidak selalu terlihat jelas hanya dari tangkapan paket, yaitu jeda antara selesainya jabat tangan TCP di level kernel dengan saat data benar benar diproses oleh aplikasi.

---

## Struktur Folder

| File / Folder | Keterangan |
|---|---|
| `server_s1_iteratif.c` | Server TCP iteratif (blocking), hanya melayani satu klien penuh sebelum menerima klien berikutnya |
| `server_s2_poll.c` | Server TCP berbasis `poll()`, mampu melayani hingga 32 klien sekaligus dalam satu proses |
| `klien_sim.py` | Klien simulasi Python dengan pencatatan waktu presisi milidetik |
| `run_demo.sh` | Script yang mengompilasi server, menjalankan skenario uji, dan menyimpan log ke folder `hasil/` |
| `hasil/` | Log asli hasil eksekusi (server dan klien, timestamp milidetik) |
| `Laporan_Advanced_Network_Concept.docx` | Laporan lengkap (pendahuluan, source code, analisis, tabel perbandingan, kesimpulan) |

---

## 1. Tahap 1: Server Iteratif (Blocking)

Server pada tahap ini menangani satu koneksi klien secara penuh di dalam satu putaran loop, mulai dari menerima data (`recv`), mengembalikan data tersebut (`send`, bersifat echo), hingga klien tersebut menutup koneksinya, sebelum server bersedia memanggil `accept()` untuk klien berikutnya.

### Hasil Eksekusi

Pengujian dilakukan dengan dua klien, KlienA dan KlienB. KlienA terhubung lebih dulu dan menahan koneksinya selama dua detik sebelum mengirim pesan (mensimulasikan sesi aktif panjang). Di tengah masa tahan tersebut, KlienB mencoba terhubung ke server yang sama.

```
[21:25:14.475] [S1] server iteratif mendengarkan di 127.0.0.1:5000
[21:25:14.866] [S1] klien diterima 127.0.0.1:52516 (mode blocking, klien lain harus menunggu)
[21:25:16.867] [S1] terima 11 byte dari 127.0.0.1:52516 -> "halo dari A"
[21:25:16.867] [S1] echo 11 byte ke 127.0.0.1:52516
[21:25:17.669] [S1] terima 13 byte dari 127.0.0.1:52516 -> "pesan kedua A"
[21:25:17.669] [S1] echo 13 byte ke 127.0.0.1:52516
[21:25:18.871] [S1] klien 127.0.0.1:52516 terputus, server siap accept() klien berikutnya
[21:25:18.871] [S1] klien diterima 127.0.0.1:52518 (mode blocking, klien lain harus menunggu)
[21:25:18.871] [S1] terima 11 byte dari 127.0.0.1:52518 -> "halo dari B"
[21:25:18.871] [S1] echo 11 byte ke 127.0.0.1:52518
[21:25:19.573] [S1] klien 127.0.0.1:52518 terputus, server siap accept() klien berikutnya
```

### Analisis

KlienB berhasil menyelesaikan `connect()` pada pukul 21:25:15.437, hanya 2,4 milidetik setelah proses `connect()` dimulai, menunjukkan jabat tangan TCP tetap diselesaikan dengan cepat oleh kernel karena koneksi masuk ke antrean `listen()` backlog. Akan tetapi, KlienB baru menerima balasan pada level aplikasi pada pukul 21:25:18.871, yaitu 3,434 detik kemudian, tepat setelah KlienA menutup koneksinya. Ini membuktikan bahwa server iteratif baru memanggil `accept()` untuk KlienB setelah KlienA selesai dilayani, bukan pada saat jabat tangan TCP KlienB selesai.

---

## 2. Tahap 2: Server Chat Multi Klien dengan poll()

Server pada tahap ini menggunakan system call `poll()` untuk memantau socket listener dan seluruh socket klien yang terhubung secara bersamaan, dalam satu proses dan satu thread, sehingga mampu menerima klien baru dan memproses pesan klien lain tanpa saling menunggu.

### Hasil Eksekusi

Pengujian dilakukan dengan tiga klien (Andi, Budi, Citra) yang terhubung hampir bersamaan dan saling mengirim pesan ke ruang chat yang sama (broadcast).

```
[21:25:19.889] [S2] server poll() mendengarkan di 127.0.0.1:5001 (maks 32 klien simultan)
[21:25:20.246] [S2] klien baru diterima 127.0.0.1:45888 (total klien aktif=1)
[21:25:20.246] [S2] klien 127.0.0.1:45888 mendaftar sebagai nama="Citra"
[21:25:20.261] [S2] klien baru diterima 127.0.0.1:45900 (total klien aktif=2)
[21:25:20.261] [S2] klien 127.0.0.1:45900 mendaftar sebagai nama="Andi"
[21:25:20.262] [S2] pesan dari Citra: "ikut gabung"
[21:25:20.262] [S2] pesan dari Andi: "halo semua"
[21:25:20.270] [S2] klien baru diterima 127.0.0.1:45908 (total klien aktif=3)
[21:25:20.270] [S2] klien 127.0.0.1:45908 mendaftar sebagai nama="Budi"
[21:25:20.971] [S2] pesan dari Andi: "apa kabar?"
[21:25:20.972] [S2] pesan dari Budi: "halo juga"
```

### Analisis

Ketiga klien diterima dalam rentang waktu kurang dari 25 milidetik satu sama lain (Citra di 20.246, Andi di 20.261, Budi di 20.270) tanpa saling menunggu, berbeda jauh dengan pola server iteratif yang membuat klien baru menunggu hingga ribuan milidetik. Isolasi antar sesi juga terlihat pada pertukaran pesan: Andi mengirim pesan pada 20.971 dan hanya satu milidetik kemudian Budi turut mengirim pesan tanpa menunggu giliran. Hasil ini membuktikan model `poll()` berhasil melakukan I/O multiplexing, meningkatkan konkurensi secara signifikan dibandingkan server iteratif.

---

## 3. Perbandingan Komparatif Arsitektur S1 vs S2

| Parameter Evaluasi | Tahap 1: Server Iteratif (S1) | Tahap 2: Server poll() (S2) |
|---|---|---|
| Model Arsitektur | Sekuensial / Iteratif | Event-Driven I/O Multiplexing |
| System Call Utama | `socket`, `bind`, `listen`, `accept`, `recv`, `send` | `poll()`, `socket`, `accept`, `recv`, `send` per klien |
| Maksimum Konkurensi | 1 klien aktif, klien lain antre di backlog kernel | Hingga 32 klien simultan (dibatasi array `MAKS_KLIEN`, bukan batas sistem operasi) |
| Waktu tunggu balasan klien baru saat server sibuk | Tertunda signifikan (pada pengujian ini 3,434 detik) | Hampir seketika (pada pengujian ini di bawah 25 milidetik) |
| Status handshake TCP saat server sibuk | Tetap berhasil, masuk antrean `listen` backlog | Tetap berhasil dan langsung diproses aplikasi |
| Keterbatasan | Blocking total saat melayani satu klien aktif | `send()` per klien masih blocking, pencarian klien pada array masih bersifat linear |

---

## Kesimpulan

Arsitektur server sangat menentukan kemampuan sebuah sistem dalam melayani banyak klien secara bersamaan. Server iteratif (S1) sederhana untuk diimplementasikan, namun memiliki keterbatasan besar karena hanya mampu melayani satu klien pada satu waktu, sehingga klien lain harus menunggu meskipun koneksi TCP mereka sudah berhasil terjalin. Sebaliknya, server berbasis `poll()` (S2) mampu melayani banyak klien sekaligus dalam satu proses dan satu thread, terbukti dari selisih waktu penerimaan klien yang hanya berkisar puluhan milidetik dan pertukaran pesan antar klien yang berlangsung hampir seketika.

---

## Cara Menjalankan Ulang

```bash
chmod +x run_demo.sh klien_sim.py
./run_demo.sh
```

Membutuhkan `gcc` dan `python3`. Hasil log akan tersimpan otomatis di folder `hasil/`.

---

## Lisensi

Tugas ini dibuat untuk keperluan akademik. © [ISI TAHUN] [ISI NAMA MAHASISWA].
