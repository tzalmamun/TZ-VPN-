# Ubuntu VPS-এ 3X-UI ইনস্টলেশন গাইড (শুরু থেকে শেষ)

এই README-তে একটি নতুন Ubuntu 24.04 LTS VPS-এ SSH দিয়ে লগইন, প্রাথমিক সিকিউরিটি, SSH port `22022`, Firewall, 3X-UI ইনস্টলেশন, IP-ভিত্তিক Admin Panel ও Subscription সেটআপ এবং সাধারণ Error সমাধানের ধাপ দেওয়া হয়েছে।

**এই গাইডের লক্ষ্য:**
- কোনো Domain ব্যবহার করা হবে না।
- Admin Panel সরাসরি VPS-এর Public IP দিয়ে খোলা হবে।
- Client Subscription URL-এও VPS-এর Public IP থাকবে।
- Nginx, Certbot বা Domain SSL প্রয়োজন হবে না।
- প্রতিটি Client-এর নিজস্ব Subscription Token থাকবে।

> **গুরুত্বপূর্ণ:** কমান্ডগুলো নতুন Ubuntu 24.04 VPS-এর জন্য। আগে থেকে ব্যবহৃত সার্ভারে চালালে বিদ্যমান কনফিগারেশন নষ্ট হতে পারে। নতুন VPS হলে Provider Dashboard থেকে Ubuntu 24.04 LTS Reinstall করুন। Reinstall করলে সার্ভারের পুরোনো ডেটা মুছে যাবে।

---

## সূচিপত্র

1. VPS প্রস্তুতি
2. প্রয়োজনীয় তথ্য
3. SSH দিয়ে প্রথম Login
4. Ubuntu Update
5. SSH Port `22022` সেট করা
6. Firewall চালু করা
7. 3X-UI Install
8. Admin Panel Settings
9. IP দিয়ে Admin Panel খোলা
10. Subscription Settings
11. Client তৈরি ও Subscription পরীক্ষা
12. দরকারি Management Commands
13. Error Troubleshooting
14. Backup ও Security
15. Final Checklist

---

## ১. VPS প্রস্তুতি

VPS Provider থেকে নতুন Server তৈরি করুন।

প্রস্তাবিত:
- OS: Ubuntu 24.04 LTS 64-bit
- RAM: প্রয়োজন অনুযায়ী (ছোট পরীক্ষার জন্য 1–2 GB হতে পারে; ব্যবহারকারীর সংখ্যা ও ট্রাফিক অনুযায়ী বেশি লাগতে পারে)
- Root access
- Public IPv4 address

Provider Dashboard থেকে নিচের তথ্য সংগ্রহ করুন:

| তথ্য | উদাহরণ |
|---|---|
| Server IP | `YOUR_SERVER_IP` |
| Initial SSH port | `22` (Provider অনুযায়ী আলাদা হতে পারে) |
| SSH username | `root` |
| New SSH port | `22022` |
| Admin Panel port | `2053` |
| Subscription port | `2096` |

**Provider Firewall / Security Group-এ** শুরুতেই Initial SSH port অনুমোদিত রাখুন। পরে `22022` যোগ করবেন। Panel, Subscription এবং Xray Inbound-এর প্রয়োজনীয় TCP/UDP port-ও Provider Firewall-এ খুলতে হবে।

---

## ২. নিজের তথ্য বসান

এই গাইডে `YOUR_SERVER_IP` লেখা থাকলে সেটির জায়গায় নিজের VPS-এর Public IP বসাবেন।

উদাহরণ:
```text
YOUR_SERVER_IP = 203.0.113.10
```

`203.0.113.10` শুধু উদাহরণ IP; নিজের সার্ভারের IP ব্যবহার করবেন।

---

## ৩. SSH দিয়ে প্রথম Login

নিজের কম্পিউটারের Terminal / PowerShell / Windows Terminal খুলুন:

```bash
ssh root@YOUR_SERVER_IP
```

যদি Provider অন্য SSH port দিয়ে থাকে:

```bash
ssh -p INITIAL_SSH_PORT root@YOUR_SERVER_IP
```

