# 056 — SSH Keys

**VERIQTA Linux World | Public-Facing Junior Engineer Follow-Along Lab**

**Canonical path:** `Junior/03-Networking/JUN-056-SSH-Keys/README.md`  
**Environment:** Ubuntu Server 24.04 LTS; non-root account; an isolated or explicitly authorized lab machine  
**Level:** Junior Engineer, progressing from fundamentals to production evidence  
**Duration:** 5–8 hours of reading, execution, evidence collection, and reporting (variable by experience)  
**Changes:** No production configuration changes; all temporary lab resources are confined to an authorized local environment  
**Prerequisites:** All preceding JUN networking labs through JUN-055  
**Next:** JUN-057 — SSH Configuration Basics

> **How to use this lab:** Follow every step on your own Ubuntu lab VM. Commands that include placeholders such as `LAB_HOST` must be adapted to your authorized environment. Output blocks are **illustrative**, not evidence from your host. Record your real results in the report. Never publish real private IPs, hostnames, usernames, fingerprints, command arguments containing secrets, or internal topology without approval. This is a teaching resource for everyone using VERIQTA Linux World, not an instruction to alter production systems.

## Operating rules and evidence standard

1. **Read before running.** Understand the effect of a command, its privileges, and whether it sends traffic or changes a file.
2. **Stay within scope.** Use loopback or an authorized training host. Never probe third-party systems or enumerate networks without permission.
3. **Protect access.** Never change production SSH listeners, firewall rules, default routes, DNS settings, or authentication policies in these junior labs.
4. **Use bounded operations.** Prefer explicit connection timeouts, overall deadlines, and small request counts; avoid flooding, stress testing, or unattended loops.
5. **Preserve evidence.** Timestamp each observation in UTC, include the exact command, and distinguish *Observed*, *Illustrative*, *Hypothesis*, and *Confirmed*.
6. **Protect credentials.** Do not put passwords, private keys, tokens, cookies, authorization headers, or raw internal configuration in public reports or shell history.
7. **Verify cleanup.** Remove only resources you created; never kill unrelated processes. Report any remaining temporary file or service.

**Safe starting point:**

```bash
cat /etc/os-release
hostname
id
ip -br addr show
ip -4 route show
mkdir -p "$HOME/veriqta-linux-labs/JUN-056-SSH-Keys"
cd "$HOME/veriqta-linux-labs/JUN-056-SSH-Keys"
date -u '+%Y-%m-%dT%H:%M:%SZ'
```

**Checkpoint:** Identify your host, user, interfaces, lab directory, and whether you are working locally or over SSH. If remotely connected, do not perform an experiment that could interrupt the session.

## 1. Production mission: a key was exposed and access must be restored safely

**Simulated incident:** A developer accidentally commits an SSH private key to a repository. The team removes the file in a later commit and assumes the incident is over. It is not: Git history and clones may still contain the secret. Your mission is to understand key pairs, generate a **lab-only** identity, inspect fingerprints, authenticate to an approved lab account if available, diagnose common failures, and design a secure revocation and rotation workflow. Do not reuse the training key for real accounts. Never publish private keys or paste their contents into chat, tickets, or repositories.

**Deliverable:** an SSH key lifecycle report covering generation, protection, authorization, use, agent handling, incident response, revocation, verification, and cleanup.

## 2. Understand public-key authentication and cryptographic roles

An SSH key pair has a private key and corresponding public key. The private key is kept secret; the public key may be distributed to approved servers or identity systems. During public-key authentication, the client proves possession of the private key by signing protocol data; it does **not** send the private key to the server. A passphrase protects a private key stored on disk but does not replace server-side authorization. Modern OpenSSH supports Ed25519 keys and other algorithms; organizational/FIPS policies can constrain acceptable types. The **host key** that authenticates a server is separate from the **user authentication key** used to log in.

**Checkpoint:** explain why copying your private key into `authorized_keys` is both incorrect and dangerous.

## 3. Inspect existing keys safely, without opening secrets

Run:

