---
title: What is SSRF?

---

# What is SSRF?
- Lợi dụng server mục tiêu làm proxy để gửi request thay cho hacker
- Mục tiêu là hệ thống mạng nội bộ mà ban đầu bị chặn bởi firewall đối với người dùng bên ngoài
    - Các dịch vụ của bên thứ 3 mà server mục tiêu có quyền truy cập

# What is the impact of SSRF attacks?
- Đọc được dữ liệu như file config, metadata, tương tác với các API nội bộ không cần xác thực, hoặc RCE  thông qua các service nội bộ

# SSRF attacks against the server
- Dev thường thiết lập các quy tắc trong cấu hình nginx, apache, middleware của ứng dụng để bỏ qua việc kiểm tra JWT/session cookie nếu request có IP nguồn là `127.0.0.1` hoặc `::1`
- Các API nội bộ giao tiếp với nhau qua localhost thường không implement cơ chế authentication nghiêm ngặt để giảm độ trễ, từ đó hacker lợi dụng endpoint bị lỗi SSRF làm proxy để giả mạo một service nội bộ hợp lệ
``` java!
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1


POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```

- Lỗ hổng nằm ở tham số `stockApi`. Ứng dụng nhận đầu vào là một URL hoàn chỉnh và `fetch` nó mà không có cơ chế kiểm tra tính hợp lệ hay whitelist nào

- Lỗi không nằm ở code của tính năng admin mà nằm ở kiến trúc mạng và phân quyền
- Reverse proxy bypass: trong các hệ thống như nginx,  hoặc WAF thường đứng trước, xử lý xác thực, kiểm tra cookie/JWT. Backend ở sau mặc định tin tưởng mọi request lọt qua được nginx. Khi dùng SSRF ta đi vòng qua nginx và trực tiếp vào backend -> backend sẽ cho rằng ta thuộc nội bộ

- Tính năng disaster recovery là một dạng backdoor hợp pháp do dev bỏ lại. Họ nghĩ chỉ có người SSH được vào server mới dùng được localhost, nhưng SSRF có thể biến web server thành một terminal mini

- Firewall bên ngoài chỉ mở port 80/443. Nhưng bên trong server có thể đang mở port 8080 (tomcat admin), 5000 (flask), hoặc 27017 (mongoDB) mà không cần password

**[LAB 1](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)**

