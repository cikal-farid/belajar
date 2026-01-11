# CI / CD

1. Pertama download terlebih dahulu java nya dengan perintah dibawah ini

```
apt-get install openjdk-17-jdk-headless
```

Cek

```
javac --version
```

<figure><img src="../.gitbook/assets/image (810).png" alt=""><figcaption></figcaption></figure>

2. Download Git

```
apt-get install git
```

<figure><img src="../.gitbook/assets/image (811).png" alt=""><figcaption></figcaption></figure>

3. Download jenkinsnya di situs jenkinsnya

```
https://www.jenkins.io/download/
```

Klik kanan dan copy ink address

<figure><img src="../.gitbook/assets/image (812).png" alt=""><figcaption></figcaption></figure>

Dan jalankan perintah dibawah ini untuk download jenkinsnya

```
wget https://get.jenkins.io/war-stable/2.528.2/jenkins.war
```

<figure><img src="../.gitbook/assets/image (813).png" alt=""><figcaption></figcaption></figure>

4. Buka firewall port 9090 karena saya akan menggunakan port 9090

```
ufw allow 9090
```

<figure><img src="../.gitbook/assets/image (21) (1).png" alt=""><figcaption></figcaption></figure>

5. Tambahkan dibawah ini agar Github dapat berjalan dengan baik

```
nano ~/.ssh/config
```

```
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_rsa
```

```
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

```
ssh -T git@github.com
```

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

6. Kemudian jalankan jenkins.war dengan menggunakan perintah dibawah ini

```
java -jar jenkins.war --httpPort=9090 &
```

Jika ingin menggunakan port default cukup jalankan perintah dibwah ini

```
java -jar jenkins.war &
```

Agar log di simpan di file jalankan perintah dibawah ini

```
nohup java -jar jenkins.war --httpPort=9090 > jenkins.out 2>&1 &
```

Note :&#x20;

a. menggunakan " & " di akhir agar jenkins berjalan di background

b. menggunakan nohup agar jenkins tetap berjalan walau terminal di tutup

c. `> jenkins.out 2>&1` → output dan error disimpan di file `jenkins.out`&#x20;



7. Lanjut buka browser dengan url dibawah ini

```
http://192.168.56.41:9090/
```

8. Tampilan pertama kali akses jenkins akan seperti gambar dibawah ini

<figure><img src="../.gitbook/assets/image (11) (1) (1).png" alt=""><figcaption></figcaption></figure>

Isi kan Administrator password dengan melihat isi file dari /root/.jenkins/secrets/initialAdminPassword dan lanjut klik Continue

```
cat /root/.jenkins/secrets/initialAdminPassword
```

<figure><img src="../.gitbook/assets/image (12) (1) (1).png" alt=""><figcaption></figcaption></figure>

9. Tampilan awal login seperti gambar dibawah ini dan karena saya tidak ingin menginstall plugin apapun maka perhatikan seperti di gambar

<figure><img src="../.gitbook/assets/image (13) (1) (1).png" alt=""><figcaption></figcaption></figure>

10. Disini pilih None dan Install

<figure><img src="../.gitbook/assets/image (16) (1).png" alt=""><figcaption></figcaption></figure>

11. Isi kan data Admin User seperti gambar dibawah ini dan Save continue

Username

```
cinosta
```

Password

```
cinosta123
```

Confirm Password

```
cinosta123
```

Full Name

```
cikal farid
```

<figure><img src="../.gitbook/assets/image (17) (1).png" alt=""><figcaption></figcaption></figure>

12. Isi Jenkins URL dan Continue

```
http://192.168.56.41:9090/
```

<figure><img src="../.gitbook/assets/image (18) (1).png" alt=""><figcaption></figcaption></figure>

13. Terakhir klik Start Using Jenkins

<figure><img src="../.gitbook/assets/image (19) (1).png" alt=""><figcaption></figcaption></figure>

14. Tampilan Jenkins

<figure><img src="../.gitbook/assets/image (20) (1).png" alt=""><figcaption></figcaption></figure>

15. Disini saya akan setup SSH Public Key Server ke Github, jika belum ada jalankan perintah dibawah ini
16. Generate SSH key pair di Ubuntu

```
ssh-keygen -t rsa -b 4096
```

<figure><img src="../.gitbook/assets/image (22) (1).png" alt=""><figcaption></figcaption></figure>

Setelah perintah dijalankan kita akan memasukkan beberapa kondisi dan disini saya pilih enter saja

17. Copas isi dari id\_rsa.pub ke github

```
cat ~/.ssh/id_rsa.pub
```

Karena ini tutorial saya paste disini isi didalam file id\_rsa.pub nya (jangan sampai semua orang tahu)

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCzjpx+OvpnedQfEbXu6A84tuOUw4LLc+wG+mytphUdsBKZ23oWi9Xuipgs6E0ftDaRrW8qKkmpP/Aay3QC4qzzxptV4lO+TX8ybBCePijnXkNEnlq/jDBv/+ybJOwcYO+AJTeZJP5XNtAtePV5l9R4ZpKmOEG6JdVhxvBO/+eBHt+kUizjI0Iw0FXqs9+Y2VoJtq3mde6QjAI1/Z6uGKEtAcb6AfYln1H/ENrGbVHSk+7fSBnLiccS8touqgWPMqz9mHJ1jkG/bgFFTXRkZRNXP2QiGCnx2Xla/PtpUM/vxoqePxfXwfFeD75vT7/JNw5IsYyEXpgdAdsdym5W/wKX0CAWB0/JEkEm6y70/6N+duiAw71r+clRC0AbK/YrcdE6y30X8VtEl4mpsVYDUwxJllcZRmoBduo2XYaO7DTTz9J7hP897Sv6xN6cxwyf9fmHcZOaCXkyrxo1P4nfzsNMy/mlipwpfjW46qYXajZL5kUFhY7o6bPJOE5wSjK4BFuAonJj2lc1E7HkiCeX+8JrlpLoOY7Wi0Ck6dyTGCcpU+Lm+FJADTzqKWtTmxhSuvYc9CfEZvQMYlv50/kAVt1RciqOwDqE+g208syLujeyREYx4CxmlsiOcHtLnwQ1iEzM5X/9LprkNJSbub2eB1WeGigBl4824L3J+IpKvd1xow== root@cinosta
```