```bash
ssh -V
ssh-keygen -h 2>&1 | head -n 15 || true
ls -ld "$HOME/.ssh" 2>/dev/null || true
find "$HOME/.ssh" -maxdepth 1 -type f -printf '%f
' 2>/dev/null || true
```

`ssh-keygen -h` behavior may vary; use `man ssh-keygen` or `ssh-keygen -?` to consult usage if necessary. File names may include `id_ed25519`, `id_ed25519.pub`, `known_hosts`, and `config`, but never assume a particular file exists. **Do not `cat` a private key.** If you find an existing identity, do not overwrite it, rename it, or change its permissions as part of this lab. **Checkpoint:** distinguish `.pub` public-key files from private-key files and known-host data.

## 4. Create a dedicated lab-only key directory

Use a private workspace with restrictive permissions:

```bash
mkdir -p "$HOME/veriqta-linux-labs/JUN-056/keys"
chmod 700 "$HOME/veriqta-linux-labs/JUN-056/keys"
cd "$HOME/veriqta-linux-labs/JUN-056"
umask 077
pwd
```

`umask 077` affects new files in the current shell session. The directory is not `~/.ssh` because this lab should not accidentally replace a working identity. **Checkpoint:** verify directory mode with `stat -c '%a %n' keys`. Expected `700`. Never commit `keys/` to Git.

## 5. Generate an Ed25519 training key with a passphrase

Run on your own lab VM:

```bash
ssh-keygen -t ed25519 -a 100   -f "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519"   -C 'veriqta-jun056-lab-only'
```

`-t ed25519` selects the algorithm; `-a 100` sets the number of key derivation function rounds for OpenSSH private-key passphrase protection; `-f` chooses the output file; `-C` adds a comment. Enter a unique training passphrase when prompted; **do not put it on the command line**. The passphrase protects the private key at rest; it is not sent to the server. If your environment's security policy prohibits Ed25519, follow the organization's approved algorithm and record the deviation rather than overriding policy. **Checkpoint:** confirm two files exist: `jun056_ed25519` (private) and `jun056_ed25519.pub` (public).

## 6. Verify permissions and metadata

Run:

```bash
stat -c '%a %n' keys/jun056_ed25519 keys/jun056_ed25519.pub
ls -l keys/
```

Private keys should be readable only by their owner; mode `600` is a common result under restrictive umask. Public keys are not secrets but the containing directory should remain protected. Fix permissions **only on your newly created lab key** if needed:

```bash
chmod 600 keys/jun056_ed25519
chmod 644 keys/jun056_ed25519.pub
```

Do not run recursive chmod against a production home directory. **Checkpoint:** explain why SSH may refuse to use a private key accessible to other users.

## 7. Inspect the public key and fingerprint

Run:

```bash
cat keys/jun056_ed25519.pub
ssh-keygen -lf keys/jun056_ed25519.pub -E sha256
```

The `.pub` file is safe to inspect in the local lab. It typically contains algorithm, base64-encoded public-key material, and comment. `ssh-keygen -l` prints fingerprint; `-f` selects the file; `-E sha256` selects fingerprint hash. The fingerprint is a compact identifier for comparing public keys; it is not a secret, but it can reveal which identity is in use. Record your actual fingerprint in your **private** report, and use a redacted or simulated value in public examples. **Checkpoint:** explain why a key fingerprint is not the private key.

## 8. Validate the key pair without exposing private material

Run:

```bash
ssh-keygen -y -f keys/jun056_ed25519 > keys/derived_public.pub
ssh-keygen -lf keys/derived_public.pub -E sha256
ssh-keygen -lf keys/jun056_ed25519.pub -E sha256
```

`ssh-keygen -y` derives the public key from the private key and may prompt for its passphrase. Compare the fingerprints: they should match. This validates correspondence but does not prove the key is authorized on a remote server. Remove `derived_public.pub` after recording the result if you wish. **Checkpoint:** distinguish cryptographic key-pair validity from account authorization.

## 9. Understand authorized_keys and remote authorization

