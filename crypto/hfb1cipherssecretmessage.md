# Cipher's Secret Message

**Room:** [Cipher's Secret Message](https://tryhackme.com/room/hfb1cipherssecretmessage)
**Category:** Crypto | **Difficulty:** Easy

## Scenario

A secret message from the Ciphers group was recovered from an old system, together with the script that encrypted it. My job was to decode the message and wrap it in the `THM{}` flag format.

The ciphertext:

```
a_up4qr_kaiaf0_bujktaz_qm_su4ux_cpbq_ETZ_rhrudm
```

The encryption algorithm:

```python
from secret import FLAG

def enc(plaintext):
    return "".join(
        chr((ord(c) - (base := ord('A') if c.isupper() else ord('a')) + i) % 26 + base)
        if c.isalpha() else c
        for i, c in enumerate(plaintext)
    )

with open("message.txt", "w") as f:
    f.write(enc(FLAG))
```

## Step 1: Reading the algorithm

I went through `enc()` piece by piece before writing any code:

- `enumerate(plaintext)` gives me each character `c` together with its position `i`, starting at 0.
- `base` is `ord('A')` for uppercase and `ord('a')` for lowercase, so case is preserved.
- `ord(c) - base` turns the letter into a number from 0 to 25.
- `+ i` shifts that number by the character's position, and `% 26` wraps it around the alphabet.
- `+ base` turns it back into a letter.
- Anything that isn't a letter (`_`, digits) is copied unchanged, **but it still increases `i`**.

So this is a Caesar cipher with a key that grows by one for every character, basically a progressive shift (a simple Vigenère-style cipher where the key is `0, 1, 2, 3...`). There's no secret key at all: the shift depends only on the position, and I can see every position.

## Step 2: Reversing it

Since encryption adds `i`, decryption subtracts `i`. Everything else stays exactly the same, including counting positions across the whole string and not just across letters. `% 26` handles the wrap-around, so a letter shifted below `a` comes back from `z` (Python's modulo always returns a non-negative result here).

I saved this as `decode-cipher.py`:

```python
msg = "a_up4qr_kaiaf0_bujktaz_qm_su4ux_cpbq_ETZ_rhrudm"

def dec(ciphertext):
    return "".join(
        chr((ord(c) - (base := ord('A') if c.isupper() else ord('a')) - i) % 26 + base)
        if c.isalpha() else c
        for i, c in enumerate(ciphertext)
    )

print("THM{" + dec(msg) + "}")
```

## Step 3: Running it

```
┌──(m㉿kali)-[~/Desktop/thm/cipher]
└─$ python3 decode-cipher.py
THM{a_sm4ll_*****_*******_**_*****_****_***_******}
```

The output reads as a normal leetspeak sentence, which confirmed the decryption was right on the first try. Wrapped in `THM{}`, it's the flag for the task (masked here).

## Recommendations

- **Don't build your own crypto.** A position-based shift has no secret key, so anyone holding the algorithm can reverse it instantly. Use a vetted authenticated cipher like AES-GCM or ChaCha20-Poly1305 from a standard library.
- **Assume the algorithm will leak (Kerckhoffs's principle).** Security has to rely on a secret key, not on the code staying hidden. Here the script was recovered together with the ciphertext, and that was enough.
- **Don't keep old ciphertexts and encryption scripts together** on legacy systems. If a weak scheme was used, retire it and re-encrypt the data properly.

## IOC

| Type | Value | Note |
|---|---|---|
| Ciphertext | `a_up4qr_kaiaf0_bujktaz_qm_su4ux_cpbq_ETZ_rhrudm` | Recovered message (`message.txt`) |
| File | `message.txt` | Output of the encryption script |
| Module | `secret.FLAG` | Source of the plaintext imported by the script |
| Technique | Progressive Caesar shift, key = character index | Reversible without any key |

## Mapping

| Framework | ID | Name | Why it fits |
|---|---|---|---|
| CWE | CWE-327 | Use of a Broken or Risky Cryptographic Algorithm | A homemade shift cipher instead of a real one |
| CWE | CWE-1240 | Use of a Cryptographic Primitive with a Risky Implementation | Custom primitive with no key |
| CWE | CWE-656 | Reliance on Security Through Obscurity | Protection only lasts while the algorithm stays secret |
| OWASP Top 10 (2021) | A02 | Cryptographic Failures | Sensitive data protected with weak encryption |
| MITRE ATT&CK | T1027 | Obfuscated Files or Information | The message is obfuscated, not really encrypted |
| MITRE ATT&CK | T1140 | Deobfuscate/Decode Files or Information | What I did to recover the plaintext |

## Takeaways

- Read the algorithm line by line before writing code. Once I understood that the shift equals the position, the fix was one character: `+ i` became `- i`.
- Non-letters don't get shifted, but they still count as positions. If I'd counted only letters, the output would have been garbage.
- Python's `%` with a negative number still returns a value from 0 to 25, so the wrap-around works without extra logic.
- Without a secret key, "encryption" is just encoding. If you have the algorithm, you have the plaintext.
