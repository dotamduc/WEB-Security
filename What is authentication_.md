---
title: What is authentication?

---

# What is authentication?
- Là quá trình xác minh danh tính của một user hoặc client. Các trang web luôn có nguy cơ bị tiếp cận bởi bất kỳ ai có kết nối internet. Điều này cần có các cơ chế xác thực mạnh mẽ
- Ba loại xác thực chính:
    - Something you know: mật khẩu hoặc câu trả lời cho một câu hỏi bảo mật (yếu tố kiến thức)
    - Something you have: một vật thể vật lý như điện thoại hoặc token bảo mật (yếu tố sở hữu)
    - Something you are or do: sinh trắc học hoặc thói quen hành vi (yếu tố vốn có)

# What is the difference between authentication and authorization?
- Xác thực là quá trình xác minh một user đúng là người mà họ tự nhận
- Phân quyền liên quan đến việc xác minh xem user đó có được phép thực hiện một hành động cụ thể nào đó hay không

# How do authentication vulnerabilities arise?
- Cơ chế phòng thủ yếu hoặc không có: không có Rate Limit, Account Lockout hay Captcha.
- Logic flaw: dev tự implement luồng đăng nhập nhưng xử lý biến,trạng thái sai ví dụ tin tưởng dữ liệu từ client-side, check lỏng lẻo bằng `==` thay vì `===` trong PHP,v.v.v.v.

# What is the impact of vulnerable authentication?
- Account takeover: chiếm tài khoản user thường bằng cách bruteforce, lộ session, hoặc XSS lấy cookie => đọc được thông tin cá nhân, lịch sử giao dịch
- Admin panel to RCE: khi đã có tài khoản admin, các tính năng hợp lệ thường bị lợi dụng để tấn công hệ thống nội bộ

# Vulnerabilities in password-based login
- Endpoint đăng nhập cho phép gửi hàng ngàn request mỗi phút mà không chặn IP hay yêu cầu CAPTCHA.
- Bypass IP ban: server đếm số lần đăng nhập sai dựa trên IP, nhưng lại tin vào các HTTP header giả do client gửi lên (như X-Forwarded-For, X-Originating-IP, Client-IP)
- HTTP basic authentication: cơ chế này không có trang đăng nhập HTML mà bật popup của trình duyệt. Nó không hỗ trợ bảo vệ CSRF và thường thiếu rate limit ở server (Apache/nginx) nếu admin không cấu hình kỹ

## Brute-forcing usernames
- Rỉ thông tin qua API/HTTP response: các endpoint API như `/api/users` hoặc `/api/comments` trả về toàn bộ object user bao gồm cả email hoặc username nội bộ, thay vì chỉ trả về tên hiển thị
- IDOR: Trang web cho phép xem hồ sơ bằng ID như `/profile?id=1`, và ID 1 thường là admin
- Predictable credential: sử dụng cấu trúc email có thể đoán được (vd: `johndoe@company.com`) kết hợp với việc công ty công khai danh sách trên trang About Us

## Brute-forcing passwords
- Đoán từ password pattern: mặc dù policy ép có chữ hoa, số và ký tự đặc biệt, người dùng thường dùng cách: `[Tên công ty/Tên user viết hoa chữ đầu]` + `[Năm]` + `[! / @]`. Ví dụ: `Admin@2024`, `Company!123`
- Thiếu rate limit: password policy chỉ tăng thời gian bẻ khóa. Nếu endpoint login không giới hạn số lần thử, hacker có thể chạy các custom wordlist với các ruleset hàng triệu dòng cho đến khi thành công
- Password expiry: nếu ta kiếm được một mật khẩu cũ bị lộ từ db năm ngoái là Admin@2023, khả năng cao mật khẩu hiện tại của user đó sẽ là `Admin@2024` hoặc `Admin!2024`
- Leetspeak: người dùng nghĩ rằng `P@ssw0rd` là an toàn, nhưng hacker đã bơm sẵn toàn bộ quy luật biến đổi leetspeak vào các file rule của công cụ bruteforce
- Nếu hệ thống khóa tài khoản sau 3 lần nhập sai, thay vì thử các mật khẩu ngẫu nhiên, ta chỉ thử đúng 3 biến thể dễ đoán nhất của từ khóa liên quan đến mục tiêu

## Username enumeration
- Dựa trên thông báo lỗi: 
    - User sai: Invalid username
    - User đúng nhưng sai pass: Incorrect password
    - Đăng ký tài khoản: Username already taken

- Length-based: thông báo lỗi có thể giống hệt nhau `Invalid username or password`, nhưng `Content-Length` trả về trong HTTP Response lại chênh lệch vài byte do server xử lý template HTML khác nhau
- Dựa trên thời gian : nếu user không tồn tại, database trả về kết quả ngay lập tức. Nếu user tồn tại database phải lấy hash mật khẩu ra và dùng thuật toán bcrypt/argon2 để so sánh dẫn đến thời gian phản hồi chậm hơn

