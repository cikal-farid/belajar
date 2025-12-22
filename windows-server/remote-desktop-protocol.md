# Remote Desktop Protocol

1. Selanjutnya akan muncul Initial Configuration Tasks, kita ke Customize This Server lalu klik Enable Remote Desktop agar bisa di remote.

<figure><img src="../.gitbook/assets/image (724).png" alt=""><figcaption></figcaption></figure>

2. Selanjutnya kita pilih Allow connections from computers running any version of Remote Desktop (less secure).

<figure><img src="../.gitbook/assets/image (725).png" alt=""><figcaption></figcaption></figure>

3. Lalu akan muncul pesan peringatan untuk memberi pengecualian pada firewall maka klik Windows Firewall with Advanced Security.

<figure><img src="../.gitbook/assets/image (726).png" alt=""><figcaption></figcaption></figure>

4. Pada Overview pilih Windows Firewall Properties.

<figure><img src="../.gitbook/assets/image (727).png" alt=""><figcaption></figcaption></figure>

5. Pada menu Domain Profile Inbound connections pilih Allow, lalu Allow juga pada menu Private Profile dan Public Profile selanjutnya klik Apply dan ok.

<figure><img src="../.gitbook/assets/image (728).png" alt=""><figcaption></figcaption></figure>

6. Selanjutnya kita ke View and create firewall rules. Klik Inbound Rules.

<figure><img src="../.gitbook/assets/image (729).png" alt=""><figcaption></figcaption></figure>

7. Pilih Netlogon Service, Network Discovery, Remote Administration, Remote Desktop, Remote Event Log Management, Remote Scheduled Task Management, dan Remote Service Management lalu di Enable Rules.

<figure><img src="../.gitbook/assets/image (730).png" alt=""><figcaption></figcaption></figure>

8. Selanjutnya pada View and create firewall rules kita pilih Outbound Rules.

<figure><img src="../.gitbook/assets/image (731).png" alt=""><figcaption></figcaption></figure>

9. Kita pilih Network Discovery dan Enable Rule.

<figure><img src="../.gitbook/assets/image (732).png" alt=""><figcaption></figcaption></figure>

10. Selanjutnya kembali ke pesan peringatan Remote Desktop Connection, klik ok.

<figure><img src="../.gitbook/assets/image (733).png" alt=""><figcaption></figcaption></figure>

11. Lalu pada System Properties klik Apply dan ok.

<figure><img src="../.gitbook/assets/image (734).png" alt=""><figcaption></figcaption></figure>

12. Selanjutnya cek ip server melalui cmd.

<figure><img src="../.gitbook/assets/image (735).png" alt=""><figcaption></figcaption></figure>

13. Selanjutnya kita coba untuk remote ke alamat ip tersebut.

<figure><img src="../.gitbook/assets/image (736).png" alt=""><figcaption></figcaption></figure>

14. Jika berhasil akan seperti tampilan berikut. Windows server berhasil di install.

<figure><img src="../.gitbook/assets/image (737).png" alt=""><figcaption></figcaption></figure>











































