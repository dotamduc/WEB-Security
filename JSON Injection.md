---
title: JSON Injection

---

---
title: JSON Injection
---

# overview
- Cho phép attacker thay đổi hướng logic của truy vấn CSDL bằng cách inject các đối tượng JSON nguy hiểm thay vì dữ liệu thông thường => authentication bypass hoặc trích xuất dữ liệu

# cơ chế
- Thông thường khi một API đăng nhập sẽ thường trả về dữ liệu đầu vào dưới dạng JSON với cấu trúc đơn giản, trong đó các giá trị là string

Ví dụ
```jsonld
{
    "username": "user",
    "password": "pass"
}
```

# thực tế
- nếu backend không kiểm tra kỹ kiểu dữ liệu để đảm bảo `password` phải là một string, ta có thể thay thế nó bằng một JSON object
- Trong các CSDL NoSQL (mongodb), các JSON object này có thể chứa các toán tử logic đặc biệt

```jsonld
{
    "username": "admin",
    "password": {
        "$ne": "wrong"
    }
}
```

- `"$ne": "wrong"`: `$ne` là viết tắt của nt equal (Không bằng)
- truy vấn lúc này trửo thành: tìm người dùng có tên admin và mật khẩu không bằng `wrong`
=> điều kiện này trả về kết quả `true` => cho phép đăng nhập mà không cần biết mật khẩu

# Forcing JSON
- trong một số trường hợp, web có thể trông giống như chỉ nhận dữ liệu qua tham số URL hoặc form `application/x-www-form-urlencoded`

- Nếu thấy một request như sau:
```jsonld
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=user&password=pass
```
- Ta có thể ép server xử lý JSON bằng cách:
    - thay đổi Header: đổi `Content-Type` thành `application/json`
    - thay đổi body: chuyển đổi cú pháp từ `key=value` sang dạng JSON `{"key":"value"}`

- Request sau khi sửa: 

```jsonld
POST /login HTTP/1.1
Content-Type: application/json

{
    "username": "user",
    "password": "pass"
}
```

- nếu server trả về `200 OK` hoặc xử lý dữ liệu này tương tự với request gốc => forcing JSON thành công

# steps
- Bước 1: xác định target: tìm các điểm đầu vào như form đăng nhập, API search, API reset password

- Bước 2: thử forcing JSON: nếu request đang ở dạng URL parameters, thử chuyển sang JSON xem server có chấp nhận không

- Bước 3: inject toán tử NoSQL: thay đổi giá trị của trường nhạy cảm (như password) thành một object chứa toán tử

    - thử `$ne` (Not Equal): `{"$ne": "random_string"}` -> dùng để Bypass Login

    - Thử `$gt` (Greater Than): `{"$gt": ""}` -> Có thể dùng để liệt kê dữ liệu (blind NoSQLi)

- Bước 4: nếu đăng nhập thành công vào tài khoản admin mà không cần mật khẩu đúng, lỗ hổng đã được xác nhận