**[LAB 1](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)**
![image](https://hackmd.io/_uploads/B19CiQDtZe.png)
![image](https://hackmd.io/_uploads/Sylpan7DKbe.png)
![image](https://hackmd.io/_uploads/SkN-k4PtWg.png)
![image](https://hackmd.io/_uploads/H1uGkEvtWg.png)

**[LAB 2](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)**
- Gửi một username và password không hợp lệ. Bôi đen tham số username trong request `POST /login` và gửi sang Intruder
- Ở mục Payloads đảm bảo loại payload Simple list được chọn và thêm danh sách các tên username
- Chọn Settings, dưới mục Grep - Extract, nhấp Add. Trong hộp thoại xuất hiện cuộn xuống qua phần phản hồi cho đến khi tìm thấy thông báo lỗi `Invalid username or password.`. Sử dụng chuột để bôi đen nội dung văn bản của thông báo. Các cài đặt khác sẽ tự động được điều chỉnh. Nhấp OK và sau đó Start Attack
- Hãy nhìn kỹ vào response và nhận thấy nó chứa một lỗi đánh máy trong thông báo lỗi, thay vì dấu chấm, lại có một khoảng trắng ở cuối. Ghi chú lại tên người dùng này
- Chèn username vừa xác định và thêm một vị trí payload vào tham số password: `username=root&password=§abc`
- Lặp lại các bước trên nhận thấy rằng một trong các request nhận được phản hồi 302 => đây là password đúng
- Đăng nhập bằng tên người dùng và mật khẩu mà ta đã xác định

**[LAB 3](https://portswigger.net/web-security/learning-paths/authentication-vulnerabilities/password-based-vulnerabilities/authentication/password-based/lab-username-enumeration-via-response-timing)**
- Gửi một username và password không hợp lệ rồi gửi request `POST /login` sang Repeater. Thử với các username và password khác nhau. Lưu ý là IP sẽ bị chặn nếu thực hiện quá nhiều lần đăng nhập không hợp lệ
- Xác định rằng header `X-Forwarded-For` được hỗ trợ, cho phép spoof IP của mình và bypass bảo vệ chống brute-force dựa trên IP
- Tiếp tục thử nghiệm với các username và password. Chú ý đến thời gian phản hồi. Nhận thấy rằng khi username không hợp lệ, thời gian phản hồi gần như giống nhau. Nhưng khi nhập một tên người dùng hợp lệ, thời gian phản hồi sẽ tăng lên tùy thuộc vào độ dài của mật khẩu bạn đã nhập
- Gửi request này sang Intruder và chọn kiểu tấn công Pitchfork từ menu thả xuống. Thêm header `X-Forwarded-For`
- Thêm các vị trí payload cho header `X-Forwarded-For` và tham số username. Đặt mật khẩu thành một chuỗi ký tự rất dài
- Ở mục Payloads chọn vị trí 1. Chọn loại payload là Numbers. Nhập dải số từ 1 - 100 và đặt step là 1. Đặt số chữ số thập phân tối đa là 0. Payload này sẽ được dùng để giả mạo IP
- Chọn vị trí 2 từ danh sách thả xuống sau đó thêm danh sách username. Start Attack
- Khi kết thúc, ở đầu hộp thoại nhấp vào Columns và chọn các tùy chọn Response received và Response completed. Hai cột này bây giờ sẽ hiển thị trong bảng kết quả
- Nhận thấy rằng một trong các thời gian phản hồi dài hơn đáng kể so với các phản hồi khác. Lặp lại request này vài lần để chắc chắn rằng nó luôn mất nhiều thời gian hơn, sau đó ghi chú lại username này
- Tạo một attack Intruder mới cho cùng một request. Thêm lại header `X-Forwarded-For` và thêm một vị trí payload cho nó. Chèn username ta vừa xác định và thêm một vị trí payload vào tham số password
- Ở mục Payloads thêm danh sách các số vào vị trí payload 1 và thêm danh sách mật khẩu vào vị trí payload 2. Start Attack
- Khi kết thúc hãy tìm phản hồi có trạng thái 302 => password để login

## Flawed brute-force protection
- Lỗi logic làm mới bộ đếm: server gắn bộ đếm số lần đăng nhập sai với địa chỉ IP thay vì gắn với session hoặc username cụ thể đang bị tấn công. Nhưng việc đăng nhập thành công lại clear toàn bộ lịch sử vi phạm của IP đó bất kể username là gì
- Nếu hệ thống chỉ block IP mà không khóa tài khoản thì việc bypass IP block đồng nghĩa với việc tài khoản mục tiêu sẽ bị bruteforce mật khẩu

**[LAB 4](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)**
- Thấy rằng IP sẽ bị chặn tạm thời nếu gửi 3 lần đăng nhập sai liên tiếp. Nhưng ta có thể reset bộ đếm số lần đăng nhập thất bại bằng cách đăng nhập vào tài khoản đúng trước khi đạt đến giới hạn này
- Ở mục Resource pool thêm attack vào một resource pool với số lượng request đồng thời tối đa (Maximum concurrent requests) được đặt thành 1
- Chọn pos 1 của Payload position. Thêm một danh sách các payload xen kẽ giữa tên `wiener` và `carlos`. Dưới đây là code để gen

``` python
for i in range(150):
    if i%3:
        print("carlos")
    else:
        print("wiener")
        
        
with open('password.txt', 'r') as f:
    lines = f.readlines()
    
i = 0
for pwd in lines:
    if i%3:
        print(pwd.strip('\n'))
    else:
        print("peter")
        print(pwd.strip('\n'))
        i=i+1
    i=i+1
```

- Copy danh sách mật khẩu vừa gen và paste vào pos 2 của Payload position rồi Start attack
- Khi kết thúc thì lọc các kết quả trả về 302 cho request của carlos => lấy mật khẩu trả về 302 rồi đăng nhập

![image](https://hackmd.io/_uploads/ryjqYQuF-l.png)

## Account locking
- Username enumeration: thông báo Tài khoản của bạn đã bị khóa khẳng định tài khoản đó tồn tại trong hệ thống
- Password spraying: cơ chế này chỉ bảo vệ được một tài khoản cụ thể khi bị vertical bruteforce. Nhưng nó vô dụng nếu hacker lấy 1 mật khẩu phổ biến nhất và thử trên 1000 tài khoản khác nhau

**[LAB 5](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock)**
- Attack type: Cluster bomb
- Request: username=§HEHE§&password=abc§§
- Payload 1: candidate usernames
- Payload 2: chọn Null payloads -> chọn generate 5 payloads
- Nó sẽ chạy tuần tự: carlos (thử 5 lần), wiener (thử 5 lần), v.v. request nào sinh ra lỗi `too many incorrect` thì đó là user xịn
- Chờ 1 phút để hệ thống nhả trạng thái khóa
- Cố định username vừa tìm được chuyển thành kiểu Sniper rồi spam danh sách mật khẩu và tìm request không có thông báo lỗi

## User rate limiting
- Chặn IP thay vì account locking. Việc này giúp chặn các tấn công DOS
- Nếu endpoint `/login` cho phép định dạng JSON thay vì Form URL-Encoded, và backend sử dụng framework như NodeJS/Express hoặc PHP xử lý JSON yếu. Thay vì gửi 1 chuỗi string thì ta gửi 1 array chứa hàng trăm password. Hệ thống chỉ ghi nhận đây là 1 HTTP Request hoàn toàn không vi phạm rate limit, nhưng backend lại mang cả array đó đi so sánh ở vòng lặp

## HTTP basic authentication
- HTTP basic auth: thông tin đăng nhập chỉ được encoding bằng Base64 dưới định dạng `username:password`, chứ không phải là encryption hay hash
- Mit): gửi đi gửi lại token tĩnh trong mọi request dễ bị bắt gói tin nếu không có mã hóa
- Thiếu bảo vệ bruteforce: token tĩnh hoàn toàn có thể bị bruteforce
- CSRF: không có cơ chế phòng thủ CSRF tích hợp.
- Việc lộ thông tin ở một trang rác có thể dẫn đến việc tái sử dụng mật khẩu để attack các hệ thống lõi

# Vulnerabilities in multi-factor authentication
- Lỗi triển khai:  2FA được tạo ra để an toàn hơn, nhưng nếu luồng xử lý lỏng lẻo thì nó có thể bị bypass dễ dàng
- 2FA Email: nếu 2FA gửi qua email nó không phải là something you have, nó vẫn là something you know (biết mật khẩu đăng nhập email). Nếu người dùng dùng chung 1 mật khẩu cho cả website và email, 2FA qua email trở nên vô dụng

## Two-factor authentication tokens
- Dùng RSA token cứng hoặc Google Authenticator (TOTP). Mã được sinh ra ngay trên thiết bị bằng thuật toán mã hóa và thời gian thực, không truyền qua mạng, khó bị đánh chặn trực tiếp
- Network transmission: dùng SMS OTP. Server tạo mã và gửi qua mạng viễn thông. Đây là chỗ hacker dễ dàng tấn công vì mạng viễn thông (đặc biệt là giao thức SS7) không bảo mật, ngoài ra còn có nguy cơ SIM Swapping từ đó có thể nhận tất cả các tin nhắn SMS gửi đến victim bao gồm cả tin nhắn chữa mã xác minh

## Bypassing two-factor authentication
**Forced Browsing**
- Nhập pass đúng -> đẩy sang trang nhập OTP -> nhập OTP đúng -> đẩy vào Dashboard. Nhưng ở backend ngay khi nhập đúng mật khẩu ở bước 1, hệ thống đã cho ta một session cookie hợp lệ có đầy đủ quyền. Việc bị chặn ở trang 2FA chỉ là hình thức. Nếu dùng Burp Suite hoặc tự gõ URL `/admin` trên thanh địa chỉ ta có thể qua bước kiểm tra OTP.

**[LAB 6](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)**
![image](https://hackmd.io/_uploads/S1jvyBuFbe.png)
![image](https://hackmd.io/_uploads/HJN5JButZx.png)
![image](https://hackmd.io/_uploads/S1DsJBut-x.png)
![image](https://hackmd.io/_uploads/rJUlgButZg.png)
![image](https://hackmd.io/_uploads/Sy6MeH_Y-e.png)
![image](https://hackmd.io/_uploads/rJy4gruYWl.png)
![image](https://hackmd.io/_uploads/Hkx8gH_Y-e.png)

## Flawed two-factor verification logic
- IDOR/Cookie tmpering: cookie account không được sign bằng mã MAC và không được mã hóa. Nó có thể bị giả mạo từ phía client
- Logic flaw: backend không ràng buộc phiên của bước 1 với bước 2. Nó tách rời 2 bước và chỉ tin vào Cookie truyền lên
- Thiếu rate limiting ở endpoint 2FA: tạo điều kiện để bruteforce mã PIN 4 chữ soos

**[LAB 7](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic)**
![image](https://hackmd.io/_uploads/S1IOXHOFZx.png)
![image](https://hackmd.io/_uploads/SypiQBOFZg.png)
![image](https://hackmd.io/_uploads/rJC67B_tbe.png)
![image](https://hackmd.io/_uploads/HyWf4SOYWg.png)
![image](https://hackmd.io/_uploads/HJ5IVS_K-x.png)
![image](https://hackmd.io/_uploads/Hks9EH_t-g.png)
![image](https://hackmd.io/_uploads/B1iGSSOtWg.png)
![image](https://hackmd.io/_uploads/S156HBOtbe.png)
![image](https://hackmd.io/_uploads/H1VtoBuKbe.png)

- Right click => Show response in browser => Copy URL

# Vulnerabilities in other authentication mechanisms
- Password reset: server tin tưởng vào HTTP Host header để tạo link reset. Ta đổi Host thành server của mình rồi yêu cầu reset pass của admin. Link reset gửi vào mail admin sẽ trỏ về server của ta
- IDOR khi đổi mật khẩu: ở form `Change Password` endpoint API lại nhận một tham số ẩn {`"user_id": 2, "new_pass": "123"}`. Đổi `user_id` thành `1` (admin) là xong.
- Token gửi qua email để reset pass không phải là chuỗi ngẫu nhiên mã hóa an toàn, mà chỉ là `MD5(username + timestamp)`. Hacer có thể tự tạo token này và chiếm quyền mà không cần đọc email của victim
- Chức năng `Đổi mật khẩu` không yêu cầu nhập mật khẩu hiện tại. Dẫn đến việc có thể khai thác qua CSRF hoặc XSS để ép victim tự đổi mật khẩu thành mật khẩu của hacker

## Keeping users logged in
- Tính năng `Remember me`: là một cookie có thời hạn dài giúp người dùng không phải nhập lại mật khẩu
- Một cookie `Remember me` mạnh phải là một chuỗi ngẫu nhiên (như UUID) được lưu trữ và đối chiếu trong DB. Nhưng dev lại hay dùng giải pháp không lưu DB bằng cách băm,mã hóa trực tiếp thông tin người dùng vào cookie
- Quy trình tấn công: register 1 acc -> chọn `Remember me` -> phân tích cookie trả về
- Encoding và encryption: Base64 là mã hóa định dạng => encoding), không phải là encryption, ai cũng có thể dịch ngược Base64 mà không cần khóa bí mật
- Sử dụng hash nhưng quên salt: Hàm băm (như `MD5, SHA-1`) là một chiều nhưng nếu băm chuỗi đơn giản như `admin:123456` mà không cộng thêm một chuỗi bí mật salt thì hacker chỉ cần tự băm một wordlist mật khẩu và đem đi so sánh là sẽ ra kết quả
- Các hệ thống có thể khóa IP nếu gõ sai pass 3 lần ở trang `/login` nhưng lại không giới hạn số lần gửi một cookie `Remember me` sai.










