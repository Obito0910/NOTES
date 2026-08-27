# Byte Lotus Hotel — Do Not Disturb — Detailed Notes

**Challenge:** Hacker Holidays · The Byte Lotus Hotel · Do Not Disturb **Category:** Boot2Root (Web) **Difficulty:** Medium **Solved by:** Obito Uchiha — Team AKATSUKI

---

## 1. What This Challenge Is About (Topic Overview)

This is a **web-based boot2root** machine — meaning we have to attack a website, use that to get inside the server, and then escalate privileges until we reach `root` (the most powerful user on the system).

Think of it like this: there's a hotel with an **online booking portal**. Inside that portal:

- There's a **login page** (guests or staff can sign in)
- Staff have access to a **special panel** where they can customize a guest confirmation message
- There are **hidden background programs** running on the server that we can't see directly from the website, but that can still be exploited

Our job: find the weaknesses in each of these pieces, one at a time, and chain them together to eventually reach root.

---

## 2. Technologies Table

|Technology|What It Is|Role in This Challenge|
|---|---|---|
|**Express.js**|A Node.js web framework — similar to how Laravel is for PHP, Express is for JavaScript/Node|The entire web server is built on it|
|**NeDB (@seald-io/nedb)**|A lightweight, file-based/in-memory database — like a smaller version of MongoDB|Stores users (username/password/role), uses MongoDB-style query syntax|
|**EJS (Embedded JavaScript)**|A templating engine — lets you embed JavaScript inside HTML (e.g. `<%= variable %>`) to generate dynamic content|Used by the staff panel to insert the guest's name into a confirmation message|
|**express-session**|Session management library — creates a cookie-based session to remember a logged-in user|Issues the `connect.sid` cookie after login|
|**Node.js Inspector (`--inspect`)**|Node's built-in debugging tool — developers use it to step through code, connecting via browser DevTools|A background service (`processor.js`) was running in this debug mode, which we exploited|
|**debugfs**|A Linux tool that reads ext2/ext3/ext4 filesystems **directly from the raw disk**, without mounting it|Used to read the root filesystem directly, bypassing normal permission checks|

---

## 3. Vulnerabilities Table (What We Found, How It Works)

|#|Vulnerability|Simple Explanation (Real-Life Analogy)|
|---|---|---|
|1|**NoSQL Injection**|Imagine asking a security guard "is the password not null?" — the guard just says "yes" and lets you in without actually checking the real password, because you phrased the question so its answer is always "yes"|
|2|**SSTI (Server-Side Template Injection)**|This is like a restaurant letting you write your own order slip, then handing it straight to the kitchen robot that blindly follows every instruction on it — if you write "burn down the kitchen" on the slip, it will do it, because it has no idea what's safe and what isn't|
|3|**Exposed Debugger (Node Inspector)**|Like leaving a factory's control panel unlocked — anyone who gets inside the factory can walk up to that panel and directly control the machines, with no authorization needed|
|4|**`disk` group privilege escalation**|Imagine you should only have the key to your own apartment, but by mistake you were given the **building's master maintenance key**. With that key you can bypass any apartment's door and look inside, without ever having that apartment's actual key|

---

## 4. Beginner-to-Challenge Progression, Step by Step

Let's assume you know nothing, and walk through exactly how we got here, step by step — as if explaining it to a younger sibling with no prior knowledge.

### Step A: First, see what the website actually is (Recon)

Whenever attacking a website, the first thing to do is "explore" it — like walking into a new house and checking how many rooms there are and which doors are open.

- **`nmap`** is a tool that shows which "doors" (ports) are open on the server. We found: port 22 (SSH) and port 80 (website).
- **`whatweb`** revealed the site was built on Node.js/Express.
- **`ffuf`** (a directory fuzzer) was used to find hidden pages — like searching a house for hidden rooms. We found `/staff` and `/logout`.

### Step B: Looked at and understood the login form

The website had a login form (username + password). We looked at the page's **source code** — like flipping a form over to read what's written on the back. This revealed the form submits to `/login`.

### Step C: Broke the first lock (NoSQL Injection)

We tried logging in normally with fake credentials — it failed, as expected.

Then we reasoned: if the backend database is **MongoDB-like** (NoSQL), normal SQL-style tricks (`' OR '1'='1`) won't work, because that syntax belongs to a different "language" (SQL). NoSQL databases have their own special "operators" like `$ne` (not equal), `$gt` (greater than), etc. We sent, in the password field:

```json
{"password": {"$ne": null}}
```

This means _"find a password that is not null"_ — since a real password is never NULL, this condition is always TRUE, and we logged in **without ever knowing the actual password**.

It's the same as asking a security guard "is the guest list not empty?" — obviously it isn't, so the guard says "correct" and lets you in, without ever checking whether you're actually on that list.

**Result:** we got a "staff" role session cookie.

### Step D: Found a second lock inside the staff panel (SSTI)

