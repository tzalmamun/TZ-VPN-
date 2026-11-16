# Ubuntu 24.04 VPS-এ 3X-UI Installation Guide (বাংলা)

**শুরু থেকে শেষ পর্যন্ত Setup, Firewall, SSH Security, IP-Based Admin Panel,
Subscription, VPN Inbound, DNS Fix এবং Troubleshooting Guide।**

এই গাইডটি নতুন Ubuntu 24.04 LTS VPS-এর জন্য। প্রতিটি ধাপ মনোযোগ দিয়ে অনুসরণ
করুন। VPS Provider, 3X-UI Version এবং VPN Protocol অনুযায়ী কিছু সেটিংস
আলাদা হতে পারে।

> **গুরুত্বপূর্ণ নিরাপত্তা নোট:** এই গাইডে IP-ভিত্তিক HTTP Panel ও
> Subscription-এর উদাহরণ দেওয়া হয়েছে। HTTP-তে Username, Password ও
> Subscription Data এনক্রিপ্টেড থাকে না। নিরাপত্তা গুরুত্বপূর্ণ হলে HTTPS/TLS
> ব্যবহার করুন অথবা Admin Panel-এ SSH Tunnel দিয়ে প্রবেশ করুন।

## সূচিপত্র

1.  VPS প্রস্তুতি
2.  SSH দিয়ে Login
3.  Ubuntu Update
4.  SSH Port 22022
5.  UFW ও Provider Firewall
6.  3X-UI Install
7.  Admin Panel
8.  Subscription Settings
9.  VPN Inbound
10. Client ও Subscription পরীক্ষা
11. DNS Fix
12. Troubleshooting
13. Management Commands
14. Backup ও Security
15. TZ VPN App
16. Final Checklist

------------------------------------------------------------------------

## ১. VPS প্রস্তুতি

VPS Provider থেকে Ubuntu 24.04 LTS 64-bit Server তৈরি করুন। Provider
Dashboard থেকে সংগ্রহ করুন:

  বিষয়                উদাহরণ
  ------------------- --------------------------------------
  Server IP           `YOUR_SERVER_IP`
  SSH Username        `root`
  Initial SSH Port    `22` (Provider অনুযায়ী ভিন্ন হতে পারে)
  নতুন SSH Port        `22022`
  Admin Panel Port    `2053` (উদাহরণ)
  Subscription Port   `2096` (উদাহরণ)
  VPN Inbound Port    নিজের Inbound অনুযায়ী

`YOUR_SERVER_IP`-এর জায়গায় নিজের VPS-এর Public IP বসাবেন।
`203.0.113.10`-এর মতো ডকুমেন্টেশন IP বাস্তব সার্ভারের IP নয়---নিজের IP
ব্যবহার করুন।

> নতুন করে OS Reinstall করলে পুরোনো সার্ভারের ডেটা মুছে যেতে পারে। প্রয়োজনীয়
> Backup আগে নিন।

## ২. SSH দিয়ে প্রথম Login

নিজের কম্পিউটারে Terminal / PowerShell খুলুন:

``` bash
ssh root@YOUR_SERVER_IP
```

Provider-এর SSH Port 22 না হলে:

``` bash
ssh -p INITIAL_SSH_PORT root@YOUR_SERVER_IP
```

Login-এর পর যাচাই করুন:

``` bash
whoami
hostname
hostname -I
```

`whoami`-এর ফলাফল `root` হওয়া উচিত।

## ৩. Ubuntu Update ও প্রয়োজনীয় Package

``` bash
apt update
apt -y full-upgrade
apt -y install curl wget nano sudo ufw fail2ban ca-certificates sqlite3
timedatectl set-timezone UTC
timedatectl
```

Kernel Update-এর কারণে Reboot প্রয়োজন হলে:

``` bash
reboot
```

Server চালু হলে আবার SSH দিয়ে Login করুন।

## ৪. SSH Port 22022 Configure

**সতর্কতা:** Provider Firewall-এ আগে `22022/TCP` Allow করুন। বর্তমান SSH
Session বন্ধ করবেন না। নতুন Port দিয়ে Login পরীক্ষা না করে পুরোনো Port বন্ধ
করবেন না।

প্রথমে UFW-তে নতুন ও পুরোনো SSH Port Allow করুন:

``` bash
ufw allow 22022/tcp
ufw allow 22/tcp
```