A server commonly permits a public key by placing it in the intended account's `~/.ssh/authorized_keys` or by consulting an approved central identity system. A key added to the wrong account will not authorize the intended username. Server policies, certificate requirements, authorized principals, restrictions, and permissions can affect acceptance. The private key stays on the client. **Do not edit a production `authorized_keys` file in this lab.** If you have an approved training account and permission to install your lab public key, follow that environment's documented onboarding procedure and record who authorized it. **Checkpoint:** explain why placing a public key on a server is an access-control change.

## 10. Authenticate with an explicitly selected identity

Only if you have an approved SSH lab server/account **and your lab public key is authorized**, run:

```bash
ssh -i "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519"   -o IdentitiesOnly=yes -o ConnectTimeout=5   USER@APPROVED_HOST 'id -un; hostname'
```

`-i` chooses the private identity file; `IdentitiesOnly=yes` limits identity selection to configured/explicit identities and relevant certificate behavior; `ConnectTimeout` bounds connection setup. Independently verify the server host key before accepting a new host. If no authorized server is available, **do not manufacture a successful login**; document that generation and local validation were completed, and use the supplied simulated server evidence for authentication analysis. **Checkpoint:** record which stage succeeded and whether the remote command was actually executed.

## 11. Understand ssh-agent and safe agent usage

Inspect whether an agent is available:

```bash
ssh-add -l
printf 'agent_socket_set=%s
' "${SSH_AUTH_SOCK:+yes}"
```

`ssh-add -l` may report no agent or no identities; both are valid. For an **isolated lab shell**, you may start a temporary agent and load the training key with a lifetime:

```bash
eval "$(ssh-agent -s)"
ssh-add -t 300 "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519"
ssh-add -l
```

This loads the key for roughly five minutes, subject to agent behavior; you may be prompted for the passphrase. **Never paste private key text into the agent.** Agent forwarding (`ssh -A`) introduces additional trust risks; do not enable it for this lab. Record the agent PID printed when starting it so you can stop **your own agent**. **Checkpoint:** explain why an agent is a convenience and trust boundary, not a replacement for server authorization.

## 12. Remove the lab identity and stop only your agent

If you started a dedicated lab agent, remove the identity and stop the agent:

```bash
ssh-add -d "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519" 2>/dev/null || true
ssh-agent -k
```

**Do not run `ssh-agent -k` against a pre-existing shared agent you did not start.** If you did not start a dedicated agent, skip this step. Agent cleanup does not revoke server-side authorization. **Checkpoint:** distinguish removal from local agent memory from revocation of access on the server.

## 13. Diagnose wrong identity and too many authentication attempts

A client may offer the wrong key, or too many identities, before reaching the intended one. Inspect evaluated configuration:

```bash
ssh -G -i "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519"   -o IdentitiesOnly=yes USER@APPROVED_HOST   | grep -E '^(user|hostname|identityfile|identitiesonly|port) '
```

This command does not connect. For authorized troubleshooting, use `ssh -vv` and redact output. A key being **offered** is not proof it was **accepted**. The server can reject it due to account policy or missing authorization. **Checkpoint:** identify the exact client-side identity and remote username used.

## 14. Diagnose key-file permissions and ownership

**Simulated symptom:** `WARNING: UNPROTECTED PRIVATE KEY FILE!` or a client refuses to load the key. **Possible cause:** permissions expose the private key to other users. For your training key only, inspect `stat`, owner, and directory permissions; correct the single file if necessary. Do not apply `chmod -R` to a production `.ssh` directory. On the server, ownership and modes of home directory, `.ssh`, and `authorized_keys` may also matter under StrictModes; a server administrator must verify these within policy. **Root cause** is not confirmed from client error alone if server-side authorization is also involved.

## 15. Understand passphrases, backup, and key handling

A strong passphrase reduces exposure if an encrypted private key file is stolen, but it cannot protect a key already unlocked in memory or copied from an untrusted endpoint. Avoid reusing passphrases. Store production private keys only in approved protected locations or hardware-backed systems; never email or chat them. Backup and recovery policies are organization-specific. For this training key, keep the private file within the restricted lab directory and remove it after completing the exercises. **Checkpoint:** list three places private key material must not appear.

## 16. Understand key rotation and revocation

