# Looking for `robots.xml` and `sitemap.xml` files
These files give a lot of clue about the website's directory structure

```bash
root@ip-10-112-103-4:~# curl -I $URL/robots.txt
HTTP/1.0 404 NOT FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 3424
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 09:23:12 GMT
```

```bash
curl -I $URL/sitemap.xml
HTTP/1.0 404 NOT FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 3424
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 09:23:17 GMT
```

**Conclusion**: 
- It's worth checking for the existence of these two files. It takes few seconds only.
- In this case, both files are not present on this web server/ website.

# Variations of `feroxbuster`
## Using API specific wordlist
As the website is using an API sort of application (i.e., `Werkzeug`), then use an API dedicated wordlist instead of spraying it with a large file (like `/usr/share/wordlists/dirb/big.txt`). This wastes time and sprays a lot of noise in the webserver logs.

Below is an example of using an API specific wordlist from `https://github.com/chrislockard/api_wordlist`:

```bash
# Start from the correct directory
cd $HOME/challenge/feroxbuster

# Download the API specific wordlist
git clone https://github.com/chrislockard/api_wordlist.git

cd api_wordlist

# Merge the files to make it easy to run the request once 
cat actions.txt actions-uppercase.txt actions-lowercase.txt api_seen_in_wild_paths.txt api_seen_in_wild.txt objects-uppercase.txt objects-lowercase.txt common_paths.txt objects.txt > all.txt

# Set the wordlist variable so it is used later
WORDLIST=$HOME/challenge/feroxbuster/api_wordlist/all.txt
```

## Paying attention to what is white/black listed

At first, let's create a word list fro scrapping the contents of the website:
```bash
cewl $URL -w list.txt  
cewl --lowercase $URL -w list_lowercase.txt  
cat list.txt list_lowercase.txt > all_lists.txt  
WORDLIST=/root/challenge/feroxbuster/all_lists.txt  
```

The interesting part starts here:

Using a non-existing page, identify the size of a `404` page. In this example, it is 3424:

```bash
curl -I $URL/test
HTTP/1.0 404 NOT FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 3424
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 10:00:38 GMT
```

Then use it with `feroxbuster`:
Notice:
- the "Status Codes" says "All Status Codes!"
- Size filter: 3424

The result? it finds a single item!

The "Status Codes" says "All Status Codes!" means that the tool will accept and display responses for **every HTTP status code** (such as `200`, `301`, `403`, `404`, `500`, etc.) rather than restricting the output to a specific allowlist.

```bash
./feroxbuster -u $URL -w $WORDLIST -S 3424 -d 3 -o challenge_site_scrap_1.txt  

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher                    ver: 2.13.1
───────────────────────────┬──────────────────────
     Target Url            │ http://10.112.191.197:5050/
     In-Scope Url          │ 10.112.191.197
     Threads               │ 50
     Wordlist              │ /root/challenge/feroxbuster/all_lists.txt
     Status Codes          │ All Status Codes!
     Timeout (secs)        │ 7
     User-Agent            │ feroxbuster/2.13.1
     Size Filter           │ 3424
     Extract Links         │ true
     Output File           │ challenge_site_scrap_1.txt
     HTTP methods          │ [GET]
     Recursion Depth       │ 3
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET       64l      325w     3424c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      370l      980w     9259c http://10.112.191.197:5050/
[####################] - 1s       159/159     0s      found:1       errors:0      
[####################] - 0s       159/159     413/s   http://10.112.191.197:5050/                               
```

Now do the same, except instead of using the `-S 3424`, use `--filter-similar-to $URL/dash`. The `/dash` doesn't exist either, so it will filter out pages that don't exist (in theory!)
Notice:
- "Status Codes" is "All Status Codes!"
- "Similarity Filter" is "http://10.112.191.197:5050/dash"

```bash
./feroxbuster -u $URL -w $WORDLIST --filter-similar-to $URL/dash -d 3 -o challenge_site_scrap_2.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher                    ver: 2.13.1
───────────────────────────┬──────────────────────
     Target Url            │ http://10.112.191.197:5050/
     In-Scope Url          │ 10.112.191.197
     Threads               │ 50
     Wordlist              │ /root/challenge/feroxbuster/all_lists.txt
     Status Codes          │ All Status Codes!
     Timeout (secs)        │ 7
     User-Agent            │ feroxbuster/2.13.1
     Similarity Filter     │ http://10.112.191.197:5050/dash
     Extract Links         │ true
     Output File           │ challenge_site_scrap_2.txt
     HTTP methods          │ [GET]
     Recursion Depth       │ 3
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET       64l      325w     3424c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      370l      980w     9259c http://10.112.191.197:5050/
[####################] - 1s       159/159     0s      found:1       errors:0      
[####################] - 0s       159/159     399/s   http://10.112.191.197:5050/                                
```

Do repeat and use the `-S 3424`, however, add specific status codes: `-s 200,301,302`
Notice:
- "Status Codes" is `[200, 301, 302]`

This time, we get a hit `http://10.112.191.197:5050/internal`:

```bash
./feroxbuster -u $URL -w $WORDLIST -S 3424 -s 200,301,302 -d 3 -o challenge_site_scrap_3.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher                    ver: 2.13.1
───────────────────────────┬──────────────────────
     Target Url            │ http://10.112.191.197:5050/
     In-Scope Url          │ 10.112.191.197
     Threads               │ 50
     Wordlist              │ /root/challenge/feroxbuster/all_lists.txt
     Status Codes          │ [200, 301, 302]
     Timeout (secs)        │ 7
     User-Agent            │ feroxbuster/2.13.1
     Size Filter           │ 3424
     Extract Links         │ true
     Output File           │ challenge_site_scrap_3.txt
     HTTP methods          │ [GET]
     Recursion Depth       │ 3
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET      370l      980w     9259c http://10.112.191.197:5050/
200      GET      264l      912w     8770c http://10.112.191.197:5050/internal
[####################] - 1s       160/160     0s      found:2       errors:0      
[####################] - 0s       159/159     398/s   http://10.112.191.197:5050/                                

```

The `-s 200,301,302` means: _only report responses with these status codes._ It’s a **positive filter** - a whitelist. Everything else is suppressed from output. This is why `/internal` suddenly surfaced: it was returning a 200 all along, but in the earlier runs it was presumably getting swept up by the `--filter-similar-to`/length logic, or buried. By explicitly saying “show me 200/301/302,” you cut the noise to just the interesting codes.

**Conclusions**: 
- Understand what the technology the website/web server is based on so you can use the correct wordlist.
- Understand what the tool does and how the different options behave so the correct one can be used.
- Don't assume the first option used gave the correct results and give up on that path. Utilise the other options as well.