SSH Configuration Backup:

``` bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

Ubuntu-এর SSH drop-in directory-তে Port যোগ করুন:

``` bash
mkdir -p /etc/ssh/sshd_config.d
cat > /etc/ssh/sshd_config.d/00-custom-port.conf <<'EOF'
Port 22022
EOF
```

Configuration পরীক্ষা:

``` bash
sshd -t
```

কোনো Error না এলে:

``` bash
systemctl reload ssh
```

এখন বর্তমান Session চালু রেখেই নিজের কম্পিউটারে নতুন Terminal খুলুন:

``` bash
ssh -p 22022 root@YOUR_SERVER_IP
```

নতুন Port দিয়ে Login সফল হলে কার্যকর Port পরীক্ষা করুন:

``` bash
sshd -T | grep '^port '
```

> যদি এখানে 22 এবং 22022 দুটোই দেখা যায়, SSH এখনও দুটো Port-এ Listen করতে
> পারে। পুরোনো Port বন্ধ করতে চাইলে কার্যকর SSH Configuration যাচাই করে
> তারপর পরিবর্তন করুন। নতুন SSH Login নিশ্চিত না করে কখনো পুরোনো Port বন্ধ
> করবেন না।

নতুন Port দিয়ে Login নিশ্চিত হওয়ার পর Provider Firewall থেকে পুরোনো SSH
Port সরাতে পারেন। UFW-তে পুরোনো Port-এর Rule থাকলে:

``` bash
ufw delete allow 22/tcp
```

Provider-এর Initial SSH Port 22 না হলে সেই Port-ও যাচাই করুন।

## ৫. UFW Firewall Configure

SSH, Panel ও Subscription-এর উদাহরণ Port Allow করুন:

``` bash
ufw allow 22022/tcp
ufw allow 2053/tcp
ufw allow 2096/tcp
ufw default deny incoming
ufw default allow outgoing
ufw --force enable
ufw status verbose
```

এখানে `2053` ও `2096` এই গাইডের উদাহরণ। আপনার 3X-UI-তে Port আলাদা হলে
প্রকৃত Port ব্যবহার করুন।

### VPN Inbound Port

VPN Inbound তৈরি করার পর তার **সঠিক Port এবং Protocol** অনুযায়ী Rule যোগ
করুন।

TCP 443-এর উদাহরণ:

``` bash
ufw allow 443/tcp
```

UDP 443-এর উদাহরণ:

``` bash
ufw allow 443/udp
```

WireGuard যদি সত্যিই UDP 51820 ব্যবহার করে:

``` bash
ufw allow 51820/udp
```

**সতর্কতা:** `51820/UDP` সব VPN বা Xray Inbound-এর Port নয়। নিজের Inbound
Configuration না দেখে এই Port খুলবেন না।

### Provider Firewall / Security Group

VPS Provider Dashboard-এ Firewall / Security Group / Network Rules খুলুন।
UFW-তে যে Port ও Protocol খুলেছেন, Provider Firewall-এও প্রয়োজন অনুযায়ী
সেগুলো Allow করুন:

-   SSH: TCP `22022`
-   Admin Panel: TCP `2053` (অথবা প্রকৃত Panel Port)
-   Subscription: TCP `2096` (অথবা প্রকৃত Subscription Port)
-   VPN: নিজের Inbound-এর প্রকৃত Port ও Protocol

শুধু UFW-তে Port Allow করলেই Provider Firewall-এর Block উঠে যায় না।

## ৬. Official 3X-UI Installation

Official Project:

https://github.com/MHSanaei/3x-ui

Official Installer চালান:

``` bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Installer-এর নির্দেশনা অনুসরণ করুন। Version অনুযায়ী Menu বা প্রশ্ন বদলাতে
পারে।

-   শক্তিশালী Admin Username ও Password দিন।
-   Random এবং দীর্ঘ Web Base Path রাখুন।
-   Panel Port চাইলে `2053` দিতে পারেন, যদি Portটি ব্যবহারযোগ্য থাকে।
-   Installer যদি Random Port তৈরি করে, সেটি লিখে রাখুন।
-   Domain বা SSL ব্যবহার না করলে সেগুলো Configure করবেন না।
-   Installer শেষে দেখানো Panel URL, Username, Password ও Base Path
    নিরাপদে সংরক্ষণ করুন।

Service পরীক্ষা:

