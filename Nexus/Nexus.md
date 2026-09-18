---
title: Nexus

---

**Hack The Box Machines - Nexus**
- Catagory: Linux
- Rating: Easy
- Description: Nexus is an easy-difficulty Linux machine that features an exposed Gitea repository leaking credentials and a job posting that reveals valid usernames. The leaked credentials provide access to Krayin CRM, which is vulnerable to CVE-2026-38526, leading to a shell as www-data. Further enumeration of the Krayin CRM configuration files reveals additional credentials that allow SSH access. Service enumeration reveals a Gitea template sync service vulnerable to directory traversal, which is leveraged to gain a shell as root.

**Question1:** `How many open TCP ports are listening on Nexus?`
- Sử dụng **nmap** để quét các cổng dịch vụ đang chạy, kết quả tìm ra có hai cổng đang mở là **22(SSH)** và **80(Http)**.
<img width="403" height="92" alt="Screenshot 2026-09-18 095958" src="https://github.com/user-attachments/assets/841b3479-f54d-49eb-aa11-61e59019544a" />

- Sửa đổi file `/etc/hosts` để cho máy phân giải địa chỉ ip trỏ về domain của bài.
<img width="311" height="65" alt="Screenshot 2026-09-18 100035" src="https://github.com/user-attachments/assets/e756e252-590a-4550-859b-b36985a1a412" />

**Question3:** `What is the name of the additional subdomain hosting the Git service discovered during enumeration of nexus.htb?` 
- Với gợi ý liên quan đến subdomain, ta sẽ sử dụng công cụ để **fuzz** tìm kiếm các subdomain khác để khai thác.
<img width="908" height="125" alt="Screenshot 2026-09-18 105031" src="https://github.com/user-attachments/assets/cddaf10c-888f-48aa-891d-f95d90e9f8bc" />

- Kết quả tìm được `git` và `billing`, `git` chứa hệ thống **Gitea** có file `.env` làm lộ mật khẩu đăng nhập trong lịch sử commit.
<img width="199" height="221" alt="Screenshot 2026-09-18 105324" src="https://github.com/user-attachments/assets/c82c64d6-963a-46a7-a0c3-86f16e610383" />

- `billing` có hệ thống đang chạy **Krayin CRM 2.2.0**, có liên quan đến CVE-2026-38526, lỗ hổng cho phép user `upload file php` nhưng giả dạng file ảnh.

**Question7**: `What is the password for jones discovered during post-exploitation?`
- Mục tiêu cần khai thác là đính kèm file trong email, khi thử upload 1 file bình thường thì thấy trả về kết quả file được lưu tại `/storage/emails/ten_file`.
<img width="697" height="164" alt="Screenshot 2026-09-18 105822" src="https://github.com/user-attachments/assets/76148655-195a-456f-b393-a488e5f44a36" />

- Ý tưởng sẽ là đính kèm 1 file php với chức năng sẽ tạo 1 reverse_shell từ server về máy của attacker tại cổng đang lắng nghe, attacker nhận được quyền của web server **(www)**.
- Chuyển đổi file thành đuôi `.png` để đi qua filter, chặn request lại bằng `Burp Intercept` để sửa tên file trở lại thành `.php`, sau khi thấy path lưu trữ file vừa upload, gửi request để kích hoạt file `.php`.
<img width="701" height="187" alt="Screenshot 2026-09-18 111045" src="https://github.com/user-attachments/assets/b20b7af7-ac2e-4da9-80fb-b607e812973a" />

- Trên máy attacker, sử dụng netcat để mở cổng lắng nghe, sau khi nhận được shell với quyền `www`, nâng cấp shell lên bằng `script /dev/null -c /bin/bash`.
<img width="632" height="124" alt="Screenshot 2026-09-18 111117" src="https://github.com/user-attachments/assets/ed3dc6b9-514b-44e4-98f2-dc86406685a2" />

- Đọc file `.env` thu được mật khẩu và sử dụng mật khẩu này để SSH user `jones`, do mật khẩu **DB** được tái sử dụng.
<img width="314" height="83" alt="Screenshot 2026-09-18 111312" src="https://github.com/user-attachments/assets/aa6cd8c0-ff58-4bab-b322-49f97872de9f" />

**Question9:** `What systemd timer triggers the template synchronization service?`
- Dùng `systemctl list-timers` để liệt kệ các bộ hẹn giờ và các dịch vụ đi kèm của các tác vụ tự động. Kết quả trả về `gitea-template-sync.timer`.
- `systemctl cat gitea-template-sync.service`, sẽ xem được dịch vụ này chạy file `.py`, nội dung chính của file là clone các template repository 2 phút 1 lần.

**Question10:** `Submit the flag located in the root user's home directory?`
- Flow hoạt động là duyệt danh sách file bằng `git ls-tree` rồi sao chép vào `/home/git/template-staging/<owner>/<repo>/`, nhưng vấn đề ở trong cách xử lý đường dẫn, script lấy trực tiếp giá trị truyền vào từ user mà không có cơ chế validate. Nếu người dùng lợi dụng `../` thì có thể khiến file được ghi ra ngoài thư mục `staging`.
<img width="368" height="43" alt="Screenshot 2026-09-12 161619" src="https://github.com/user-attachments/assets/e77d0ed7-2d19-4cac-990e-ed8ff6be6627" />

- Tạo 1 repo template mới, trên máy attacker sinh 1 cặp khóa công khai, ghi khóa công khai vào `/root/.ssh/authorized_keys` bằng việc lợi dụng cơ chế clone `template_repo`, nhưng thông thường **git** có hàm verify file path "..", nên cần tạo Git object trong `.git/objects`bằng script python.
- `git push -u origin main --force`, đợi 2 phút khi systemd chạy, nó sẽ clone `public key` và attacker có thể ssh vào máy server, sau đó lấy được flag.


