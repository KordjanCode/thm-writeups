# TryHackMe Write-ups

Hi, I'm Marcin ([@KordjanCode](https://github.com/KordjanCode)). This is where I keep my notes from [TryHackMe](https://tryhackme.com) rooms.

I'm mostly interested in **AI and LLM security**: how models get attacked, how supply chains get poisoned and how to investigate it when something goes wrong. I also do web security and incident response.

Each write-up walks through a room step by step, the way I actually solved it: what I ran, what I saw and what it told me. At the end I add IOCs, a mapping to MITRE ATT&CK / ATLAS, OWASP and CWE, and what I'd recommend to fix the problem.

For premium rooms I mask the answers and flags. The point is to show the process, not to give away solutions.

## AI Security

| Room | What it's about |
|---|---|
| [Payload](./ai/payload/README.md) ([room](https://tryhackme.com/room/payload)) | Investigating a backdoored ML model: a malicious pickle running `os.system`, a hidden Keras `Lambda` layer and a C2 beacon split across two artefacts |
| [Checkpoint](./ai/checkpoint/README.md) ([room](https://tryhackme.com/room/checkpoint)) | Evaluating four AI code review models before production: a malicious pickle, a Keras `Lambda` loader, an opaque hosted API and a poisoned prompt template linked by one campaign ID |
| [UnIndexed](./ai/unindexedchallenge.md) ([room](https://tryhackme.com/room/unindexedchallenge)) | Probing Cloudwright Labs' staff-level assistant Atlas and retrieving board-level records through ordinary questions, because the RAG store had no document-level access control |
| [Lockdown](./ai/lockdownai.md) ([room](https://tryhackme.com/room/lockdownai)) | Auditing Meridian Security Group's assistant Bastion: confirming three RAG flaws (unfiltered retrieval, sensitive logs, no tenant isolation) and describing the right control for each |
| [BankGPT](./ai/bankgpt.md) ([room](https://tryhackme.com/room/bankgpt)) | Getting SecureBank's staff assistant BankGPT to leak a secret key by asking it to write an internal admin memo that quotes, exactly, the values staff must never share |

More coming as I work through the AI Security path.

## Web Security

| Room | What it's about |
|---|---|
| [Message to Garcia](./web/messagetogarcia.md) ([room](https://tryhackme.com/room/messagetogarcia)) | Turning an unvalidated resource fetcher into local file read via `file://`, dumping the Flask source through `/proc/self/cwd`, and forging a valid Fernet message from the hardcoded key |

## Crypto

| Room | What it's about |
|---|---|
| [Cipher's Secret Message](./crypto/hfb1cipherssecretmessage.md) ([room](https://tryhackme.com/room/hfb1cipherssecretmessage)) | Reversing a homemade progressive Caesar cipher, where each letter is shifted by its position, with a short Python decoder |
| [Order](./crypto/hfb1order.md) ([room](https://tryhackme.com/room/hfb1order)) | Breaking a repeating-key XOR cipher with a known-plaintext attack: the fixed `ORDER:` header reveals the whole key |
| [The Impossible Challenge](./crypto/theimpossiblechallenge.md) ([room](https://tryhackme.com/room/theimpossiblechallenge)) | Peeling ROT13, ROT47, hex and Base64 to a hint, then decoding zero-width Unicode characters hidden in the task text to get the ZIP password |
| [Scripting](./crypto/scripting.md) ([room](https://tryhackme.com/room/scripting)) | Unwrapping a file encoded in Base64 50 times (Bash and Python), scripting a port-hopping arithmetic chain, and decrypting AES-GCM flags over UDP until a SHA-256 checksum matched |

## Certifications

- TryHackMe Cyber Security 101 (SEC1): passed
- TryHackMe AI Security (AI1): in progress

## Get in touch

If you're working on AI security or want to talk about a project, find me here on GitHub: [@KordjanCode](https://github.com/KordjanCode).
