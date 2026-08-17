---
name: auto-audit-security
description: "Use when a codebase needs an adversarial security review or compliance readiness assessment — web UI and API security, authentication and access control, licensing or paywall bypass, secrets exposure, dependency vulnerabilities, and SOC 2 / ISO 27001 / HIPAA / PCI DSS / GDPR readiness; when the user mentions security audit, pen test, SOC 2, compliance, Vanta, hardening, or says \"/auto-audit-security\"; or when about to call a system secure or compliant."
---

# /auto-audit-security

An adversarial security review of this codebase, plus an honest compliance-readiness assessment —
findings first, fixes only when you approve them.

**The boundary, stated once and never blurred: this skill cannot make anything compliant or
certified.** SOC 2 is an attestation issued by a licensed CPA firm after examining an organization
over a period of time. ISO 27001 is certified by an accredited body. What this skill produces is a
**readiness assessment**: which technical controls are evidenced in the code, which are missing, and
what an auditor will ask for that no repository can answer. Anyone who tells you a scan produces SOC 2
is selling something.

## Step 1 — Scope And Authorization

Establish before running anything:

1. **What is in scope** — repo only, or deployed environments too? Which environments?
2. **Authorization for active testing.** Static review of code you own needs none. **Any active scan
   (ZAP, nuclei, fuzzing, credential testing) requires explicit authorization for the specific target,
   and must never be pointed at production, at shared infrastructure, or at a third party.** Staging,
   or a local instance, or not at all.
3. **Which frameworks matter** — SOC 2, ISO 27001, HIPAA, PCI DSS, GDPR/CCPA, or none. Ask; do not
   assume, since the applicable set drives everything downstream.
4. **Where findings may be written.** Not a public issue tracker. See Handling below.

## Step 2 — Map The Attack Surface

You cannot audit what you have not enumerated. Produce the inventory first:

- **Every route/endpoint**, with method, and whether it is authenticated
- **Every trust boundary** — where untrusted input enters, where privilege changes
- **Every authentication path**, including recovery, SSO, API keys, and service-to-service
- **Every entitlement check** — what gates a paid feature, and where that check executes
- **All data stores**, and what class of data each holds (PII, credentials, payment, health)
- **All third-party integrations** and what each is trusted with
- **The shipped client bundle** and any mobile binary, treated as fully readable by an attacker

This inventory is the denominator for everything after it. A review that samples routes will miss the
one unprotected endpoint, and that is the one that matters.

## Step 3 — Access Control, Route By Route

**This is the class scanners systematically miss, and the class most likely to be exploitable.** It
cannot be sampled and it cannot be delegated to a tool.

For **every** route in the inventory, answer explicitly:

- Does it authenticate? Does it *authorize*, separately?
- Is the authorization check **server-side**? A check in the UI layer is not a check.
- Does it verify the caller owns the object it operates on, or only that the caller is logged in?
  (That gap is IDOR/BOLA, and it is the most common serious finding in real applications.)
- Can a parameter change the identity or scope of the operation (mass assignment, tampered
  `user_id`, tampered price or quantity)?
- Are administrative functions protected by a role check on the server, or only unlinked in the UI?

Produce a table with a row per route and an explicit verdict per column. "Probably fine" is not a
verdict — trace the code path to the check, or record it as unverified.

## Step 4 — Licensing And Paywall Bypass

Treat the client as fully hostile and fully readable — it is.

- **Entitlement evaluated client-side.** If the client decides whether the user is premium, the user
  decides whether the user is premium. The server must gate the data, not just the UI.
- **Feature flags that only hide UI.** Hidden is not disabled. Call the endpoint directly and see what
  comes back.
- **Trial or license state stored on the device** — local storage, preferences, a file, a registry key.
  All user-writable.
- **In-app purchase receipts validated on device.** Receipt validation belongs server-side, against the
  store's API.
- **Secrets in the shipped bundle**: API keys, signing keys, admin endpoints, or private hostnames in
  JS bundles, source maps, or a mobile binary. Extract and grep the actual artifact rather than the
  source tree.
- **Premium data on unauthenticated endpoints** — the paywalled screen calls an API; call it directly
  without a session.