``` bash
systemctl status x-ui --no-pager
```

অথবা:

``` bash
x-ui status
```

## ৭. Admin Panel খুলুন

Installer-এর দেওয়া প্রকৃত Port ও Web Base Path ব্যবহার করুন:

``` text
http://YOUR_SERVER_IP:2053/YOUR_WEB_BASE_PATH/
```

উদাহরণ:

``` text
http://YOUR_SERVER_IP:2053/example-random-path/
```

Panel না খুললে পরীক্ষা করুন:

``` bash
x-ui status
ss -lntp
ufw status verbose
journalctl -u x-ui -n 100 --no-pager
```

Provider Firewall-এ Panel Port Allow আছে কি না যাচাই করুন।

**Security:** Public HTTP Panel-এ Login করলে Credentials এনক্রিপ্টেড থাকে
না। নিরাপদ বিকল্প হিসেবে SSH Tunnel ব্যবহার করুন (Security অংশ দেখুন)।

## ৮. Subscription Settings (IP Only)

3X-UI Panel-এ Login করে Subscription Settings খুলুন। Version অনুযায়ী
Field-এর নাম বা অবস্থান ভিন্ন হতে পারে।

যদি সংশ্লিষ্ট Field থাকে, উদাহরণস্বরূপ:

  Setting                 Value
  ----------------------- -------------------------------------
  Public Address / Host   `YOUR_SERVER_IP`
  Subscription Port       `2096`
  Scheme                  `http` (শুধু Plain HTTP Listener হলে)
  Domain                  খালি
  Reverse Proxy URL       খালি
  TLS Certificate         HTTP Setup-এ ব্যবহার হবে না

Host Field-এ শুধু IP লিখুন---`http://` বা `https://` নয়, যদি Field-টি শুধু
Host/IP চায়।

Save করার পর:

``` bash
x-ui restart
```

Subscription Listener চলছে কি না:

``` bash
ss -lntp | grep 2096
```

> আপনার 3X-UI Version-এ Public Address Field না থাকলে Database-এর অজানা
> Key অনুমান করে পরিবর্তন করবেন না। Version-সংশ্লিষ্ট Official Documentation
> যাচাই করুন।

**HTTP/HTTPS:** Plain HTTP Listener-এর URL `http://` দিয়ে শুরু হবে। Plain
HTTP Port-এ `https://` ব্যবহার করলে SSL Protocol Error হতে পারে। HTTPS
চাইলে বৈধ TLS Certificate ও HTTPS Listener/Reverse Proxy সঠিকভাবে
Configure করতে হবে।

## ৯. VPN Inbound তৈরি

3X-UI Panel থেকে **Inbounds → Add Inbound** নির্বাচন করুন। প্রয়োজন অনুযায়ী
Protocol বেছে নিন, যেমন:

-   VLESS
-   VMess
-   Trojan
-   Shadowsocks
-   WireGuard
-   Hysteria2

Inbound-এর Port, Transport, Security এবং অন্যান্য Option আপনার ব্যবহৃত
Protocol ও Client App অনুযায়ী Configure করুন।

উদাহরণ:

  Inbound        Port Protocol
  ----------- ------- --------------
  VLESS           443 TCP (উদাহরণ)
  WireGuard     51820 UDP (উদাহরণ)

এগুলো বাধ্যতামূলক মান নয়। আপনার Inbound-এ যে Port ও Protocol সেট করা আছে,
Firewall-এ সেটিই খুলবেন।

Listening Port পরীক্ষা:

``` bash
ss -lntup
```

### Subscription Port বনাম VPN Inbound Port

এ দুটি আলাদা:

-   **Subscription Port `2096`:** Client Configuration/Subscription আনার
    জন্য।
-   **VPN Inbound Port:** VPN Tunnel Connection-এর জন্য, যেমন আপনার
    Inbound অনুযায়ী `443/TCP` বা `51820/UDP`।

VPN App-এর Endpoint-এ Subscription Port বসাবেন না। Client App-এ
Subscription Import করলে App সাধারণত Configuration থেকে Inbound-এর তথ্য
নেয়। WireGuard Profile হলে Profile-এর Endpoint ও Peer Settings অনুসরণ
করুন।

## ১০. Client তৈরি ও Subscription পরীক্ষা

3X-UI থেকে Inbound-এর Client অংশে **Add Client** নির্বাচন করুন। প্রয়োজন
অনুযায়ী:

