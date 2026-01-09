# Selinux

**SELinux** singkatan dari **Security-Enhanced Linux**.\
Ini adalah **lapisan keamanan tambahan** di sistem operasi Linux yang dikembangkan oleh **NSA (National Security Agency)**.\
Fungsinya adalah untuk **mengontrol apa yang boleh dan tidak boleh dilakukan oleh program di sistem**.

Biasanya di Linux, sistem keamanan bekerja berdasarkan **user dan permission**:

* File punya owner (pemilik), group, dan permission (read, write, execute).
* Kalau permission cocok → boleh akses. Kalau tidak → ditolak.

Nah, **SELinux menambahkan aturan keamanan yang lebih ketat lagi** di atas itu.\
Jadi walaupun permission biasa sudah mengizinkan, **kalau kebijakan SELinux melarang, tetap tidak bisa.**

#### Akan tetapi SELinux ini akan bekerja maksimal dan aman pada os CentOS, RHEL, Fedora karena secara default framework securitynya memang SELinux pada os yang disebutkan diatas, beda dengan default framework security nya OS Ubuntu ini adalah AppArmor, akan tetap berjalan menggunakan SELinux namun belum maksimal karena akan stuck pada saat server off atau restart, hanya bisa mode permissive dan jika server sudah on baru jalankan "sudo setenforce 1"

| Distro                     | Default Security Framework | Efek jika pakai SELinux                                                              |
| -------------------------- | -------------------------- | ------------------------------------------------------------------------------------ |
| **Ubuntu**                 | AppArmor                   | SELinux bisa jalan, tapi **butuh konfigurasi manual** dan rawan error jika enforcing |
| **CentOS / RHEL / Fedora** | SELinux                    | Sudah stabil, aman untuk enforcing                                                   |

| Mode           | Aman di Ubuntu?                          | Risiko Boot Stuck            | Saran                             |
| -------------- | ---------------------------------------- | ---------------------------- | --------------------------------- |
| **Disabled**   | Aman                                     | ❌ Tidak pakai SELinux        | Hanya untuk testing               |
| **Permissive** | ✅ Aman sekali                            | ⚪ Hampir tidak ada           | Cocok untuk belajar & audit       |
| **Enforcing**  | ⚠️ Bisa stuck jika relabel belum selesai | ⚠️ Perlu hati-hati di Ubuntu | Gunakan setelah semua label benar |

* **Ubuntu secara default tidak memakai SELinux**,\
  melainkan sistem keamanan lain bernama **AppArmor**.
* Paket `libselinux1` hanya dibutuhkan agar aplikasi yang “tahu SELinux” tidak error,\
  **bukan berarti SELinux aktif**.
