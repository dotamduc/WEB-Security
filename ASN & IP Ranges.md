---
title: ASN & IP Ranges

---

# ASN & IP Ranges
Nhập Domain $\rightarrow$ Tự động tìm IP $\rightarrow$ Tra cứu ASN $\rightarrow$ Dump toàn bộ Subdomain nội bộ

`amass intel -asn <ASN> | tee domains.txt`

# Cào subdomain
`gospider -s https://target.com --subs -d 2 cat subs.txt | grep -F '[subdomains]' | cut -d'/' -f3 | sort -u | tee linked-subdomains.txt`

- `-s https://target.com`: chỉ định mục tiêu gốc

- `--subs`: báo cho gospider biết rằng ta đang cào subdomain -> gom chúng lại

- `-d 2`: depth (độ sâu) bằng 2. Nghĩa là tool sẽ bò từ trang chủ (depth 0) -> click vào link trên trang chủ (depth 1) -> click tiếp link trên trang con (depth 2) rồi dừng

- `grep -F '[subdomains]'`: lọc ra những dòng được tag là chứa subdomain

- `cut -d'/' -f3`: băm nhỏ URL dựa trên dấu `/` (ví dụ: https://api.target.com/path) và lấy cột thứ 3, tức là lấy được đúng cục `api.target.com`

- `sort -u`: sắp xếp và loại bỏ các subdomain trùng lặp

# Scraping (cào subdomain)
`subfinder -d gitbook.com -o scraped-subdomains.txt`
- `-d gitbook.com`: chỉ định tên miền mục tiêu cần tìm subdomain

- `-o scraped-subdomains.txt`: output danh sách subdomain tìm được và lưu vào tệp văn bản có tên `scraped-subdomains.txt`

# Brute force
- chuẩn bị Resolver)
```jsonld!
dnsvalidator -tL https://public-dns.info/nameservers.txt -threads 20 -o resolvers.txt
```
- tìm và lọc các DNS resolver công cộng đang hoạt động tốt (chạy 20 luồng) để phân tán request, tránh bị chặn

```jsonld!
puredns bruteforce best-dns-wordlist.txt gitbook.com -r resolvers.txt -w brute-subdomains.txt
```
- dùng `puredns` ghép các từ khóa trong `best-dns-wordlist.txt` với `gitbook.com`, đi qua các `resolvers.txt` trung gian để ép máy chủ nhả ra các subdomain đang tồn tại

# Confirming Status
```jsonld!
subfinder -d gitbook.com -silent | httpx -silent | tee online-subdomains.txt
```
- lấy output tĩnh, từ `subfinder` và đẩy thẳng (pipe) qua `httpx`
- `httpx` sẽ ping thử HTTP/HTTPS. Domain nào trả về phản hồi (tức là online) sẽ được lưu vào file `online-subdomains.txt`

# Port scan
## Masscan
```
sudo masscan 192.168.1.0/24 -p- --rate 100000 -oX out.xml
```
- Quét toàn bộ 65535 cổng

## Nmap
- quét TCP Stealth:
```jsonld!
sudo nmap -sV -O -sC -Pn -n -sS -T4 -oN nmap.txt -p- -vv 10.10.10.10
```

- quét UDP:
```jsonld!
sudo nmap -Pn -n -sV -sC -O -vv -oN nmap-udp.txt --top-ports 100 -sU --version-intensity 0 -T4 10.10.10.10
```

`-sV`: Dò phiên bản phần mềm (Service/Version detection).

`-O`: Dò hệ điều hành.

`-sC`: Chạy script mặc định an toàn (NSE).

`-Pn`: Không ping (coi như mục tiêu luôn sống để bypass block ping).

`-n`: Bỏ qua tra cứu DNS cho nhanh.

`-sS`: Stealth scan (TCP SYN), mở nửa kết nối để tránh ghi log.

`-sU`: Quét UDP (chậm nhưng hay có lọt lưới).

`-T4`: Tốc độ quét nhanh (từ T0 đến T5).

`-p-`: Quét full cổng. --top-ports 100: Quét 100 cổng phổ biến nhất.

`-vv`: Bật chế độ lắm mồm (verbose) để xem trực tiếp tiến trình.

`-oN`: Lưu kết quả dạng text thường.

# Gobuster
- Tìm admin panel, file backup, hay source code leak
```jsonld!
gobuster dir -u https://www.youtube.com -w FUZZ.txt -t 40 -o gobuster-dir.txt
```

# Vẫn là quét cổng
- quét lỗ hổng dựa trên template
```jsonld!
nuclei -target http://chat.rtaf.mi.th
```

- 
```jsonld!
nmap -sU -p3671 --script ./knx-gateway-info.nse 119.46.201.138
```
`-sU`: quét giao thức UDP

`-p3671`: chỉ định đích danh port 3671. (Đây là port mặc định của giao thức KNX - dùng trong hệ thống nhà thông minh / tự động hóa tòa nhà)

`--script ./knx-gateway-info.nse`: chạy một script Nmap (NSE) do người dùng tự viết/tải về để moi thông tin cụ thể từ con KNX Gateway này

`119.46.201.138`: IP mục tiêu 

```jsonld!
host yahoo.com
```
- Dùng để tra DNS

# Kiểm tra thông tin đăng ký
```jsonld!
whois tesla.com
```
- Truy vấn thông tin đăng ký của domain
    - nhả ra nhà đăng ký (registrar)
    - Name Servers
    - ngày tạo/hết hạn
    - **Email quản trị** 
# Đào bới DNS Records
```jsonld!
dig tesla.com
```
- Tìm IP chính - A record

```jsonld!
dig tesla.com ANY 
```
- Kéo toàn bộ bản ghi DNS

```jsonld!
nslookup -type=ns tesla.com
```
- Ép lấy riêng bản ghi Name Server để biết tổ chức đang dùng dịch vụ DNS nào (Cloudflare, AWS, v.v.)

# Cào subdomain siêu tốc
```jsonld!
subfinder -d tesla.com -silent

assetfinder --subs-only tesla.com
```

# Fuzzing
```jsonld!
ffuf -u https://www.tesla.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fc 404
```

# Nhận diện Công Nghệ & Framework
```jsonld!
whatweb https://www.tesla.com
```
- xuất CMS, Web Server, Framework đang chạy ở mục tiêu.

https://builtwith.com/

```jsonld!
httpx -l domains.txt -tech-detect -title -status-code -web-server
```
- Công nghệ web (`-tech-detect`)
- Tiêu đề trang (`-title`)
- Mã trạng thái (`-status-code`)
- Server Header, IP, CNAME và TLS info

# Cào vét URL & Tham số
```jsonld!
sudo ./gau mta.edu.vn > baby.txt
```
- cào toàn bộ lịch sử đường link của mục tiêu từng tồn tại trên Wayback Machine, AlienVault OTX, và Common Crawl, sau đó lưu vào baby.txt


```jsonld!
gau example.com | tee all_urls.txt
cat all_urls.txt | sort -u | httpx -silent -status-code -follow-redirects | grep -E "200|301|302|307|308" > alive.txt
```
1. Cào mọi URL lịch sử ném vào `all_urls.txt`
2. Đọc file (`cat`) và lọc trùng lặp (`sort -u`)
3. Ống pipe chuyển sang `httpx` để kiểm tra trạng thái, yêu cầu tool đi theo các chuyển hướng (`-follow-redirects`)
4. Dùng `grep -E` để chỉ nhặt lại những URL trả về mã sống hoặc chuyển hướng (`200, 301, 302, 307, 308`)
5. Lấy thành quả cuối cùng lưu vào `alive.txt`



