-   Client Name
-   UUID বা Protocol-সংশ্লিষ্ট Credentials
-   Traffic Limit
-   Expiry Date
-   Enable Status

সেট করে Save করুন।

Client-এর Subscription URL Panel থেকে Copy করুন। সাধারণ Format:

``` text
http://YOUR_SERVER_IP:2096/YOUR_SUBSCRIPTION_PATH/CLIENT_TOKEN
```

প্রতিটি Client-এর Token আলাদা রাখুন। Token কাউকে প্রকাশ করবেন না।

VPS-এর ভিতর থেকে পরীক্ষা:

``` bash
curl -i "http://127.0.0.1:2096/YOUR_SUBSCRIPTION_PATH/YOUR_CLIENT_TOKEN"
```

নিজের কম্পিউটার থেকে পরীক্ষা:

``` bash
curl -i "http://YOUR_SERVER_IP:2096/YOUR_SUBSCRIPTION_PATH/YOUR_CLIENT_TOKEN"
```

সাধারণত সফল Response-এ `HTTP 200` এবং Subscription Data থাকবে। শুধু Status
Code নয়, Response Body খালি কি না সেটিও দেখুন।

## ১১. DNS Fix

VPN Connected দেখালেও Internet না চললে Client App-এর DNS Settings-এ
পরীক্ষা হিসেবে নিচের DNS ব্যবহার করতে পারেন:

``` text
1.1.1.1
1.0.0.1
```

অথবা:

``` text
8.8.8.8
8.8.4.4
```

Client App-এ DNS ছাড়াও Routing, IPv4/IPv6 এবং DNS Routing Settings
পরীক্ষা করুন।

### VPS-এর DNS পরীক্ষা

``` bash
ping -c 3 1.1.1.1
getent hosts github.com
resolvectl status
ip route
```

-   IP Ping চলে কিন্তু `getent hosts github.com` ব্যর্থ হলে DNS সমস্যা থাকতে
    পারে।
-   IP Ping-ও ব্যর্থ হলে Gateway, Provider Network বা Routing সমস্যা হতে
    পারে।

### systemd-resolved DNS Fix

Ubuntu-তে `systemd-resolved` ব্যবহৃত হলে এবং Provider-এর নিজস্ব DNS প্রয়োজন
না হলে:

``` bash
mkdir -p /etc/systemd/resolved.conf.d
cat > /etc/systemd/resolved.conf.d/dns.conf <<'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=1.0.0.1 8.8.4.4
EOF
systemctl restart systemd-resolved
```

পরীক্ষা:

``` bash
resolvectl status
getent hosts github.com
```

Provider-এর Private DNS বা Custom Network থাকলে DNS পরিবর্তনের আগে
Provider-এর নির্দেশনা যাচাই করুন।

## ১২. Common Problems ও সমাধান

### সমস্যা ১: Admin Panel খুলছে না

সম্ভাব্য কারণ: ভুল IP/Port/Path, Service বন্ধ, UFW বা Provider Firewall
Block।

``` bash
x-ui status
ss -lntp
ufw status verbose
journalctl -u x-ui -n 100 --no-pager
```

Installer-এর প্রকৃত Panel Port ও Base Path ব্যবহার করছেন কি না যাচাই করুন।

### সমস্যা ২: Connection Timed Out

সম্ভাব্য কারণ: Provider Firewall Block, UFW Rule নেই, ভুল Port/Protocol।

``` bash
ufw status numbered
ss -lntup
```

Provider Firewall-এও একই Port ও Protocol Allow আছে কি না দেখুন।

### সমস্যা ৩: Connection Refused

সম্ভাব্য কারণ: Service বা Listener চালু নেই, ভুল Port, Inbound Disable।

``` bash
systemctl status x-ui --no-pager
x-ui restart
ss -lntup
journalctl -u x-ui -n 100 --no-pager
```

### সমস্যা ৪: VPN Connected কিন্তু Internet নেই

সম্ভাব্য কারণ:

-   Client DNS সমস্যা
-   Client Routing ভুল
-   Inbound Port/Protocol ভুল
-   Provider Firewall-এ UDP/TCP Block
-   Server IP Forwarding বা Routing সমস্যা
-   Protocol-specific Configuration ভুল

