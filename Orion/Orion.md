---
title: Orion

---

**Hack The Box Machines - Orion**
- Catagory: Linux
- Rating: Easy
- Description: Orion is a very easy Linux machine that features CSRF Validation Bypass and exploration of CraftCMS and Telnetd. The foothold includes achieving remote code execution by exploiting CVE-2025-32432 in a vulnerable version of CraftCMS. Then the default Craft environment variable file exposes the credentials for its MySQL database, which contains a crackable password. The password has been reused and leads to SSH access to the user on the machine. Finally, privilege escalation is achieved by finding and exploiting a vulnerable version of telnetd (CVE-2026-24061), allowing authentication bypass to root.

**Question1:** `How many open TCP ports are listening on Orion?`
- Sử dụng nmap để quét các cổng dịch vụ đang chạy, kết quả tìm ra có hai cổng đang mở là 22(SSH) và 80(Http).
 <img width="437" height="99" alt="Screenshot 2026-09-18 123112" src="https://github.com/user-attachments/assets/b34b7ef3-fe4e-4950-9592-13aedd541394" />

- Sửa đổi file /etc/hosts để cho máy phân giải địa chỉ ip trỏ về domain của bài.
<img width="322" height="152" alt="Screenshot 2026-09-18 123154" src="https://github.com/user-attachments/assets/60635315-3c07-4d88-a847-e5de335f6f08" />

**Question2:** `What is the version of CraftCMS running on the target?`
- Sau khi fuzzing thử các endpoint phổ biến, kết quả trả về được `/admin` với `status code = 302`, điều này có nghĩa khi truy cập endpoint này, nó sẽ điều hướng đến endpoint khác.
 <img width="829" height="129" alt="Screenshot 2026-09-18 123452" src="https://github.com/user-attachments/assets/dba11175-4d12-471d-ade2-421a1cf892d3" />

- Nhận ra endpoint `/admin` này sẽ chuyển hướng đến trang `login`, ở đây ta thấy được phiên bản **Craft CMS 5.6.16**, khi tra cứu trên mạng thì ta thấy có 1 CVE liên quan đến hệ thống này.
 <img width="266" height="359" alt="Screenshot 2026-09-18 123539" src="https://github.com/user-attachments/assets/8d48109f-2e22-4b50-802a-d7288146bcf3" />

- CVE-2025-32432: CraftCMS xây dựng trên `Yii framework`, Yii có cơ chế cho cấu hình object bằng array/configuration, đơn giản có thể hiểu là tạo 1 object mới bằng mảng
- Trong CraftCMS tồn tại các action routes: Bắt đầu bằng tiền tố actions/… để gọi trực tiếp các controller xử lý chức năng ngầm của hệ thống hoặc plugin. Ví dụ: `actions/assets/generate-transform`. Cơ chế được diễn tả trong payload như sau:
<img width="472" height="117" alt="Screenshot 2026-09-13 212704" src="https://github.com/user-attachments/assets/60d717d0-57be-42cd-8376-6afa49fbfcb4" />

- `class: "craft\\behaviors\\FieldLayoutBehavior"`: Đây được coi như một lá chắn hợp lệ, hệ thống CraftCMS trước khi xử lý mảng sẽ kiểm tra khi thấy từ khóa `class`, nhưng nó thấy đây là 1 class hợp lệ vì vậy mà không loại bỏ, `_class` cũng được bỏ qua vì bộ lọc không biết từ khóa (**option**) này dùng để làm gì.
- "__class": "GuzzleHttp\\Psr7\\FnStream": Đây là mục tiêu thật (object) mà attacker muốn tạo ra vì FnStream là một class của thư viện GuzzleHttp. Class này cho phép định nghĩa hàm callback bằng chuỗi. Hiểu đơn giản là khi object bị đóng hoặc hủy, nó sẽ tự động gọi hàm đó -> RCE.
-	Trong cơ chế **DI Container** của Yii2, `_class` là từ khóa đặc biệt có **priority** > `class`,`_class` được dùng để chỉ định rõ cấu hình cho DI Container. Vì vậy, FieldLayoutBehavior không được tạo ra, mà class được tạo là **FnStream**.
-	`"__construct()"`: [ [] ]”: Class FnStream của thư viện Guzzle đòi hỏi tham số đầu vào của hàm `__construct` phải là một mảng **(array $methods)**. Do đó, cấu trúc [ [] ] nghĩa là truyền một mảng rỗng làm tham số đầu tiên cho hàm tạo.
-	`"_fn_close": "phpinfo"`: Gán cho thuộc tính `_fn_close` cụm `phpinfo`, mục đích để khi đối tượng trong php không được sử dụng nữa thì sẽ tự động gọi hàm `_destruct()` -> tự động gọi tiếp đến hàm được lưu trong biến `close()`-> `close()` kích hoạt code được gán trong `_fn_close`.
- Sau khi truyền payload vào, ta tìm trong response trả về thì thấy php thường lưu session trong đường dẫn: /var/lib/php/sessions. Trong CraftCSM sessionID chính là CraftSessionID, tương ứng: `/var/lib/php/sessions/sess_<Session_ID>`.
- Sau khi tìm được đường dẫn cụ thể của file Session, thực hiện biến file này thành một file chứa mã độc. Khi attacker gửi 1 request **GET** chứa chuỗi như `a=<?eval($_GET[‘cmd’]);die()?>`, cơ chế của php sẽ tự động lưu lại lịch sử request hoặc các tham số URL này vào trong file Session của chính session đó trên ổ cứng.
- `<?eval($_GET[‘cmd’])>`: là nội dung độc hại, còn `die()` đóng vai trò để kết thúc chương trình php ngay lập tức, tránh lỗi Syntax bởi các giá trị đằng sau.
-	Gửi 1 request bình thường đến `/admin/login` để lấy **Cookie** và **X-CSRF-Token**. Payload chỉ thành công khi có đủ `CraftSesionId`, `CRAFT_CSRF_TOKEN` và `X-CSRF-TOKEN`.