প্রথমবার সংযোগ করলে host authenticity confirmation এলে যাচাই করে `yes` লিখুন। এরপর VPS-এর root password দিন। Password টাইপ করার সময় স্ক্রিনে অক্ষর দেখা যাবে না—এটাই স্বাভাবিক।

Login সফল হলে সাধারণত এমন Prompt দেখবেন:

```text
root@server:~#
```

Server-এর তথ্য যাচাই:

```bash
whoami
hostname
hostname -I
ip -4 addr
```

`whoami`-এর ফলাফল `root` হওয়া উচিত।

---

## ৪. Ubuntu Update এবং প্রয়োজনীয় Package

```bash
apt update
apt -y full-upgrade
apt -y install curl wget ca-certificates gnupg nano sudo ufw fail2ban
```

সময় UTC করা:

```bash
timedatectl set-timezone UTC
timedatectl
```

Kernel update হয়ে থাকলে:

```bash
reboot
```

Reboot-এর পর আবার Initial SSH port দিয়ে Login করুন।

---

## ৫. SSH Port `22022` সেট করা

**সতর্কতা:** এই ধাপ ভুল হলে SSH Login বন্ধ হয়ে যেতে পারে। বর্তমান SSH Session বন্ধ করবেন না। Provider Firewall-এ আগে `22022/TCP` অনুমোদন করুন।

প্রথমে UFW-তে নতুন SSH port অনুমোদন:

```bash
ufw allow 22022/tcp
```

SSH configuration backup:

```bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

নতুন SSH configuration drop-in তৈরি:

```bash
mkdir -p /etc/ssh/sshd_config.d