প্রথমে Client DNS `1.1.1.1` দিয়ে পরীক্ষা করুন। তারপর Inbound-এর
Port/Protocol, UFW, Provider Firewall এবং Log যাচাই করুন:

``` bash
journalctl -u x-ui -n 100 --no-pager
ss -lntup
ufw status verbose
```

সব Protocol-এর জন্য একই IP Forwarding বা NAT Command প্রযোজ্য নয়। Protocol
ও Network Design নিশ্চিত না হয়ে Kernel বা Firewall Rule পরিবর্তন করবেন না।

### সমস্যা ৫: Subscription কাজ করছে না

``` bash
ss -lntp | grep 2096
ufw status verbose
curl -i "http://127.0.0.1:2096/YOUR_PATH/YOUR_TOKEN"
journalctl -u x-ui -n 100 --no-pager
```

Subscription Service Enabled, Port সঠিক, Path/Token সঠিক এবং Client
Enabled কি না যাচাই করুন। External Test ব্যর্থ কিন্তু Local Test সফল হলে
Provider Firewall বা Network Path পরীক্ষা করুন।

### সমস্যা ৬: ERR_SSL_PROTOCOL_ERROR

Plain HTTP Listener হলে:

``` text
http://YOUR_SERVER_IP:2096/YOUR_PATH/YOUR_TOKEN
```

ব্যবহার করুন। Plain HTTP Port-এ `https://` দেবেন না। HTTPS-এর জন্য TLS
Certificate এবং HTTPS Listener প্রয়োজন।

### সমস্যা ৭: DNS Resolve হচ্ছে না

``` bash
ping -c 3 1.1.1.1
getent hosts google.com
resolvectl status
```

IP Connection কাজ করে কিন্তু Domain Resolve না হলে DNS/Resolver
Configuration পরীক্ষা করুন।

### সমস্যা ৮: SSH Login হচ্ছে না

Provider Web Console, VNC বা Serial Console দিয়ে Server-এ প্রবেশ করুন।

``` bash
sshd -t
systemctl status ssh --no-pager
ss -lntp | grep ssh
```

Backup থেকে আগের Configuration ফিরিয়ে আনতে হতে পারে। Drop-in File-এ ভুল
Port থাকলে সেটিও সংশোধন করুন। তারপর:

``` bash
sshd -t
systemctl restart ssh
```

SSH Port পরিবর্তনের সময় নতুন Port দিয়ে Login নিশ্চিত না করে পুরোনো Session
বন্ধ করবেন না।

### সমস্যা ৯: Service বারবার বন্ধ হচ্ছে

``` bash
systemctl status x-ui --no-pager
journalctl -u x-ui -n 200 --no-pager
free -h
df -h
```

Log-এর আসল Error শনাক্ত করুন। শুধু বারবার Restart করে মূল কারণ উপেক্ষা করবেন
না।

### সমস্যা ১০: No space left on device

``` bash
df -h
apt clean
apt autoremove -y
```

Database, Client Data বা Log না বুঝে Delete করবেন না।

## ১৩. দরকারি Management Commands

  কাজ               Command
  ----------------- ----------------------------------------
  Management Menu   `x-ui`
  Status            `x-ui status`
  Start             `x-ui start`
  Stop              `x-ui stop`
  Restart           `x-ui restart`
  Service Details   `systemctl status x-ui --no-pager`
  Service Log       `journalctl -u x-ui -n 100 --no-pager`
  Firewall          `ufw status verbose`
  Listening Ports   `ss -lntup`
  Memory            `free -h`
  Disk              `df -h`
  DNS               `resolvectl status`

## ১৪. Backup ও Security

### Database Backup

``` bash
mkdir -p /root/3x-ui-backup
cp -a /etc/x-ui/x-ui.db "/root/3x-ui-backup/x-ui-$(date +%F-%H%M%S).db"
```

Backup-এ Client Credentials, Token ও অন্যান্য গোপন তথ্য থাকতে পারে।
GitHub-এ Database, Backup বা Secret Upload করবেন না। Backup VPS-এর বাইরে
নিরাপদ স্থানে রাখুন।

### SSH Tunnel দিয়ে Admin Panel Access

Public HTTP Panel-এ Password এনক্রিপ্টেড থাকে না। SSH Tunnel একটি নিরাপদ
বিকল্প হতে পারে।

নিজের কম্পিউটার থেকে:

``` bash
ssh -p 22022 -L 2053:127.0.0.1:2053 root@YOUR_SERVER_IP
```