**Question5:** `What is the password that can be obtained from the MySQL database?`
- Để lấy được những thông tin trong Database, cần lợi dụng lỗ hổng như trên, nhưng mục tiêu nhắm vào **PhpManager**, vì đây là lớp quản lý phân quyền của Yii Framework.
<img width="455" height="216" alt="Screenshot 2026-09-13 214347" src="https://github.com/user-attachments/assets/9f08b89d-5aa0-4982-a7f0-87b04100562e" />

- Vì nó được lập trình sẵn tính năng khi khởi tạo sẽ tìm tệp tin được khai báo ở `itemFile` và ép hệ thống chạy lệnh `include_once($itemFile)` để nạp dữ liệu.
-	Ta sẽ truyền đường dẫn vào file Session mà ta đã đầu độc ở trên, mã độc được thực thi trực tiếp bằng quyền Server.
-	`base64endoded_reversephpPAYLOAD`: Đây là đoạn code php có chức năng tạo kết nối ngược, nói đơn giản là máy chủ victim sẽ mở một đường truyền mạng kết nối đến địa chỉ IP của attacker (Phải encodeBase64 vì để bản rõ có thể bị WAF chặn).
-	Khi Payload được gửi thành công, `fileSession` độc hại được thực thì `eval($_GET[‘cmd’])`, `cmd` được gán ở trên payload chính là đoạn code php để mở kết nối từ victim đến máy attacker. Phía máy attacker mở lắng nghe tại cổng đã định trước, và có được quyền truy cập vào máy chủ Orion.
-	Thực hiện tự động bằng Metasploit, dùng `getuid` thì thấy shell lấy được có quyền của `www`, nâng cấp shell lên terminal bằng `script /dev/null -c /bin/bash`.
<img width="450" height="128" alt="Screenshot 2026-09-13 214936" src="https://github.com/user-attachments/assets/2f616573-02e9-4146-b6b9-461c649c3db9" />

- Truy cập vào file môi trường .env để đọc nội dung bên trong: tên db, mật khẩu db, người dùng db,… Sau đó xác thực db để xem thông tin của db: `mysql -u root -p orion`.
 <img width="515" height="287" alt="Screenshot 2026-09-18 124723" src="https://github.com/user-attachments/assets/4ba73397-cbe9-409d-b4db-741b0bece114" />

-   Truy vấn toàn bộ cơ sở dữ liệu user `Select *from users;`, tìm được 1 user tên Adam.
<img width="929" height="204" alt="Screenshot 2026-09-18 125734" src="https://github.com/user-attachments/assets/327adf36-3de1-4d70-a88c-b7e81f190318" />

-    Mật khẩu đã bị bcrypt, vì vậy sử dụng `hashcat` với `rockyou.txt` để tìm ra mật khẩu dạng rõ.
<img width="777" height="266" alt="Screenshot 2026-09-18 125859" src="https://github.com/user-attachments/assets/74a3f7a9-adf6-4a21-a304-5b400234daf5" />

**Question10:** `Submit the flag located in the root user's home directory?`
- Đăng nhập bằng ssh với `adam@orion.htb` và password, đọc file user.txt như bình thường. Sau khi thử quét các cổng và dịch vụ và hỗ trợ thì có thấy xuất hiện cổng ==23== tương ứng với telnet, một giao thức không an toàn.
<img width="502" height="195" alt="Screenshot 2026-09-18 130256" src="https://github.com/user-attachments/assets/a7df3268-8a4d-468a-b0dd-0a4f1e551d45" />

- Kiểm tra version của telnet là **2.7** thì thấy có bị lỗ hổng liên quan đến **CVE-2026-24061**.
<img width="432" height="101" alt="Screenshot 2026-09-18 130415" src="https://github.com/user-attachments/assets/78506419-19a9-41a6-87e9-89ef2f7db4e0" />

- Khai thác bằng payload `USER="-f root" telnet -a 127.0.0.1`. `USER="-f root"`: Tạo ra một biến môi trường tạm thời trên máy mình với giá trị là `-f root`. `telnet -a 127.0.0.1`: Tham số -a (**Autologin**) ra lệnh cho chương trình Telnet tự động bốc biến môi trường USER vừa tạo để gửi sang bên Server trong quá trình thiết lập kết nối (**handshake**).
- Kết quả ta gửi đi `-f root` lên server. Bản chất của `telnetd` là sẽ gọi một chương trình có sẵn của OS Linux là `/usr/bin/login` để xử lý kiểm tra tài khoản/mật khẩu. Khi kết hợp lại thì attacker sẽ đăng nhập với quyền root mà không cần mật khẩu. Từ đó ta đọc root.txt bình thường là lấy được flag.
 <img width="410" height="248" alt="Screenshot 2026-09-18 130618" src="https://github.com/user-attachments/assets/61e12e08-3763-4001-a88c-46419aad0695" />

- **Note**:
> Sử dụng `“shell”` để chuyển từ môi trường Metasploit sang Terminal của victim
`“$ script /dev/null -c /bin/bash”`: Nâng cấp Terminal để có thể sử dụng Tab để tự động điền nhanh,…
`“netstat -tulnp”`: Liệt kê tất cả các cổng mạng đang mở và đang Listening, cùng với thông tin tiến trình, bao gồm cả các dịch vụ chạy ngầm.

