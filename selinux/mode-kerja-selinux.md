# Mode kerja SELinux

#### Mode kerja SELinux

1. **Enforcing** → aktif penuh, aturan dijalankan dan pelanggaran diblokir.
2. **Permissive** → hanya mencatat pelanggaran di log (tidak diblokir).
3. **Disabled** → SELinux dimatikan total.

cek mode sekarang

```
getenforce
```

setenforce 0 # jadi permissive\
setenforce 1 # aktif enforcing lagi

```
setenforce 0
```

```
setenforce 1
```
