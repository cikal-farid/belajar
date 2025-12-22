# Mode Koneksi Data

#### Mode Koneksi Data

Bagaimana "Data Channel" ini dibuat menentukan dua mode operasi FTP:

**A. Active Mode (Mode Aktif)**

1. Klien (misal port 3000) terhubung ke port 21 server (Control Channel).
2. Klien memberi tahu server, "Hai, aku akan mendengarkan di port 3001 untuk data." (Menggunakan perintah `PORT`).
3. Server kemudian secara aktif membuat koneksi _baru_ dari port 20 miliknya ke port 3001 klien (Data Channel).
4. Data ditransfer.

* Masalah: Mode ini sering gagal jika klien berada di belakang Firewall atau NAT (Router). Firewall di sisi klien akan memblokir koneksi masuk yang "asing" dari port 20 server.

**B. Passive Mode (Mode Pasif)**

1. Klien (misal port 3000) terhubung ke port 21 server (Control Channel).
2. Klien mengirim perintah `PASV` (Passive).
3. Server menjawab, "Oke, silakan hubungkan dirimu ke port _saya_ yang acak, misalnya port 55000."
4. Klien secara aktif membuat koneksi _baru_ dari port 3001 miliknya ke port 55000 server (Data Channel).
5. Data ditransfer.

* Keuntungan: Ini adalah mode yang paling umum dan disukai saat ini karena koneksi _selalu_ diinisiasi dari sisi klien, sehingga lebih ramah terhadap firewall.