cat >/etc/ssh/sshd_config.d/99-custom-port.conf <<'EOF'
Port 22022
EOF
```

Configuration পরীক্ষা:

```bash
sshd -t
```

কোনো Error না এলে SSH service reload করুন:

```bash
systemctl reload ssh
```

এখন **বর্তমান Session বন্ধ করবেন না**। নিজের কম্পিউটারে নতুন Terminal খুলে দ্বিতীয় Login পরীক্ষা করুন:

```bash
ssh -p 22022 root@YOUR_SERVER_IP
```

দ্বিতীয় Login সফল হলেই নতুন SSH port কাজ করছে।

> Ubuntu-তে `/etc/ssh/sshd_config`-এ আগে থেকেই `Port 22` সক্রিয় থাকলে সেটিও কার্যকর থাকতে পারে। `sshd -T | grep '^port '` দিয়ে কার্যকর Port দেখুন। নতুন Port নিশ্চিত না করে পুরোনো SSH port বন্ধ করবেন না।

কার্যকর SSH Port দেখুন:

```bash
sshd -T | grep '^port '
```

দ্বিতীয় Login সফল হওয়ার পর Provider Firewall থেকে Initial SSH port বন্ধ করতে পারেন। UFW-তে আগে থেকে `22/tcp` অনুমতি দিয়ে থাকলে এবং আর প্রয়োজন না হলে:

```bash
ufw delete allow 22/tcp
```

Provider-এর Initial SSH port `22` না হলে সেই port-এর UFW rule-ও যাচাই করে সরান।

---

## ৬. Firewall (UFW) Configure

প্রথমে SSH port অনুমোদন করুন:

```bash
ufw allow 22022/tcp
```

Admin Panel port:

```bash
ufw allow 2053/tcp
```

Subscription port:

```bash
ufw allow 2096/tcp
```

Default Firewall policy:

```bash
ufw default deny incoming
ufw default allow outgoing
```

Firewall চালু:

```bash
ufw --force enable
```

Status:

```bash
ufw status verbose
```

**গুরুত্বপূর্ণ:** Xray Inbound যে Port ও Protocol (TCP/UDP) ব্যবহার করবে, সেগুলোও UFW এবং Provider Firewall—দুই জায়গাতেই অনুমোদন করতে হবে।

উদাহরণস্বরূপ, যদি আপনার Inbound সত্যিই TCP `443` ব্যবহার করে:

```bash
ufw allow 443/tcp
```

যদি UDP `443`-ও প্রয়োজন হয়:

```bash
ufw allow 443/udp
```

নিজের Inbound configuration না দেখে উদাহরণের Port খুলবেন না।

---

## ৭. 3X-UI Install

শুধু Official Project-এর Installer ব্যবহার করুন:

Project:
https://github.com/MHSanaei/3x-ui

Official installation documentation:
https://github.com/MHSanaei/3x-ui/blob/main/docs/content/docs/en/guide/installation.mdx

Installer চালান:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Installer-এর নির্দেশনা অনুসরণ করুন। Version অনুযায়ী প্রশ্ন বা Menu পরিবর্তিত হতে পারে।

- Panel port হিসেবে `2053` ব্যবহার করতে পারেন।
- Random ও শক্তিশালী Web Base Path রাখুন।
- শক্তিশালী Admin Username ও Password দিন।
- কোনো Domain সেট করবেন না।
- Domain SSL Certificate তৈরি করবেন না।
- Installer-এর শেষে দেখানো Panel URL, Username, Password ও Web Base Path নিরাপদ জায়গায় লিখে রাখুন।

Install শেষে Service status:

```bash
systemctl status x-ui --no-pager
```

অথবা:

```bash
x-ui status
```

---

## ৮. Admin Panel Settings

Management Menu খুলুন:

```bash
x-ui
```

Menu-তে Panel Settings / Configuration অপশন নির্বাচন করুন। Menu নম্বর Version অনুযায়ী আলাদা হতে পারে।

যা সেট করবেন:

| Setting | Value |
|---|---|
| Panel Port | `2053` |
| Web Base Path | Random / দীর্ঘ Path |
| Admin Username | নিজের শক্তিশালী Username |
| Admin Password | নিজের শক্তিশালী Password |
| Domain | খালি / ব্যবহার করবেন না |
| Domain SSL | ব্যবহার করবেন না |

Panel URL-এর ধরন:

```text
http://YOUR_SERVER_IP:2053/YOUR_RANDOM_PANEL_PATH/
```

এখানে `YOUR_RANDOM_PANEL_PATH`-এর জায়গায় Installer বা Panel-এ দেখানো আসল Path বসাবেন।

**মনে রাখবেন:** Admin Panel-এর Path এবং Subscription Path এক জিনিস নয়।

---

## ৯. IP দিয়ে Admin Panel খোলা

Browser-এ খুলুন:

```text
http://YOUR_SERVER_IP:2053/YOUR_RANDOM_PANEL_PATH/
```

Login না হলে পরীক্ষা করুন:

```bash
systemctl status x-ui --no-pager
ss -lntp | grep 2053
ufw status verbose
```

Provider Firewall-এ `2053/TCP` অনুমোদিত কি না দেখুন।

---

## ১০. Subscription Settings (IP Only)

3X-UI Admin Panel-এ Login করুন।

**Subscription Settings** খুলুন। Version অনুযায়ী Setting-এর নাম কিছুটা ভিন্ন হতে পারে।

যদি Public Address / Subscription Domain / Host নামে Field থাকে:

| Setting | Value |
|---|---|
| Public Address / Host | `YOUR_SERVER_IP` |
| Subscription Port | `2096` |
| Subscription Path | Panel-এর তৈরি Random Path |
| Scheme | Plain HTTP হলে `http` |
| Domain | খালি |
| Reverse Proxy URL | খালি |
| Domain Certificate | ব্যবহার করবেন না |

**Host Field-এ শুধু IP লিখবেন।** `http://` বা `https://` যোগ করবেন না, যদি Field-টি শুধু Host/IP নেওয়ার জন্য হয়।

যদি আপনার Version-এ Subscription-এর Public Address Field না থাকে, Database-এ অনুমান করে Key পরিবর্তন করবেন না। আগে ওই Version-এর Settings UI এবং Official Documentation যাচাই করুন।

Save করার পর প্রয়োজন হলে:

```bash
x-ui restart
```

