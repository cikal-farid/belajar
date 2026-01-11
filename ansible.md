# Ansible

**Ansible** adalah tool **automation** untuk mengelola banyak server sekaligus (install, konfigurasi, deploy, restart service, copy file, dll) **dari satu “control node”** (laptop/VM kamu), dengan cara **push** perintah ke server-server target.

Ciri khas Ansible:

* **Agentless**: di VM target **tidak perlu install agent** khusus (cukup bisa **SSH**).
* Pakai **YAML Playbook** (mudah dibaca pemula).
* **Idempotent**: kalau playbook dijalankan berkali-kali, hasilnya tetap konsisten (tidak “ngulang-ngulang” bikin rusak).
* Banyak “module” siap pakai: `apt`, `service`, `copy`, `template`, `ufw`, dll.

***

### Sejarah Ansible (ringkas + penting)

* **2012**: Ansible dibuat oleh **Michael DeHaan** (tujuannya bikin automation yang simpel, agentless, dan mudah dipakai).
* Setelah itu Ansible cepat populer karena “tinggal SSH + YAML”.
* **2015**: **Red Hat mengakuisisi Ansible** dan menjadikannya bagian besar dari ekosistem automation enterprise (platform/produk enterprise-nya berkembang, tapi core open-source tetap ada).
* Sekarang Ansible dipakai luas untuk: provisioning, konfigurasi server, patching, deployment aplikasi, hingga otomasi operasional.

**Konsep Ansible itu “remote automation”**: kamu menjalankan perintah/konfigurasi dari 1 mesin **control node** ke mesin lain **managed node** lewat **SSH** (jadi mirip remote), bedanya Ansible bikin semuanya **rapi, bisa diulang, dan konsisten** (pakai playbook YAML + modul).

***

## Runbook (Ubuntu 24) — VM 192.168.56.19 sebagai Ansible, remote ke 192.168.56.20

### Target

* Control node: `192.168.56.19` (Ubuntu 24)
* Managed node: `192.168.56.20` (sudah terinstall nginx)
* Goal: dari `.19` kamu bisa:
  * cek nginx di `.20`
  * pastikan nginx running + enabled
  * deploy `index.html` sederhana (biar keliatan hasilnya)

> Semua command di bawah dijalankan di **VM 192.168.56.19** kecuali disebutkan lain.

***

### 1) Pastikan koneksi jaringan & SSH

```bash
ping -c 2 192.168.56.20
```

Tes SSH (ganti `<user_vm>` sesuai user di VM .20, misal `cikal`):

```bash
ssh cikal@192.168.56.20 "hostname && whoami"
```

Kalau belum ada SSH server di `.20`, di VM `.20` install:

```bash
sudo apt update
sudo apt install -y openssh-server
```

***

### 2) Install Ansible di VM 192.168.56.19 (Ubuntu 24)

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

***

### 3) Setup SSH key (paling enak untuk pemula)

Buat SSH key di `.19` (kalau belum ada):

```bash
ssh-keygen -t ed25519 -C "ansible"
```

Tekan **Enter** terus sampai selesai.

Copy key ke `.20`:

```bash
ssh-copy-id cikal@192.168.56.20
```

Tes login tanpa password:

```bash
ssh cikal@192.168.56.20 "echo OK"
```

***

### 4) Buat folder project Ansible

```bash
mkdir -p ~/ansible-nginx-onevm/{inventory,playbooks}
cd ~/ansible-nginx-onevm
```

***

### 5) Buat Inventory (daftar host target)

Buat file:

```bash
nano inventory/hosts.ini
```

Isi seperti ini (ganti `<user_vm>`):

```ini
[nginx_target]
192.168.56.20

[nginx_target:vars]
ansible_user=cikal
ansible_become=true
ansible_become_method=sudo
```

***

### 6) Tes koneksi Ansible ke VM target

```bash
ansible -i inventory/hosts.ini nginx_target -m ping
```

Kalau sukses, akan ada `SUCCESS` dari `192.168.56.20`.

> Kalau gagal karena sudo minta password, jalankan command/playbook nanti dengan `--ask-become-pass`.

***

### 7) “Cek sederhana” Nginx di VM target via Ansible (command-line)

#### 7.1 Cek status nginx (active/enabled)

```bash
ansible -i inventory/hosts.ini nginx_target -b -m shell -a "systemctl is-active nginx && systemctl is-enabled nginx"
```

#### 7.2 Cek versi nginx

```bash
ansible -i inventory/hosts.ini nginx_target -m shell -a "nginx -v"
```

#### 7.3 Cek port 80 listen

```bash
ansible -i inventory/hosts.ini nginx_target -b -m shell -a "ss -lntp | grep -E ':80\s' || true"
```

***

### 8) Playbook: pastikan nginx running + enabled + deploy index.html

Buat file:

```bash
nano playbooks/nginx.yml
```

Isi:

```yaml
---
- name: Manage Nginx on single remote VM
  hosts: nginx_target
  become: true

  tasks:
    - name: Ensure nginx is installed (aman walau sudah terinstall)
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy simple index.html
      copy:
        dest: /var/www/html/index.html
        content: |
          <html>
          <head><title>Nginx via Ansible</title></head>
          <body>
            <h1>OK - Nginx managed by Ansible</h1>
            <p>Target: {{ inventory_hostname }}</p>
          </body>
          </html>
        owner: root
        group: root
        mode: "0644"
```

Jalankan:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/nginx.yml
```

Kalau butuh password sudo:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/nginx.yml --ask-become-pass
```

***

### 9) Validasi hasil (dari VM 192.168.56.19)

```bash
curl -s http://192.168.56.20 | head -n 20
```

Harusnya muncul HTML dengan teks “OK - Nginx managed by Ansible”.

***

### 10) Troubleshooting cepat

#### A) SSH masih minta password terus

Pastikan `ssh-copy-id` berhasil dan kamu bisa:

```bash
ssh cikal@192.168.56.20 "echo OK"
```

#### B) Sudo minta password saat Ansible jalan

Pakai:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/nginx.yml --ask-become-pass
```

#### C) Nginx aktif tapi curl gagal

Cek dari Ansible:

```bash
ansible -i inventory/hosts.ini nginx_target -b -m shell -a "systemctl status nginx --no-pager -l | head -n 40"
```

***