Key rotation means provisioning a replacement identity and retiring the old one through an approved process. Revocation means removing or invalidating authorization for a compromised or retired credential, possibly through `authorized_keys`, a certificate authority, central identity system, or access-control platform. Merely deleting the local private key does **not** revoke a copy that an attacker obtained. A safe migration typically: authorize new key → verify new login → remove old authorization → verify old identity fails → monitor access logs → document completion. The exact sequence must follow organizational change control, especially for break-glass accounts. **Checkpoint:** explain why deleting a committed secret from the latest Git commit is insufficient.

## 17. Production incident: private key leaked in Git history

**Simulated timeline:** 09:00 key committed; 09:05 pushed; 09:15 removed from current branch; 09:30 security alert. **Impact:** key confidentiality is lost or cannot be trusted; clones and history may retain it. **Immediate response:** notify security/incident owner; identify affected accounts and authorized destinations; revoke the key's authorization; rotate replacement through approved workflow; review access logs for misuse; preserve incident evidence securely; assess whether history rewriting and secret-scanning remediation are required. **Do not paste the compromised key into the incident ticket.** Root cause may involve secret-handling failure and absent pre-commit controls; confirm with evidence. Recovery is not complete until old authorization is invalidated and the replacement works.

## 18. Production incident: public key installed for wrong account

**Symptom:** TCP and host-key verification succeed; user auth fails. **Evidence:** intended account is `deploy`, but the public key was provisioned to a different account. **Hypotheses:** wrong username, wrong authorization target, policy restriction, wrong identity. **Tests:** approved account mapping and server logs; effective client config. **Supported cause:** account mismatch if independently verified. **Authorized remediation:** provision public key to correct least-privilege account through approved process. **Verification:** intended account works, unintended account access is not granted.

## 19. Production incident: agent forwarding across a bastion

**Simulated risk:** an engineer enables agent forwarding to an untrusted intermediate host. A compromised host can potentially abuse forwarded agent access to request signatures while the forwarding is active, depending on controls. **Do not enable `ssh -A` in this lab.** Prefer approved bastion/ProxyJump designs and constrained identities or hardware-backed authentication. Record trust boundaries, destination restrictions, and policy requirements. The presence of an agent is not proof that forwarding is safe.

## 20. Integrated incident: rotate a compromised deploy identity

**Simulated environment:** `deploy@build-lab-01` uses a key found in a public repository. You are not asked to manipulate production access. Draft an approved runbook: (1) classify exposure and preserve evidence, (2) identify every account authorized for that public key, (3) coordinate incident owner, (4) create a new approved identity in a secure location, (5) provision public key, (6) verify host identity and least-privilege login, (7) revoke old key across every authorization location, (8) verify old key is rejected, (9) review access logs, (10) close with prevention controls and ownership. **Root cause status:** credential exposure confirmed in the scenario; unauthorized use not established without logs. **Checkpoint:** distinguish containment, recovery, and prevention.

## 21. Cleanup and final verification

If you installed the lab public key on an authorized training server, remove that authorization using the lab's approved procedure **before** deleting the local private key, and verify removal. If you started your own agent, stop it. Then remove only the lab key files you created:

```bash
rm -f "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519"       "$HOME/veriqta-linux-labs/JUN-056/keys/jun056_ed25519.pub"       "$HOME/veriqta-linux-labs/JUN-056/keys/derived_public.pub"
```

Do not remove real user keys or unrelated authorization. Document whether any lab server authorization remains (expected: none). **Checkpoint:** deleting local key files is not equivalent to revoking an identity that may have been copied.

## Command-and-flag reference — review immediately before the final challenge