**HTTP / HTTPS বিষয়ে:**
- Subscription listener যদি Plain HTTP হয়, URL হবে `http://...`
- Plain HTTP listener-এ `https://...` ব্যবহার করলে `ERR_SSL_PROTOCOL_ERROR` হতে পারে।
- HTTPS ব্যবহার করতে হলে সঠিক TLS Certificate ও HTTPS Listener/Reverse Proxy আলাদাভাবে configure করতে হবে।
- এই README-এর IP-only সহজ Setup-এ Domain SSL বা Nginx Reverse Proxy ব্যবহার করা হচ্ছে না।

---

## ১১. Client তৈরি ও Subscription পরীক্ষা

3X-UI থেকে একটি Test Client তৈরি করুন। Client-এর Subscription URL Copy করুন।

IP-only URL-এর প্রত্যাশিত Format:

```text
http://YOUR_SERVER_IP:2096/YOUR_SUBSCRIPTION_PATH/CLIENT_TOKEN
```

এখানে:
- `YOUR_SERVER_IP` = VPS-এর Public IP
- `YOUR_SUBSCRIPTION_PATH` = Panel-এ তৈরি Subscription Path
- `CLIENT_TOKEN` = ওই নির্দিষ্ট Client-এর Token

প্রতিটি Client-এর Token আলাদা থাকবে। একটি Token সব Client-এর জন্য ব্যবহার করবেন না।

### VPS-এর ভিতর থেকে Test

নিজের প্রকৃত Subscription Path ও Token বসিয়ে চালান:

```bash
curl -i "http://127.0.0.1:2096/YOUR_SUBSCRIPTION_PATH/YOUR_CLIENT_TOKEN"
```

### নিজের Computer থেকে External Test

```bash
curl -i "http://YOUR_SERVER_IP:2096/YOUR_SUBSCRIPTION_PATH/YOUR_CLIENT_TOKEN"
```

সাধারণত সফল Response-এ `HTTP/1.1 200 OK` দেখা যায় এবং Response Body-তে Subscription Data থাকে।

শুধু HTTP 200 হলেই যথেষ্ট নয়—Body খালি কি না তাও যাচাই করুন।

---

## ১২. দরকারি 3X-UI Management Commands

```bash
x-ui
x-ui status
x-ui start
x-ui stop
x-ui restart
systemctl status x-ui --no-pager
journalctl -u x-ui -n 100 --no-pager
```

Listening Port দেখুন:

```bash
ss -lntup
```

Firewall দেখুন:

```bash
ufw status numbered
ufw status verbose
```

Disk ও Memory:

```bash
df -h
free -h
```

CPU ও Process:

```bash
top
```

---

## ১৩. Common Errors এবং সমাধান

### Error 1: `Connection refused`

সম্ভাব্য কারণ:
- 3X-UI Service বন্ধ।
- ভুল Port ব্যবহার করা হয়েছে।
- Panel বা Subscription Listener চালু নেই।

পরীক্ষা:

```bash
systemctl status x-ui --no-pager
x-ui status
ss -lntup
journalctl -u x-ui -n 100 --no-pager
```

প্রয়োজনে:

```bash
x-ui restart
```

### Error 2: `Connection timed out`

সম্ভাব্য কারণ:
- Provider Firewall-এ Port বন্ধ।
- UFW-তে Port অনুমোদিত নয়।
- ভুল IP বা Port ব্যবহার করা হচ্ছে।

পরীক্ষা:

```bash
ufw status verbose
ss -lntup
```

Provider Dashboard-এর Firewall/Security Group-ও পরীক্ষা করুন।

### Error 3: `ERR_SSL_PROTOCOL_ERROR`

সম্ভাব্য কারণ:
- Plain HTTP Listener-এ HTTPS দিয়ে প্রবেশ করা হচ্ছে।
- ভুল Port-এ HTTPS Request পাঠানো হচ্ছে।
- TLS Listener সঠিকভাবে configure করা নেই।

Plain HTTP Subscription হলে:

```text
http://YOUR_SERVER_IP:2096/YOUR_SUBSCRIPTION_PATH/YOUR_CLIENT_TOKEN
```

