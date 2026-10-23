# Cisco Routing — Komunikasi dan Jaringan Komputer

> **Mata Kuliah:** Komunikasi dan Jaringan Komputer  
> **Dosen:** Dr. Ferry Astika Saputra, ST, M.Sc  
> **Mahasiswa:** Galeh Ariya Irwana  
> **NRP:** 1226800017  
> **Program Studi:** Pascasarjana Teknik Informatika dan Komputer  
> **Institusi:** Politeknik Elektronika Negeri Surabaya (PENS)  
> **Tahun:** 2026

---

## Deskripsi

Repository ini berisi tugas praktikum **Cisco Routing** yang disimulasikan menggunakan **Cisco Packet Tracer**. Terdapat dua topologi jaringan yang dirancang untuk membuktikan konektivitas antar subnet menggunakan routing statis maupun dinamis.

---

## Tugas 1 — Topologi dengan 1 Router dan 2 Switch

### Deskripsi Topologi

Topologi ini terdiri dari:
- **1 Router** (Main Router) sebagai penghubung antar jaringan
- **2 Switch** (Switch-A dan Switch-B) sebagai penghubung perangkat dalam LAN
- **6 PC**, masing-masing 3 PC per switch

| Segmen | Switch | PC | Subnet |
|--------|--------|----|--------|
| Kiri | Switch-A | PC-A1, PC-A2, PC-A3 | 192.168.1.0 |
| Kanan | Switch-B | PC-B1, PC-B2, PC-B3 | 192.168.2.0 |

### Hasil Analisis

Berdasarkan hasil simulasi jaringan di Cisco Packet Tracer, topologi yang dirancang terdiri dari dua segmen jaringan lokal (LAN) yang masing-masing terhubung melalui Switch-A dan Switch-B, lalu kedua switch tersebut dihubungkan ke sebuah Main Router sebagai penghubung antar jaringan. Jaringan sisi kiri menggunakan subnet **192.168.1.0** (PC-A1, PC-A2, PC-A3), sedangkan jaringan sisi kanan menggunakan subnet **192.168.2.0** (PC-B1, PC-B2, PC-B3).

Hasil pengujian dari PC-A3 menunjukkan bahwa ping ke `192.168.1.1` (gateway sisi A) berhasil dengan reply 4 dari 4 paket, TTL=255, dan waktu respons di bawah 1ms, begitu pula ping ke `192.168.2.1` (gateway sisi B) juga berhasil dengan statistik serupa — membuktikan bahwa routing antar kedua subnet berjalan dengan baik.

Keberhasilan komunikasi lintas jaringan ini dikonfirmasi lebih lanjut melalui PDU List Window yang mencatat tiga skenario pengiriman paket ICMP, yakni dari PC-A1 ke PC-B1, PC-A3 ke PC-B3, dan PC-A2 ke PC-B1, yang semuanya berstatus **Successful**. Hal ini menunjukkan bahwa konfigurasi routing pada Main Router telah berjalan dengan benar, sehingga setiap perangkat di subnet 192.168.1.0 dapat berkomunikasi dengan perangkat di subnet 192.168.2.0 tanpa hambatan.

---

## Tugas 2 — Topologi dengan 2 Router dan 2 Switch

### Deskripsi Topologi

Topologi ini terdiri dari:
- **2 Router** (Router-A dan Router-B) yang saling terhubung melalui jaringan transit
- **2 Switch** (Switch-C dan Switch-D) sebagai penghubung perangkat dalam LAN
- **6 PC**, masing-masing 3 PC per switch

| Segmen | Router | Switch | PC | Subnet |
|--------|--------|--------|----|--------|
| Kiri | Router-A | Switch-C | PC-C1, PC-C2, PC-C3 | 192.168.1.0 |
| Transit | Router-A ↔ Router-B | — | — | 192.168.3.0 |
| Kanan | Router-B | Switch-D | PC-D1, PC-D2, PC-D3 | 192.168.2.0 |

### Hasil Analisis

Topologi pada tugas nomor 2 ini menggunakan arsitektur yang lebih kompleks dibandingkan sebelumnya, yaitu dua router (Router-A dan Router-B) yang saling terhubung melalui jaringan transit **192.168.3.0**, di mana sisi kiri (subnet 192.168.1.0) menghubungkan PC-C1, PC-C2, dan PC-C3 melalui Switch-C ke Router-A, sedangkan sisi kanan (subnet 192.168.2.0) menghubungkan PC-D1, PC-D2, dan PC-D3 melalui Switch-D ke Router-B.

Hasil pengujian dari PC-C2 menunjukkan bahwa ping ke `192.168.2.3` (PC-D3) maupun ke `192.168.2.4` (PC-D2) keduanya berhasil dengan 4 dari 4 paket diterima (0% loss), dengan rata-rata round trip time sebesar **2ms** dan nilai **TTL=126** — nilai TTL yang lebih rendah dari 255 ini mengindikasikan bahwa paket melewati dua hop router (Router-A dan Router-B) sebelum sampai ke tujuan, yang sesuai dengan topologi yang dirancang.

Keberhasilan komunikasi antar subnet ini juga dikonfirmasi melalui PDU List Window yang mencatat tiga skenario pengiriman paket ICMP — dari PC-C1 ke PC-D3, PC-C3 ke PC-D1, dan PC-C2 ke PC-D3 — dengan seluruhnya berstatus **Successful**. Hal ini membuktikan bahwa konfigurasi routing antar Router-A dan Router-B, termasuk pengaturan rute untuk masing-masing subnet, telah berjalan dengan benar sehingga setiap host di jaringan 192.168.1.0 mampu berkomunikasi dengan host di jaringan 192.168.2.0 secara penuh melalui jalur transit 192.168.3.0.

---

## Tools yang Digunakan

- [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) — simulasi jaringan

---

## Lisensi

Tugas ini dibuat untuk keperluan akademik. © 2026 Galeh Ariya Irwana — PENS.