| Command/option | Purpose | Safety note |
|---|---|---|
| `ssh-keygen -t ed25519` | Generate Ed25519 pair | Follow organizational algorithm policy |
| `-a 100` | KDF rounds | Protects encrypted private-key file |
| `-f FILE` | Output/input key path | Avoid overwriting existing keys |
| `-C COMMENT` | Public key comment | Comment is not an authorization control |
| `ssh-keygen -lf FILE -E sha256` | Print SHA-256 fingerprint | Fingerprint ≠ private key |
| `ssh-keygen -y -f FILE` | Derive public key | Prompts for passphrase if protected |
| `chmod 600 PRIVATE` | Restrict file mode | Apply only to lab key |
| `ssh -i PRIVATE` | Select user identity | Verify server host key separately |
| `-o IdentitiesOnly=yes` | Limit identity selection | Does not override server policy |
| `ssh-add -l` | List agent identities | Agent may be absent |
| `ssh-add -t 300 FILE` | Add identity temporarily | Only to your own agent |
| `ssh-add -d FILE` | Remove one identity | Does not revoke server authorization |
| `ssh-agent -k` | Stop agent | **Only** agent you started |
| `ssh -G` | Effective client config | Does not connect |
| `ssh -A` | Agent forwarding | **Do not use in this lab** |

## Troubleshooting decision tree — review immediately before the final challenge

```text
PUBLIC-KEY LOGIN FAILS
  |
  +-- TCP / host identity fails? --> JUN-055; stop if host key changed
  +-- Wrong username? --> confirm intended remote account
  +-- Wrong key offered? --> ssh -G; explicit -i + IdentitiesOnly
  +-- Private key unreadable? --> check owner, mode, passphrase, file
  +-- Key offered but rejected? --> approved authorized_keys/CA/policy/log review
  +-- Key exposed? --> INCIDENT RESPONSE: revoke + rotate + review access
  v
VERIFY NEW AUTHORIZATION -> VERIFY OLD KEY REVOKED -> DOCUMENT CLEANUP
```
**Never fix an authorization failure by copying private keys or disabling host authentication.**

## Integrated final challenge — evidence-based engineering assessment

**Deliverable:** `report/JUN-056-SSH-Key-Lifecycle-and-Compromise-Response.md`.

1. Inventory SSH tools and existing filenames without printing private key material.
2. Create a private lab directory; document `umask` and modes.
3. Generate a new passphrase-protected lab-only Ed25519 key; explain every option.
4. Record actual public-key fingerprint and compare it to a public key derived from the private key.
5. Explain host key vs user key vs `known_hosts` vs `authorized_keys`.
6. If approved training server exists, install public key through its onboarding process and authenticate with `-i` and `IdentitiesOnly`; otherwise document why not executed.
7. Inspect an agent; optionally create and clean up your own temporary agent.
8. Diagnose simulated wrong-key, wrong-account, permissions, and policy failures.
9. Write a stepwise private-key-exposure response with containment, revocation, rotation, audit, and prevention.
10. Verify no real credentials entered public artifacts and all temporary authorization removed.
11. Explain how to verify the old key **cannot** authenticate after revocation.
12. Submit a redacted report and cleanup record.

**Pass:** private key is never printed or shared; fingerprint and file permissions are correct; authentication is never conflated with host identity; the incident response revokes authorization rather than merely deleting local copies.

## Knowledge check with worked answers

**1. What is a key pair?**  
A mathematically related public key and private key.

**2. Does SSH send the user private key to the server?**  
No; it proves possession through cryptographic protocol operations.

**3. What is Ed25519?**  
A modern digital-signature algorithm supported by OpenSSH, subject to policy.

**4. What does `-t ed25519` select?**  
Key algorithm.

**5. What does `-a 100` affect?**  
Private-key passphrase KDF rounds in OpenSSH format.

**6. What does `-f` specify?**  
Key filename/path.

**7. What does `-C` specify?**  
A key comment, not a permission.

**8. What is a fingerprint?**  
A compact digest used to identify/compare public keys.

**9. Can a fingerprint replace the private key?**  
No.

**10. What does `ssh-keygen -y` do?**  
Derives public key from a private key.

**11. What should private-key file permissions be?**  
Owner-only access is typical, such as 600, with appropriate ownership.

**12. What is authorized_keys?**  
A common per-account server authorization file for public keys.

**13. Is known_hosts the same as authorized_keys?**  
No; known_hosts records server identity, authorized_keys permits user identities.

**14. What does a passphrase protect?**  
An encrypted private key at rest.

**15. Does deleting local key revoke server access?**  
No; copies may still authenticate while authorized.

