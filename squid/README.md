# Squid

Squid adalah sebuah perangkat lunak yang digunakan sebagai [_proxy server_](https://id.wikipedia.org/wiki/Proxy_server) dan [_web cache_](https://id.wikipedia.org/wiki/Web_cache?action=edit\&redlink=1). Squid memiliki banyak jenis penggunaan, mulai dari mempercepat [server web](https://id.wikipedia.org/wiki/Server_web) dengan melakukan _caching_ permintaan yang berulang-ulang, _caching_ [DNS](https://id.wikipedia.org/wiki/DNS), caching situs web, dan caching pencarian [komputer](https://id.wikipedia.org/wiki/Komputer) di dalam [jaringan](https://id.wikipedia.org/wiki/Jaringan_komputer) untuk sekelompok komputer yang menggunakan sumber daya jaringan yang sama, hingga pada membantu [keamanan](https://id.wikipedia.org/wiki/Keamanan_komputer) dengan cara melakukan penyaringan (_filter_) lalu lintas.

Secara Teknis: Squid adalah server Web Proxy Cache. Ia menerima permintaan (request) halaman web dari pengguna, lalu memutuskan apakah akan mengambil data baru dari internet atau memberikan data yang sudah ia simpan (cache) sebelumnya.

Fungsi Utamanya:

* Caching (Mempercepat Akses): Menyimpan data web yang sering dibuka agar tidak perlu didownload berulang kali.
* Filtering (Menyaring Konten): Memblokir situs-situs tertentu (misal: situs judi, iklan, atau media sosial di jam kerja).
* Security & Anonymity: Menyembunyikan IP asli pengguna agar lebih aman saat berselancar.

Arsitektur Squid bisa kita lihat dari dua sisi: Sisi Luar (Posisi di Jaringan) dan Sisi Dalam (Cara Otaknya Bekerja).

**A. Arsitektur Eksternal (Posisi di Jaringan)**

Ini adalah bagaimana Squid ditempatkan dalam jaringan kantor atau rumah Anda.

1. Client (Pengguna): Laptop/HP Anda yang ingin browsing.
2. Squid Proxy Server: Komputer/Server di tengah yang terinstall Squid.
3. Internet / Origin Server: Server tujuan (misal: server Google, Facebook, Detik).

Alur Data:

> `Komputer Anda` <---> `Squid Proxy` <---> `Internet`

_Tanpa Squid, komputer Anda langsung terhubung ke Internet. Dengan Squid, semua lalu lintas "mampir" dulu di server Squid._

**B. Arsitektur Internal (Isi "Otak" Squid)**

Bagian ini menjelaskan apa yang terjadi di dalam Squid saat menerima permintaan. Squid memiliki beberapa komponen kunci:

1. ACL (Access Control List) - "Si Satpam Pintu": Ini adalah komponen pertama yang bekerja. Saat ada permintaan masuk, ACL mengecek aturan:
   * _Apakah IP komputer ini boleh internetan?_
   * _Apakah situs yang dituju diblokir?_
   * _Apakah ini jam kerja yang dilarang buka YouTube?_
   * Jika lolos, lanjut ke proses berikutnya. Jika tidak, akses ditolak.
2. Storage Manager (Cache) - "Si Pengelola Gudang": Bagian ini mengatur penyimpanan data.
   * Memori (RAM): Untuk data yang _sangat sering_ diminta dan berukuran kecil (akses super cepat).
   * Disk (Hard Drive): Untuk data yang lebih besar dan agak jarang diminta (akses lebih lambat dari RAM, tapi kapasitas besar).
3. DNS Client - "Si Buku Telepon": Jika Squid harus mengambil data baru dari internet, komponen ini bertugas menerjemahkan nama domain (misal: `google.com`) menjadi alamat IP agar bisa dihubungi.
4. Protocol Handler: Penerjemah bahasa komunikasi (HTTP/HTTPS/FTP) agar Squid mengerti apa yang diminta oleh browser Anda.

#### Ringkasan Keuntungan Arsitektur Ini

* Hemat Bandwidth: Karena banyak data diambil dari laci lokal (cache), kuota internet kantor lebih irit.
* Internet Lebih Cepat (Low Latency): Mengambil data dari server lokal (Squid) jauh lebih cepat daripada mengambil dari server di luar negeri.
* Kontrol Penuh: Admin bisa melihat siapa membuka apa dan memblokir yang tidak perlu.