ব্যবহার করুন। `https://` দিয়ে পরীক্ষা করবেন না।

### Error 4: Subscription URL-এ Domain দেখা যায়

সম্ভাব্য কারণ:
- Subscription Settings-এর Public Host/Address-এ Domain রয়ে গেছে।
- Custom Subscription URL বা Reverse Proxy URL সেট করা আছে।

সমাধান:
1. Subscription Settings খুলুন।
2. Public Host/Address-এ VPS-এর Numeric IP দিন।
3. Domain/Reverse Proxy URL খালি রাখুন।
4. Save করুন।
5. প্রয়োজনে `x-ui restart` করুন।
6. নতুন করে Client Subscription URL Copy করে পরীক্ষা করুন।

Database-এর অজানা Key অনুমান করে পরিবর্তন করবেন না।

### Error 5: Subscription খুললে Blank Page

সম্ভাব্য কারণ:
- ভুল Subscription Path বা Token।
- Response Body খালি।
- Proxy Rewrite বা Listener configuration ভুল।
- Client-এর Subscription Data তৈরি হয়নি।

Test:

```bash
curl -i "http://127.0.0.1:2096/YOUR_SUBSCRIPTION_PATH/YOUR_CLIENT_TOKEN"
```

Service Log:

```bash
journalctl -u x-ui -n 100 --no-pager
```

URL-এর Path ও Token Panel থেকে আবার Copy করে পরীক্ষা করুন।

### Error 6: `sqlite3: command not found`

Ubuntu-তে SQLite CLI না থাকলে:

```bash
apt update
apt install -y sqlite3
```

শুধু Database দেখতে চাইলে Python-ও ব্যবহার করা যায়:

```bash
python3 - <<'PY'
import sqlite3
db = "/etc/x-ui/x-ui.db"
con = sqlite3.connect(db)
for (table,) in con.execute("SELECT name FROM sqlite_master WHERE type='table'"):
    print("TABLE:", table)
    for row in con.execute(f'PRAGMA table_info("{table}")'):
        print(row)
con.close()
PY
```

Database পরিবর্তনের আগে Backup নিন এবং সঠিক Schema নিশ্চিত করুন।

### Error 7: SSH Login হচ্ছে না

সম্ভাব্য কারণ:
- ভুল IP বা Port।
- Provider Firewall-এ SSH Port বন্ধ।
- SSH Configuration ভুল।
- SSH Service বন্ধ।

Provider Console/VNC/Serial Console দিয়ে VPS-এ প্রবেশ করে পরীক্ষা করুন:

```bash
sshd -t
systemctl status ssh --no-pager
ss -lntp | grep ssh
```

SSH Config ঠিক থাকলে:

```bash
systemctl restart ssh
```

SSH Port পরিবর্তনের সময় পুরোনো Session বন্ধ করার আগে নতুন Port দিয়ে Login পরীক্ষা করা বাধ্যতামূলক।

### Error 8: `Could not resolve host`

সম্ভাব্য কারণ:
- VPS-এর DNS সমস্যা।
- Network/Gateway সমস্যা।
- Provider-এর Network সমস্যা।

পরীক্ষা:

```bash
ip route
cat /etc/resolv.conf
ping -c 3 1.1.1.1
getent hosts github.com
```

IP Ping কাজ করে কিন্তু Domain Resolve না হলে DNS configuration পরীক্ষা করুন।

### Error 9: `No space left on device`

পরীক্ষা:

```bash
df -h
du -sh /var/log/* 2>/dev/null
```

অপ্রয়োজনীয় Package Cache পরিষ্কার:

```bash
apt clean
apt autoremove -y
```

Log বা Database না বুঝে Delete করবেন না।

### Error 10: Service বারবার বন্ধ হয়ে যায়

```bash
systemctl status x-ui --no-pager
journalctl -u x-ui -n 200 --no-pager
free -h
df -h
```

Log-এ যে Error দেখাবে সেটি অনুযায়ী সমাধান করতে হবে। শুধু Restart করে প্রকৃত কারণ উপেক্ষা করবেন না।

