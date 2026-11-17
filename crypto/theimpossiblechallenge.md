# The Impossible Challenge

**Room:** [The Impossible Challenge](https://tryhackme.com/room/theimpossiblechallenge)
**Category:** Crypto / Steganography | **Difficulty:** Easy

## Scenario

The task had one line, "Download the file, and find the Flag!", a lone `-`, and a strange string:

```
qo qt q` r6 ro su pn s_ rn r6 p6 s_ q2 ps qq rs rp ps rt r4 pu pt qn r4 rq pt q` so pu ps r4 sq pu ps q2 su rn on oq o_ pu ps ou r5 pu pt r4 sr rp qt pu rs q2 qt r4 r4 ro su pq o5
```

The task file was a small ZIP archive.

## Step 1: Looking at the ZIP

```
$ unzip -l <file>.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
       91  2020-04-16 12:05   flag.txt

$ zipinfo -v <file>.zip | grep -i security
  file security status:                           encrypted
```

One 91-byte `flag.txt`, password protected. So the string probably leads to the password.

## Step 2: Peeling the string

The string only used 14 characters: `2 4 5 6 _ ` n o p q r s t u`. That looked like hex digits after some rotation. Lowercase letters `n`–`u` plus digits and `_`/`` ` `` pointed at ROT13 (letters only) combined with ROT47 (all printable ASCII).

```python
import codecs, base64

s = "qo qt q` r6 ro su pn s_ ..."  # full string from the task
step1 = codecs.encode(s, "rot13")
step2 = "".join(chr(33 + (ord(c) - 33 + 47) % 94) if c != " " else " " for c in step1)
print(step2)
step3 = bytes.fromhex(step2.replace(" ", ""))
print(step3)
print(base64.b64decode(step3))
```

```
53 58 51 6e 63 79 42 70 62 6e 4e 70 5a 47 55 67 ...
b'SXQncyBpbnNpZGUgdGhlIHRleHQsIGluIGZyb250IG9mIHlvdXIgZXllcyE='
b"It's inside the text, in front of your eyes!"
```

Four layers: ROT13, ROT47, hex, Base64. The result wasn't the password but a hint.

## Step 3: Following the hint

"Inside the text, in front of your eyes" means something hidden in the task text itself. The suspicious part was the lone `-`. A classic way to hide data in plain sight is zero-width Unicode characters: they render as nothing but survive in the page text.

Copying the text through my VM clipboard into a file lost them, so I checked straight in the browser console on the task page:

```js
[...document.body.innerText].filter(c => c.charCodeAt(0) > 127).map(c => c.charCodeAt(0).toString(16))
```

That returned 160 invisible characters, all from just four code points:

| Code point | Name |
|---|---|
| `U+200C` | Zero Width Non-Joiner |
| `U+200D` | Zero Width Joiner |
| `U+202C` | Pop Directional Formatting |
| `U+FEFF` | Zero Width No-Break Space |

## Step 4: Decoding the zero-width characters

Four symbols means base 4. This scheme encodes every character as 8 base-4 digits, so 160 symbols give 20 characters. Mapping `200C=0, 200D=1, 202C=2, FEFF=3`:

```python
codes = "200c 200c 200c 200c 200d feff ...".split()  # all 160 from the console
m = {"200c": "0", "200d": "1", "202c": "2", "feff": "3"}
q = "".join(m[c] for c in codes)
print("".join(chr(int(q[i:i+8], 4)) for i in range(0, len(q), 8)))
```

```
password is ********
```

(The same result comes out of the online [Unicode Steganography with Zero-Width Characters](https://330k.github.io/misc_tools/unicode_steganography.html) decoder.)

## Step 5: Opening the ZIP

```
$ unzip -P '********' <file>.zip
$ cat flag.txt
You have solved the Impossible Challenge! Here is your flag THM{****_*****_**********_****}
```

## Recommendations

- **Normalize and filter untrusted text.** Strip or flag zero-width and bidirectional control characters (`U+200B`–`U+200F`, `U+202A`–`U+202E`, `U+2060`–`U+2064`, `U+FEFF`) in user input, comments and documents. The same trick is used to hide prompt-injection payloads from human reviewers of LLM inputs.
- **Encoding is not encryption.** ROT13, ROT47, hex and Base64 add zero security; anyone can reverse them without a key.
- **Use strong archive encryption.** Legacy ZipCrypto is weak; use AES-256 (e.g. `7z -mem=AES256`) and never deliver the password through the same channel as the file.
- **Show invisible characters in review tools.** Editors and diff tools that highlight zero-width characters make this kind of hiding obvious.

## IOC

| Type | Value | Note |
|---|---|---|
| File | `flag.txt` (91 bytes) | Inside a password-protected ZIP |
| Encoded string | `qo qt q` r6 …` | ROT13 → ROT47 → hex → Base64 |
| Decoded hint | `It's inside the text, in front of your eyes!` | Points to hidden characters |
| Hidden characters | `U+200C`, `U+200D`, `U+202C`, `U+FEFF` (160 total) | Base-4 zero-width steganography in the task text |
| ZIP password | masked | Recovered from the zero-width payload |

## Mapping

| Framework | ID | Name | Why it fits |
|---|---|---|---|
| MITRE ATT&CK | T1027.003 | Obfuscated Files or Information: Steganography | Password hidden in invisible Unicode characters |
| MITRE ATT&CK | T1027 | Obfuscated Files or Information | Stacked ROT13/ROT47/hex/Base64 encoding |
| MITRE ATT&CK | T1140 | Deobfuscate/Decode Files or Information | What I did to recover the hint and password |
| CWE | CWE-176 | Improper Handling of Unicode Encoding | Invisible characters pass unnoticed through text handling |
| CWE | CWE-327 | Use of a Broken or Risky Cryptographic Algorithm | Encodings used in place of encryption |
| OWASP LLM Top 10 (2025) | LLM01 | Prompt Injection | Zero-width characters are a known way to hide injected instructions |

## Takeaways

- A limited character set is a strong clue: 14 symbols that rotate into hex pointed straight at ROT13 + ROT47.
- When a decoded message is a hint rather than an answer, re-read the task. Here the answer was literally on the page.
- Copy-paste can silently drop zero-width characters (VM clipboards, terminals). Inspecting the text in the browser console is more reliable.
- Four distinct invisible characters means base 4; the decoding is then just grouping digits into bytes.
