# Sisterhood of the Traveling Packets — Detailed Notes (English)

## What the challenge was, basic understanding

This was a **ransomware leak site simulation**. In the real world, when ransomware gangs (like LockBit, ALPHV, etc.) steal a company's data and the company refuses to pay the ransom, the gang **publishes the data on a dark web "leak site"** to apply pressure. This challenge was exactly this kind of fake site, built by a fictional gang called "pantalones" (for CTF purposes).

Our job was: **find these "hackers'" own mistakes (OPSEC mistakes) inside this site** and use them to gain access to their admin panel — where the flag was found.

**OPSEC** (Operational Security) refers to the precautions an attacker/criminal takes to hide their identity and methods. When they make a mistake in those precautions (like sharing a password in chat, or accidentally leaving credentials in a file), it's called an "OPSEC failure" — and that's exactly what we exploited.

## Technologies/Tools used

|Thing|What it is / why it was used|
|---|---|
|**Tor Browser**|A special browser that can open `.onion` addresses — these are "dark web" addresses that aren't accessible from a normal browser (Chrome/Firefox). Tor encrypts traffic and routes it through multiple servers (relays) for anonymity.|
|**curl**|A command-line tool that sends HTTP requests without a browser — like "go to this url and give me the response" but from the terminal|
|**SOCKS5 proxy**|Tor's internal proxy (port 9050) that lets tools like curl also use the Tor network|
|**base64**|An encoding (NOT encryption) that converts text into a random-looking string of letters/numbers. It is **reversible** — anyone can decode it, which is why it isn't sufficient for security|
|**robots.txt**|Every website has this file that tells search engines (Google, Bing) "don't crawl these pages." But this file itself is publicly readable — so for an attacker it becomes a **map** of the hidden pages the site owner wanted to hide|
|**Directory listing**|When a web server shows the list of files inside a folder (like a web version of Windows Explorer), if the developer hasn't disabled this feature, anyone can open the folder and see every file, even ones with no direct link|
|**jq**|A tool for pretty-printing/formatting JSON data into a readable format|

## Real-life analogy to understand it

Imagine a thief robbed someone's house and set up a stall in an alley to sell the stolen goods (this is the leak site). Behind the stall is a small office where he keeps his accounts (this is the admin panel). Now:

- The stall has a building directory board saying "don't look here: office door, accounts file" (this is `robots.txt`) — but this board itself can be read publicly!
- The thief put the loot in a sack, but by mistake also put his own diary in the same sack, containing his whole plan, contact number, and passwords (this is the `.exfil.sh` script)
- The thief's gang members use a "hidden" code when sending passwords to each other on WhatsApp (like writing it backwards) — but this code is so simple that anyone can figure it out in 2 seconds (this is base64)

This is exactly what we exploited.

## From beginner to challenge — step-by-step journey

If you're a complete beginner doing your first CTF, here's how we thought through it step by step:

**Step 1 — Explore the site like a normal visitor** The very first thing to do in any CTF/pentest is **"view page source"** (Ctrl+U or right-click → View Source). Here we found a base64 comment that decoded to "not_the_flag_keep_looking" — the designer deliberately placed this so beginners wouldn't stop early thinking "this is the flag."

**Step 2 — Check every nav link** Open every link at the top of the website (Leaks, Crew, About) and check their source too. On `crew.php` we found 4 names: vex, crypt, mora, skid — along with their roles. The clue "vex — panel dev" was important, meaning there's an admin panel somewhere that vex built.

**Step 3 — Check robots.txt (always do this)** This is a standard step in any website recon. Go to `site.com/robots.txt` and check — whatever the developer has "Disallowed" is exactly what you should check first, because those are the things they were trying to hide. We found `/admin.php` and `/api.php`.

**Step 4 — Check file listings, don't rely only on the download button** The site had a "download leak" button that gave a `.zip` file, but we noticed that if you go directly to the folder path of that zip (`downloads/aetherflow/`) in the browser, the full folder listing shows up — including extra files that weren't included in the zip. This is a developer mistake — "directory listing" should have been disabled in the web server settings.

**Step 5 — Look for hidden/dot files** When we checked the `quantumcore` folder, we found a file called `.exfil.sh` — files starting with a dot (.) are "hidden" in Linux, meaning a normal `ls` command doesn't show them, only `ls -la` (all files) does. This was a bash script written by the attacker (vex) himself to upload his own data — and it had his **API key** (`X-Panel-Key: pantalonesgroup`) and a **second .onion address** hardcoded inside it.

**Step 6 — Send an authenticated request to the API** Now that we had the key, we tried `api.php?action=status` with that key in the header — and it worked! This told us which operators were "online" (vex, crypt) — meaning both of them have valid login accounts.

**Step 7 — Read the messages/chat logs** `api.php?action=messages&conversation_id=X` gave us the crew's internal chat (by trying X as 1, 2, 3...). Conversation 2 contained an exchange where crypt gave mora her password, but "encoded":

```
UGFudGFsMG4zc19SdWwzeiE=
```

**Step 8 — Decode the base64** This string looks random at a glance, but its pattern (letters, numbers, an `=` sign at the end) is a signature of base64 encoding. In the terminal:

```bash
echo "UGFudGFsMG4zc19SdWwzeiE=" | base64 -d
```

This gave us: `Pantal0n3s_Rul3z!`

**Step 9 — Log in** Now we had:

- Username: `mora` (the chat told us this password belongs to mora)
- Password: `Pantal0n3s_Rul3z!` (the decoded value)

We entered these on `admin.php` and **the login succeeded!**

**Step 10 — Extract the flag** The admin dashboard opened, showing a list of every "victim" along with their ransom amount and "decryption key." The entry for our own simulated meta-victim company, "Sisterhood of the Travelling Packets," had the flag in place of a decryption key:

```
flare{pantal0n3s_g0t_pantsed_2026}
```

## Key takeaways (for beginners)

1. **Check every small detail** — page source, robots.txt, folder listings, hidden files — these are all "low hanging fruit" that even experienced pentesters check first
2. **Base64 encoding is not security** — if any string ends in `=` and is a random mix of letters/numbers, it's probably base64, always try decoding it
3. **These same mistakes happen in the real world too** — companies leave directory listing enabled on their cloud storage (S3 buckets), developers leave hardcoded API keys in `.env` files or scripts — exactly like what was simulated in this CTF
4. **A chain of small clues = the big win** — no single step alone led to the flag; each step gave the next clue. This is exactly how real pentesting/CTF work goes — patience and a systematic approach

## Command cheat-sheet used

```bash
# Check the Tor proxy
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip

# Check robots.txt
curl --socks5-hostname 127.0.0.1:9050 http://<onion-address>/robots.txt

# Authenticated API status check
curl -s "http://<onion>/api.php?action=status" \
    -H "X-Panel-Key: pantalonesgroup" \
    --socks5-hostname 127.0.0.1:9050 | jq

# Reading messages
curl -s "http://<onion>/api.php?action=messages&conversation_id=2" \
    -H "X-Panel-Key: pantalonesgroup" \
    --socks5-hostname 127.0.0.1:9050 | jq

# Base64 decode
echo "UGFudGFsMG4zc19SdWwzeiE=" | base64 -d
```