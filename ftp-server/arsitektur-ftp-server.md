# Arsitektur FTP Server

Arsitektur FTP adalah model Client-Server yang unik karena menggunakan dua koneksi (channel) terpisah antara klien dan server. Ini adalah poin terpenting dalam arsitekturnya.

1. Control Channel (Koneksi Kontrol)
   * Port: Selalu menggunakan port 21 di sisi server.
   * Tujuan: Koneksi ini dibuat saat klien pertama kali terhubung ke server. Fungsinya adalah untuk mengirimkan perintah (command) dan menerima respons.
   * Contoh Perintah: `USER` (kirim username), `PASS` (kirim password), `LIST` (minta daftar file), `RETR` (minta download file), `STOR` (kirim upload file).
   * Koneksi ini tetap terbuka selama sesi FTP berlangsung.
2. Data Channel (Koneksi Data)
   * Port: Port-nya _dinamis_ (berubah-ubah).
   * Tujuan: Koneksi ini hanya digunakan untuk mentransfer data yang sebenarnya, yaitu isi file yang diunduh/diunggah atau daftar direktori (hasil perintah `LIST`).
   * Koneksi ini dibuka setiap kali ada transfer data dan ditutup setelah selesai.

* **Client** → komputer yang meminta layanan (misalnya komputer siswa).
* **Server** → komputer yang menyediakan layanan (misalnya komputer server sekolah).

<figure><img src="../.gitbook/assets/image (632).png" alt=""><figcaption></figcaption></figure>

Cara Kerjanya :

1. **Client** terhubung ke **Server** melalui jaringan.
2. Client **login** menggunakan username & password (atau login anonymous).
3. Setelah login, client bisa:
   * **Upload file** → kirim file ke server.
   * **Download file** → ambil file dari server.
4. Setelah selesai, client **disconnect** dari server.