<figure><img src="../.gitbook/assets/image (23) (1).png" alt=""><figcaption></figcaption></figure>

18. Login Github dan klik Profile dan masuk ke menu Settings dan klik SSH and GPG Keys

<figure><img src="../.gitbook/assets/image (24) (1).png" alt=""><figcaption></figcaption></figure>

19. Kemudian klik New SSH Key

<figure><img src="../.gitbook/assets/image (25) (1).png" alt=""><figcaption></figcaption></figure>

20. Isi Title dan Key yang sebelumnya kita generate di server dan salin serta pindahkan seperti gambar dibawah ini, kemudian klik Add SSH key

<figure><img src="../.gitbook/assets/image (26) (1).png" alt=""><figcaption></figcaption></figure>

21. Menambahkan Credential SSH di Jenkins dengan cara menyalin terlebih dahulu isi dari file id\_rsa (private key)

```
cat ~/.ssh/id_rsa
```

Salin semua

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

22. Masuk ke Jenkins dan install plugin Git karena sebelumnya saya tidak ingin install plugin apapun, dan lanjut klik Restart Jenkins (Refresh Page Browser tekan F5 jika Jenkins belum update)

<figure><img src="../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

23. Masuk ke setting

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

24. Masuk ke menu Credential

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

25. Klik Global

<figure><img src="../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

26. Klik Add Credential

<figure><img src="../.gitbook/assets/image (3) (1) (1).png" alt=""><figcaption></figcaption></figure>

27. Pilih SSH Username

<figure><img src="../.gitbook/assets/image (4) (1) (1).png" alt=""><figcaption></figcaption></figure>

28. Isi data seperti di gambar lalu Create

<figure><img src="../.gitbook/assets/image (5) (1) (1).png" alt=""><figcaption></figcaption></figure>

Note :&#x20;

1. Scope = Global agar siapa saja dapat akses
2. ID = saya isi github-cikal
3. Deskripsi = Github Cikal
4. Username = cikal-farid (user github)
5. Add Private Key = id\_rsa











Install Plugin Pipeline dan Restart Jenkins

<figure><img src="../.gitbook/assets/image (6) (1) (1).png" alt=""><figcaption></figcaption></figure>







Kita mulai membuat job di jenkins dengan pipeline juga dari sini sudah dimulai CI/CD nya

Klik Create a Job

<figure><img src="../.gitbook/assets/image (7) (1) (1).png" alt=""><figcaption></figcaption></figure>







Menggunakan Post Build Action

Install Slack notification

<figure><img src="../.gitbook/assets/image (9) (1) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

Jika menggunakan VPS dan menemukan error pada tampilan jenkins seperti teks dibawah ini

**HTTP Error 403 No Valid CRUMB was included...**

Maka untuk memperbaikinya kita masuk security dan enable proxy compabilitynya seperti gambar dibawah ini

<figure><img src="../.gitbook/assets/image (8) (1) (1).png" alt=""><figcaption></figcaption></figure>









































































































