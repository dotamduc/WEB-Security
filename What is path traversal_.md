---
title: What is path traversal?

---

# What is path traversal?
- Lỗi kiểm soát đầu vào đối với các tham số xử lý file, dẫn đến việc thoát khỏi thư mục gốc của web
- Read: leak source code, file cấu hình (DB, API keys), file hệ thống (`/etc/passwd`,..v..v..). Đủ để thu thập thông tin leo thang
- Write: nếu có quyền ghi kết hợp path traversal -> RCE 

# Reading arbitrary files via path traversal
- Các endpoint xử lý tệp tin tĩnh (hình ảnh, tài liệu) thông qua tham số (VD: `?filename=`).
- Ứng dụng nối chuỗi trực tiếp user input vào một base directory mà không sanitize các ký tự duyệt thư mục
- Lợi dụng đặc tính của hệ điều hành cho phép dùng `../` hoặc `..\` để trỏ ngược lên thư mục cha, từ đó thoát khỏi  webroot (`/var/www/images/`) và trỏ thẳng về thư mục gốc hệ thống (`/` hoặc `C:\`)

**[LAB 1](https://portswigger.net/web-security/file-path-traversal/lab-simple)**
- Sử dụng burp suite để chỉnh sửa một request  đang tải hình ảnh sản phẩm 
- Sửa đổi tham số filename, gán cho nó giá trị sau:
`../../../etc/passwd`
- Thấy response trả về chứa toàn bộ nội dung của tệp `/etc/passwd`
![image](https://hackmd.io/_uploads/B10bh5rcZg.png)

# Common obstacles to exploiting path traversal vulnerabilities
- Gặp phải hệ thống có filter chỉ lo chặn các ký tự tương đối như `../` hoặc `..\` mà không block các đường dẫn bắt đầu từ gốc
- Sử dụng absolute path. Thay vì lùi từng bước từ thư mục hiện tại ta chỉ định thẳng vị trí chính xác của file trên ổ đĩa hệ điều hành
- Nhiều hàm xử lý file cấp thấp như `open()` trong python hay `file_get_contents()` trong PHP khi nhận vào một chuỗi bắt đầu bằng `/` hoặc ổ đĩa `C:\` chúng sẽ tự động ghi đè luôn base directory của ứng dụng và ưu tiên đường dẫn tuyệt đối

**[LAB 2](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)**
![image](https://hackmd.io/_uploads/SJUHJjHc-e.png)
![image](https://hackmd.io/_uploads/HJa81oBcZe.png)

- Lab này đã có filter hoặc regex để chặn các chuỗi lùi thư mục cơ bản `../` hoặc `..\`
- Chặn `../` nhưng không kiểm tra xem input có phải là absolute path hay không. Nếu ta nối một thư mục gốc `/var/www/images/` với một đường dẫn tuyệt đối `/etc/passwd`, hàm sẽ bỏ qua hoàn toàn phần thư mục gốc và trả về thẳng đường dẫn tuyệt đối đó

- Khi ta gửi `....//`, phần `../` ở giữa bị xóa đi, hai phần còn lại `..` (ở đầu) và `/` (ở cuối) sẽ tự động nối lại với nhau thành một chuỗi `../` hoàn chỉnh và hợp lệ đối với hệ điều hành
**[LAB 3](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)**