- **Quota and rate limits enforced only in the client.**
- **Price, quantity or plan sent from the client** and trusted by the server.

For each: the correct design is that the **server independently determines entitlement on every
request** from state the user cannot write.

## Step 5 — The Rest Of The Classes

Cover the current OWASP Top 10 and API Security Top 10 — **pull the current edition and its category
codes at audit time rather than reciting them, because the editions and codes change** and a stale
category reference undermines the report.

By class, with the concrete question to answer in this codebase:

- **Injection** — SQL/NoSQL/command/template/LDAP. Is every query parameterized? Is any shell command
  built from input?
- **Authentication** — session fixation, rotation on privilege change, lifetime, logout invalidation.
  JWTs: is the signature actually verified, is `alg` pinned, are `exp`/`aud`/`iss` checked, is the
  secret strong? OAuth/OIDC: strict `redirect_uri` matching, `state`, PKCE.
- **Password storage** — a current memory-hard KDF with current parameters. Verify against present-day
  guidance rather than a number remembered from years ago.
- **Cryptographic failures** — legacy algorithms, ECB mode, static or reused IVs, homegrown crypto, keys
  in source, missing TLS verification, certificate pinning that was disabled "temporarily".
- **SSRF** — any server-side fetch of a user-supplied URL, including webhooks, previews and importers.
- **XSS and CSP** — output encoding by context, `dangerouslySetInnerHTML` and equivalents, and whether
  CSP is real or `unsafe-inline`.
- **CSRF and SameSite**, on every state-changing endpoint that accepts cookies.
- **Deserialization, XXE, file upload** — type and size validation, storage location, execution
  prevention.
- **Logging** — are credentials, tokens or PII being logged? Is there enough logging to reconstruct an
  incident? Both failures are findings.
- **Error handling** — stack traces or internal paths returned to clients.

## Step 6 — Run The Scanners, Then Distrust Them

Automation is the floor, not the audit. It is good at known-vulnerable dependencies and hardcoded
secrets, and poor at logic and access control.

```bash
# SAST
pipx run semgrep scan --config=auto --sarif -o semgrep.sarif .
pipx run bandit -r src/ -f json -o bandit.json          # Python
gosec -fmt=json -out=gosec.json ./...                    # Go

# dependencies — prefer reachability where available
osv-scanner scan source -r .
govulncheck ./...                                        # Go, reachability-aware
pip-audit -r requirements.txt
cargo audit
npm audit --omit=dev
trivy fs --scanners vuln,secret,misconfig .

# secrets — history, not just the working tree
gitleaks detect --source . --redact --report-format sarif --report-path gitleaks.sarif
trufflehog git file://. --no-verification --results=unverified,unknown

# IaC / container
checkov -d . --compact
trivy config .
```

Two rules about the tooling:

**`trufflehog` verification mode makes outbound requests using the credentials it finds.** That is
sometimes what you want and sometimes a disclosure. Default to `--no-verification` unless liveness
checking is explicitly authorized.

**A security gate must fail closed.** Check exit codes deliberately — a pipeline that treats any
non-zero as "tool error, continue" turns a finding into a pass.

**Active scanning (ZAP, nuclei) only against an authorized non-production target**, and never as a
default step.

Then triage: reachability beats raw severity, and a CVE in a dev-only dependency that never ships is
not the same finding as one in the shipped artifact. Say which is which.

## Step 7 — Compliance Readiness, Honestly

For each framework in scope, **read the current criteria from the authoritative source at audit time**
rather than reciting identifiers from memory, then classify every control:

| Verdict | Meaning |
|---|---|
| **Evidenced in code** | The control exists and you can point to it — file, config, or CI job |
| **Partially evidenced** | Present but incomplete or inconsistently applied |
| **Not evidenced** | Applicable, absent |
| **Not answerable from a repository** | Organizational — HR onboarding and offboarding, vendor management, physical security, policy documents, board oversight, incident-response process, background checks, training |

That last row is the honest half, and it is usually the larger half. Say so plainly: a repository can
speak to change management, access control, encryption in transit and at rest, logging and monitoring,
vulnerability management, and secure development practices. It cannot speak to whether offboarding
actually revokes access, whether vendors were assessed, or whether anyone reviewed the policy.

