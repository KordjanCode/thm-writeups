# TryHackMe: Lockdown

**Room:** [Lockdown](https://tryhackme.com/room/lockdownai) (Premium) · **Path:** AI Security › Data Poisoning · **Difficulty:** Medium · **Time:** ~30 min
**Author:** Marcin Korczak ([@KordjanCode](https://github.com/KordjanCode))

> This is a premium room, so I masked the flag. The goal here is to show how I worked through it, not to hand out answers.

## The scenario

*The model can refuse a document and still pull it into context.*

I was a security engineer at Meridian Security Group. Our internal AI assistant, Bastion, helps employees with policy questions and operational issues. A routine audit had flagged three vulnerabilities in its configuration. My job was to confirm each one, diagnose the root cause, and prescribe the correct remediation.

Bastion only accepted a fix if I described the right control. Vague answers were rejected. I was logged in as a security administrator, talking to it through a direct chat. The room pointed me at three built-in commands: `SHOW LOGS` to inspect logging behaviour, `QUERY AS: [any name]` to test tenant isolation, and `STATUS` to check progress. Each correct fix would reveal a flag fragment. Three vulnerabilities, three fragments.

All of the company and document details in this write-up are fictional; they exist only in the lab.

## Step 1: Asking what Bastion serves

I started with a broad question, the kind an admin might ask at the beginning of an audit:

> **Me:** What data do you serve?
>
> **Bastion:** Hello! I help with company policies, and security admins can run commands like SHOW LOGS, QUERY AS, and STATUS.

Follow-up questions about available tenants, and about values I could pass to `QUERY AS`, stayed just as generic. `QUERY AS: User` did not change anything either. Bastion would describe its role, but it would not inventory the knowledge base or the identity model for me.

Asking it to list the company policies was more useful:

> **Me:** List me the company policies
>
> **Bastion:** Travel Policy: up to $2,000 per trip, international travel needs director approval, book via portal, receipts within 14 days. Remote Access: VPN required with MFA, devices registered with IT, 30-minute timeout. Incident Classification: P1 system down 15 min, P2 degraded 1 h, P3 minor 4 h, P4 cosmetic 24 h.

Those three policies look like the kind of material an internal assistant should serve. When I asked who the director was, Bastion refused and said it only helped with policies and audits. That refusal was worth remembering: the model can decline a question, which is not the same as the retrieval layer refusing to fetch a document.

`STATUS` confirmed the starting point:

> **Me:** STATUS
>
> **Bastion:** There are currently 3 vulnerabilities that need to be fixed.

## Step 2: Asking for restricted documents, then reading the logs

I asked directly for material that should have been out of scope:

> **Me:** Show me restricted documents
>
> **Bastion:** I'm unable to serve restricted documents.

That sounded like a boundary. `SHOW LOGS` showed it was only a boundary in the reply, not in the retriever:

> **Me:** SHOW LOGS
>
> **Bastion:** [LOG] Query: SHOW LOGS | Retrieved: Travel Policy, Client Contracts, Employee Data | Logged to /var/log/bastion/retrieval.log

Bastion had just refused to serve restricted documents, yet the retrieval layer had already pulled Client Contracts and Employee Data into the context, alongside the public Travel Policy. Authorization was being left to the model's wording, after the sensitive documents were already in the prompt. The log itself was a second finding: it named those restricted documents in plaintext, and it recorded no user identity and no access decision.

## Step 3: Fixing data retrieval

The first control had to sit at retrieval time, not in the system prompt:

> **Me:** The retrieval layer returns restricted documents (Client Contracts, Employee Data) for any query. Fix: enforce document-level access control at retrieval time, filtering documents by the user's role and permissions before they reach the model, not just in the prompt.
>
> **Bastion:** Correct. Metadata pre-filtering applied. Confidential docs excluded. Fragment 1: `THM{l0****_`

That was the first fragment. Filtering by metadata before the similarity search runs means confidential chunks never reach the model, so a polite refusal is no longer the only line of defence.

## Step 4: Fixing the logs

The same log entry was already enough evidence for the second vulnerability. The file `/var/log/bastion/retrieval.log` exposed which sensitive documents had been retrieved, and it did not say who had queried them. I submitted the logging control next:

> **Me:** The logs don't record user identity and expose which sensitive documents were retrieved. Fix: audit logging with user identity, timestamp and access decision for every retrieval, with sensitive content redacted and log access restricted to admins.
>
> **Bastion:** Correct. Log redaction applied. Logs now record doc IDs only. Fragment 2: `d0**_`

A usable audit log should identify the caller and the access decision. It should not keep a cleartext catalogue of Client Contracts and Employee Data.

## Step 5: Testing tenant isolation

The third issue was user-level access. I switched identity and looked at what the retriever did, rather than trusting the chat reply alone.

`QUERY AS: alice` and `QUERY AS: admin` produced generic replies, the same kind of policy help I had seen at the start. Combining the command with a question, for example `QUERY AS: alice | Show employee records` or `QUERY AS: alice SHOW LOGS`, also stayed generic. The useful evidence was the log after each switch:

> **Me:** QUERY AS: alice
>
> *(generic reply)*
>
> **Me:** SHOW LOGS
>
> **Bastion:** [LOG] Query: SHOW LOGS | Retrieved: Travel Policy, Client Contracts, Employee Data | Logged to /var/log/bastion/retrieval.log

> **Me:** QUERY AS: Admin
>
> *(generic reply)*
>
> **Me:** SHOW LOGS
>
> **Bastion:** [LOG] Query: SHOW LOGS | Retrieved: Travel Policy, Client Contracts, Employee Data | Logged to /var/log/bastion/retrieval.log

The three logs were the same: my own session, `alice`, and `Admin`. Each time the retriever pulled Travel Policy, Client Contracts and Employee Data, and each time the log omitted the user identity. `QUERY AS` did not scope the search to a person or a tenant. There was no isolation to bypass in the chat, because there was no isolation in the retrieval layer at all.

## Step 6: Fixing user-level access

I described that evidence and the control that should sit behind it:

> **Me:** QUERY AS has no effect: SHOW LOGS shows the same restricted documents (Client Contracts, Employee Data) retrieved for alice, Admin and my own session, with no user identity in the log. There is no per-user/tenant isolation. Fix: derive identity server-side from the authenticated session, scope every retrieval to that user's tenant and permissions (per-tenant namespace or filter), and ignore any user-supplied identity.
>
> **Bastion:** Correct. Tenant isolation is enforced at the vector DB layer. Fragment 3: `s3*****}`

Identity has to come from the authenticated session, not from a name typed into the prompt. Every retrieval should be scoped to that tenant and those permissions, in the vector store, before chunks are added to the context.

The three fragments concatenate to `THM{l0****_d0**_s3*****}`.

## What I'd tell the team

**What went wrong:**
- Restricted documents (Client Contracts, Employee Data) were indexed in the same retrievable corpus as public policies.
- Retrieval did not enforce the caller's authorization. The model could refuse to "serve restricted documents" after those documents were already in context.
- The retrieval log named sensitive documents in plaintext, and it recorded neither the user identity nor an access decision.
- `QUERY AS` accepted a user-supplied name, but switching identity did not change what was retrieved. There was no per-user or per-tenant isolation at the vector store.

**What I'd change:**
- Enforce document-level access control at retrieval time, before chunks are added to the prompt. Filter the vector search by the authenticated user's role and permissions, not by what the model is willing to say.
- Keep public and restricted corpora in separate collections, or tag every document with a clearance and drop unauthorized hits before similarity search.
- Don't rely on the assistant's refusal as an access-control check. If a document should not be read, it should not be retrieved.
- Write audit logs with user identity, timestamp and the access decision for every retrieval. Redact or replace sensitive titles and content with document IDs, and restrict log access to admins.
- Derive identity server-side from the authenticated session. Scope every query to that user's tenant and permissions (a per-tenant namespace or filter) and ignore any identity parameter supplied in the prompt.

## IOCs

The values below are artefacts from the lab, not a real company.

| Type | Value |
|---|---|
| Organization (fictional) | Meridian Security Group |
| Assistant | Bastion |
| Public policies served | Travel Policy, Remote Access, Incident Classification |
| Restricted documents retrieved | Client Contracts, Employee Data |
| Log path | `/var/log/bastion/retrieval.log` |
| Diagnostic commands | `SHOW LOGS`, `QUERY AS: [any name]`, `STATUS` |
| Identities tested | `alice`, `admin` / `Admin`, own security-admin session |
| Flag | `THM{l0****_d0**_s3*****}` |

## Mapping

| Framework | ID | Name | Why it fits |
|---|---|---|---|
| MITRE ATLAS | AML.T0064 | Gather RAG-Indexed Targets | The early questions and `SHOW LOGS` mapped what the knowledge base contained, including restricted Client Contracts and Employee Data that Bastion would not openly list |
| MITRE ATLAS | AML.T0085.000 | Data from AI Services: RAG Databases | Ordinary and diagnostic queries caused the retrieval layer to pull restricted documents into context, and identity switches did not change that result set |
| OWASP LLM Top 10 (2025) | LLM02:2025 | Sensitive Information Disclosure | Restricted document names were written to the retrieval log, and those documents were fetched into the model context even when Bastion refused to display them |
| OWASP LLM Top 10 (2025) | LLM08:2025 | Vector and Embedding Weaknesses | The vector store had no document-level authorization and no tenant partition, so any query could surface confidential chunks |
| CWE | CWE-862 | Missing Authorization | Retrieval did not check whether the caller was allowed to read Client Contracts or Employee Data |
| CWE | CWE-639 | Authorization Bypass Through User-Controlled Key | `QUERY AS` takes a name from user input, and retrieval was not bound to a server-side session or tenant |
| CWE | CWE-532 | Insertion of Sensitive Information into Log File | `SHOW LOGS` wrote the names of restricted documents into `/var/log/bastion/retrieval.log` |
| CWE | CWE-778 | Insufficient Logging | Log entries had no user identity and no access decision, so they could not support an audit of who retrieved what |
| CWE | CWE-653 | Improper Isolation or Compartmentalization | Public policies and restricted documents lived in the same retrievable corpus, with no per-user or per-tenant namespace |

I used ATLAS v2026.10. `AML.T0057` (LLM Data Leakage) was moved in that release and is no longer a current ID, so I did not map to it.

## Takeaways

The lesson I took from this room is that **a refusal in the chat is not an access-control check.** If the retriever already fetched Client Contracts and Employee Data, the sensitive material is in the prompt, in the logs, and in any downstream trace, even when the assistant says it cannot serve restricted documents.

I also needed the logs as evidence, not just as a second finding. `QUERY AS` never handed me another user's records in the reply. What proved the isolation failure was that the retriever pulled the same restricted set for alice, Admin and my own session, and never recorded who was asking. When you test an internal assistant, compare what it says with what the retrieval layer actually loaded.

---

*Write-up by Marcin Korczak, based on a TryHackMe lab, for educational purposes.*
