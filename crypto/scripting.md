# TryHackMe: Scripting

**Room:** [Scripting](https://tryhackme.com/room/scripting) (Premium) · **Category:** Crypto / Scripting · **Difficulty:** Medium · **Time:** ~75 min
**Author:** Marcin Korczak ([@KordjanCode](https://github.com/KordjanCode))

> This is a premium room, so I masked the flags and answers. The goal here is to show how I worked through it, not to hand out solutions.

## The scenario

*Three small scripts, and a server that hands you its own AES key.*

The room is a set of scripting challenges, labelled Easy, Medium and Hard. I worked on Kali in `~/Desktop/scripting`. Task 1 is a downloaded file. Tasks 2 and 3 talk to the room machine: a port-hopping HTTP service, then a UDP server that claims to encrypt its flags.

## Task 1: [Easy] Base64

The task file was `b64_1550406728131.txt`. It had been Base64-encoded 50 times. The room asked for a loop in both Bash and Python: read the file, decode, repeat.

Bash:

```
┌──(m㉿kali)-[~/Desktop/scripting]
└─$ data=$(cat b64_1550406728131.txt); for i in $(seq 1 50); do data=$(echo "$data" | base64 -d); done; echo "$data"
Hack*********
```

Python:

```
┌──(m㉿kali)-[~/Desktop/scripting]
└─$ python3 -c 'import base64; d=open("b64_1550406728131.txt").read().strip()
for _ in range(50): d=base64.b64decode(d)
print(d.decode())'
Hack*********
```

Both printed the same 13-character string, which is the answer for this task (masked here). Fifty layers of Base64 are still just Base64. There is no key, so a loop is enough.

## Task 2: [Medium] Gotta Catch'em All

This task starts a web server that hops from port to port. Each port stays up for about four seconds, and the page body is a single instruction in the form `operation number nextport`. I started a running total at `0`, began at port `1337`, and stopped when the next port was `9765` (or if the body said `STOP`). The start page was `http://10.80.177.59:3010`. The answer has to be rounded to two decimal places, but only at the end.

I saved this as `ports.py`. It opens a raw TCP socket, sends a simple HTTP GET, and polls about once a second if the port is down. If I see the same body twice I ignore it and wait. The operations in this room were `add`, `minus`, `multiply` and `divide`.

```python
import socket, time, re

HOST = "10.80.177.59"
port = 1337
num = 0.0
last = None

def get(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(2)
    s.connect((HOST, port))
    s.sendall(f"GET / HTTP/1.1\r\nHost: {HOST}:{port}\r\nConnection: close\r\n\r\n".encode())
    data = b""
    while True:
        chunk = s.recv(4096)
        if not chunk:
            break
        data += chunk
    s.close()
    return data.decode(errors="ignore").split("\r\n\r\n", 1)[-1].strip()

while True:
    try:
        body = get(port)
    except (ConnectionRefusedError, socket.timeout, OSError):
        time.sleep(1)
        continue
    if body == last:
        time.sleep(1)
        continue
    print(f"[{port}] {body}")
    if "STOP" in body.upper():
        break
    m = re.match(r"(\w+)\s+(-?[\d.]+)\s+(\d+)", body)
    if not m:
        time.sleep(1)
        continue
    last = body
    op, val, nxt = m.group(1).lower(), float(m.group(2)), int(m.group(3))
    if op == "add": num += val
    elif op == "minus": num -= val
    elif op == "multiply": num *= val
    elif op == "divide": num /= val
    print(f"    -> {op} {val} = {num}")
    port = nxt
    if port == 9765:
        break

print(f"RESULT: {num:.2f}")
```

```
┌──(m㉿kali)-[~/Desktop/scripting]
└─$ python3 ports.py
[1337] add 900 23456
    -> add 900.0 = 900.0
[23456] minus 2 8888
    -> minus 2.0 = 898.0
[8888] multiply 4 9823
    -> multiply 4.0 = 3592.0
[9823] divide 2 9887
    -> divide 2.0 = 1796.0
[9887] add 456 7823
    -> add 456.0 = 2252.0
[7823] minus 43 10456
    -> minus 43.0 = 2209.0
...
RESULT: 3447**.**
```

The chain ran for 34 hops and finished when the last instruction pointed at port `9765`. A few later steps were divisions, and Python's float started to show the usual binary residue. I left the running total alone and only rounded when I printed `RESULT`. That value (masked here) is the answer for the task.

## Task 3: [Hard] Encrypted Server Chit Chat

The last task is a UDP server on port `4000` (machine IP `10.81.183.123`). The room said to send a packet with the payload `hello`, then follow the instructions, and that the flags were encrypted with AES-GCM. The server sends the ciphertext and the GCM tag as two separate replies, in that order.

I probed the protocol first.

```
┌──(m㉿kali)-[~/Desktop/scripting]
└─$ python3 -c '
import socket
s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM); s.settimeout(5)
s.sendto(b"hello",("10.81.183.123",4000))
print(s.recvfrom(4096)[0])'
b"You've connected to the super secret server, send a packet with the payload ready to receive more information"
```

Then `ready`:

```
┌──(m㉿kali)-[~/Desktop/scripting]
└─$ python3 -c '
import socket
s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM); s.settimeout(5)
s.sendto(b"ready",("10.81.183.123",4000))
print(s.recvfrom(4096)[0])'
b"key:thisisaverysecretkeyl337 iv:secureivl337 to decrypt and find the flag that has a SHA256 checksum of ]w\xf0\x18\xd2... [32-byte checksum] send final in the next payload to receive all the encrypted flags"
```

The greeting handed me the key, the IV and a 32-byte SHA-256 of the real flag, all in one cleartext UDP reply. The key is 24 bytes, so this is AES-192. The IV is 12 bytes, which is the usual GCM nonce length, but it is a fixed string: every flag is encrypted under the same key and the same nonce.

I saved the rest as `chit.py`. After the same `hello` / `ready` handshake it sends `final` to get a ciphertext, sends `final` again to get the tag, decrypts with AES-GCM, and compares `sha256(plaintext)` to the checksum from the server.

```python
import socket, hashlib, time
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

HOST, PORT = "10.81.183.123", 4000
KEY = b"thisisaverysecretkeyl337"
IV = b"secureivl337"
TARGET = b"<32-byte checksum from server>"

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.settimeout(5)
aes = AESGCM(KEY)

s.sendto(b"hello", (HOST, PORT)); s.recvfrom(4096)
s.sendto(b"ready", (HOST, PORT)); s.recvfrom(4096)

for i in range(200):
    s.sendto(b"final", (HOST, PORT))
    ct = s.recvfrom(4096)[0]
    s.sendto(b"final", (HOST, PORT))
    tag = s.recvfrom(4096)[0]
    try:
        flag = aes.decrypt(IV, ct + tag, None)
    except Exception as e:
        print(f"[{i}] decrypt fail: {e}")
        continue
    print(f"[{i}] {flag}")
    if hashlib.sha256(flag).digest() == TARGET:
        print("FLAG:", flag.decode())
        break
    time.sleep(0.2)
```

```
┌──(m㉿kali)-[~/Desktop/scripting]
└─$ python3 chit.py
[0] b'THM{78OI****}'
[1] b'THM{LZD1****}'
[2] b'THM{FEFR****}'
[3] b'THM{MMEY****}'
[4] b'THM{3Q2D****}'
[5] b'THM{MK5V****}'
[6] b'THM{NIB1****}'
[7] b'THM{W85N****}'
[8] b'THM{D1BN****}'
[9] b'THM{IIGL****}'
[10] b'THM{eW-s*******************}'
FLAG: THM{eW-s*******************}
```

The first ten plaintexts were decoys, each an eight-character `THM{...}` string. Index 10 was the one whose SHA-256 matched the checksum from the `ready` reply. That is the flag (masked here).

AES-GCM itself is fine. What is not fine is a static IV. Nonce reuse under the same key is a real-world failure: it can leak plaintext relationships and undermine GCM's authenticity guarantee. Combined with a key and IV delivered in cleartext, the "encrypted" channel was only a filter for which candidate to keep.

## Recommendations

- **Encoding is not encryption.** Fifty rounds of Base64 still reverse with a loop and no key. If a file has to stay secret, use a real cipher and keep the key off the shared channel.
- **Don't hide a service by hopping ports.** A four-second window on an unpublished port is security through obscurity. Anyone who can poll the first port can walk the whole chain. Authenticate the service and keep it on a stable, firewalled port.
- **Never ship the AES key and IV in the clear.** After `ready`, the UDP server handed me everything I needed to decrypt every flag. Use a proper handshake (TLS or DTLS, or an authenticated key exchange) and do not print secrets into a protocol greeting.
- **Never reuse a nonce with AES-GCM.** `secureivl337` is 12 bytes, which is the right length, but it never changes. Generate a fresh random nonce for every message and send it alongside the ciphertext.
- **Don't hard-code keys.** A 24-byte AES-192 key baked into a greeting, and then into a client, is a key you should assume is public.

## IOC

The values below are artefacts from the lab.

| Type | Value | Note |
|---|---|---|
| File | `b64_1550406728131.txt` | Base64-encoded 50 times |
| Task 1 answer | `Hack*********` | Same result from Bash and Python |
| Start page | `http://10.80.177.59:3010` | Task 2 entry point |
| Start / end ports | `1337` → `9765` | HTTP service hops every ~4 seconds |
| Operations | `add`, `minus`, `multiply`, `divide` | Body format: `operation number nextport` |
| Task 2 answer | `3447**.**` | 34 hops, rounded to 2 decimals at the end |
| UDP service | `10.81.183.123:4000` | Task 3 |
| Payloads | `hello`, `ready`, `final` | Cleartext UDP |
| AES key | `thisisaverysecretkeyl337` (24 bytes) | AES-192, sent in the `ready` reply |
| AES-GCM IV | `secureivl337` (12 bytes) | Static nonce, reused for every flag |
| SHA-256 checksum | 32 raw bytes in the `ready` reply | Identifies the real flag among decoys |
| Flag | `THM{eW-s*******************}` | Decrypted candidate that matched the checksum |

## Mapping

| Framework | ID | Name | Why it fits |
|---|---|---|---|
| MITRE ATT&CK | T1132.001 | Data Encoding: Standard Encoding | The first flag was wrapped in Base64 50 times, which hides the string but does not protect it |
| MITRE ATT&CK | T1571 | Non-Standard Port | The HTTP service hopped across high ports every few seconds, and the flag server listened on UDP/4000 |
| MITRE ATT&CK | T1573.001 | Encrypted Channel: Symmetric Cryptography | Flags were sent under AES-192-GCM, but the key and IV were already on the wire |
| MITRE ATT&CK | T1059.006 | Command and Scripting Interpreter: Python | The port-hopping chain and the UDP decrypt loop are small Python clients; that is also how I solved the room |
| OWASP Top 10 (2021) | A02:2021 | Cryptographic Failures | Layered Base64, a hard-coded key, a reused GCM nonce, and secrets sent in cleartext |
| CWE | CWE-321 | Use of Hard-coded Cryptographic Key | The AES-192 key is a fixed string in the server greeting |
| CWE | CWE-323 | Reusing a Nonce, Key Pair in Encryption | Every flag is encrypted with the same key and the same IV `secureivl337` |
| CWE | CWE-319 | Cleartext Transmission of Sensitive Information | The key, IV and flag checksum arrived in an unencrypted UDP datagram |
| CWE | CWE-311 | Missing Encryption of Sensitive Data | Those secrets were not protected in transit at all |
| CWE | CWE-656 | Reliance on Security Through Obscurity | The arithmetic service tried to stay hidden by changing ports every four seconds |

## Takeaways

The lesson I took from Task 1 is the obvious one, and it is still worth stating: **stacked encoding is not a defence.** If I can write a 50-step loop, I can read the file.

Task 2 was less about the arithmetic than about making the client reliable. I had to retry while a port was down, skip a body I had already applied, and leave the float alone until the final `print`. Rounding in the middle would have hardened early errors into the result.

Task 3 looked like cryptography until the `ready` reply. After that I was only iterating AES-GCM candidates under a reused nonce. The SHA-256 checksum told me which plaintext to keep. It did not protect the key, and the stream of decoy flags did not either.

---

*Write-up by Marcin Korczak, based on a TryHackMe lab, for educational purposes.*