Where the code *can* provide evidence, name the artifact an auditor would accept: branch protection
settings, required reviews, CI gates, the dependency-scanning job, encryption configuration, log
retention, access-control tests.

**Never output the words "compliant", "certified", or "passing" about a framework.** The output is
readiness, gaps, and the evidence you can hand an auditor.

## Step 8 — The Report

Write to `.audit/security-report.md` — and treat it as sensitive (see Handling).

Per finding:

```
id · title · severity (with the reasoning, not just a label) · CWE where it applies
location: file:line, or endpoint + method
precondition: what an attacker needs (unauthenticated? a free account? a specific role?)
reproduction: the minimal concrete steps
impact: what is actually reachable — data, funds, privilege
remediation: the specific change, not "validate input"
verification: the test that proves it fixed
```

Order by **exploitability and impact**, not by scanner severity. An unauthenticated IDOR exposing
customer records outranks a dozen medium-severity dependency advisories in code that never executes.

Separate the compliance-readiness section, with the four-way verdict per control and a clear statement
of scope and method.

State coverage honestly at the top: what was reviewed, what was scanned, what was skipped, and what
could not be verified without an environment you did not have.

## Step 9 — Then Ask

**Present the report and ask what to fix.** Do not start editing.

Offer, in order:

1. **Critical and exploitable now** — recommend fixing these immediately, one at a time, with a
   regression test per fix that fails before and passes after.
2. **Structural** — an access-control pattern applied in one place should be applied at a chokepoint;
   propose the refactor rather than patching each caller.
3. **Hygiene** — dependency bumps, header hardening, lint rules that prevent recurrence.
4. **Compliance gaps** with a code answer — CI gates, branch protection, logging, encryption config.

For each fix applied: verify it, and **add the check that prevents recurrence** — a test, a CI gate, a
lint rule. A security fix without a gate returns.

## Handling The Report

A document describing live, exploitable vulnerabilities is one of the most sensitive artifacts a
project has.

- **Never commit it to a public repository**, and never paste findings into a public issue, PR, or chat.
- Keep it out of the working tree if the repo is public — write it somewhere gitignored and say where.
- Do not put reproduction detail for an unfixed vulnerability into any tracker that is not private.
- Redact credentials, tokens and customer data from every excerpt.
- When a finding is fixed and released, the detail can be shared; before that it is a weapon.

If findings involve customer data exposure, note that breach-notification obligations may have clocks
attached, and that determination belongs to counsel — flag it, do not adjudicate it.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "The scan came back clean" | Scanners barely find access-control bugs. That is the class that gets exploited |
| "We're SOC 2 compliant now" | An audit firm says that after examining the organization. This is readiness |
| "The UI hides it from free users" | Hidden is not disabled. Call the endpoint |
| "The client checks the license" | Then the user controls the license. Gate the data server-side |
| "The API key is only in the mobile app" | The binary is readable. That key is public |
| "It's behind login, so it's fine" | Authentication is not authorization. Check object ownership |
| "It's an internal endpoint" | Verify that claim from the network config, not the name |
| "I'll run ZAP against prod to be thorough" | Never. Authorized non-production targets only |
| "That CVE is in a dev dependency" | Then say so and rank it accordingly — but confirm it does not ship |
| "I'll open an issue with the repro" | Not in a public tracker for an unfixed vulnerability |
| "Severity is medium, the scanner said so" | Rank by exploitability and impact in *this* system |
| "We can fix it later, it's theoretical" | Then demonstrate it is unreachable. Otherwise it is a finding |

## Red Flags — Stop

- Using the words "compliant" or "certified" about any framework
- Running an active scan without explicit authorization for that specific target
- Pointing any scanner at production
- Sampling routes for access control instead of enumerating all of them
- Reciting a framework criterion id or OWASP category code without checking the current edition
- Committing the findings report to a public repo, or putting a repro in a public issue
- A CI security gate that continues on a non-zero exit
- `trufflehog` with verification enabled against credentials you did not intend to test
- Fixing anything before the user has seen the report and chosen
- Declaring a fix done without a regression test and a gate against recurrence