তারপর Browser-এ:

``` text
http://127.0.0.1:2053/YOUR_WEB_BASE_PATH/
```

এটি তখনই কাজ করবে যখন Panel `127.0.0.1`-এ Listen করছে বা Local Port
Forwarding গ্রহণ করছে। Panel যদি শুধু অন্য Interface-এ Bind করা থাকে,
Tunnel-এর Target Address সেই অনুযায়ী ঠিক করতে হবে। Tunnel পরীক্ষা না করে
Public Panel Access বন্ধ করবেন না।

**Subscription Security:** Plain HTTP Subscription Link-এ Subscription
Data এনক্রিপ্টেড নয়। নিরাপত্তা গুরুত্বপূর্ণ হলে HTTPS/TLS ব্যবহার করুন।
Subscription Token-কে Password-এর মতো গোপন রাখুন।

## ১৫. TZ VPN App

### আমাদের App ব্যবহার করুন

**TZ VPN -- সহজে VPN Configuration ব্যবহার করার জন্য আমাদের অ্যাপ।**

অ্যাপ ডাউনলোড:

**[Download TZ VPN
APK](https://github.com/tzalmamun/TZ-VPN-/releases/download/Apk/TZ.VPN_1.0.apk)**

GitHub Releases:

https://github.com/tzalmamun/TZ-VPN-/releases

**ব্যবহার করার আগে:** - APK কেবল বিশ্বস্ত উৎস থেকে ডাউনলোড করুন। -
Android-এর নিরাপত্তা সতর্কতা মনোযোগ দিয়ে পড়ুন। - নিজের 3X-UI Server থেকে
তৈরি করা বৈধ Client Configuration ব্যবহার করুন। - Subscription URL বা
Client Token অন্য কারও সঙ্গে শেয়ার করবেন না। - অ্যাপের বর্তমান Version ও
Compatibility Release Page-এ যাচাই করুন।

## ১৬. Final Checklist

-   [ ] Ubuntu 24.04 LTS Install হয়েছে।
-   [ ] SSH Login সফল।
-   [ ] SSH Port 22022 দিয়ে দ্বিতীয় Login পরীক্ষা হয়েছে।
-   [ ] Provider Firewall Configure হয়েছে।
-   [ ] UFW Firewall Configure হয়েছে।
-   [ ] Official 3X-UI Installer ব্যবহার হয়েছে।
-   [ ] Admin Username, Password ও Web Base Path নিরাপদে রাখা হয়েছে।
-   [ ] Admin Panel Access পরীক্ষা হয়েছে।
-   [ ] Subscription Service Enable এবং Port যাচাই হয়েছে।
-   [ ] VPN Inbound তৈরি হয়েছে।
-   [ ] Inbound-এর সঠিক Port ও Protocol Firewall-এ Allow হয়েছে।
-   [ ] Client তৈরি হয়েছে।
-   [ ] Subscription URL-এর HTTP Response ও Body পরীক্ষা হয়েছে।
-   [ ] DNS পরীক্ষা হয়েছে।
-   [ ] VPN Connection ও Internet পরীক্ষা হয়েছে।
-   [ ] Database Backup নিরাপদ স্থানে রাখা হয়েছে।
-   [ ] HTTP ব্যবহারের Security Risk বোঝা হয়েছে।

------------------------------------------------------------------------

## Official Documentation

-   3X-UI GitHub: https://github.com/MHSanaei/3x-ui
-   3X-UI Installation Guide:
    https://github.com/MHSanaei/3x-ui/wiki/Installation
-   Ubuntu Server Documentation: https://ubuntu.com/server/docs
-   Ubuntu UFW Help: https://help.ubuntu.com/community/UFW

------------------------------------------------------------------------

**Guide Version:** 1.0\
**Operating System:** Ubuntu 24.04 LTS\
**Panel:** MHSanaei 3X-UI\
**Installation Source:** Official 3X-UI GitHub Repository

**Disclaimer:** এই গাইড সাধারণ Setup নির্দেশনা। 3X-UI Version, VPS
Provider, Network এবং Inbound Protocol অনুযায়ী সেটিংস ভিন্ন হতে পারে। কোনো
সমস্যা হলে Error Log পরীক্ষা করুন। অনুমান করে Database, SSH বা Network
Configuration পরিবর্তন করবেন না।
