# Order

**Room:** [Order](https://tryhackme.com/room/hfb1order)
**Category:** Crypto | **Difficulty:** Easy

## Scenario

We intercepted a message from Cipher with their next target. It was encrypted with a repeating-key XOR cipher, but they made one critical mistake: every message always starts with the same header, `ORDER:`. My job was to decrypt it and find the target.

The intercepted ciphertext (hex, two lines of one message):

```
1c1c01041963730f31352a3a386e24356b3d32392b6f6b0d323c22243f6373
1a0d0c302d3b2b1a292a3a38282c2f222d2a112d282c31202d2d2e24352e60
```

## Step 1: Understanding the weakness

Repeating-key XOR encrypts every byte as `ciphertext = plaintext ^ key[i % len(key)]`. XOR is its own inverse, so if I know a piece of the plaintext I also know the key at those positions:

```
key = ciphertext ^ plaintext
```

The fixed `ORDER:` header gives me 6 known plaintext bytes at the start of the message. This is a known-plaintext attack (a crib). If the key is 6 bytes or shorter, the header reveals the entire key, and since the key repeats, it then decrypts the whole message.

## Step 2: Writing the decoder

I joined both hex lines into one ciphertext, XORed the first 6 bytes with `ORDER:` to recover the key, then used it to decrypt everything. I saved it as `decode-order.py`:

```python
ct = bytes.fromhex(
    "1c1c01041963730f31352a3a386e24356b3d32392b6f6b0d323c22243f6373"
    "1a0d0c302d3b2b1a292a3a38282c2f222d2a112d282c31202d2d2e24352e60"
)
known = b"ORDER:"

# 1. Key = ciphertext XOR known header
key = bytes(c ^ p for c, p in zip(ct, known))
print("Key:", key)

# 2. Decrypt with the repeating key
pt = bytes(c ^ key[i % len(key)] for i, c in enumerate(ct))
print(pt.decode())
```

## Step 3: Running it

```
┌──(m㉿kali)-[~/Desktop/thm/order]
└─$ python3 decode-order.py
Key: b'SNEAKY'
ORDER: Attack at dawn. Target: THM{the_*********_**********}.
```

The recovered key was a readable 6-letter word, which told me right away that the key length matched the header length. The rest of the message decrypted cleanly with no garbage, so there was no need to guess other key lengths. The target is in the flag (masked here).

## Recommendations

- **Never use repeating-key XOR for real data.** It falls to known-plaintext and frequency analysis in seconds. Use an authenticated cipher like AES-GCM or ChaCha20-Poly1305.
- **Never reuse a key stream.** If XOR-style encryption is needed (a stream cipher), the key stream must be at least as long as the message, unique for every message and never repeat (a fresh nonce for every message).
- **Assume message structure is known.** Fixed headers, file magic bytes and protocol fields are all cribs. Good modern ciphers stay secure even when the attacker knows most of the plaintext.
- **Don't use dictionary words as keys.** A key like this can be guessed even without a crib.

## IOC

| Type | Value | Note |
|---|---|---|
| Ciphertext (hex) | `1c1c0104…24352e60` | Intercepted message, 62 bytes |
| Known plaintext | `ORDER:` | Fixed header on every message (crib) |
| Key | `SNEAKY` (6 bytes) | Recovered from the header |
| Technique | Repeating-key XOR | Broken with a known-plaintext attack |

## Mapping

| Framework | ID | Name | Why it fits |
|---|---|---|---|
| CWE | CWE-327 | Use of a Broken or Risky Cryptographic Algorithm | Repeating-key XOR instead of a real cipher |
| CWE | CWE-323 | Reusing a Nonce, Key Pair in Encryption | The same short key is reused across the whole message and every message |
| CWE | CWE-321 | Use of Hard-coded Cryptographic Key | A fixed dictionary-word key shared by all messages |
| OWASP Top 10 (2021) | A02 | Cryptographic Failures | Sensitive data protected with weak encryption |
| MITRE ATT&CK | T1573.001 | Encrypted Channel: Symmetric Cryptography | The group's communication relies on a weak symmetric cipher |
| MITRE ATT&CK | T1140 | Deobfuscate/Decode Files or Information | What I did to recover the plaintext |

## Takeaways

- With XOR, knowing the plaintext means knowing the key. A single fixed header is enough to break repeating-key XOR.
- If the crib is at least as long as the key, I get the full key in one step. If it were shorter, I'd recover part of the key and have to guess the rest with frequency analysis.
- A readable key is a good sanity check: random-looking bytes would have suggested a wrong key length or a wrong crib.
- Both hex lines belong to one message, so they have to be joined before decryption, otherwise the key position resets in the middle.