![image](https://hackmd.io/_uploads/rJ1LQiSqZl.png)
![image](https://hackmd.io/_uploads/ByYLQiH9bx.png)
![image](https://hackmd.io/_uploads/HyGBQoS9-g.png)

- Giả sử các victim không nằm ở tầng ứng dụng mà nằm ở tầng web server (Nninx, apache, Tomcat) hoặc WAF. Các tầng này tự động normalize URL hoặc cắt mất `../` trước khi request chạm tới backend
- Sử dụng các kỹ thuật encode ký tự để giấu payload khỏi các bộ lọc chuỗi tĩnh
- WAF kiểm tra và decode lần 1 thấy `%2e%2e%2f` không phải `../` nên bỏ qua. Khi request vào tới backend, framework (VD: java spring, PHP) tự động decode thêm lần 2, biến `%2e%2e%2f` trở về `../` và thực thi thành công

**[LAB 4](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)**

![image](https://hackmd.io/_uploads/rJpQ9iScZg.png)
![image](https://hackmd.io/_uploads/ByiV9oSqZg.png)
![image](https://hackmd.io/_uploads/BymUciHc-x.png)

- Hệ thống có đặt filter chặn path traversal. Dev biết rằng web server tự động decode URL 1 lần nên đã viết code để chặn cả chuỗi raw `../` lẫn chuỗi single encode `%2e%2e%2f`
- Dev gọi thêm hàm `urldecode()` ở backend sau khi đã đi qua bộ lọc WAF. (nguyên tắc vàng bị phá vỡ: luôn phải validate dữ liệu sau khi đã hoàn tất mọi quá trình decode)
- Bọc payload bằng 2 lớp mã hóa URL. Lớp thứ nhất bypass WAF, lớp thứ hai được backend tự decode và thực thi

- nhận ra việc chặn `../` có thể bị bypass bằng nhiều cách, nên họ đổi cách khác. Họ sử dụng hàm kiểm tra xem chuỗi input có bắt đầu bằng thư mục chuẩn mực hay không (VD: dùng `strpos($path, '/var/www/images/') === 0` trong PHP hoặc `path.startswith('/var/www/images/')` trong python)
- Logic bug: dev chỉ kiểm tra phần đầu của chuỗi mà không normalize toàn bộ đường dẫn trước khi kiểm tra. Việc nối `../` ngay sau thư mục cơ sở hợp lệ sẽ giúp đoạn đầu của payload bypass, sau đó hệ điều hành sẽ tự động xử lý phần `../` để thoát ra ngoài
- Nó muốn thấy thư mục nào ở đầu, ta cho nó thư mục đó, sau đó tiếp nối bằng `../`

**[LAB 5](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)**
![image](https://hackmd.io/_uploads/r1hHToS9bg.png)

- chỉ kiểm tra xem chuỗi đầu vào có bắt đầu bằng chuỗi `/var/www/images/` hay không (sử dụng các hàm như `startsWith()`). Nó quên việc kiểm tra các ký tự nhảy thư mục `../` nằm ở phần thân của chuỗi

- áp dụng cơ chế suffix validation (iểm tra hậu tố). Họ dùng code để ép buộc chuỗi đầu vào phải kết thúc bằng một đuôi file cụ thể (như `.png, .jpg, .pdf`) nhằm ngăn chặn việc đọc các file hệ thống (`.txt`, `.conf`, file không có đuôi)

- Null byte `%00`: trong C/C++ ký tự null (được biểu diễn là `\0` hoặc `%00` trên URL) đóng vai trò là ký tự kết thúc chuỗi. Khi thêm `%00` vào giữa bộ lọc của web vẫn đọc toàn bộ chuỗi để pass validation nhưng hệ điều hành bên dưới sẽ chặt chuỗi ngay tại `%00` và bỏ qua phần đuôi file

**[LAB 6](https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass)**

![image](https://hackmd.io/_uploads/HyBH13HqZg.png)
![image](https://hackmd.io/_uploads/B1nH13HqWg.png)

- Sửa đổi tham số filename, gán cho nó giá trị sau: `../../../etc/passwd%00.png`
- Thấy response trả về chứa nội dung của tệp `/etc/passwd`

# How to prevent a path traversal attack
- Không tin tưởng user input. Dùng định danh gián tiếp (VD: truyền `id=1` vào DB, DB trả ra đường dẫn `/var/www/images/avatar.jpg` ở backend)
- DiD: nếu bắt buộc phải nhận tên file từ user thì phải kết hợp 2 thứ:
    - whitelist, filter: chỉ cho phép `[a-zA-Z0-9]`, block dấu chấm `.`, gạch chéo `/`
    - Chuẩn hóa đường dẫn: sử dụng hàm của hệ điều hành (như `getCanonicalPath()` trong java, `realpath()` trong PHP/python) để tự động tính toán, block toàn bộ `../`, `./` hay symlink thành một đường dẫn tuyệt đối cuối cùng. Sau đó mới dùng hàm `startsWith()` để đối chiếu xem nó có nằm đúng trong base directory




