# FTP Server

FTP adalah singkatan dari File Transfer Protocol. Sesuai namanya, ini adalah protokol (aturan) standar di jaringan komputer yang digunakan untuk mentransfer atau bertukar file dari satu komputer ke komputer lain melalui jaringan TCP/IP (seperti internet atau intranet).

FTP Server adalah perangkat lunak (software) yang berjalan di sebuah komputer (server) yang bertugas untuk "mendengarkan" permintaan koneksi dari klien FTP.

Fungsi utamanya adalah sebagai berikut:

* Menyediakan Penyimpanan: Menjadi repositori pusat tempat file-file disimpan.
* Mengatur Akses: Mengelola siapa saja (autentikasi pengguna) yang boleh terhubung dan apa yang bisa mereka lakukan (otorisasi), seperti:
  * Mengunduh (Download): Mengambil file dari server ke komputer klien.
  * Mengunggah (Upload): Mengirim file dari komputer klien ke server.
  * Melihat Direktori (List): Menampilkan daftar file dan folder di server.
  * Menghapus/Mengganti Nama: Memodifikasi file di server (jika diizinkan).

Komputer yang meminta koneksi dan file dari FTP Server disebut FTP Client. Contoh FTP Client populer adalah FileZilla, WinSCP, atau bahkan command line (`ftp`) di komputer Anda.