---

## ১৪. Backup এবং Security

### 3X-UI Database Backup

```bash
mkdir -p /root/3x-ui-backup
cp -a /etc/x-ui/x-ui.db "/root/3x-ui-backup/x-ui-$(date +%F-%H%M%S).db"
```

Backup-এ Client Configuration, Token বা Sensitive Information থাকতে পারে। GitHub-এ কখনো Database বা Backup Upload করবেন না।

### Admin Panel Security

IP-only Plain HTTP Admin Panel ইন্টারনেটে প্রকাশ করলে Username ও Password এনক্রিপ্টেড থাকবে না। নিরাপত্তার জন্য Panel Port নিজের নির্দিষ্ট Public IP-তে সীমাবদ্ধ করুন, অথবা SSH Tunnel ব্যবহার করুন।

নিজের Computer থেকে SSH Tunnel:

```bash
ssh -p 22022 -L 2053:127.0.0.1:2053 root@YOUR_SERVER_IP
```

তারপর Browser-এ খুলুন:

```text
http://127.0.0.1:2053/YOUR_RANDOM_PANEL_PATH/
```

SSH Tunnel ব্যবহার করলে Provider Firewall এবং UFW থেকে Public `2053/TCP` Access বন্ধ করতে পারেন। তবে SSH `22022/TCP` অবশ্যই খোলা রাখতে হবে।

**মনে রাখবেন:** Subscription URL যদি Plain HTTP হয়, Subscription Data-ও এনক্রিপ্টেড নয়। নিরাপত্তা গুরুত্বপূর্ণ হলে TLS-সহ সঠিক HTTPS Setup ব্যবহার করুন।

---

## ১৫. Final Checklist

- [ ] Ubuntu 24.04 LTS নতুন করে Install করা হয়েছে।
- [ ] SSH Login সফল।
- [ ] SSH Port `22022` দিয়ে দ্বিতীয় Login পরীক্ষা করা হয়েছে।
- [ ] Provider Firewall-এ প্রয়োজনীয় Port খোলা।
- [ ] UFW সঠিকভাবে Configure করা হয়েছে।
- [ ] 3X-UI Official Installer দিয়ে Install হয়েছে।
- [ ] Admin Panel IP দিয়ে খোলা যাচ্ছে।
- [ ] Admin Panel-এর Random Web Base Path রাখা হয়েছে।
- [ ] কোনো Domain Configure করা হয়নি।
- [ ] Subscription Public Host-এ VPS-এর IP দেওয়া হয়েছে।
- [ ] Subscription Port `2096` Listener চালু।
- [ ] Client Subscription URL-এ Domain নয়, IP দেখা যাচ্ছে।
- [ ] প্রতিটি Client-এর আলাদা Token আছে।
- [ ] Subscription URL HTTP 200 এবং Non-empty Body দিচ্ছে।
- [ ] Xray Inbound-এর প্রয়োজনীয় TCP/UDP Port Firewall-এ অনুমোদিত।
- [ ] Database Backup নিরাপদ স্থানে রাখা হয়েছে।

---

## Official Links
App For Use https://github.com/tzalmamun/TZ-VPN-/releases/download/Apk/TZ.VPN_1.0.apk

- 3X-UI GitHub: https://github.com/MHSanaei/3x-ui
- Official Installation Guide: https://github.com/MHSanaei/3x-ui/blob/main/docs/content/docs/en/guide/installation.mdx
- Ubuntu Server Documentation: https://ubuntu.com/server/docs
- UFW Documentation: https://help.ubuntu.com/community/UFW

---

**লেখক/রক্ষণাবেক্ষণ:** নিজের VPS, 3X-UI Version এবং Provider Firewall অনুযায়ী Port ও Settings যাচাই করে ব্যবহার করুন। 3X-UI-এর Menu এবং Subscription Settings Version অনুযায়ী পরিবর্তিত হতে পারে। কোনো Error হলে সম্পূর্ণ Error Log দেখে সমাধান করুন; অনুমান করে Database বা System File পরিবর্তন করবেন না।