**16. What does ssh-agent do?**  
Brokers signing with loaded identities.

**17. What does ssh-add -l show?**  
Identities currently available through the agent.

**18. Why use IdentitiesOnly?**  
To constrain which configured identities are offered.

**19. What does permission denied (publickey) prove?**  
The server did not accept the attempted user public-key authentication.

**20. Why is Git history a risk?**  
A removed secret can remain in prior commits and clones.

**21. What is revocation?**  
Invalidating server/identity-system authorization for a credential.

**22. What is rotation?**  
Provisioning replacement credential and retiring the old one.

**23. Why not copy a private key to a server?**  
It exposes secret signing material and violates intended trust separation.

**24. Why avoid agent forwarding by default?**  
A compromised intermediate host may misuse forwarded agent access.

**25. What verifies a successful compromise response?**  
Old key revoked, new access validated, logs reviewed, and exposure documented.

## Interview preparation with worked answers

**Interview 1: How would you generate a secure lab key?**  
Use approved algorithm, unique passphrase, restricted directory, and explicit output path; verify fingerprint and permissions.

**Interview 2: Why does a private key require protection?**  
Anyone possessing usable private material may impersonate the key holder where it is authorized.

**Interview 3: How do you verify a public/private pair?**  
Derive public key with `ssh-keygen -y` and compare fingerprints.

**Interview 4: What causes publickey denial?**  
Wrong user/key, missing authorization, policy restrictions, permissions, or expired/revoked credentials.

**Interview 5: What is wrong with committing a private key?**  
Repository history and clones can expose credentials; revoke and rotate.

**Interview 6: How do you rotate without losing access?**  
Follow approved change control: provision new, verify, revoke old, verify old rejection, audit.

**Interview 7: What does a passphrase not protect against?**  
A key already unlocked on a compromised endpoint or a leaked unencrypted copy.

**Interview 8: What is the difference between agent removal and revocation?**  
Agent removal changes local loaded identities; revocation changes authorization.

**Interview 9: How do you use an explicit identity?**  
`ssh -i FILE -o IdentitiesOnly=yes ...` after verifying host identity.

**Interview 10: What do you do if a key is exposed?**  
Escalate, contain, revoke, rotate, investigate access, and prevent recurrence.

## Portfolio submission, redaction, and completion criteria

Use this layout in your public portfolio:

```text
lab-submission/
├── README.md                   # what you tested, architecture, conclusions
├── evidence/
│   ├── README.md               # evidence index and sanitization policy
│   └── sanitized-observations.md
├── report/
│   └── incident-investigation.md
└── cleanup.md                  # resources created, stopped, and removed
```

Your final report must contain **(A) scope and authorization, (B) host and time, (C) expected behavior, (D) commands and observed outputs, (E) competing hypotheses, (F) discriminating tests, (G) supported root cause or explicitly unresolved status, (H) approved remediation plan, (I) recovery verification, (J) prevention, (K) limitations, and (L) cleanup record**.

**Acceptance criteria:**

- [ ] I ran each safe follow-along exercise that my lab supports, and explicitly marked unavailable exercises.
- [ ] I recorded actual outputs rather than copying illustrative examples as evidence.
- [ ] I explained each command's important options and why I used them.
- [ ] I differentiated network, transport, security, and application failures.
- [ ] I tested competing explanations rather than choosing the first plausible story.
- [ ] I did not treat one successful command as proof of complete service health.
- [ ] I documented the original user-visible operation to verify after any approved fix.
- [ ] I made no unauthorized production changes and disclosed any temporary lab artifacts.
- [ ] I sanitized all evidence intended for a public GitHub repository.
- [ ] I can answer the knowledge and interview questions without reading the answers.

**Engineering principle:** A good investigation says precisely what the evidence proves, what it cannot prove, and what you would test next. An incomplete result documented accurately is better than a confident invented conclusion.

## Next lab

**JUN-057 — SSH Configuration Basics**. Continue in sequence; carry forward your evidence discipline and the distinction between connectivity, authentication, configuration, and application health.

**End of 056 — SSH Keys.**