The staff panel had a feature — customizing a guest confirmation message using EJS templates (e.g. writing `<%= guest %>` would automatically insert the guest's name).

We tested: if we submit `<%= 7*7 %>`, does the server calculate it and show `49`? **Yes, it did** — meaning the server was treating our input as **executable code**, not just as plain text to display.

This is an important distinction: normally, only specific **variables** (like a name or date) get rendered by a template. But if the **entire user-supplied string** is being treated as the template itself, the user can inject arbitrary JavaScript code — and in EJS, JavaScript code can call system commands through the `child_process` module.

We crafted a payload that made the server run the `id` command, and got a result back — meaning our code was executing. We then used the same technique to send a **reverse shell** — a payload that connects back to our Kali machine, giving us a proper interactive terminal inside the server (as the `poolside` user).

**This is where we captured `user.txt` (the user flag).**

### Step E: Further exploration inside the server (a hidden second process)

Once inside, we looked for **other running programs** — `ps aux` revealed a `processor.js` Node.js program running under a **different user account** (`pipelinesvc`).

This program was running in `--inspect` mode — a Node.js **debugging feature** developers use to step through their code, similar to setting breakpoints in VS Code. The problem is that if this feature is left enabled on a production server, anyone who can reach that port can **remotely control** the program — like leaving a machine's control panel open from the inside.

We noticed this debugger was only bound to `127.0.0.1` (localhost) — meaning it's not reachable from outside, but we were **already inside** (as the `poolside` user), so we could reach it locally.

### Step F: Exploiting the debugger (a second RCE)

Node's debugger uses a **WebSocket protocol**. We manually wrote a Python WebSocket client (since the server had no internet access to install the `ws` library), which:

1. Connects to the debugger
2. Sends it a command: "run this JavaScript code"
3. Reads back the result

Using this, we achieved command execution again — this time as the `pipelinesvc` user. We used the same method to get a reverse shell as that user.

### Step G: Reaching root (abusing the `disk` group)

Checking the new user `pipelinesvc`'s `id` revealed it was a member of the `disk` group. This is an **extremely powerful group** in Linux — members of this group can read the **raw disk device** (the entire hard drive), bypassing normal file permission checks entirely.

Normally, if you try to read root's files, Linux blocks you ("permission denied"). But if you have permission to read the **entire disk in raw form**, you can read any file, belonging to anyone, "from below" (at the raw byte level) — file permissions as a concept get bypassed completely.

We used `debugfs` (a tool normally meant for disk repair/inspection) to directly extract the contents of `/root/root.txt` from `/dev/nvme0n1p1` (the raw root partition device) — **without mounting it, and without any normal permission check.**

```bash
debugfs -R "cat /root/root.txt" /dev/nvme0n1p1
```

**This is where we captured the root flag.**

---

## 5. Full Command Walkthrough (The "Why" Behind Each Command)

|Command|What It Did|Why We Did It|
|---|---|---|
|`nmap -A <IP>`|Scanned open ports and services|Reveals what's running on the server, where to start attacking|
|`whatweb <IP>`|Fingerprinted the website's technology|Confirmed an Express/Node backend|
|`ffuf -u <IP>/FUZZ -w wordlist`|Brute-forced hidden directories/pages|Found `/staff` and `/logout`, which aren't linked anywhere visible|
|`curl -v <IP>/`|Viewed the homepage's raw HTML|Revealed the login form's target URL (`/login`) and field names (`username`, `password`)|
|`curl -X POST /login -d "username=test&password=test"`|Tried a normal login with form-encoded data|Established baseline behavior — whether a cookie is set, what the error looks like|
|`curl -H "Content-Type: application/json" -d '{"password":{"$ne":null}}'`|Sent a NoSQL injection payload|If the backend passes user input directly into a MongoDB-style query, injecting an operator can bypass the check|
|`curl -b "connect.sid=..." /staff/preview -d 'template=<%= 7*7 %>'`|Tested for SSTI|Confirms whether the server treats user input as "code" or just "text"|
|`curl ... 'template=<%= process.mainModule.require("child_process").execSync("id") %>'`|Ran a system command through EJS|Jumped from SSTI to RCE — confirmed we could execute commands on the server|
|`nc -lvnp 4444` + reverse shell payload|Started a listener on our machine and had the server connect back|Sending commands one at a time via curl is limited; we needed a proper interactive terminal|
|`python3 -c 'import pty; pty.spawn("/bin/bash")'`|Upgraded the shell|A basic reverse shell is very limited (no tab-complete, poor Ctrl+C handling) — `pty` gives a real terminal experience|
|`ps auxww`|Listed all running processes|Looking for other users/programs that could be exploited|
|`grep -E "^p" /etc/passwd`|Listed system users|Needed to confirm the full username `pipelinesvc`|
|`curl http://127.0.0.1:9229/json`|Checked the Node debugger's info endpoint|Confirmed the debugger was reachable and got its WebSocket URL|
|Python raw WebSocket script|Manually performed the WebSocket handshake and sent `Runtime.evaluate`|The `ws` npm library wasn't available, so the protocol was implemented from scratch using raw sockets|
|`id` (as pipelinesvc)|Checked the new user's group memberships|Finding the `disk` group was the key clue for privilege escalation|
|`lsblk`|Listed disk partitions|Needed to confirm the root partition's device path (`/dev/nvme0n1p1`)|
|`which debugfs`|Checked whether the tool was available|Required for the final exploit|
|`debugfs -R "cat /root/root.txt" /dev/nvme0n1p1`|Read the root flag directly from the raw disk|Leveraged `disk` group permissions to read the file without mounting it or passing normal permission checks|

---

## 6. Key Takeaways

1. **NoSQL can be just as vulnerable as SQL** — only the syntax differs; the underlying concept is the same: if user input flows unvalidated into a query, an attacker can alter the query's structure.
2. **Templates and code should never mix carelessly** — never pass a user-supplied string directly into a template engine. Only pass in **data** (like a name); keep the template itself fixed.
3. **Never leave debug tooling enabled in production** — features like `--inspect` should be used locally only, never left open on a live server.
4. **Group permissions must be assigned carefully** — service accounts should never be added to groups like `disk`, `root`, or `docker` unless absolutely necessary, since that can grant full control over the system.
5. **Chaining matters** — no single vulnerability here was enough on its own; four separate issues (auth bypass → RCE → lateral movement → privesc) had to be linked together to reach root. This mirrors how real-world penetration testing usually works.

---

_Notes by Obito Uchiha — Team AKATSUKI_
