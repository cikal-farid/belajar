# Page 1

[Lewati ke konten](https://chatgpt.com/c/694901f2-e9e0-8323-8ba4-2a217bad04df#main)

### Riwayat obrolan

Undang anggota tim

**Anda bilang:**

cikal@vm-docker:\~/three-body-problem-main$ git commit --allow-empty -m "chore: trigger pipeline" git push gitlab main \[main 1da2be5] chore: trigger pipeline ssh: Could not resolve hostname altssh.gitlab.com: Temporary failure in name resolution fatal: Could not read from remote repository. Please make sure you have the correct access rights and the repository exists. aku mau baru hidupkan vm lagi ada 4 vm yaitu terdiri dari vm-docker, vm-harbor, vm k8s, vm-worker dan aku ingin running pipeline di vm-docker kok ada error ya? tolong bantu perbaiki dan cek secara detail dan berikan arahan yang mudah aku pahami ya

**ChatGPT bilang:**

Selesai berpikir selama 27s

Masalah utamanya ada di sini:

> `ssh: Could not resolve hostname altssh.gitlab.com: Temporary failure in name resolution`

Itu **bukan error akses repo / SSH key dulu**, tapi **VM kamu gagal resolve DNS (nama domain → IP)**. Jadi `git push` berhenti sebelum sempat autentikasi.

Di bawah ini aku kasih **cek detail + cara perbaikan** yang paling gampang diikuti (urut dari yang paling cepat).

***

### 1) Pastikan remote Git kamu benar (biar nggak salah target)

Di `vm-docker`, dari folder repo:

```bash
git remote -v
git remote show gitlab
```

Yang perlu kamu lihat:

* Remote `gitlab` itu menunjuk ke mana?
* Kalau kamu pakai GitLab SaaS, kadang URL-nya bentuknya:
  * `ssh://git@altssh.gitlab.com:443/namespace/repo.git` (SSH via port 443)
  * atau `git@gitlab.com:namespace/repo.git` (SSH normal port 22)

Kalau ternyata kamu **pakai GitLab self-hosted** (mis. `gitlab.company.local`), tapi remotemu malah `altssh.gitlab.com`, ya itu keliru → harus diganti.

***

### 2) Cek apakah network VM hidup (tes IP dulu, baru DNS)

Jalankan:

```bash
ip a
ip r
ping -c 3 1.1.1.1
```

Interpretasi cepat:

* ✅ Kalau `ping 1.1.1.1` **berhasil** → internet/routing OK, lanjut cek DNS
* ❌ Kalau `ping 1.1.1.1` **gagal** → problem routing/gateway (bagian 5)

***

### 3) Cek DNS (ini yang paling mungkin penyebabnya)

Tes resolve domain:

```bash
getent hosts altssh.gitlab.com
getent hosts gitlab.com
```

Kalau output kosong / error → DNS bermasalah.

Lihat resolver yang dipakai VM:

```bash
cat /etc/resolv.conf
```

Hal yang sering kejadian setelah VM “baru dihidupkan”:

* nameserver kosong / mengarah ke DNS internal yang sedang tidak bisa diakses
* `/etc/resolv.conf` mengarah ke `127.0.0.53` tapi service `systemd-resolved` mati

Cek status `systemd-resolved`:

```bash
systemctl status systemd-resolved --no-pager
```

***

### 4) Fix cepat (paling gampang) untuk DNS

Kalau kamu butuh cepat biar bisa `git push`, coba set DNS publik.

#### Opsi A — kalau VM pakai systemd-resolved (umum di Ubuntu)

Edit config:

```bash
sudo nano /etc/systemd/resolved.conf
```

Isi / pastikan bagian ini ada:

```ini
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9
```

Restart:

```bash
sudo systemctl restart systemd-resolved
```

Lalu tes lagi:

```bash
getent hosts altssh.gitlab.com
```

> Kalau `/etc/resolv.conf` bukan symlink yang benar, rapikan dengan:

```bash
ls -l /etc/resolv.conf
```

Biasanya yang sehat itu symlink ke stub milik systemd-resolved. Kalau kamu lihat file “biasa” yang ke-lock, bilang ya—nanti aku arahkan rapihin yang aman sesuai distro kamu.

#### Opsi B — fix darurat (langsung edit resolv.conf)

Ini cepat tapi kadang ketimpa DHCP/Netplan setelah reboot:

```bash
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
printf "nameserver 1.1.1.1\nnameserver 8.8.8.8\n" | sudo tee /etc/resolv.conf
```

Tes lagi:

```bash
getent hosts altssh.gitlab.com
```

***

### 5) Kalau ping ke IP saja gagal (problem gateway/routing)

Kalau di langkah (2) `ping 1.1.1.1` gagal, lakukan:

```bash
ip r
```

Cari baris `default via ... dev ...`

Kalau tidak ada `default` route:

* kemungkinan DHCP tidak jalan / interface belum up
* solusi cepat (tergantung service yang dipakai):

Coba ini (Ubuntu server biasanya):

```bash
sudo systemctl restart systemd-networkd
sudo systemctl restart systemd-resolved
```

Atau kalau pakai NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

Kalau kamu pakai netplan:

```bash
ls /etc/netplan
sudo netplan try
# atau
sudo netplan apply
```

***

### 6) Setelah DNS beres, pastikan port akses GitLab tidak keblok

Karena kamu pakai `altssh.gitlab.com`, biasanya itu **SSH over port 443**. Tes:

```bash
nc -vz altssh.gitlab.com 443
```

Kalau sukses, tes SSH:

```bash
ssh -T -p 443 git@altssh.gitlab.com
```

Kalau port 443 keblok dari jaringan kamu, alternatifnya:

* pakai SSH normal ke `gitlab.com:22` (kalau 22 boleh), atau
* pindah ke HTTPS remote.

***

### 7) Alternatif paling stabil: pakai HTTPS remote (kalau SSH ribet)

Kalau kamu mau bypass isu SSH/DNS-port tertentu (setelah DNS normal), kamu bisa switch remote ke HTTPS:

```bash
git remote set-url gitlab https://gitlab.com/<namespace>/<repo>.git
git push gitlab main
```

Nanti GitLab biasanya minta username + token (Personal Access Token), bukan password.

***

### 8) Checklist cepat biar pipeline ke-trigger

Setelah jaringan beres:

```bash
git status
git log -1 --oneline
git push gitlab main
```

Kalau sukses push, pipeline harus ke-trigger (karena commit baru sudah naik ke branch `main`).

***

## Biar aku bisa “cek detail” dengan akurat

Kamu jalankan dan paste output 6 command ini aja (cukup copy-paste):

```bash
git remote -v
ping -c 3 1.1.1.1
getent hosts altssh.gitlab.com
cat /etc/resolv.conf
systemctl status systemd-resolved --no-pager
ip r
```

Dari output itu, aku bisa bilang persis: masalahnya DNS/service resolved/netplan/gateway, dan langkah fix mana yang paling tepat buat setup `vm-docker` kamu.

**Anda bilang:**

sebelum ada perbaikan tolong cek terlebih dahulu ya cikal@vm-docker:\~/three-body-problem-main$ git remote -v git remote show gitlab gitlab git@gitlab-443:cikalfarid/three-body-problem-main.git (fetch) gitlab git@gitlab-443:cikalfarid/three-body-problem-main.git (push) origin https://github.com/cikal-farid/three-body-problem-main.git (fetch) origin https://github.com/cikal-farid/three-body-problem-main.git (push) \* remote gitlab Fetch URL: git@gitlab-443:cikalfarid/three-body-problem-main.git Push URL: git@gitlab-443:cikalfarid/three-body-problem-main.git HEAD branch: main Remote branch: main tracked Local branch configured for 'git pull': main merges with remote main Local ref configured for 'git push': main pushes to main (fast-forwardable) cikal@vm-docker:\~/three-body-problem-main$ ip a ip r ping -c 3 1.1.1.1 1: lo: \<LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 inet 127.0.0.1/8 scope host lo valid\_lft forever preferred\_lft forever inet6 ::1/128 scope host noprefixroute valid\_lft forever preferred\_lft forever 2: enp0s3: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP group default qlen 1000 link/ether 08:00:27:52:ac:ef brd ff:ff:ff:ff:ff:ff inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3 valid\_lft 84866sec preferred\_lft 84866sec inet6 fd17:625c:f037:2:ea06:788d:c04c:45ce/64 scope global temporary dynamic valid\_lft 86098sec preferred\_lft 14098sec inet6 fd17:625c:f037:2:a00:27ff:fe52:acef/64 scope global dynamic mngtmpaddr valid\_lft 86098sec preferred\_lft 14098sec inet6 fe80::a00:27ff:fe52:acef/64 scope link valid\_lft forever preferred\_lft forever 3: enp0s8: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP group default qlen 1000 link/ether 08:00:27:56:f9:19 brd ff:ff:ff:ff:ff:ff inet 192.168.56.42/24 brd 192.168.56.255 scope global noprefixroute enp0s8 valid\_lft forever preferred\_lft forever inet6 fe80::a00:27ff:fe56:f919/64 scope link valid\_lft forever preferred\_lft forever 4: br-df396568552e: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP group default link/ether 5e:7b:7c:71:02:ca brd ff:ff:ff:ff:ff:ff inet 172.18.0.1/16 brd 172.18.255.255 scope global br-df396568552e valid\_lft forever preferred\_lft forever inet6 fe80::5c7b:7cff:fe71:2ca/64 scope link valid\_lft forever preferred\_lft forever 5: docker0: \<NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default link/ether da:e1:82:1d:31:16 brd ff:ff:ff:ff:ff:ff inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0 valid\_lft forever preferred\_lft forever 7: veth081d1b4@if2: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master br-df396568552e state UP group default link/ether 6a:15:b3:f8:3d:25 brd ff:ff:ff:ff:ff:ff link-netnsid 1 inet6 fe80::6815:b3ff:fef8:3d25/64 scope link valid\_lft forever preferred\_lft forever 12: veth38fe387@if2: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master br-df396568552e state UP group default link/ether 9e:9a:70:32:06:c5 brd ff:ff:ff:ff:ff:ff link-netnsid 2 inet6 fe80::9c9a:70ff:fe32:6c5/64 scope link valid\_lft forever preferred\_lft forever 13: veth44c44a7@if2: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master br-df396568552e state UP group default link/ether 06:0a:0e:17:c0:b0 brd ff:ff:ff:ff:ff:ff link-netnsid 3 inet6 fe80::40a:eff:fe17:c0b0/64 scope link valid\_lft forever preferred\_lft forever 14: vethdf3b9c8@if2: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master br-df396568552e state UP group default link/ether 6e:a5:14:2b:33:e2 brd ff:ff:ff:ff:ff:ff link-netnsid 4 inet6 fe80::6ca5:14ff:fe2b:33e2/64 scope link valid\_lft forever preferred\_lft forever 16: veth33b2510@if2: \<BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master br-df396568552e state UP group default link/ether 5e:e6:8c:9f:e2:07 brd ff:ff:ff:ff:ff:ff link-netnsid 0 inet6 fe80::5ce6:8cff:fe9f:e207/64 scope link valid\_lft forever preferred\_lft forever default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 default via 192.168.56.1 dev enp0s8 proto static metric 101 10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 172.18.0.0/16 dev br-df396568552e proto kernel scope link src 172.18.0.1 192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.42 metric 101 PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data. 64 bytes from 1.1.1.1: icmp\_seq=1 ttl=255 time=97.8 ms 64 bytes from 1.1.1.1: icmp\_seq=2 ttl=255 time=79.5 ms 64 bytes from 1.1.1.1: icmp\_seq=3 ttl=255 time=355 ms --- 1.1.1.1 ping statistics --- 3 packets transmitted, 3 received, 0% packet loss, time 2005ms rtt min/avg/max/mdev = 79.525/177.491/355.164/125.854 ms cikal@vm-docker:\~/three-body-problem-main$ getent hosts altssh.gitlab.com getent hosts gitlab.com 2606:4700:90:0:f0ff:e6a3:2ac:f7ef 075e5c94415c4071840fa96497443943.pacloudflare.com altssh.gitlab.com 2606:4700:90:0:f22e:fbec:5bed:a9b9 gitlab.com cikal@vm-docker:\~/three-body-problem-main$ cat /etc/resolv.conf # This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8). # Do not edit. # # This file might be symlinked as /etc/resolv.conf. If you're looking at # /etc/resolv.conf and seeing this text, you have followed the symlink. # # This is a dynamic resolv.conf file for connecting local clients to the # internal DNS stub resolver of systemd-resolved. This file lists all # configured search domains. # # Run "resolvectl status" to see details about the uplink DNS servers # currently in use. # # Third party programs should typically not access this file directly, but only # through the symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a # different way, replace this symlink by a static file or a different symlink. # # See man:systemd-resolved.service(8) for details about the supported modes of # operation for /etc/resolv.conf. nameserver 127.0.0.53 options edns0 trust-ad search . cikal@vm-docker:\~/three-body-problem-main$ systemctl status systemd-resolved --no-pager ● systemd-resolved.service - Network Name Resolution Loaded: loaded (/usr/lib/systemd/system/systemd-resolved.service; enabled; preset: enabled) Active: active (running) since Mon 2025-12-22 08:07:01 UTC; 26min ago Docs: man:systemd-resolved.service(8) man:org.freedesktop.resolve1(5) https://www.freedesktop.org/wiki/Software/systemd/writing-network-configuration-managers https://www.freedesktop.org/wiki/Software/systemd/writing-resolver-clients Main PID: 513 (systemd-resolve) Status: "Processing requests..." Tasks: 1 (limit: 2266) Memory: 7.4M (peak: 8.0M) CPU: 1.086s CGroup: /system.slice/systemd-resolved.service └─513 /usr/lib/systemd/systemd-resolved Dec 22 08:07:35 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 8.8.8.8. Dec 22 08:07:39 vm-docker systemd-resolved\[513]: Clock change detected. Flushing caches. Dec 22 08:22:59 vm-docker systemd-resolved\[513]: Grace period over, resuming full feature set (UDP+EDNS0) for DNS se….8.8.8. Dec 22 08:23:04 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 8.8.4.4. Dec 22 08:28:47 vm-docker systemd-resolved\[513]: Grace period over, resuming full feature set (UDP+EDNS0) for DNS se….8.4.4. Dec 22 08:29:02 vm-docker systemd-resolved\[513]: Using degraded feature set UDP instead of UDP+EDNS0 for DNS server 8.8.8.8. Dec 22 08:29:07 vm-docker systemd-resolved\[513]: Using degraded feature set UDP instead of TCP for DNS server 10.0.2.3. Dec 22 08:29:07 vm-docker systemd-resolved\[513]: Using degraded feature set UDP instead of UDP+EDNS0 for DNS server 8.8.4.4. Dec 22 08:29:18 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 10.0.2.3. Dec 22 08:29:23 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 8.8.8.8. Hint: Some lines were ellipsized, use -l to show in full. cikal@vm-docker:\~/three-body-problem-main$ git remote -v ping -c 3 1.1.1.1 getent hosts altssh.gitlab.com cat /etc/resolv.conf systemctl status systemd-resolved --no-pager ip r gitlab git@gitlab-443:cikalfarid/three-body-problem-main.git (fetch) gitlab git@gitlab-443:cikalfarid/three-body-problem-main.git (push) origin https://github.com/cikal-farid/three-body-problem-main.git (fetch) origin https://github.com/cikal-farid/three-body-problem-main.git (push) PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data. 64 bytes from 1.1.1.1: icmp\_seq=1 ttl=255 time=480 ms 64 bytes from 1.1.1.1: icmp\_seq=2 ttl=255 time=268 ms 64 bytes from 1.1.1.1: icmp\_seq=3 ttl=255 time=359 ms --- 1.1.1.1 ping statistics --- 3 packets transmitted, 3 received, 0% packet loss, time 2413ms rtt min/avg/max/mdev = 267.671/369.117/480.198/87.030 ms 2606:4700:90:0:f0ff:e6a3:2ac:f7ef 075e5c94415c4071840fa96497443943.pacloudflare.com altssh.gitlab.com # This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8). # Do not edit. # # This file might be symlinked as /etc/resolv.conf. If you're looking at # /etc/resolv.conf and seeing this text, you have followed the symlink. # # This is a dynamic resolv.conf file for connecting local clients to the # internal DNS stub resolver of systemd-resolved. This file lists all # configured search domains. # # Run "resolvectl status" to see details about the uplink DNS servers # currently in use. # # Third party programs should typically not access this file directly, but only # through the symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a # different way, replace this symlink by a static file or a different symlink. # # See man:systemd-resolved.service(8) for details about the supported modes of # operation for /etc/resolv.conf. nameserver 127.0.0.53 options edns0 trust-ad search . ● systemd-resolved.service - Network Name Resolution Loaded: loaded (/usr/lib/systemd/system/systemd-resolved.service; enabled; preset: enabled) Active: active (running) since Mon 2025-12-22 08:07:01 UTC; 26min ago Docs: man:systemd-resolved.service(8) man:org.freedesktop.resolve1(5) https://www.freedesktop.org/wiki/Software/systemd/writing-network-configuration-managers https://www.freedesktop.org/wiki/Software/systemd/writing-resolver-clients Main PID: 513 (systemd-resolve) Status: "Processing requests..." Tasks: 1 (limit: 2266) Memory: 7.5M (peak: 8.0M) CPU: 1.096s CGroup: /system.slice/systemd-resolved.service └─513 /usr/lib/systemd/systemd-resolved Dec 22 08:07:35 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 8.8.8.8. Dec 22 08:07:39 vm-docker systemd-resolved\[513]: Clock change detected. Flushing caches. Dec 22 08:22:59 vm-docker systemd-resolved\[513]: Grace period over, resuming full feature set (UDP+EDNS0) for DNS se….8.8.8. Dec 22 08:23:04 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 8.8.4.4. Dec 22 08:28:47 vm-docker systemd-resolved\[513]: Grace period over, resuming full feature set (UDP+EDNS0) for DNS se….8.4.4. Dec 22 08:29:02 vm-docker systemd-resolved\[513]: Using degraded feature set UDP instead of UDP+EDNS0 for DNS server 8.8.8.8. Dec 22 08:29:07 vm-docker systemd-resolved\[513]: Using degraded feature set UDP instead of TCP for DNS server 10.0.2.3. Dec 22 08:29:07 vm-docker systemd-resolved\[513]: Using degraded feature set UDP instead of UDP+EDNS0 for DNS server 8.8.4.4. Dec 22 08:29:18 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 10.0.2.3. Dec 22 08:29:23 vm-docker systemd-resolved\[513]: Using degraded feature set TCP instead of UDP for DNS server 8.8.8.8. Hint: Some lines were ellipsized, use -l to show in full. default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 default via 192.168.56.1 dev enp0s8 proto static metric 101 10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 172.18.0.0/16 dev br-df396568552e proto kernel scope link src 172.18.0.1 192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.42 metric 101 cikal@vm-docker:\~/three-body-problem-main$

**ChatGPT bilang:**

Selesai berpikir selama 30s

Oke, aku **cek dulu dari output yang kamu kirim** — dan ini kesimpulannya:

### 1) Dari sisi Git remote: aman, tapi pakai alias SSH `gitlab-443`

Remote `gitlab` kamu bukan langsung `altssh.gitlab.com`, tapi alias:

* `gitlab git@gitlab-443:cikalfarid/three-body-problem-main.git`

Biasanya ini artinya di `~/.ssh/config` ada rule seperti:

* `Host gitlab-443`
  * `Hostname altssh.gitlab.com`
  * `Port 443`
  * `User git`

Jadi saat `git push gitlab main`, **SSH tetap akan resolve `altssh.gitlab.com`** (sesuai error kamu kemarin).

✅ Remote-nya sendiri terlihat benar.

***

### 2) Dari sisi network: internet & routing OK

* `ping 1.1.1.1` sukses (0% packet loss) ✅
* Routing ada default gateway:
  * `default via 10.0.2.2 dev enp0s3 metric 100` ✅ (ini NAT VirtualBox biasanya)
  * Ada default kedua via `enp0s8 metric 101` (lebih “lemah”, jadi tidak dominan)

Jadi **koneksi keluar VM hidup**.

***

### 3) Dari sisi DNS: sekarang resolve jalan, tapi ada indikasi “flaky/intermittent”

Sekarang `getent hosts altssh.gitlab.com` berhasil ✅ (malah dapat IPv6).\
`systemd-resolved` juga running ✅

Tapi log `systemd-resolved` menunjukkan bolak-balik “degraded feature set” untuk DNS server:

* `8.8.8.8`, `8.8.4.4`, dan `10.0.2.3`

Nah ini _nyambung banget_ dengan error kemarin:

> `Temporary failure in name resolution`

Artinya: **kadang resolver gagal jawab / timeout / error sesaat**, walaupun beberapa menit kemudian normal lagi (seperti sekarang).

***

## Langkah cek yang “validasi akar masalahnya” (tanpa ubah apa-apa dulu)

Jalankan ini di `vm-docker`:

#### A) Pastikan `gitlab-443` memang ngarah ke altssh + port 443

```bash
ssh -G gitlab-443 | egrep -i '^(hostname|port|user) '
```

#### B) Test SSH koneksi pakai alias yang sama persis dengan git

```bash
ssh -T gitlab-443
```

Kalau sukses biasanya outputnya semacam “Welcome…” / “You are authenticated…” (meski tidak dapat shell).

#### C) Test port 443 ke altssh

```bash
nc -vz altssh.gitlab.com 443
```

Kalau `nc` belum ada:

```bash
sudo apt-get update && sudo apt-get install -y netcat-openbsd
```

***

## Perbaikan yang paling masuk akal (biar nggak kambuh): stabilkan DNS yang dipakai systemd-resolved

Karena internet OK, dan DNS kadang degrade, target kita: **paksa VM pakai DNS publik yang stabil** dan (kalau bisa) **tidak bergantung ke `10.0.2.3`**.

### Opsi 1 (recommended): set DNS di systemd-resolved

1. Edit config:

```bash
sudo nano /etc/systemd/resolved.conf
```

2. Isi / aktifkan bagian ini:

```ini
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 8.8.4.4
```

3. Restart:

```bash
sudo systemctl restart systemd-resolved
```

4. Bersihkan cache + cek server DNS yang kepakai:

```bash
sudo resolvectl flush-caches
resolvectl status | sed -n '1,120p'
```

5. Uji resolve:

```bash
resolvectl query altssh.gitlab.com
```

> Catatan: `/etc/resolv.conf` kamu sudah benar (stub ke `127.0.0.53`), jadi **nggak perlu diedit manual**.

***

## Uji ulang push untuk trigger pipeline

Setelah cek/beres:

```bash
git push gitlab main
```

Kalau mau paksa detail log SSH (biar kelihatan macet di DNS atau auth):

```bash
GIT_SSH_COMMAND="ssh -vv" git push gitlab main
```

***

### Kemungkinan hasil & artinya (biar kamu gampang baca)

* Jika error balik lagi **“Could not resolve hostname …”** → 99% DNS masih intermittent → lakukan “stabilkan DNS” di atas.
* Jika jadi error **“Permission denied (publickey)”** → DNS sudah beres, tinggal urusan SSH key/akses GitLab.
* Jika error **timeout / no route** → cek port 443 diblok atau IPv6/IPv4 issue.

***

Kalau kamu jalankan **3 perintah validasi** ini dan paste outputnya:

1. `ssh -G gitlab-443 | egrep -i '^(hostname|port|user) '`
2. `ssh -T gitlab-443`
3. `nc -vz altssh.gitlab.com 443`

…aku bisa pinpoint: ini murni DNS, port 443 keblok, atau SSH key yang belum kepasang.

**Anda bilang:**

dari konsep dibawah ini 1. vm-docker = GitLab Runner + build/push image + edge Nginx (HTTPS + rate-limit) + staging (docker compose) 2. vm-harbor = Harbor registry (harbor.local) 3. vm-k8s = Kubernetes control-plane (API server) 4. vm-worker = Kubernetes worker node (tempat pod prod jalan) bisa bantu aku cek pipeline dibawah ini? karena jika aku cek stage nya tidak ada pull atau deploy ke kuber nya, atau boleh kamu rombak tapi hati-hati jangan sampai ada error dan rombak jadi ada stage terdiri dari build di docker, push ke harbor, pull dan deploy ke kubernetes nya stages: - build - deploy\_staging - deploy\_prod workflow: rules: - if: '$CI\_COMMIT\_BRANCH == "main"' - when: never variables: DOCKER\_BUILDKIT: "1" TAG: "$CI\_COMMIT\_SHORT\_SHA" HARBOR\_HOST: "harbor.local" HARBOR\_PROJECT: "threebody" REGISTRY: "$HARBOR\_HOST/$HARBOR\_PROJECT" # Prod k8s settings K8S\_NS\_PROD: "threebody-prod" KUBECTL\_VERSION: "v1.30.14" K8S\_ROLLOUT\_TIMEOUT: "10m" K8S\_JOB\_TIMEOUT: "10m" K8S\_FORCE\_DELETE\_TERMINATING: "0" default: tags: \["deploy"] before\_script: - set -euo pipefail build\_images: stage: build script: | echo "==> \[build] login Harbor (robot)..." : "${HARBOR\_USERNAME:?Missing HARBOR\_USERNAME}" : "${HARBOR\_PASSWORD:?Missing HARBOR\_PASSWORD}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_HOST" -u "$HARBOR\_USERNAME" --password-stdin echo "==> \[build] build images..." docker build \ --build-arg REACT\_APP\_GO\_API\_BASE=/go \ --build-arg REACT\_APP\_LARAVEL\_API\_BASE=/laravel \ -t "$REGISTRY/frontend:$TAG" \ -f frontend/Dockerfile frontend docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel echo "==> \[build] push images..." docker push "$REGISTRY/frontend:$TAG" docker push "$REGISTRY/go:$TAG" docker push "$REGISTRY/laravel:$TAG" deploy\_staging: stage: deploy\_staging needs: \["build\_images"] script: | echo "==> \[staging] validasi variables..." : "${HARBOR\_USERNAME:?Missing HARBOR\_USERNAME}" : "${HARBOR\_PASSWORD:?Missing HARBOR\_PASSWORD}" : "${EDGE\_TLS\_CRT:?Missing EDGE\_TLS\_CRT (GitLab Variables Type: File)}" : "${EDGE\_TLS\_KEY:?Missing EDGE\_TLS\_KEY (GitLab Variables Type: File)}" : "${MYSQL\_ROOT\_PASSWORD:?Missing MYSQL\_ROOT\_PASSWORD}" : "${MYSQL\_DATABASE:?Missing MYSQL\_DATABASE}" : "${MYSQL\_USER:?Missing MYSQL\_USER}" : "${MYSQL\_PASSWORD:?Missing MYSQL\_PASSWORD}" : "${LARAVEL\_APP\_KEY:?Missing LARAVEL\_APP\_KEY}" echo "==> \[staging] cek sudo NOPASSWD (runner harus bisa systemctl tanpa password)..." SYSTEMCTL="$(command -v systemctl || true)" if \[ -z "$SYSTEMCTL" ]; then echo "ERROR: systemctl tidak ditemukan di runner" exit 1 fi if ! sudo -n "$SYSTEMCTL" --version >/dev/null 2>&1; then echo "ERROR: runner tidak bisa sudo tanpa password untuk systemctl." echo "FIX di vm-docker: /etc/sudoers.d/gitlab-runner-threebody harus allow /usr/bin/systemctl NOPASSWD." exit 1 fi SUDO="sudo -n" echo "==> \[staging] login Harbor untuk runner..." echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_HOST" -u "$HARBOR\_USERNAME" --password-stdin echo "==> \[staging] salin docker auth ke user cikal (agar systemd pull setelah reboot)..." test -s "$HOME/.docker/config.json" || { echo "ERROR: $HOME/.docker/config.json tidak ada/kosong"; exit 1; } $SUDO install -d -m 700 -o cikal -g cikal /home/cikal/.docker $SUDO install -m 600 -o cikal -g cikal "$HOME/.docker/config.json" /home/cikal/.docker/config.json echo "==> \[staging] sync deploy files ke /opt/threebody/ (sesuai rancangan systemd)..." $SUDO install -d -m 755 /opt/threebody/edge /opt/threebody/staging $SUDO rsync -a --delete deploy/edge/ /opt/threebody/edge/ $SUDO rsync -a --delete deploy/staging/ /opt/threebody/staging/ echo "==> \[staging] pastikan symlink docker-compose.yml..." $SUDO ln -sf docker-compose.edge.yml /opt/threebody/edge/docker-compose.yml $SUDO ln -sf docker-compose.staging.yml /opt/threebody/staging/docker-compose.yml echo "==> \[staging] tulis TLS cert untuk edge..." $SUDO install -d -m 755 /opt/threebody/edge/certs $SUDO install -m 600 "$EDGE\_TLS\_CRT" /opt/threebody/edge/certs/tls.crt $SUDO install -m 600 "$EDGE\_TLS\_KEY" /opt/threebody/edge/certs/tls.key echo "==> \[staging] tulis .env staging (dibaca systemd)..." $SUDO install -o cikal -g docker -m 640 /dev/null /opt/threebody/staging/.env { echo "REGISTRY=$REGISTRY" echo "TAG=$TAG" echo "" echo "MYSQL\_ROOT\_PASSWORD=$MYSQL\_ROOT\_PASSWORD" echo "MYSQL\_DATABASE=$MYSQL\_DATABASE" echo "MYSQL\_USER=$MYSQL\_USER" echo "MYSQL\_PASSWORD=$MYSQL\_PASSWORD" echo "LARAVEL\_APP\_KEY=$LARAVEL\_APP\_KEY" } | $SUDO tee /opt/threebody/staging/.env >/dev/null echo "==> \[staging] restart service via systemd..." $SUDO systemctl daemon-reload $SUDO systemctl restart threebody-edge threebody-staging $SUDO systemctl status threebody-edge threebody-staging --no-pager -l || true echo "==> \[staging] healthcheck staging (via edge)..." for i in $(seq 1 30); do if curl -kfsS --resolve staging.local:443:127.0.0.1 https://staging.local/ >/dev/null 2>&1; then echo "OK: staging sehat" exit 0 fi sleep 2 done echo "ERROR: staging healthcheck gagal" docker ps || true docker logs --tail 200 edge-nginx || true exit 1 deploy\_prod: stage: deploy\_prod needs: \["deploy\_staging"] script: | echo "==> \[prod] (tetap sama seperti file kamu sekarang)" echo "NOTE: deploy\_prod lanjut setelah staging beres."

**ChatGPT bilang:**

Selesai berpikir selama 1m 57s

Sip — dari pipeline yang kamu kirim, diagnosisnya jelas:

### Kenapa sekarang “nggak ada pull/deploy ke Kubernetes”?

* Job **`build_images`** sudah **build + push** ke Harbor ✅
* Job **`deploy_staging`** deploy **staging via systemd + docker compose** di **vm-docker** ✅
* Job **`deploy_prod`** cuma placeholder (cuma echo) ❌ → jadi **tidak ada langkah kubectl/apply/rollout**, makanya **tidak pernah deploy ke k8s**

Selain itu, kamu juga minta stage terpisah: **build → push → pull+deploy k8s**. Di Kubernetes, “pull image” itu **otomatis dilakukan node (vm-worker) saat rollout** asalkan:

* image tag berubah, dan
* ada **imagePullSecret** yang valid untuk `harbor.local`

Jadi di pipeline kita bikin step yang memicu node untuk pull: **update image tag + rollout**.

***

## Rombakan pipeline yang aman (build → push → staging → prod/k8s)

Di bawah ini versi `.gitlab-ci.yml` yang **tetap mempertahankan staging kamu** (hampir sama), tapi menambahkan:

* stage `push`
* job `deploy_prod` beneran pakai `kubectl` + `rollout`
* memastikan **imagePullSecret** untuk Harbor ada di namespace prod

> Catatan: aku bikin push job **build ulang lalu push** (bukan mengandalkan image hasil job build sebelumnya), ini paling aman kalau suatu hari kamu punya >1 runner.

```yaml
stages:
  - build
  - push
  - deploy_staging
  - deploy_prod

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"

  HARBOR_HOST: "harbor.local"
  HARBOR_PROJECT: "threebody"
  REGISTRY: "$HARBOR_HOST/$HARBOR_PROJECT"

  # Prod k8s settings
  K8S_NS_PROD: "threebody-prod"
  KUBECTL_VERSION: "v1.30.14"
  K8S_ROLLOUT_TIMEOUT: "10m"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail

# 1) Build stage: validasi Dockerfile bisa dibuild (tanpa push)
build_check:
  stage: build
  script: |
    echo "==> [build] build-check images (no push)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel

# 2) Push stage: build ulang lalu push ke Harbor (paling aman lintas-runner)
push_images:
  stage: push
  needs: ["build_check"]
  script: |
    echo "==> [push] login Harbor (robot)..."
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [push] build + push images..."
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker push "$REGISTRY/frontend:$TAG"

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker push "$REGISTRY/go:$TAG"

    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    docker push "$REGISTRY/laravel:$TAG"

# 3) Staging: tetap pakai punyamu (cuma needs diganti dari build_images -> push_images)
deploy_staging:
  stage: deploy_staging
  needs: ["push_images"]
  script: |
    echo "==> [staging] validasi variables..."
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${EDGE_TLS_CRT:?Missing EDGE_TLS_CRT (GitLab Variables Type: File)}"
    : "${EDGE_TLS_KEY:?Missing EDGE_TLS_KEY (GitLab Variables Type: File)}"
    : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    : "${MYSQL_DATABASE:?Missing MYSQL_DATABASE}"
    : "${MYSQL_USER:?Missing MYSQL_USER}"
    : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"
    : "${LARAVEL_APP_KEY:?Missing LARAVEL_APP_KEY}"

    echo "==> [staging] cek sudo NOPASSWD (runner harus bisa systemctl tanpa password)..."
    SYSTEMCTL="$(command -v systemctl || true)"
    if [ -z "$SYSTEMCTL" ]; then
      echo "ERROR: systemctl tidak ditemukan di runner"
      exit 1
    fi
    if ! sudo -n "$SYSTEMCTL" --version >/dev/null 2>&1; then
      echo "ERROR: runner tidak bisa sudo tanpa password untuk systemctl."
      echo "FIX di vm-docker: /etc/sudoers.d/gitlab-runner-threebody harus allow /usr/bin/systemctl NOPASSWD."
      exit 1
    fi
    SUDO="sudo -n"

    echo "==> [staging] login Harbor untuk runner..."
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [staging] salin docker auth ke user cikal (agar systemd pull setelah reboot)..."
    test -s "$HOME/.docker/config.json" || { echo "ERROR: $HOME/.docker/config.json tidak ada/kosong"; exit 1; }
    $SUDO install -d -m 700 -o cikal -g cikal /home/cikal/.docker
    $SUDO install -m 600 -o cikal -g cikal "$HOME/.docker/config.json" /home/cikal/.docker/config.json

    echo "==> [staging] sync deploy files ke /opt/threebody/ (sesuai rancangan systemd)..."
    $SUDO install -d -m 755 /opt/threebody/edge /opt/threebody/staging
    $SUDO rsync -a --delete deploy/edge/ /opt/threebody/edge/
    $SUDO rsync -a --delete deploy/staging/ /opt/threebody/staging/

    echo "==> [staging] pastikan symlink docker-compose.yml..."
    $SUDO ln -sf docker-compose.edge.yml /opt/threebody/edge/docker-compose.yml
    $SUDO ln -sf docker-compose.staging.yml /opt/threebody/staging/docker-compose.yml

    echo "==> [staging] tulis TLS cert untuk edge..."
    $SUDO install -d -m 755 /opt/threebody/edge/certs
    $SUDO install -m 600 "$EDGE_TLS_CRT" /opt/threebody/edge/certs/tls.crt
    $SUDO install -m 600 "$EDGE_TLS_KEY" /opt/threebody/edge/certs/tls.key

    echo "==> [staging] tulis .env staging (dibaca systemd)..."
    $SUDO install -o cikal -g docker -m 640 /dev/null /opt/threebody/staging/.env
    {
      echo "REGISTRY=$REGISTRY"
      echo "TAG=$TAG"
      echo ""
      echo "MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD"
      echo "MYSQL_DATABASE=$MYSQL_DATABASE"
      echo "MYSQL_USER=$MYSQL_USER"
      echo "MYSQL_PASSWORD=$MYSQL_PASSWORD"
      echo "LARAVEL_APP_KEY=$LARAVEL_APP_KEY"
    } | $SUDO tee /opt/threebody/staging/.env >/dev/null

    echo "==> [staging] restart service via systemd..."
    $SUDO systemctl daemon-reload
    $SUDO systemctl restart threebody-edge threebody-staging
    $SUDO systemctl status threebody-edge threebody-staging --no-pager -l || true

    echo "==> [staging] healthcheck staging (via edge)..."
    for i in $(seq 1 30); do
      if curl -kfsS --resolve staging.local:443:127.0.0.1 https://staging.local/ >/dev/null 2>&1; then
        echo "OK: staging sehat"
        exit 0
      fi
      sleep 2
    done

    echo "ERROR: staging healthcheck gagal"
    docker ps || true
    docker logs --tail 200 edge-nginx || true
    exit 1

# 4) Prod: deploy ke Kubernetes (ini yang sebelumnya kosong)
deploy_prod:
  stage: deploy_prod
  needs: ["deploy_staging"]
  script: |
    echo "==> [prod] validasi variables..."
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD (GitLab Variables Type: File)}"

    echo "==> [prod] siapkan kubectl..."
    if ! command -v kubectl >/dev/null 2>&1; then
      curl -fsSL -o /tmp/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x /tmp/kubectl
      sudo -n install -m 0755 /tmp/kubectl /usr/local/bin/kubectl
    fi
    kubectl version --client=true

    echo "==> [prod] set kubeconfig..."
    export KUBECONFIG="$KUBECONFIG_PROD"

    echo "==> [prod] pastikan namespace ada..."
    kubectl get ns "$K8S_NS_PROD" >/dev/null 2>&1 || kubectl create ns "$K8S_NS_PROD"

    echo "==> [prod] buat/update imagePullSecret untuk Harbor..."
    kubectl -n "$K8S_NS_PROD" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] attach imagePullSecret ke default ServiceAccount (biar pod bisa pull)..."
    kubectl -n "$K8S_NS_PROD" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null || true

    echo "==> [prod] apply manifests..."
    # TODO: sesuaikan path manifest prod kamu
    kubectl -n "$K8S_NS_PROD" apply -f deploy/k8s/prod/

    echo "==> [prod] update image tag (memicu pull di vm-worker)..."
    # TODO: sesuaikan nama Deployment + nama container
    kubectl -n "$K8S_NS_PROD" set image deployment/threebody-frontend frontend="$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/threebody-go go="$REGISTRY/go:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/threebody-laravel laravel="$REGISTRY/laravel:$TAG"

    echo "==> [prod] tunggu rollout..."
    kubectl -n "$K8S_NS_PROD" rollout status deployment/threebody-frontend --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/threebody-go --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/threebody-laravel --timeout="$K8S_ROLLOUT_TIMEOUT"
```

***

## Yang wajib kamu siapin di GitLab CI/CD Variables

Minimal untuk prod Kubernetes jalan:

1. **`KUBECONFIG_PROD`** → **Type: File**\
   Isinya kubeconfig yang bisa akses API server di **vm-k8s** dari **vm-docker**.
2. **`HARBOR_USERNAME`**, **`HARBOR_PASSWORD`**\
   Robot account Harbor yang punya akses pull/push ke project `threebody`.

Dan staging variables kamu yang sudah ada (TLS, MySQL, Laravel key) tetap.

***

## Hal penting biar “pull dari harbor.local” di vm-worker tidak gagal

Walaupun pipeline sudah benar, prod bisa gagal kalau **node k8s (vm-worker)** tidak bisa akses `harbor.local`. Pastikan:

* DNS/hosts di vm-worker bisa resolve `harbor.local`
* kalau Harbor pakai TLS self-signed, node/container runtime trust cert-nya (atau registry dibuat insecure sesuai setup kamu)

Gejala kalau ini bermasalah: pod `ImagePullBackOff` / `ErrImagePull`.

***

### Poin yang perlu kamu edit (2 titik saja)

Di `deploy_prod`, kamu perlu menyesuaikan:

1. Path manifest prod: `deploy/k8s/prod/` (kalau beda)
2. Nama Deployment dan container:
   * `deployment/threebody-frontend` + container `frontend`
   * `deployment/threebody-go` + container `go`
   * `deployment/threebody-laravel` + container `laravel`

Kalau kamu paste struktur file `deploy/k8s/prod/` (list nama yaml) **atau** output `kubectl -n threebody-prod get deploy -o name`, aku bisa langsung sesuaikan nama-namanya biar _langsung match_ tanpa coba-coba.

**Anda bilang:**

aku punya runbook dibawah ini, apakah dari sini ada pipeline yang mesti kamu rombak lagi? karena aku mau stage build → push → prod/k8s, tolong bantu cek dan perhatikan secara detail ya serta bantu arahakan karena aku ini masih pemula. 0. DevOps V3 ​https://harbor.local https://staging.local https://prod.local​ https://harbor.local https://staging.local https://prod.local Di bawah ini runbook: K8s pakai kubeadm (bukan k3s) Tanpa Ingress & tanpa MetalLB Akses aplikasi lewat Edge Nginx (jalan di vm-docker) + NodePort (untuk prod K8s) Harbor wajib untuk registry + image multistage Tag image = CI\_COMMIT\_SHORT\_SHA Ada push dari GitHub → GitLab (step-by-step) Ada Logging terpusat (Loki + Grafana) + promtail Catatan penting dari error kamu sebelumnya: Harbor error Please specify hostname karena harbor.yml masih hostname: reg.mydomain.com → harus diganti harbor.local dan tidak boleh kosong. Harbor memang mewajibkan hostname di-update. (goharbor.io) scp kamu timeout karena UFW di vm-harbor belum buka SSH (22). Kamu cuma allow 80/443 → SSH ketutup → scp timeout. vm-docker kamu belum install docker → docker: command not found, docker.service not found. vm-k8s belum install containerd → /etc/containerd/config.toml memang belum ada. 0) Target Arsitektur & Nama Host Host-only network: 192.168.56.0/24 VM IP Hostname OS FQDN/akses vm-docker 192.168.56.42 vm-docker staging.local, prod.local (Edge Nginx) vm-harbor 192.168.56.43 vm-harbor harbor.local (Harbor UI/Registry) vm-k8s (control-plane) 192.168.56.44 vm-k8s - vm-worker 192.168.56.45 vm-worker - Flow CI/CD: Git push → GitLab Pipeline → build & push image ke Harbor → deploy staging Docker Compose → healthcheck → deploy prod K8s NodePort → healthcheck → logs ke Loki+Grafana. 1) “Common Step” untuk SEMUA VM (dari VM kosong) Lakukan di vm-docker, vm-harbor, vm-k8s, vm-worker. 1.1 Set hostname OS Jalankan sesuai VM: vm-harbor vm-docker vm-k8s vm-worker Cek: 1.2 Isi /etc/hosts (WAJIB biar harbor.local, staging.local, prod.local resolve) Di setiap VM, edit: Isi (tambahkan) ini: Cek: Di laptop host kamu juga (Windows/Mac/Linux) tambahkan hosts yang sama supaya browser bisa buka Harbor dan app. 1.3 Update paket & tools dasar Cek SSH aktif: 1.4 UFW (firewall) – jangan putus SSH! Install & allow SSH dulu: Ini yang bikin scp kamu timeout sebelumnya: di vm-harbor belum allow OpenSSH. 2) Install Harbor di vm-harbor (FULL + rapi) Harbor wajib update hostname di harbor.yml. ( goharbor.io ) 2.1 Install Docker Engine + Compose (di vm-harbor) 2.2 Siapkan folder & download Harbor offline installer Pilih versi (contoh kamu pakai v2.14.1): 2.3 Buat TLS (CA + cert harbor.local) 2.3.1 Buat CA 2.3.2 Buat key + CSR untuk harbor.local 2.3.3 Buat SAN config lalu sign Cek: 2.4 Konfigurasi harbor.yml (INI FIX ERROR “Please specify hostname”) Masuk folder: Edit: Minimal yang kamu ubah: hostname wajib benar dan tidak kosong. Harbor minta hostname berupa IP/FQDN yang bisa diakses klien. ( goharbor.io ) Buat data dir: 2.5 Install Harbor JANGAN docker compose down kalau docker-compose.yml belum ada. Jalankan: Cek container: Test akses dari vm-harbor: Dari laptop, buka: https://harbor.local Login: admin / Harbor12345 2.6 Buat project di Harbor Di UI Harbor: Projects → New Project Nama: threebody Visibility: bebas (Private juga OK) 3) Trust CA Harbor di VM lain (biar docker login dan pull image aman) Dokumentasi Docker untuk trust registry dengan custom CA pakai /etc/docker/certs.d/\<host>/ca.crt. ( Docker Documentation ) 3.1 Copy CA dari vm-harbor (PAKAI IP biar gak ngadat) Di vm-docker: Kalau masih timeout → pastikan di vm-harbor: 3.2 Install CA ke Docker (vm-docker) Di vm-docker: Fixed Install CA Harbor ke Docker + OS trust store (HTTPS tetap proper) Jalankan di vm-docker (runner host): Test manual (di vm-docker) Kalau login/pull aman, pipeline docker push harusnya beres. Catatan: Kalau Harbor kamu pakai self-signed cert, isi /tmp/harbor-ca.crt di atas biasanya cukup (server cert = CA). Kalau Harbor pakai CA internal + intermediate, langkah di atas juga sering works karena kita taruh full chain. 1) Test login (pakai interaktif biar gak tergantung env) Di vm-docker jalankan: Masukkan user/pass Harbor (mis. admin + password kamu, atau robot account). Kalau kamu memang mau pakai env: Bonus penting (biar nanti Kubernetes gak ImagePullBackOff) Kalau cluster Kubernetes kamu pakai containerd (umum di kubeadm), tiap node K8s juga harus trust CA Harbor. Di setiap node K8s: (Atau copy /tmp/harbor-ca.crt dari vm-docker ke node-node K8s, lalu taruh sebagai ca.crt di path itu.) 4) Push project dari GitHub → GitLab (step-by-step) GitLab mendukung “create project dengan git push” atau lewat UI. ( Dokumentasi GitLab ) 4.1 Siapkan SSH key untuk GitLab (di vm-docker) 1) Buat folder & file config 2) Isi file \~/.ssh/config Tempel ini: 3) Set permission (wajib, biar SSH nggak nolak) Copy output pubkey → GitLab: Profile → Preferences → SSH Keys → Paste → Add key 4. Preload known\_hosts (biar setelah reboot tidak prompt “authenticity”) Test: 4.2 Clone repo GitHub kamu Di vm-docker: 4.3 Buat project baru di GitLab (UI) Di GitLab: Create new → New project/repository Create blank project Nama misal: three-body-problem-main Jangan centang “Initialize with README” (biar tidak konflik history) Copy URL SSH repo GitLab-mu, contohnya: 4.4 Add remote GitLab dan push Di folder repo (vm-docker): Kalau GitLab kamu self-managed, ganti gitlab.com sesuai domain instance. 5) Rapihin repo: file yang wajib kamu tambahkan (Dockerfile multistage + deploy + CI) Di repo GitLab kamu (di vm-docker), buat struktur ini: 5.1 Tag image = CI\_COMMIT\_SHORT\_SHA CI\_COMMIT\_SHORT\_SHA adalah predefined variable GitLab. Kita akan pakai ini di .gitlab-ci.yml. 5.2 Dockerfile multistage (contoh template) Karena aku tidak bisa build repo kamu di sini, aku kasih template “aman” yang paling umum. Kalau nama folder/entrypoint berbeda, tinggal sesuaikan. (A) go/Dockerfile (B) laravel/Dockerfile (composer builder → apache runtime) (Opsional tapi disarankan) Full replace: laravel/.dockerignore Biar build context kecil & cache lebih enak: (C) frontend/Dockerfile (node build → nginx serve) 5.3 Perbaiki frontend agar tidak hardcode localhost Jalankan di vm-docker: Buat file: frontend/.env.development frontend/.env.production Lalu di code frontend (biasanya frontend/src/App.js atau sejenis): ganti URL hardcode menjadi: dan request jadi: ${GO\_BASE}/api/products ${LARAVEL\_BASE}/api/products Jika Ingin full replace untuk frontend/src/App.js jalankan perintah dibawah ini Test build lokal dulu (biar yakin sebelum push) Masih di vm-docker: Kalau ini sukses, pipeline juga bakal sukses di step frontend. 6) Edge Nginx (jalan di vm-docker) + HTTPS + Rate limiting Dokumentasi Nginx rate limiting ada limit\_req\_zone dan limit\_req. 6.1 Buat cert untuk Edge (staging.local + prod.local) Agar simpel: kita sign pakai CA yang sama dari vm-harbor, lalu copy ke vm-docker. 6.1.1 Generate cert EDGE di vm-harbor Di vm-harbor: Copy ke vm-docker: 6.1.2 Taruh cert di repo (vm-docker) Di vm-docker (repo): 6.2 Buat config Nginx edge File: deploy/edge/nginx/conf.d/edge.conf 6.3 Docker Compose untuk Edge File: deploy/edge/docker-compose.edge.yml 7) Staging Docker Compose (images dari Harbor) File: deploy/staging/docker-compose.staging.yml 8) Install Kubernetes (kubeadm) di vm-k8s + vm-worker Ikuti konsep dari dokumentasi “Installing kubeadm”. Pakai cgroup driver systemd sesuai rekomendasi kubeadm. 8.1 Di vm-k8s dan vm-worker: disable swap 8.2 Install containerd + config default Set systemd cgroups: 8.3 Install kubeadm/kubelet/kubectl (ikuti repo resmi) Ikuti langkah dari doc Installing kubeadm. Di vm-k8s & vm-worker (Ubuntu): Setelah install: 8.4 Init cluster (di vm-k8s) Aktifkan IP\_Forward di vm-k8s dan vm-worker Pilih Pod CIDR (contoh Calico): Konfigurasi kubectl untuk user: Install CNI (contoh Calico): Tunggu node ready: 8.5 Join worker (di vm-worker) Dari output kubeadm init, ambil command join. Misal: Cek node: Jika terdapat error pada saat join maka jalankan perintah dibawah ini di vm-k8s 9) K8s nodes trust Harbor CA (containerd) Containerd pakai config\_path="/etc/containerd/certs.d" + hosts.toml. Lakukan di vm-k8s dan vm-worker: 9.1 Copy CA ke node Dari vm-harbor, copy ke masing-masing node (pakai IP): 9.2 Buat certs.d + hosts.toml 9.3 Aktifkan config\_path di containerd config Edit: Cari blok: Pastikan ada: Restart: 10) Manifest Kubernetes (NodePort) untuk PROD Buat namespace threebody-prod dan service NodePort: frontend: 30080 go: 30081 laravel: 30082 MySQL untuk prod: simplest pakai StatefulSet + hostPath (karena bare kubeadm biasanya belum ada storageclass). Kalau kamu nanti mau rapi, baru pasang provisioner. (Aku bisa tuliskan YAML lengkapnya juga, tapi karena jawaban ini sudah panjang, minimal kamu butuh: namespace.yaml, mysql.yaml, go.yaml, laravel.yaml, frontend.yaml + imagePullSecret untuk Harbor.) 10.1 Namespace deploy/k8s/base/00-namespace.yaml 10.2 MySQL (hostPath, pin ke worker) Jalankan ini sekali di vm-worker supaya folder ada: deploy/k8s/base/10-mysql.yaml Catatan: HostPath ini “lab-friendly” (bukan best practice cloud), tapi cukup untuk tugas. 10.3 GO (NodePort 30081) deploy/k8s/base/20-go.yaml Kalau service Go kamu tidak punya endpoint /health, ganti probe ke path yang ada (atau hapus probe dulu biar gampang). 10.4 Laravel (NodePort 30082) deploy/k8s/base/30-laravel.yaml 10.5 Frontend (NodePort 30080) deploy/k8s/base/40-frontend.yaml 11) Install GitLab Runner di vm-docker + register Gunakan panduan install GitLab Runner untuk Linux. 11.1 Install runner (vm-docker) 11.2 Register runner (shell executor) Di GitLab project: Settings → CI/CD → Runners → “Register a runner” Ambil registration token (atau runner token sesuai UI GitLab kamu) Di vm-docker: Isi: GitLab URL: https://gitlab.com (atau domain GitLab kamu) Token: (paste) Executor: shell Tags: deploy Description: vm-docker-runner Cek: Sebelum push ke gitlab pastikan permission Docker dan lain-lainnya terlebih dahulu dan ikuti perintah dibawah ini. 11.3 Fix permission Docker untuk gitlab-runner (vm-docker) Jalankan di vm-docker: 11.4 Verifikasi akses (WAJIB sebelum rerun pipeline) Masih di vm-docker, jalankan: Pastikan output ada docker di daftar groups. Lalu test: Kalau docker ps masih permission denied, kirim output ls -l /var/run/docker.sock dan id gitlab-runner. 12) .gitlab-ci.yml (tag = CI\_COMMIT\_SHORT\_SHA) CI\_COMMIT\_SHORT\_SHA predefined variable GitLab. Buat/replace .gitlab-ci.yml di root repo: 13) GitLab CI Variables (Secret Manager = GitLab Variables) Di GitLab Project → Settings → CI/CD → Variables, set: HARBOR\_USERNAME = admin HARBOR\_PASSWORD = Harbor12345 (atau yang kamu ganti) MYSQL\_ROOT\_PASSWORD = ... MYSQL\_DATABASE = ... MYSQL\_USER = ... MYSQL\_PASSWORD = ... LARAVEL\_APP\_KEY = base64:... (generate) KUBECONFIG\_B64 = base64 dari \~/.kube/config (di vm-k8s) Generate KUBECONFIG\_B64 (di vm-k8s): Copy output ke variable KUBECONFIG\_B64. HTTPS butuh 2 file: Certificate (CRT / fullchain) → ibarat “KTP server” Private Key (KEY) → ibarat “kunci rahasia server” Pipeline kamu nyari 2 variable ini: EDGE\_TLS\_CRT (isi: file cert) EDGE\_TLS\_KEY (isi: file key) Kalau tidak ada → job gagal (yang kamu lihat sekarang). B. Cara paling mudah (pemula): pakai Self-Signed dulu (buat latihan) Kalau ini untuk lab/staging, kamu bisa pakai self-signed supaya pipeline lanjut. 1) Buat cert & key di VM runner (vm-docker) Jalankan: Hasilnya ada: \~/tls/tls.crt \~/tls/tls.key Cek: Cert kamu sekarang CN=harbor.local → sebaiknya untuk EDGE pakai staging/prod Kamu bikin cert dengan: /CN=harbor.local Padahal yang kamu pakai untuk HTTPS edge adalah: https://staging.local https://prod.local Kalau cert hanya harbor.local, browser akan “name mismatch”. Memang curl -k masih lolos, tapi untuk beneran rapi, bikin cert dengan SAN untuk staging.local dan prod.local. ✅ Cara bikin cert EDGE yang benar (pakai SAN) Jalankan di VM: Nanti yang dipakai untuk GitLab variable adalah isi: \~/tls/tls.crt \~/tls/tls.key 3) Cara set EDGE\_TLS\_CRT dan EDGE\_TLS\_KEY di GitLab (step-by-step) Buka GitLab project kamu Masuk Settings → CI/CD → Variables Klik Add variable Tambah Variable 1 (CRT) Key: EDGE\_TLS\_CRT Type: File ✅ Value: paste isi dari \~/tls/tls.crt (termasuk BEGIN/END) Environment scope: \* (All) Protected: OFF dulu (biar gampang untuk pemula) Masked: biarkan default (biasanya file variable nggak dimasking seperti string) Klik Save Tambah Variable 2 (KEY) Key: EDGE\_TLS\_KEY Type: File ✅ Value: paste isi dari \~/tls/tls.key Environment scope: \* (All) Protected: OFF dulu Klik Save ✅ Kalau ini beres, maka di job GitLab: $EDGE\_TLS\_CRT itu bukan isi cert, tapi path file sementara yang dibuat GitLab makanya cp "$EDGE\_TLS\_CRT" ... sudah benar Jika dirasa setup vm-docker sudah aman lalu lanjut push project ke gitlab dan tunggu sampai proses running pipeline berhasil Kalau kamu sudah commit sebelumnya tapi mau trigger ulang: Cara melihat “hasil akhirnya” (URL) + checklist Cek pipeline di GitLab GitLab → Build → Pipelines Pastikan: Runner status “online” Job build\_images sukses → images muncul di Harbor project threebody Job deploy\_staging sukses Job deploy\_prod sukses Cek di vm-docker (edge + staging container) 14.3 Cek di K8s (prod) Di vm-k8s: URL yang harus bisa dibuka dari laptop https://harbor.local https://staging.local https://prod.local Kalau browser warning TLS (karena CA lab), itu normal. Solusi rapi: import ca.crt ke trust store OS laptop. 14) Observability: Loki + Grafana (terpusat) + Promtail Cara paling gampang sesuai rancangan kamu: Loki + Grafana jalan di vm-docker (docker compose) Promtail: container di vm-docker (ambil log docker containers) DaemonSet di k8s (ambil log pods) → push ke Loki vm-docker (Aku bisa tuliskan file compose + promtail config + DaemonSet YAML juga kalau kamu mau aku “full tulis semua file observability”-nya sekalian.) 14.1 Compose Loki+Grafana+Promtail (vm-docker) deploy/observability/docker-compose.observability.yml deploy/observability/loki-config.yml deploy/observability/promtail-docker.yml Jalankan (vm-docker): Buka Grafana: http://192.168.56.42:3000 (atau lewat hosts kalau kamu mau) Login admin/admin Add datasource Loki: URL http://loki:3100 Kalau kamu butuh promtail di K8s juga (DaemonSet) biar log pods masuk Loki, bilang aja—aku bisa tuliskan YAML DaemonSet promtail yang push ke http://192.168.56.42:3100. 15) README.md (yang diminta tugas) Di README tulis minimal: Cara run staging (docker compose) Cara run prod (kubectl apply) Arsitektur (diagram teks + penjelasan) Screenshot: Harbor project, pipeline sukses, staging.local, prod.local, Grafana logs “Checklist cepat” biar kamu yakin end-to-end sudah bener https://harbor.local bisa dibuka vm-docker docker login harbor.local sukses push ke GitLab sukses, pipeline jalan https://staging.local tampil frontend https://prod.local tampil frontend Grafana bisa akses Loki dan ada log Kalau kamu mau, aku bisa lanjutkan dengan paket file YAML K8s lengkap (frontend/go/laravel/mysql + secrets + NodePort) dan paket observability lengkap (docker-compose Loki+Grafana + promtail docker + promtail DaemonSet k8s) dalam format yang tinggal kamu copy-paste ke repo. Previous 0. DevOps V2 Next Page Cikal Farid Last modified 4h ago

**ChatGPT bilang:**

Selesai berpikir selama 37s

Aku sudah baca runbook kamu (detailnya banyak dan cukup jelas). Dari situ, **yang perlu “dirombak” di pipeline itu utamanya 2 hal**:

1. **Alur stage**: runbook awal kamu ada staging dulu (Docker Compose) baru prod (K8s). Tapi sekarang kamu mau **build → push → prod/k8s**. Jadi **job `deploy_staging` harus dihapus atau dijadikan manual/opsional** (biar nggak mengganggu target kamu).
2. **`deploy_prod` harus benar-benar deploy ke Kubernetes** (apply manifest + set image + rollout + healthcheck). Di pipeline kamu sekarang `deploy_prod` masih placeholder.

Yang lain di runbook (Harbor hostname, UFW SSH, install docker/containerd, trust CA, /etc/hosts) itu **bukan urusan pipeline**, tapi **prasyarat environment**. Pipeline akan tetap gagal kalau prasyarat ini belum beres (contoh klasik: pod `ImagePullBackOff` kalau node belum trust CA Harbor).

***

### Konsep penting (biar kamu paham “pull” di K8s)

Di Kubernetes, kamu **nggak bikin stage “pull image” manual** seperti docker biasa.\
Yang terjadi:

* Pipeline **push** image ke Harbor dengan tag baru (`CI_COMMIT_SHORT_SHA`)
* Pipeline **deploy** ke K8s dengan cara **mengubah image tag** di Deployment
* Begitu tag berubah → **kubelet di vm-worker otomatis pull image baru** dari Harbor

Jadi stage yang benar: **build → push → deploy (yang memicu pull otomatis)**.

***

### Pipeline yang sesuai runbook + target kamu (build → push → prod/k8s)

Ini versi `.gitlab-ci.yml` yang fokus ke **prod K8s** saja. (Staging aku jadikan opsional di bagian bawah kalau kamu mau.)

> Aku ikuti runbook kamu soal variable **`KUBECONFIG_B64`** (base64 dari kubeconfig). Ini lebih “pemula friendly” karena tidak perlu File variable dulu.

```yaml
stages:
  - build
  - push
  - deploy_prod

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"

  HARBOR_HOST: "harbor.local"
  HARBOR_PROJECT: "threebody"
  REGISTRY: "$HARBOR_HOST/$HARBOR_PROJECT"

  # k8s
  K8S_NS_PROD: "threebody-prod"
  KUBECTL_VERSION: "v1.30.14"
  K8S_ROLLOUT_TIMEOUT: "10m"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

  # Nama deployment & container (biar gak hardcode di script)
  DEPLOY_FRONTEND: "threebody-frontend"
  DEPLOY_GO: "threebody-go"
  DEPLOY_LARAVEL: "threebody-laravel"
  CTR_FRONTEND: "frontend"
  CTR_GO: "go"
  CTR_LARAVEL: "laravel"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail

build_check:
  stage: build
  script: |
    echo "==> [build] build-check (no push)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel

push_images:
  stage: push
  needs: ["build_check"]
  script: |
    echo "==> [push] login Harbor..."
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [push] build+push images (tag=$TAG)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker push "$REGISTRY/frontend:$TAG"

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker push "$REGISTRY/go:$TAG"

    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    docker push "$REGISTRY/laravel:$TAG"

deploy_prod:
  stage: deploy_prod
  needs: ["push_images"]
  script: |
    echo "==> [prod] validasi variables..."
    : "${KUBECONFIG_B64:?Missing KUBECONFIG_B64}"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"

    echo "==> [prod] siapkan kubectl..."
    if ! command -v kubectl >/dev/null 2>&1; then
      curl -fsSL -o /tmp/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x /tmp/kubectl
      sudo -n install -m 0755 /tmp/kubectl /usr/local/bin/kubectl
    fi
    kubectl version --client=true

    echo "==> [prod] buat kubeconfig dari base64..."
    echo "$KUBECONFIG_B64" | base64 -d > ./kubeconfig
    export KUBECONFIG="$PWD/kubeconfig"

    echo "==> [prod] cek koneksi cluster..."
    kubectl get nodes -o wide

    echo "==> [prod] pastikan namespace ada..."
    kubectl get ns "$K8S_NS_PROD" >/dev/null 2>&1 || kubectl create ns "$K8S_NS_PROD"

    echo "==> [prod] buat/update imagePullSecret untuk Harbor..."
    kubectl -n "$K8S_NS_PROD" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] pasang imagePullSecret ke default ServiceAccount..."
    kubectl -n "$K8S_NS_PROD" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null || true

    echo "==> [prod] apply manifest prod (NodePort) ..."
    # Sesuaikan folder manifest kamu: runbook kamu menyebut deploy/k8s/base/
    kubectl -n "$K8S_NS_PROD" apply -f deploy/k8s/base/

    echo "==> [prod] update image tag (ini yang bikin node auto-pull)..."
    kubectl -n "$K8S_NS_PROD" set image "deployment/$DEPLOY_FRONTEND" "$CTR_FRONTEND=$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS_PROD" set image "deployment/$DEPLOY_GO" "$CTR_GO=$REGISTRY/go:$TAG"
    kubectl -n "$K8S_NS_PROD" set image "deployment/$DEPLOY_LARAVEL" "$CTR_LARAVEL=$REGISTRY/laravel:$TAG"

    echo "==> [prod] tunggu rollout..."
    kubectl -n "$K8S_NS_PROD" rollout status "deployment/$DEPLOY_FRONTEND" --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status "deployment/$DEPLOY_GO" --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status "deployment/$DEPLOY_LARAVEL" --timeout="$K8S_ROLLOUT_TIMEOUT"

    echo "==> [prod] cek service NodePort..."
    kubectl -n "$K8S_NS_PROD" get svc -o wide

    echo "==> [prod] healthcheck via Edge Nginx (prod.local)..."
    for i in $(seq 1 30); do
      if curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/ >/dev/null 2>&1; then
        echo "OK: prod sehat"
        exit 0
      fi
      sleep 2
    done

    echo "ERROR: prod healthcheck gagal"
    kubectl -n "$K8S_NS_PROD" get pods -o wide || true
    kubectl -n "$K8S_NS_PROD" describe pods || true
    exit 1
```

***

### Kenapa pipeline di atas “nyambung” dengan runbook kamu?

* **Tag image** pakai `CI_COMMIT_SHORT_SHA` ✅ (runbook wajib)
* **Push ke Harbor** ✅
* **Tanpa ingress/metallb** → kita rely ke manifest NodePort di `deploy/k8s/base/` ✅
* **Akses aplikasi lewat Edge Nginx di vm-docker + NodePort** ✅
* **Pull image terjadi otomatis** setelah `kubectl set image` ✅
* **ImagePullSecret** untuk Harbor dibuat dari pipeline (biar pemula nggak setup manual) ✅

***

### Yang harus kamu set di GitLab Variables (paling penting)

Di project GitLab: Settings → CI/CD → Variables:

1. `HARBOR_USERNAME` (lebih bagus robot account)
2. `HARBOR_PASSWORD`
3.  `KUBECONFIG_B64`\
    Ini dari `vm-k8s`:

    ```bash
    base64 -w 0 ~/.kube/config
    ```

    Copy hasilnya ke variable.

> Kalau kamu pakai base64 dari kubeconfig yang punya path/sertifikat relatif, kadang decode-nya tetap OK, tapi pastikan kubeconfig memang bisa dipakai dari vm-docker (akses ke API server vm-k8s tidak terblok firewall).

***

### 2 hal yang sering bikin gagal walaupun pipeline sudah benar

#### A) Pod `ImagePullBackOff` (paling sering)

Penyebab: node K8s (**vm-worker**) belum trust CA `harbor.local` atau tidak resolve `harbor.local`.

Cek cepat di vm-k8s:

```bash
kubectl -n threebody-prod get pods
kubectl -n threebody-prod describe pod <nama-pod>
```

Kalau ada `x509: certificate signed by unknown authority` atau `no such host`, itu berarti **prasyarat runbook bagian trust CA + /etc/hosts** belum beres di node.

#### B) Edge Nginx belum proxy ke NodePort prod

Runbook kamu bilang prod akses lewat edge nginx + nodeport. Pastikan `edge.conf` prod memang proxy ke:

* frontend NodePort 30080
* go 30081
* laravel 30082\
  Dan IP tujuan biasanya `192.168.56.45` (vm-worker) atau cluster service internal (tergantung desainmu).

***

### Kalau kamu masih ingin “staging” tapi tidak mengganggu target

Cara aman: bikin `deploy_staging` jadi **manual** (opsional), jadi stage kamu tetap 3 utama.

Contoh kecil:

```yaml
deploy_staging:
  stage: deploy_prod
  when: manual
  allow_failure: true
  script: echo "staging opsional"
```

(Tapi kalau target kamu murni build→push→prod, mending staging jangan dimasukkan dulu.)

***

### Checklist super pemula (biar kamu yakin end-to-end)

1. Pipeline sukses sampai `push_images` → di Harbor project `threebody` muncul 3 image dengan tag SHA
2. `deploy_prod`:
   * `kubectl get nodes` sukses
   * `kubectl apply` sukses
   * `rollout status` sukses
3. Browser/laptop bisa buka `https://prod.local` (walau warning TLS masih ok untuk lab)

***

Kalau kamu mau aku “kunci” supaya **nggak ada mismatch nama Deployment/container**, kirim salah satu ini (pilih yang paling gampang):

* Output dari `kubectl -n threebody-prod get deploy -o name` (di vm-k8s), **atau**
* Nama file YAML di folder `deploy/k8s/base/` + potongan `metadata.name` dan `containers.name` dari tiap Deployment.

Nanti aku sesuaikan variabel `DEPLOY_*` dan `CTR_*` biar pasti match dan pipeline langsung jalan tanpa tebak-tebakan.

**Anda bilang:**

Tolong bantu cek ya dari awal percakapan kita serta cek runbook ku juga ya untuk penyesuaian, pokoknya pipeline harus build di vm docker, push ke vm harbor, dan pull serta deploy di kubernetes nya. Dan berikan arahan secara detail dan lengkap ya, karena aku ini masih pemula cikal@vm-k8s:\~$ kubectl -n threebody-prod get pods NAME READY STATUS RESTARTS AGE frontend-5797b47c6b-tchq5 1/1 Running 2 (70m ago) 22h go-588dd459bb-d45dj 1/1 Running 5 (70m ago) 22h laravel-5b9f78c886-z2shp 1/1 Running 2 (70m ago) 22h mysql-0 1/1 Running 2 (70m ago) 22h cikal@vm-k8s:\~$ kubectl -n threebody-prod describe pod frontend-5797b47c6b-tchq5 Name: frontend-5797b47c6b-tchq5 Namespace: threebody-prod Priority: 0 Service Account: default Node: vm-worker/192.168.56.45 Start Time: Sun, 21 Dec 2025 10:50:35 +0000 Labels: app=frontend pod-template-hash=5797b47c6b Annotations: cni.projectcalico.org/containerID: 9bb2c0ce8881dc996436927fc4f94cdadb369acc9a26a384852433392b86508a cni.projectcalico.org/podIP: 192.168.171.235/32 cni.projectcalico.org/podIPs: 192.168.171.235/32 Status: Running IP: 192.168.171.235 IPs: IP: 192.168.171.235 Controlled By: ReplicaSet/frontend-5797b47c6b Containers: frontend: Container ID: containerd://5cc2d258d55f8bfe6675bf4a7624b72ac18f94939321d0c5f6990ad1a80fd214 Image: harbor.local/threebody/frontend:bd17c4d5 Image ID: harbor.local/threebody/frontend@sha256:321dec4e5f88ff03fa158f8ac86db4c54cdf2f4a7aa6988ebf8fd2808cff225c Port: 80/TCP Host Port: 0/TCP State: Running Started: Mon, 22 Dec 2025 08:05:32 +0000 Last State: Terminated Reason: Unknown Exit Code: 255 Started: Sun, 21 Dec 2025 13:42:49 +0000 Finished: Mon, 22 Dec 2025 08:04:15 +0000 Ready: True Restart Count: 2 Environment: \<none> Mounts: /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gmz6j (ro) Conditions: Type Status PodReadyToStartContainers True Initialized True Ready True ContainersReady True PodScheduled True Volumes: kube-api-access-gmz6j: Type: Projected (a volume that contains injected data from multiple sources) TokenExpirationSeconds: 3607 ConfigMapName: kube-root-ca.crt ConfigMapOptional: \<nil> DownwardAPI: true QoS Class: BestEffort Node-Selectors: \<none> Tolerations: node.kubernetes.io/not-ready:NoExecute op=Exists for 300s node.kubernetes.io/unreachable:NoExecute op=Exists for 300s Events: Type Reason Age From Message ---- ------ ---- ---- ------- Warning NodeNotReady 57m (x2 over 57m) node-controller Node is not ready Warning NodeNotReady 37m node-controller Node is not ready Warning NodeNotReady 4m7s node-controller Node is not ready cikal@vm-k8s:\~$ kubectl -n threebody-prod describe pod go-588dd459bb-d45dj Name: go-588dd459bb-d45dj Namespace: threebody-prod Priority: 0 Service Account: default Node: vm-worker/192.168.56.45 Start Time: Sun, 21 Dec 2025 10:50:35 +0000 Labels: app=go pod-template-hash=588dd459bb Annotations: cni.projectcalico.org/containerID: 5904c3d15df60bbda74c744f54eb67cd32a88725923d0e1cf7273602571ffd7d cni.projectcalico.org/podIP: 192.168.171.234/32 cni.projectcalico.org/podIPs: 192.168.171.234/32 Status: Running IP: 192.168.171.234 IPs: IP: 192.168.171.234 Controlled By: ReplicaSet/go-588dd459bb Containers: go: Container ID: containerd://1bb339e008e5cb1049ffdec30b47235d802563b7bb9ee63c13e9f1312fe4cbd4 Image: harbor.local/threebody/go:bd17c4d5 Image ID: harbor.local/threebody/go@sha256:0bece7142716524b4fc2ebc25916b7ea81677bd243cafa91b1c15f3bbc1bf981 Port: 8080/TCP Host Port: 0/TCP State: Running Started: Mon, 22 Dec 2025 08:05:28 +0000 Last State: Terminated Reason: Unknown Exit Code: 255 Started: Sun, 21 Dec 2025 13:42:35 +0000 Finished: Mon, 22 Dec 2025 08:04:15 +0000 Ready: True Restart Count: 5 Liveness: http-get http://:8080/health delay=20s timeout=1s period=10s #success=1 #failure=3 Readiness: http-get http://:8080/health delay=5s timeout=1s period=5s #success=1 #failure=3 Environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_NAME: \<set to the key 'MYSQL\_DATABASE' in secret 'app-secrets'> Optional: false DB\_USER: \<set to the key 'MYSQL\_USER' in secret 'app-secrets'> Optional: false DB\_PASS: \<set to the key 'MYSQL\_PASSWORD' in secret 'app-secrets'> Optional: false Mounts: /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-z9bgh (ro) Conditions: Type Status PodReadyToStartContainers True Initialized True Ready True ContainersReady True PodScheduled True Volumes: kube-api-access-z9bgh: Type: Projected (a volume that contains injected data from multiple sources) TokenExpirationSeconds: 3607 ConfigMapName: kube-root-ca.crt ConfigMapOptional: \<nil> DownwardAPI: true QoS Class: BestEffort Node-Selectors: \<none> Tolerations: node.kubernetes.io/not-ready:NoExecute op=Exists for 300s node.kubernetes.io/unreachable:NoExecute op=Exists for 300s Events: Type Reason Age From Message ---- ------ ---- ---- ------- Warning Unhealthy 57m (x3 over 69m) kubelet Readiness probe failed: Get "http://192.168.171.234:8080/health": context deadline exceeded (Client.Timeout exceeded while awaiting headers) Warning NodeNotReady 57m (x2 over 57m) node-controller Node is not ready Warning Unhealthy 44m (x3 over 69m) kubelet Liveness probe failed: Get "http://192.168.171.234:8080/health": context deadline exceeded (Client.Timeout exceeded while awaiting headers) Warning NodeNotReady 38m node-controller Node is not ready Warning NodeNotReady 4m21s node-controller Node is not ready cikal@vm-k8s:\~$ kubectl -n threebody-prod describe pod laravel-5b9f78c886-z2shp Name: laravel-5b9f78c886-z2shp Namespace: threebody-prod Priority: 0 Service Account: default Node: vm-worker/192.168.56.45 Start Time: Sun, 21 Dec 2025 10:50:35 +0000 Labels: app=laravel pod-template-hash=5b9f78c886 Annotations: cni.projectcalico.org/containerID: f3392305a605140e6ab0022c1180e6525bcee10ee7ed789ef2d59a7cdf8bc947 cni.projectcalico.org/podIP: 192.168.171.232/32 cni.projectcalico.org/podIPs: 192.168.171.232/32 Status: Running IP: 192.168.171.232 IPs: IP: 192.168.171.232 Controlled By: ReplicaSet/laravel-5b9f78c886 Containers: laravel: Container ID: containerd://78f144fca3f3d2f277bc16214f509a6a65814b799c4e9361f8d88fa31dc1c755 Image: harbor.local/threebody/laravel:bd17c4d5 Image ID: harbor.local/threebody/laravel@sha256:746f47618362967129dab3ccd10ac4fab1ae2ca28a655f60c5c0408dd98fdf56 Port: 80/TCP Host Port: 0/TCP State: Running Started: Mon, 22 Dec 2025 08:05:29 +0000 Last State: Terminated Reason: Unknown Exit Code: 255 Started: Sun, 21 Dec 2025 13:42:47 +0000 Finished: Mon, 22 Dec 2025 08:04:15 +0000 Ready: True Restart Count: 2 Environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: \<set to the key 'MYSQL\_DATABASE' in secret 'app-secrets'> Optional: false DB\_USERNAME: \<set to the key 'MYSQL\_USER' in secret 'app-secrets'> Optional: false DB\_PASSWORD: \<set to the key 'MYSQL\_PASSWORD' in secret 'app-secrets'> Optional: false APP\_KEY: \<set to the key 'LARAVEL\_APP\_KEY' in secret 'app-secrets'> Optional: false APP\_ENV: production APP\_DEBUG: false Mounts: /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jzlww (ro) Conditions: Type Status PodReadyToStartContainers True Initialized True Ready True ContainersReady True PodScheduled True Volumes: kube-api-access-jzlww: Type: Projected (a volume that contains injected data from multiple sources) TokenExpirationSeconds: 3607 ConfigMapName: kube-root-ca.crt ConfigMapOptional: \<nil> DownwardAPI: true QoS Class: BestEffort Node-Selectors: \<none> Tolerations: node.kubernetes.io/not-ready:NoExecute op=Exists for 300s node.kubernetes.io/unreachable:NoExecute op=Exists for 300s Events: Type Reason Age From Message ---- ------ ---- ---- ------- Warning NodeNotReady 57m (x2 over 58m) node-controller Node is not ready Warning NodeNotReady 38m node-controller Node is not ready Warning NodeNotReady 4m35s node-controller Node is not ready cikal@vm-k8s:\~$ kubectl -n threebody-prod describe pod mysql-0 Name: mysql-0 Namespace: threebody-prod Priority: 0 Service Account: default Node: vm-worker/192.168.56.45 Start Time: Sun, 21 Dec 2025 10:50:35 +0000 Labels: app=mysql apps.kubernetes.io/pod-index=0 controller-revision-hash=mysql-564c45656b statefulset.kubernetes.io/pod-name=mysql-0 Annotations: cni.projectcalico.org/containerID: a26e1931659162558c2d86787aa16f7886f2b099ad8026dc1a75358be4885464 cni.projectcalico.org/podIP: 192.168.171.233/32 cni.projectcalico.org/podIPs: 192.168.171.233/32 Status: Running IP: 192.168.171.233 IPs: IP: 192.168.171.233 Controlled By: StatefulSet/mysql Init Containers: init-mysql-perms: Container ID: containerd://38c8a6b69e2b84bc5100c2c0f9826f6c874a86bc2de8340da15fd7f0eb04470f Image: busybox:1.36 Image ID: docker.io/library/busybox@sha256:6b219909078e3fc93b81f83cb438bd7a5457984a01a478c76fe9777a8c67c39e Port: \<none> Host Port: \<none> Command: sh -c mkdir -p /var/lib/mysql chown -R 999:999 /var/lib/mysql || true State: Terminated Reason: Completed Exit Code: 0 Started: Mon, 22 Dec 2025 08:05:19 +0000 Finished: Mon, 22 Dec 2025 08:05:19 +0000 Ready: True Restart Count: 2 Environment: \<none> Mounts: /var/lib/mysql from mysql-data (rw) /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4dxxc (ro) Containers: mysql: Container ID: containerd://c5235e1164be81186fda2da5a42727acbe48e933e8c88483db2553bba7a2b934 Image: mysql:8.0 Image ID: docker.io/library/mysql@sha256:0275a35e79c60caae68fac520602d9f6897feb9b0941a1471196b1a01760e581 Port: 3306/TCP Host Port: 0/TCP State: Running Started: Mon, 22 Dec 2025 08:05:29 +0000 Last State: Terminated Reason: Unknown Exit Code: 255 Started: Sun, 21 Dec 2025 13:42:35 +0000 Finished: Mon, 22 Dec 2025 08:04:15 +0000 Ready: True Restart Count: 2 Liveness: exec \[sh -c MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -h 127.0.0.1 -uroot --silent] delay=30s timeout=5s period=10s #success=1 #failure=10 Readiness: exec \[sh -c MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -h 127.0.0.1 -uroot --silent] delay=10s timeout=3s period=5s #success=1 #failure=30 Environment: MYSQL\_ROOT\_PASSWORD: \<set to the key 'MYSQL\_ROOT\_PASSWORD' in secret 'app-secrets'> Optional: false MYSQL\_DATABASE: \<set to the key 'MYSQL\_DATABASE' in secret 'app-secrets'> Optional: false MYSQL\_USER: \<set to the key 'MYSQL\_USER' in secret 'app-secrets'> Optional: false MYSQL\_PASSWORD: \<set to the key 'MYSQL\_PASSWORD' in secret 'app-secrets'> Optional: false Mounts: /var/lib/mysql from mysql-data (rw) /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4dxxc (ro) Conditions: Type Status PodReadyToStartContainers True Initialized True Ready True ContainersReady True PodScheduled True Volumes: mysql-data: Type: PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace) ClaimName: mysql-data ReadOnly: false kube-api-access-4dxxc: Type: Projected (a volume that contains injected data from multiple sources) TokenExpirationSeconds: 3607 ConfigMapName: kube-root-ca.crt ConfigMapOptional: \<nil> DownwardAPI: true QoS Class: BestEffort Node-Selectors: kubernetes.io/hostname=vm-worker Tolerations: node.kubernetes.io/not-ready:NoExecute op=Exists for 300s node.kubernetes.io/unreachable:NoExecute op=Exists for 300s Events: Type Reason Age From Message ---- ------ ---- ---- ------- Warning NodeNotReady 58m (x2 over 58m) node-controller Node is not ready Warning Unhealthy 56m (x2 over 58m) kubelet Readiness probe failed: command "sh -c MYSQL\_PWD=\\"$MYSQL\_ROOT\_PASSWORD\\" mysqladmin ping -h 127.0.0.1 -uroot --silent" timed out Warning NodeNotReady 38m node-controller Node is not ready Warning NodeNotReady 4m52s node-controller Node is not ready cikal@vm-k8s:\~$ cikal@vm-docker:\~/three-body-problem-main$ ll deploy/k8s/base/ total 28 drwxrwxr-x 2 cikal docker 4096 Dec 20 19:49 ./ drwxrwxr-x 4 cikal docker 4096 Dec 20 20:18 ../ -rw-rw-r-- 1 cikal docker 64 Dec 20 14:09 00-namespace.yaml -rw-rw-r-- 1 cikal docker 3246 Dec 20 18:35 10-mysql.yaml -rw-rw-r-- 1 cikal docker 1544 Dec 20 14:10 20-go.yaml -rw-rw-r-- 1 cikal docker 1533 Dec 20 14:11 30-laravel.yaml -rw-rw-r-- 1 cikal docker 658 Dec 20 14:11 40-frontend.yaml cikal@vm-docker:\~/three-body-problem-main$

**ChatGPT bilang:**

Selesai berpikir selama 42s

Siap. Aku rangkum dulu hasil “cek dari awal percakapan” + penyesuaian dengan runbook kamu, lalu aku kasih **pipeline final** yang sesuai target kamu: **build di vm-docker → push ke Harbor di vm-harbor → (node K8s auto) pull + deploy di Kubernetes (vm-worker)**.

***

### 1) Cek kondisi dari awal percakapan kamu

#### A. Error `Could not resolve hostname altssh.gitlab.com`

Itu **murni DNS** di `vm-docker` saat itu. Dari output yang kamu kirim belakangan:

* `ping 1.1.1.1` sukses ✅ (internet/routing OK)
* `getent hosts altssh.gitlab.com` sukses ✅ (DNS sudah resolve)

Jadi masalah “push ke GitLab” yang awal itu **sudah lewat** (sekarang tinggal memastikan pipeline-nya benar).

#### B. Dari K8s kamu: aplikasi sudah jalan, tapi ada warning besar

Pods kamu **Running**, image dari Harbor juga **sudah kepakai**:

* `harbor.local/threebody/frontend:bd17c4d5`
* `harbor.local/threebody/go:bd17c4d5`
* `harbor.local/threebody/laravel:bd17c4d5`

Artinya:

* **Harbor OK**
* **Cluster K8s bisa pull image dari Harbor** ✅
* Secret `app-secrets` juga sudah ada (karena env dari secret) ✅

Tapi di setiap pod ada event:

* `NodeNotReady` (vm-worker sering “not ready”)
* Probe Go/MySQL sempat timeout

> Ini penting: **pipeline deploy boleh benar, tapi kalau node sering NotReady, deploy jadi tidak stabil**. Jadi selain pipeline, kamu juga perlu cek kesehatan node `vm-worker`.

***

### 2) Penyesuaian dengan runbook kamu (yang paling relevan ke pipeline)

Dari runbook kamu, target flow awalnya:

> build & push image → deploy staging (docker compose) → healthcheck → deploy prod K8s → healthcheck

Tapi kamu sekarang minta:

> **build → push → prod/k8s** (tanpa staging)

Jadi penyesuaiannya:

* **hapus stage staging** dari pipeline (atau jadikan manual opsional)
* **deploy\_prod harus beneran kubectl apply + update image + rollout + healthcheck**
* **“pull” di K8s tidak dibuat job khusus**: pull terjadi **otomatis** ketika Deployment diganti tag image (kubelet di vm-worker akan pull).

***

### 3) Pipeline final yang kamu butuhkan (build → push → deploy\_prod)

Ini versi yang **aman untuk pemula**, sesuai struktur manifest kamu (`deploy/k8s/base/`), dan cocok untuk **shell runner di vm-docker**.

> Penting: aku pakai `KUBECONFIG_B64` seperti runbook kamu (base64 dari `~/.kube/config`).

#### ✅ `.gitlab-ci.yml` (copy–paste)

```yaml
stages:
  - build
  - push
  - deploy_prod

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"

  HARBOR_HOST: "harbor.local"
  HARBOR_PROJECT: "threebody"
  REGISTRY: "$HARBOR_HOST/$HARBOR_PROJECT"

  K8S_NS_PROD: "threebody-prod"
  KUBECTL_VERSION: "v1.30.14"
  K8S_ROLLOUT_TIMEOUT: "10m"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail

build_check:
  stage: build
  script: |
    echo "==> [build] build-check images (tanpa push)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel

push_images:
  stage: push
  needs: ["build_check"]
  script: |
    echo "==> [push] login Harbor"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [push] build+push images (TAG=$TAG)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker push "$REGISTRY/frontend:$TAG"

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker push "$REGISTRY/go:$TAG"

    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    docker push "$REGISTRY/laravel:$TAG"

deploy_prod:
  stage: deploy_prod
  needs: ["push_images"]
  script: |
    echo "==> [prod] validasi variables"
    : "${KUBECONFIG_B64:?Missing KUBECONFIG_B64}"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    : "${MYSQL_DATABASE:?Missing MYSQL_DATABASE}"
    : "${MYSQL_USER:?Missing MYSQL_USER}"
    : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"
    : "${LARAVEL_APP_KEY:?Missing LARAVEL_APP_KEY}"

    echo "==> [prod] siapkan kubectl (tanpa sudo)"
    mkdir -p .bin
    if [ ! -x .bin/kubectl ]; then
      curl -fsSL -o .bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x .bin/kubectl
    fi
    export PATH="$PWD/.bin:$PATH"
    kubectl version --client=true

    echo "==> [prod] buat kubeconfig dari base64"
    echo "$KUBECONFIG_B64" | base64 -d > ./kubeconfig
    export KUBECONFIG="$PWD/kubeconfig"

    echo "==> [prod] cek cluster"
    kubectl get nodes -o wide

    echo "==> [prod] pastikan namespace"
    kubectl get ns "$K8S_NS_PROD" >/dev/null 2>&1 || kubectl create ns "$K8S_NS_PROD"

    echo "==> [prod] buat/update secret app-secrets (DB + Laravel key)"
    kubectl -n "$K8S_NS_PROD" create secret generic app-secrets \
      --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
      --from-literal=MYSQL_DATABASE="$MYSQL_DATABASE" \
      --from-literal=MYSQL_USER="$MYSQL_USER" \
      --from-literal=MYSQL_PASSWORD="$MYSQL_PASSWORD" \
      --from-literal=LARAVEL_APP_KEY="$LARAVEL_APP_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] buat/update imagePullSecret Harbor"
    kubectl -n "$K8S_NS_PROD" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] attach imagePullSecret ke default serviceaccount"
    kubectl -n "$K8S_NS_PROD" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null || true

    echo "==> [prod] apply manifests (folder kamu deploy/k8s/base)"
    kubectl apply -f deploy/k8s/base/

    echo "==> [prod] update image tag (ini yang bikin vm-worker AUTO PULL)"
    # Sesuai output kamu: deployment kemungkinan bernama frontend, go, laravel
    kubectl -n "$K8S_NS_PROD" set image deployment/frontend frontend="$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/go go="$REGISTRY/go:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/laravel laravel="$REGISTRY/laravel:$TAG"

    echo "==> [prod] tunggu rollout"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/frontend --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/go --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/laravel --timeout="$K8S_ROLLOUT_TIMEOUT"

    echo "==> [prod] healthcheck via edge nginx (prod.local di vm-docker)"
    for i in $(seq 1 30); do
      if curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/ >/dev/null 2>&1; then
        echo "OK: prod sehat"
        exit 0
      fi
      sleep 2
    done

    echo "ERROR: prod healthcheck gagal"
    kubectl -n "$K8S_NS_PROD" get pods -o wide || true
    kubectl -n "$K8S_NS_PROD" describe pods || true
    exit 1
```

#### Kenapa pipeline ini sudah memenuhi “build → push → pull+deploy”?

* **Build**: dikerjakan di runner `vm-docker` (shell executor) ✅
* **Push**: login ke `harbor.local` lalu push 3 image ✅
* **Pull**: tidak ada job “pull”, karena begitu `kubectl set image ...:$TAG`, **kubelet di vm-worker akan otomatis pull tag baru** ✅
* **Deploy**: `kubectl apply` + `rollout status` ✅

***

### 4) GitLab Variables yang wajib kamu set (pemula-friendly)

Di GitLab Project → Settings → CI/CD → Variables:

**Registry (Harbor)**

* `HARBOR_USERNAME`
* `HARBOR_PASSWORD`

**Kubernetes**

*   `KUBECONFIG_B64`\
    Dari `vm-k8s`:

    ```bash
    base64 -w 0 ~/.kube/config
    ```

    copy hasilnya ke variable `KUBECONFIG_B64`

**App Secret (sesuai manifest kamu yang pakai secret `app-secrets`)**

* `MYSQL_ROOT_PASSWORD`
* `MYSQL_DATABASE`
* `MYSQL_USER`
* `MYSQL_PASSWORD`
* `LARAVEL_APP_KEY`

***

### 5) Checklist environment (biar pipeline nggak “random gagal”)

#### A) vm-docker (runner host)

Pastikan:

```bash
docker ps
id gitlab-runner
getent hosts harbor.local
getent hosts prod.local
```

#### B) vm-worker (node prod)

Karena kamu ada `NodeNotReady` warning, cek ini dulu:\
Di `vm-k8s`:

```bash
kubectl get nodes -o wide
kubectl describe node vm-worker | egrep -i "Ready|Pressure|NetworkUnavailable|Kubelet"
```

Di `vm-worker`:

```bash
sudo systemctl status kubelet containerd --no-pager
sudo journalctl -u kubelet -n 200 --no-pager
sudo journalctl -u containerd -n 200 --no-pager
```

Kalau node sering NotReady, biasanya sebab umum:

* kubelet/containerd crash atau restart
* swap belum benar-benar off
* disk penuh / pressure
* CNI (Calico) bermasalah
* time drift / clock jump (kadang bikin weird behavior)

> Ini penting: **deploy pipeline bagus, tapi node NotReady bikin rollout/probe sering timeout**.

***

### 6) Kenapa pod kamu restart ExitCode 255 “Reason Unknown”?

Ini biasanya bukan “app error murni”, tapi efek dari:

* node menjadi NotReady / jaringan putus → container dibunuh/restart
* kubelet restart
* masalah runtime/containerd

Untuk validasi cepat:

```bash
kubectl -n threebody-prod get events --sort-by=.lastTimestamp | tail -50
kubectl -n threebody-prod logs deploy/go --tail=200
kubectl -n threebody-prod logs deploy/laravel --tail=200
```

***

### 7) Kalau kamu tetap mau staging, tapi tidak mengganggu target utama

Buat staging jadi **manual** (opsional), jadi stage utama tetap 3.\
Contoh konsep:

* stages tetap `build, push, deploy_prod`
* tambah job `deploy_staging` dengan `when: manual` dan `allow_failure: true`

Tapi kalau target kamu murni prod dulu, staging mending jangan dimasukkan.

***

### Langkah yang paling aman kamu lakukan sekarang

1. **Replace `.gitlab-ci.yml`** dengan versi final di atas
2. Set GitLab variables (Harbor + Kubeconfig + MySQL + Laravel key)
3. Jalankan pipeline
4. Kalau gagal, biasanya 2 tipe:
   * gagal `kubectl get nodes` → kubeconfig/akses API server
   * gagal `rollout` / healthcheck → node NotReady / edge config / NodePort

Kalau kamu paste output **dua perintah ini** (nggak panjang):

```bash
kubectl get nodes -o wide
kubectl -n threebody-prod get deploy -o wide
```

aku bisa “kunci” nama deployment/container agar `kubectl set image` pasti match 100% (tanpa tebak-tebakan).

**Anda bilang:**

cikal@vm-k8s:\~$ kubectl get nodes -o wide kubectl -n threebody-prod get deploy -o wide NAME STATUS ROLES AGE VERSION INTERNAL-IP EXTERNAL-IP OS-IMAGE KERNEL-VERSION CONTAINER-RUNTIME vm-k8s Ready control-plane 44h v1.30.14 192.168.56.44 \<none> Ubuntu 24.04.3 LTS 6.8.0-90-generic containerd://1.7.28 vm-worker Ready \<none> 44h v1.30.14 192.168.56.45 \<none> Ubuntu 24.04.3 LTS 6.8.0-90-generic containerd://1.7.28 NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR frontend 1/1 1 1 40h frontend harbor.local/threebody/frontend:bd17c4d5 app=frontend go 1/1 1 1 40h go harbor.local/threebody/go:bd17c4d5 app=go laravel 1/1 1 1 40h laravel harbor.local/threebody/laravel:bd17c4d5 app=laravel cikal@vm-k8s:\~$ perintah diatas aku ambil dari vm-k8s ya, harusnya kamu masih ingat nama vm ini dari semua percakapan kita

**ChatGPT bilang:**

Ya, ingat: **vm-k8s = control-plane**, **vm-worker = node tempat pod prod jalan**, **vm-harbor = registry**, **vm-docker = runner + build/push + edge nginx**.

Dari output terbaru kamu:

* `vm-k8s Ready`, `vm-worker Ready` ✅ (jadi warning `NodeNotReady` sebelumnya kemungkinan **kejadian sesaat** ketika node sempat drop)
* Deployment prod kamu bernama **frontend, go, laravel** dan **nama container-nya sama** ✅
* Image sekarang pakai tag `bd17c4d5` ✅

Ini kabar bagus, karena berarti pipeline deploy ke K8s bisa dibuat **simple dan pasti cocok** (nggak perlu tebak nama deployment/container lagi).

***

## Pipeline yang paling pas untuk target kamu (build → push → deploy\_prod)

Kamu bilang ingin:

1. **Build di vm-docker**
2. **Push ke vm-harbor (harbor.local)**
3. **Pull + deploy ke Kubernetes (vm-worker)**

Di Kubernetes, “pull” terjadi otomatis saat deploy mengubah image tag. Jadi job deploy cukup:

* `kubectl apply` (manifest NodePort + mysql + dll)
* `kubectl set image deployment/... image:TAG`
* `kubectl rollout status`

Berikut versi `.gitlab-ci.yml` yang **paling clean** sesuai kondisi cluster kamu saat ini.

### ✅ .gitlab-ci.yml (final, sesuai nama deployment kamu)

```yaml
stages:
  - build
  - push
  - deploy_prod

workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never

variables:
  DOCKER_BUILDKIT: "1"
  TAG: "$CI_COMMIT_SHORT_SHA"

  HARBOR_HOST: "harbor.local"
  HARBOR_PROJECT: "threebody"
  REGISTRY: "$HARBOR_HOST/$HARBOR_PROJECT"

  K8S_NS_PROD: "threebody-prod"
  KUBECTL_VERSION: "v1.30.14"
  K8S_ROLLOUT_TIMEOUT: "10m"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

default:
  tags: ["deploy"]     # pastikan runner vm-docker punya tag 'deploy'
  before_script:
    - set -euo pipefail

build_check:
  stage: build
  script: |
    echo "==> [build] build-check images (tanpa push)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel

push_images:
  stage: push
  needs: ["build_check"]
  script: |
    echo "==> [push] login Harbor"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    echo "==> [push] build+push images (TAG=$TAG)"
    docker build \
      --build-arg REACT_APP_GO_API_BASE=/go \
      --build-arg REACT_APP_LARAVEL_API_BASE=/laravel \
      -t "$REGISTRY/frontend:$TAG" \
      -f frontend/Dockerfile frontend
    docker push "$REGISTRY/frontend:$TAG"

    docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go
    docker push "$REGISTRY/go:$TAG"

    docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel
    docker push "$REGISTRY/laravel:$TAG"

deploy_prod:
  stage: deploy_prod
  needs: ["push_images"]
  script: |
    echo "==> [prod] validasi variables"
    : "${KUBECONFIG_B64:?Missing KUBECONFIG_B64}"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    : "${MYSQL_DATABASE:?Missing MYSQL_DATABASE}"
    : "${MYSQL_USER:?Missing MYSQL_USER}"
    : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"
    : "${LARAVEL_APP_KEY:?Missing LARAVEL_APP_KEY}"

    echo "==> [prod] siapkan kubectl (tanpa sudo, aman untuk runner shell)"
    mkdir -p .bin
    if [ ! -x .bin/kubectl ]; then
      curl -fsSL -o .bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x .bin/kubectl
    fi
    export PATH="$PWD/.bin:$PATH"
    kubectl version --client=true

    echo "==> [prod] buat kubeconfig dari base64"
    echo "$KUBECONFIG_B64" | base64 -d > ./kubeconfig
    export KUBECONFIG="$PWD/kubeconfig"

    echo "==> [prod] cek cluster"
    kubectl get nodes -o wide

    echo "==> [prod] pastikan namespace"
    kubectl get ns "$K8S_NS_PROD" >/dev/null 2>&1 || kubectl create ns "$K8S_NS_PROD"

    echo "==> [prod] create/update app-secrets (dipakai go/laravel/mysql kamu)"
    kubectl -n "$K8S_NS_PROD" create secret generic app-secrets \
      --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
      --from-literal=MYSQL_DATABASE="$MYSQL_DATABASE" \
      --from-literal=MYSQL_USER="$MYSQL_USER" \
      --from-literal=MYSQL_PASSWORD="$MYSQL_PASSWORD" \
      --from-literal=LARAVEL_APP_KEY="$LARAVEL_APP_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] create/update imagePullSecret Harbor"
    kubectl -n "$K8S_NS_PROD" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    echo "==> [prod] attach imagePullSecret ke default serviceaccount"
    kubectl -n "$K8S_NS_PROD" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null || true

    echo "==> [prod] apply manifests NodePort + mysql"
    kubectl apply -f deploy/k8s/base/

    echo "==> [prod] update image tag (memicu AUTO PULL di vm-worker)"
    kubectl -n "$K8S_NS_PROD" set image deployment/frontend frontend="$REGISTRY/frontend:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/go go="$REGISTRY/go:$TAG"
    kubectl -n "$K8S_NS_PROD" set image deployment/laravel laravel="$REGISTRY/laravel:$TAG"

    echo "==> [prod] tunggu rollout"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/frontend --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/go --timeout="$K8S_ROLLOUT_TIMEOUT"
    kubectl -n "$K8S_NS_PROD" rollout status deployment/laravel --timeout="$K8S_ROLLOUT_TIMEOUT"

    echo "==> [prod] tampilkan ringkas hasilnya"
    kubectl -n "$K8S_NS_PROD" get deploy -o wide
    kubectl -n "$K8S_NS_PROD" get svc -o wide
    kubectl -n "$K8S_NS_PROD" get pods -o wide

    echo "==> [prod] healthcheck via edge nginx (prod.local di vm-docker)"
    for i in $(seq 1 30); do
      if curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/ >/dev/null 2>&1; then
        echo "OK: prod sehat"
        exit 0
      fi
      sleep 2
    done

    echo "ERROR: prod healthcheck gagal"
    kubectl -n "$K8S_NS_PROD" describe pods || true
    exit 1
```

***

## Kenapa aku yakin pipeline ini sudah “pas” untuk setup kamu?

Karena sekarang kita sudah tahu fakta dari cluster kamu:

* Namespace: `threebody-prod` ✅
* Deployment names: `frontend`, `go`, `laravel` ✅
* Container names: `frontend`, `go`, `laravel` ✅
* Image registry: `harbor.local/threebody/...:$TAG` ✅
* Manifest memang ada di repo: `deploy/k8s/base/` ✅

Jadi `kubectl set image deployment/frontend frontend=...` akan **100% match**.

***

## Arahan langkah demi langkah (untuk pemula)

### Step 1 — Pastikan runner memang “vm-docker”

Di GitLab UI:

* Project → Settings → CI/CD → Runners
* Pastikan runner yang aktif itu **shell runner di vm-docker**
* Tag-nya ada **deploy** (karena `default.tags: ["deploy"]`)

### Step 2 — Set GitLab Variables (wajib)

Project → Settings → CI/CD → Variables:

**Harbor**

* `HARBOR_USERNAME`
* `HARBOR_PASSWORD`

**Kubernetes**

*   `KUBECONFIG_B64` (dari vm-k8s)

    ```bash
    base64 -w 0 ~/.kube/config
    ```

**App**

* `MYSQL_ROOT_PASSWORD`
* `MYSQL_DATABASE`
* `MYSQL_USER`
* `MYSQL_PASSWORD`
* `LARAVEL_APP_KEY`

> Kalau `app-secrets` sudah ada di cluster, pipeline tetap aman karena kita pakai `kubectl apply` (update, bukan create ulang dengan error).

### Step 3 — Commit & push

Di vm-docker:

```bash
git add .gitlab-ci.yml
git commit -m "ci: build push deploy to k8s"
git push gitlab main
```

### Step 4 — Verifikasi hasilnya

Di vm-k8s:

```bash
kubectl -n threebody-prod get deploy -o wide
kubectl -n threebody-prod get pods -o wide
```

Kalau tag image berubah dari `bd17c4d5` ke tag baru → berarti **pull+deploy sukses**.

***

## Catatan penting: “NodeNotReady” yang kamu lihat sebelumnya

Sekarang nodes kamu `Ready`, jadi aman. Tapi kalau nanti kejadian lagi:

* deployment bisa timeout
* liveness/readiness bisa gagal sementara

Kalau muncul lagi, cek cepat:

```bash
kubectl describe node vm-worker | egrep -i "Ready|Pressure|NetworkUnavailable|Kubelet"
sudo systemctl status kubelet containerd --no-pager   # di vm-worker
```

***

Kalau kamu mau, aku bisa bantu “final check” sebelum kamu commit:\
paste isi file `deploy/k8s/base/20-go.yaml`, `30-laravel.yaml`, `40-frontend.yaml` (cukup bagian `Deployment`-nya aja) supaya aku pastikan:

* `imagePullPolicy` aman
* env `app-secrets` key-nya konsisten
* service NodePort sesuai edge nginx kamu (30080/30081/30082)

**Anda bilang:**

prod yang error berikut error log job nya dan tolong bantu perbaiki ya secara detail, lengkap serta mudah dipahami karena aku ini masih pemula dan jangan kamu sampai lupa untuk semua percakapan kita dari awal sampai sekarang agar tidak melenceng kemana-mana dan tetap sesuai rancangan. dibawah ini error log job deploy\_prod Running with gitlab-runner 18.7.0 (901991dd) on vm-docker-runner lO1LEm61F, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on vm-docker... Getting source from Git repository 00:03 Gitaly correlation ID: 9b1e93e88460f908-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/lO1LEm61F/0/cikalfarid/three-body-problem-main/.git/ Checking out e7b9cdd9 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:02 $ set -euo pipefail $ echo "==> \[prod] validasi variables" # collapsed multi-line command ==> \[prod] validasi variables ==> \[prod] siapkan kubectl (tanpa sudo, aman untuk runner shell) Client Version: v1.30.14 Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3 ==> \[prod] buat kubeconfig dari base64 base64: invalid input Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit status 1 cikal@vm-docker:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@vm-docker:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - build - push - deploy\_prod workflow: rules: - if: '$CI\_COMMIT\_BRANCH == "main"' - when: never variables: DOCKER\_BUILDKIT: "1" TAG: "$CI\_COMMIT\_SHORT\_SHA" HARBOR\_HOST: "harbor.local" HARBOR\_PROJECT: "threebody" REGISTRY: "$HARBOR\_HOST/$HARBOR\_PROJECT" K8S\_NS\_PROD: "threebody-prod" KUBECTL\_VERSION: "v1.30.14" K8S\_ROLLOUT\_TIMEOUT: "10m" K8S\_IMAGEPULL\_SECRET: "harbor-regcred" default: tags: \["deploy"] # pastikan runner vm-docker punya tag 'deploy' before\_script: - set -euo pipefail build\_check: stage: build script: | echo "==> \[build] build-check images (tanpa push)" docker build \ --build-arg REACT\_APP\_GO\_API\_BASE=/go \ --build-arg REACT\_APP\_LARAVEL\_API\_BASE=/laravel \ -t "$REGISTRY/frontend:$TAG" \ -f frontend/Dockerfile frontend docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel push\_images: stage: push needs: \["build\_check"] script: | echo "==> \[push] login Harbor" : "${HARBOR\_USERNAME:?Missing HARBOR\_USERNAME}" : "${HARBOR\_PASSWORD:?Missing HARBOR\_PASSWORD}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_HOST" -u "$HARBOR\_USERNAME" --password-stdin echo "==> \[push] build+push images (TAG=$TAG)" docker build \ --build-arg REACT\_APP\_GO\_API\_BASE=/go \ --build-arg REACT\_APP\_LARAVEL\_API\_BASE=/laravel \ -t "$REGISTRY/frontend:$TAG" \ -f frontend/Dockerfile frontend docker push "$REGISTRY/frontend:$TAG" docker build -t "$REGISTRY/go:$TAG" -f go/Dockerfile go docker push "$REGISTRY/go:$TAG" docker build -t "$REGISTRY/laravel:$TAG" -f laravel/Dockerfile laravel docker push "$REGISTRY/laravel:$TAG" deploy\_prod: stage: deploy\_prod needs: \["push\_images"] script: | echo "==> \[prod] validasi variables" : "${KUBECONFIG\_B64:?Missing KUBECONFIG\_B64}" : "${HARBOR\_USERNAME:?Missing HARBOR\_USERNAME}" : "${HARBOR\_PASSWORD:?Missing HARBOR\_PASSWORD}" : "${MYSQL\_ROOT\_PASSWORD:?Missing MYSQL\_ROOT\_PASSWORD}" : "${MYSQL\_DATABASE:?Missing MYSQL\_DATABASE}" : "${MYSQL\_USER:?Missing MYSQL\_USER}" : "${MYSQL\_PASSWORD:?Missing MYSQL\_PASSWORD}" : "${LARAVEL\_APP\_KEY:?Missing LARAVEL\_APP\_KEY}" echo "==> \[prod] siapkan kubectl (tanpa sudo, aman untuk runner shell)" mkdir -p .bin if \[ ! -x .bin/kubectl ]; then curl -fsSL -o .bin/kubectl "https://dl.k8s.io/release/${KUBECTL\_VERSION}/bin/linux/amd64/kubectl" chmod +x .bin/kubectl fi export PATH="$PWD/.bin:$PATH" kubectl version --client=true echo "==> \[prod] buat kubeconfig dari base64" echo "$KUBECONFIG\_B64" | base64 -d > ./kubeconfig export KUBECONFIG="$PWD/kubeconfig" echo "==> \[prod] cek cluster" kubectl get nodes -o wide echo "==> \[prod] pastikan namespace" kubectl get ns "$K8S\_NS\_PROD" >/dev/null 2>&1 || kubectl create ns "$K8S\_NS\_PROD" echo "==> \[prod] create/update app-secrets (dipakai go/laravel/mysql kamu)" kubectl -n "$K8S\_NS\_PROD" create secret generic app-secrets \ --from-literal=MYSQL\_ROOT\_PASSWORD="$MYSQL\_ROOT\_PASSWORD" \ --from-literal=MYSQL\_DATABASE="$MYSQL\_DATABASE" \ --from-literal=MYSQL\_USER="$MYSQL\_USER" \ --from-literal=MYSQL\_PASSWORD="$MYSQL\_PASSWORD" \ --from-literal=LARAVEL\_APP\_KEY="$LARAVEL\_APP\_KEY" \ --dry-run=client -o yaml | kubectl apply -f - echo "==> \[prod] create/update imagePullSecret Harbor" kubectl -n "$K8S\_NS\_PROD" create secret docker-registry "$K8S\_IMAGEPULL\_SECRET" \ --docker-server="$HARBOR\_HOST" \ --docker-username="$HARBOR\_USERNAME" \ --docker-password="$HARBOR\_PASSWORD" \ --dry-run=client -o yaml | kubectl apply -f - echo "==> \[prod] attach imagePullSecret ke default serviceaccount" kubectl -n "$K8S\_NS\_PROD" patch serviceaccount default \ -p "{\\"imagePullSecrets\\": \[{\\"name\\": \\"$K8S\_IMAGEPULL\_SECRET\\"}]}" >/dev/null || true echo "==> \[prod] apply manifests NodePort + mysql" kubectl apply -f deploy/k8s/base/ echo "==> \[prod] update image tag (memicu AUTO PULL di vm-worker)" kubectl -n "$K8S\_NS\_PROD" set image deployment/frontend frontend="$REGISTRY/frontend:$TAG" kubectl -n "$K8S\_NS\_PROD" set image deployment/go go="$REGISTRY/go:$TAG" kubectl -n "$K8S\_NS\_PROD" set image deployment/laravel laravel="$REGISTRY/laravel:$TAG" echo "==> \[prod] tunggu rollout" kubectl -n "$K8S\_NS\_PROD" rollout status deployment/frontend --timeout="$K8S\_ROLLOUT\_TIMEOUT" kubectl -n "$K8S\_NS\_PROD" rollout status deployment/go --timeout="$K8S\_ROLLOUT\_TIMEOUT" kubectl -n "$K8S\_NS\_PROD" rollout status deployment/laravel --timeout="$K8S\_ROLLOUT\_TIMEOUT" echo "==> \[prod] tampilkan ringkas hasilnya" kubectl -n "$K8S\_NS\_PROD" get deploy -o wide kubectl -n "$K8S\_NS\_PROD" get svc -o wide kubectl -n "$K8S\_NS\_PROD" get pods -o wide echo "==> \[prod] healthcheck via edge nginx (prod.local di vm-docker)" for i in $(seq 1 30); do if curl -kfsS --resolve prod.local:443:127.0.0.1 https://prod.local/ >/dev/null 2>&1; then echo "OK: prod sehat" exit 0 fi sleep 2 done echo "ERROR: prod healthcheck gagal" kubectl -n "$K8S\_NS\_PROD" describe pods || true exit 1 cikal@vm-docker:\~/three-body-problem-main$ git add .gitlab-ci.yml git commit -m "ci: build push deploy to k8s" git push gitlab main \[main e7b9cdd] ci: build push deploy to k8s 1 file changed, 88 insertions(+), 83 deletions(-) Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.89 KiB | 1.89 MiB/s, done. Total 3 (delta 1), reused 0 (delta 0), pack-reused 0 To gitlab-443:cikalfarid/three-body-problem-main.git 1da2be5..e7b9cdd main -> main cikal@vm-docker:\~/three-body-problem-main$

**ChatGPT bilang:**

Masalahnya **bukan Kubernetes / Harbor**, tapi **cara kamu menyimpan `KUBECONFIG_B64` di GitLab Variables**.

Log error-nya jelas:

> `base64: invalid input`

Artinya isi variable `KUBECONFIG_B64` **bukan base64 valid** (biasanya karena:

* ada newline/space yang kepotong,
* kamu copy base64 yang “wrapped” per baris,
* atau kamu menyimpan kubeconfig mentah (YAML) tapi pipeline mengira itu base64,
* atau ada karakter aneh/terpotong karena masked/protected setting).

Kita perbaiki dengan cara yang **paling aman untuk pemula** dan **sesuai rancangan kamu** (vm-docker runner → deploy ke vm-k8s).

***

## A) Perbaikan paling gampang: ubah `KUBECONFIG_B64` jadi File Variable (recommended)

Ini paling stabil, dan tidak perlu base64 sama sekali.

### 1) Ubah `.gitlab-ci.yml` (ganti bagian kubeconfig)

Ganti blok ini:

```bash
echo "$KUBECONFIG_B64" | base64 -d > ./kubeconfig
export KUBECONFIG="$PWD/kubeconfig"
```

menjadi:

```bash
: "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD (GitLab Variables Type: File)}"
export KUBECONFIG="$KUBECONFIG_PROD"
```

#### ✅ Patch yang kamu copy-paste (hanya job deploy\_prod bagian awalnya)

```yaml
deploy_prod:
  stage: deploy_prod
  needs: ["push_images"]
  script: |
    echo "==> [prod] validasi variables"
    : "${KUBECONFIG_PROD:?Missing KUBECONFIG_PROD (GitLab Variables Type: File)}"
    : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    : "${MYSQL_DATABASE:?Missing MYSQL_DATABASE}"
    : "${MYSQL_USER:?Missing MYSQL_USER}"
    : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"
    : "${LARAVEL_APP_KEY:?Missing LARAVEL_APP_KEY}"

    echo "==> [prod] siapkan kubectl (tanpa sudo, aman untuk runner shell)"
    mkdir -p .bin
    if [ ! -x .bin/kubectl ]; then
      curl -fsSL -o .bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
      chmod +x .bin/kubectl
    fi
    export PATH="$PWD/.bin:$PATH"
    kubectl version --client=true

    echo "==> [prod] set kubeconfig dari File Variable"
    export KUBECONFIG="$KUBECONFIG_PROD"

    echo "==> [prod] cek cluster"
    kubectl get nodes -o wide

    # ... lanjutkan script kamu yang sudah ada (secret/apply/set image/rollout/healthcheck)
```

> Sisanya (kubectl apply, set image deployment/frontend/go/laravel, rollout, healthcheck) **biarkan sama**. Itu sudah cocok dengan cluster kamu.

### 2) Set variable di GitLab

GitLab Project → Settings → CI/CD → Variables → Add variable:

* Key: `KUBECONFIG_PROD`
* Type: **File** ✅
* Value: isi file `~/.kube/config` dari **vm-k8s** (paste full content YAML)
* Protected: OFF dulu (biar pemula nggak ke-skip di branch main)
* Masked: OFF (file biasanya tidak cocok masked)

Cara ambil isi kubeconfig dari vm-k8s:

```bash
cat ~/.kube/config
```

Copy semua isi, paste ke variable file.

✅ Dengan ini, error `base64: invalid input` pasti hilang.

***

## B) Kalau kamu mau tetap pakai base64 (opsi kedua)

Kalau kamu ngotot tetap `KUBECONFIG_B64`, maka kamu harus regenerate base64 yang “1 line” dan valid.

Di **vm-k8s** jalankan:

```bash
base64 -w 0 ~/.kube/config
```

Lalu copy outputnya **persis** ke GitLab variable `KUBECONFIG_B64`.

Dan di pipeline, ganti decode jadi lebih toleran:

```bash
echo "$KUBECONFIG_B64" | tr -d '\n\r ' | base64 -d > ./kubeconfig
```

Tapi untuk pemula, aku tetap rekomendasikan **File Variable**.

***

## C) Kenapa pipeline kamu sesuai rancangan? (biar kamu yakin tidak melenceng)

✅ Build dilakukan di **vm-docker** (runner shell)\
✅ Push dilakukan ke **Harbor di vm-harbor** (`harbor.local/threebody/...:$CI_COMMIT_SHORT_SHA`)\
✅ Deploy dilakukan ke **Kubernetes (vm-k8s control-plane)**\
✅ “Pull” dilakukan otomatis oleh **vm-worker** saat `kubectl set image ...:$TAG` (kubelet akan pull tag baru dari Harbor)

Dan nama deployment/container sudah kita pastikan dari output kamu:

* deployment `frontend` container `frontend`
* deployment `go` container `go`
* deployment `laravel` container `laravel`

Pipeline kamu sudah benar di bagian itu.

***

## D) Checklist cepat setelah perbaikan (biar kamu tahu suksesnya di mana)

Setelah kamu ubah ke `KUBECONFIG_PROD` (File Variable) dan rerun pipeline:

1. Job deploy\_prod harus lewat step:

* `kubectl get nodes -o wide` (harus sukses, kamu akan lihat `vm-k8s Ready`, `vm-worker Ready`)

2. Pastikan tag image berubah dari `bd17c4d5` ke tag commit baru:\
   Di vm-k8s:

```bash
kubectl -n threebody-prod get deploy -o wide
```

3. Pastikan pods rollout baru:

```bash
kubectl -n threebody-prod get pods -o wide
```

***

## E) Bonus: kalau kamu ingin staging tetap ada tapi tidak mengganggu target build→push→prod

Karena kamu minta fokus 3 stage, staging sebaiknya **dipisah manual** (opsional). Tapi sekarang kamu memang sudah pakai 3 stage dan itu sesuai target.

***

### Yang perlu kamu lakukan sekarang (step-by-step)

1. Edit `.gitlab-ci.yml`: ganti `KUBECONFIG_B64` jadi `KUBECONFIG_PROD` (File Variable) seperti patch di atas
2. Di GitLab variables: buat `KUBECONFIG_PROD` tipe **File** isi dari `cat ~/.kube/config` di vm-k8s
3. Rerun pipeline

Kalau setelah itu masih gagal, kirim log baru bagian `deploy_prod` sampai error-nya (biasanya berikutnya kalau salah adalah: `Unable to connect to the server` atau RBAC).

ChatGPT dapat membuat kekeliruan. OpenAI tidak menggunakan data workspace ChatGpt+ Business untuk melatih modelnya.<br>