![image](https://hackmd.io/_uploads/B1C9mWy5bl.png)
- Chọn 1 vật bất kỳ rồi view details, sau đó check stock để xem URL `stockApi` của nó
![image](https://hackmd.io/_uploads/r1QpQZ19bg.png)

![image](https://hackmd.io/_uploads/Bk44VZJqWx.png)
- Thay đổi URL trong tham số `stockApi` thành `http://localhost`, sẽ hiển thị giao diện chứa endpoint của admin
- Sau đó truy cập tới endpoint `/admin` để xem danh sách user
![image](https://hackmd.io/_uploads/BkOL4b1cbl.png)
![image](https://hackmd.io/_uploads/BJSdEbyqZl.png)

- Ta thấy được endpoint `/delete?usernam=carlos`, giờ hãy POST tới `http://localhost/admin/delete?username=carlos` để thực hiện xoá user carlos
![image](https://hackmd.io/_uploads/HyZ9VZycZx.png)
![image](https://hackmd.io/_uploads/Hkj0N-J9Zx.png)
![image](https://hackmd.io/_uploads/B1m1HWk5Wg.png)
![image](https://hackmd.io/_uploads/H1EqSWy5be.png)

# SSRF attacks against other back-end systems
- Firewall bên ngoài bảo vệ chặt nhưng khi đã vô được vào trong mạng nội bộ bằng SSRF thì mọi thứ đều lộ hết ra. Với suy nghĩ đã ở trong mạng LAN thì đều là cùng 1 nội bộ, không cần auth
- Các dải IP private chuẩn bao gồm `10.0.0.0/8`, `172.16.0.0/12`, và `192.168.0.0/16`
- Lỗ hổng cho phép hackker dùng máy chủ web bị lỗi làm một proxy để scan và tấn công các máy chủ khác nằm sâu trong mạng nội bộ

**[LAB 2](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)**
![image](https://hackmd.io/_uploads/SJUGTWJcWx.png)
![image](https://hackmd.io/_uploads/r1kEpW19Zx.png)
- Truy cập vào món đồ rồi check stock

![image](https://hackmd.io/_uploads/H1Z8pb15bg.png)
![image](https://hackmd.io/_uploads/Hkfup-J5Zl.png)
- Chung ta không biết IP nội bộ là gì để có thể truy cập của admin panel rồi xoá user carlos
- Nên ta sẽ scan IP để tìm ra IP nội bộ chứa admin panel
![image](https://hackmd.io/_uploads/S11L0bkqWx.png)
- Ta tìm được IP nội bộ là `192.168.0.76:8080`
![image](https://hackmd.io/_uploads/rJQP0-kc-g.png)
![image](https://hackmd.io/_uploads/r1wKC-kqZg.png)

- Từ đây ta sẽ truy cập tới endpoint `/admin` để có được giao diện của admin panel, và các bước xoá username carlos tương tự như **[LAB 1](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)**
![image](https://hackmd.io/_uploads/S1U90-y9Zx.png)
![image](https://hackmd.io/_uploads/r1tRAZ19bg.png)
![image](https://hackmd.io/_uploads/S1S-kGkcbx.png)

# SSRF with blacklist-based input filters
- Dev thường dùng regex hoặc hàm `contains()` để chặn các keyword cụ thể. Hacker chỉ cần làm cho chuỗi payload nhìn thì có vẻ khác nhưng vẫn tương đương đối với hệ thống mạng là có thể bypass
- Filter kiểm tra chuỗi dưới dạng text, nhưng thư viện gọi HTTP request như cURL, urllib lại phân tích nó dưới dạng URL/IP. Sự sai lệch giữa hai hệ thống này tạo ra vuln
- IP obfuscation, làm rối chuỗi (URL/double URL encoding), DNS rebinding/spoofing, và lợi dụng tính năng tự động theo đuổi chuyển hướng HTTP redirect

**[LAB 3](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)**
- Truy cập vào một sản phẩm rồi check stock và gửi nó sang repeater
![image](https://hackmd.io/_uploads/rJI9EzJ5Ze.png)
![image](https://hackmd.io/_uploads/SJG6EMy9Wl.png)

- Thay đổi URL trong tham số `stockApi` thành `http://127.0.0.1/` và quan sát thấy bị chặn
![image](https://hackmd.io/_uploads/SkvySMJ5Wl.png)

- Bypass filter bằng cách đổi URL thành `http://127.1/`
![image](https://hackmd.io/_uploads/B1HZSfJ9Zx.png)

- Thay đổi URL thành `http://127.1/admin` và thấy URL lại bị chặn lần nữa
![image](https://hackmd.io/_uploads/SJLsrM19bg.png)

- Obfuscate chữ `a` bằng cách encode URL hai lần thành `/%25%36%31dmin` để truy cập tới admin panel và xóa user carlos
![image](https://hackmd.io/_uploads/rJZGIzk5Zx.png)

- Bước xoá user tương tự LAB 1 và LAB 2
![image](https://hackmd.io/_uploads/rkfHLGkqbx.png)
![image](https://hackmd.io/_uploads/BJJwIM1qZe.png)

# SSRF with whitelist-based input filters
- Thay vì dùng các thư viện parse URL chuẩn để trích xuất chính xác hostname, dev lại dùng các hàm string matching như `startsWith("https://trusted.com")` hoặc `contains("trusted.com")`
- Ứng dụng thường có 2 thành phần độc lập: validator (kiểm tra xem URL có hợp lệ không) và fetcher (thực hiện HTTP request). Việc bypass whitelist dựa vào việc lừa validator thấy chuỗi `trusted.com`, nhưng ép fetcher hiểu rằng trusted.com chỉ là rác `username/password/fragment` và điểm đến thực sự là `127.0.0.1`
- Lợi dụng cấu trúc RFC 3986 chuẩn của URL: `scheme://user:password@host:port/path?query#fragment`

# Bypassing SSRF filters via open redirection
- SSRF + open tedirect = filter nypass. Một lỗ hổng low severity (open redirect) khi kết hợp với một lỗ hổng khác (SSRF) có thể tạo ra 1 thứ khủng bố
- Lớp filter/validator của ứng dụng chỉ kiểm tra URL ban đầu. Nó thấy domain `weliketoshop.net` nằm trong whitelist nên cho qua. Nhưng thư viện HTTP client ở backend như cURL, Java HttpURLConnection thường được cấu hình mặc định là follow redirect tức là tự động đi theo HTTP 301/302. Khi nhận được response chuyển hướng thì HTTP client tự động quay sang vô thẳng vào mạng nội bộ mà không qua màng lọc kiểm tra lần thứ hai
=> Sự thiếu đồng bộ giữa logic kiểm tra và logic thực thi 

- Ví dụ: endpoint SSRF bắt buộc phải là `api.target.com` và ta tìm được Open Redirect tại `api.target.com/go?url=...`
    - Payload:`stockApi=https://api.target.com/go?url=http://192.168.0.68/admin`

**LAB 4**
- Truy cập vào một sản phẩm rồi check stock và gửi nó sang repeater
![image](https://hackmd.io/_uploads/SkLb5MJ9-e.png)

- Thử giả mạo tham số `stockApi` và thấy không có cách nào khiến máy chủ thực hiện yêu cầu trực tiếp đến một host khác
![image](https://hackmd.io/_uploads/Sy75qfk5Zg.png)

- Nhấp vào `next produc`t và thấy tham số `path` được đặt vào header location của một phản hồi chuyển hướng dẫn đến lỗi open redirection
![image](https://hackmd.io/_uploads/B12sqzJcZe.png)
![image](https://hackmd.io/_uploads/BJ0Tqfkqbg.png)
![image](https://hackmd.io/_uploads/HknWjGkq-x.png)
![image](https://hackmd.io/_uploads/BJ7GizJ9be.png)

- Tạo một URL khai thác lỗ hổng open redirection chuyển hướng đến admin panel và nạp nó vào tham số `stockApi` trên trình kiểm tra tồn kho:
`/product/nextProduct?path=http://192.168.0.12:8080/admin`
- Thấy trình kiểm tra tồn kho đi theo chuyển hướng và hiển thị admin panel
- Sửa path để xóa user carlos:
`/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos`
![image](https://hackmd.io/_uploads/S1O8iMkqWl.png)
![image](https://hackmd.io/_uploads/BJcOiGyqWg.png)
![image](https://hackmd.io/_uploads/H1_6oGycbg.png)


















