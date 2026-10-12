# 055 — SSH Fundamentals

**VERIQTA Linux World | Public-Facing Junior Engineer Follow-Along Lab**

**Canonical path:** `Junior/03-Networking/JUN-055-SSH-Fundamentals/README.md`  
**Environment:** Ubuntu Server 24.04 LTS; non-root account; an isolated or explicitly authorized lab machine  
**Level:** Junior Engineer, progressing from fundamentals to production evidence  
**Duration:** 5–8 hours of reading, execution, evidence collection, and reporting (variable by experience)  
**Changes:** No production configuration changes; all temporary lab resources are confined to an authorized local environment  
**Prerequisites:** All preceding JUN networking labs through JUN-054  
**Next:** JUN-056 — SSH Keys

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
mkdir -p "$HOME/veriqta-linux-labs/JUN-055-SSH-Fundamentals"
cd "$HOME/veriqta-linux-labs/JUN-055-SSH-Fundamentals"
date -u '+%Y-%m-%dT%H:%M:%SZ'
```

**Checkpoint:** Identify your host, user, interfaces, lab directory, and whether you are working locally or over SSH. If remotely connected, do not perform an experiment that could interrupt the session.

## 1. Production mission: remote access fails after a deployment

**Simulated incident:** An engineer cannot connect to `bastion.lab.example` using SSH. Monitoring says the host is online, but the client reports `Permission denied (publickey)`. Another team member says to disable host-key checking or enable password authentication globally. Neither suggestion is justified. Your mission is to distinguish name resolution, route, TCP reachability, server availability, host identity, user authentication, and remote shell behavior. **Never weaken authentication to make a diagnostic pass.**

| Stage | Example failure | Evidence needed |
|---|---|---|
| DNS | Could not resolve hostname | `getent`, resolver evidence |
| TCP | Connection refused / timed out | endpoint, route, `ss` where authorized |
| Host identity | Unknown/changed host key | approved fingerprint out-of-band |
| User authentication | Permission denied | account, permitted auth methods, authorized logs |
| Session | Connection closes after auth | shell policy, server logs, resource state |

The deliverable is an SSH access investigation with exact client commands, safe verification, and escalation criteria.

## 2. Understand SSH architecture and trust boundaries

SSH (Secure Shell) is a family of protocols for authenticated, encrypted remote access. An SSH client establishes a connection to an SSH server, negotiates algorithms, authenticates the server using a host key, authenticates the user using a permitted method, and opens a session or another authorized channel. TCP port 22 is conventional, not mandatory. The SSH transport protects confidentiality and integrity but cannot guarantee that a compromised endpoint behaves honestly.

**Two different identities:** the server's **host key** helps the client authenticate the server; the user's **authentication key** can prove the user is authorized. Never confuse them. SSH can also support command execution, file transfer, and forwarding, but this lab focuses on safe fundamentals. **Checkpoint:** explain why a valid user private key does not remove the need to verify the server's host key.

## 3. Inventory the SSH client and local environment

Run:

```bash
command -v ssh
ssh -V
command -v ssh-keygen
command -v ssh-add
id -un
hostname
getent passwd "$(id -un)"
ls -ld "$HOME" "$HOME/.ssh" 2>/dev/null || true
```

`ssh -V` commonly prints its version to stderr; record it accurately. `ssh-keygen` and `ssh-add` are installed with common OpenSSH client tooling, but their presence does not prove an agent is running. The `.ssh` directory may not exist on a new account. **Do not create credentials or modify configuration merely to pass this inventory.** Record user, host, client version, and whether an authorized lab SSH server is available.

## 4. Determine whether an SSH server exists without changing it

On your **own lab VM**, inspect:

```bash
ss -ltn '( sport = :22 )'
systemctl status ssh --no-pager 2>&1 | head -n 35
```

On Ubuntu the service unit is often named `ssh`, while the daemon executable is `sshd`. `systemctl` may report that the unit is absent or inactive, especially in containers. This is a valid observation. **Do not start, enable, or install an SSH server on production.** If you have an approved training server, ask the lab administrator for hostname, port, username, and independently verified host-key fingerprint. Never assume localhost has an SSH server.

## 5. Read the structure of an SSH command

General form:

```bash
ssh -p 22 -o ConnectTimeout=5 -o BatchMode=yes USER@APPROVED_HOST
```

`-p` selects destination port; `-o` sets a client option; `ConnectTimeout` bounds connection setup; `BatchMode=yes` prevents interactive password/passphrase prompting, useful for a non-destructive connectivity test. `USER@HOST` chooses the remote login identity and destination. **Do not run the placeholder literally.** For your authorized host, begin with a normal interactive login when you expect an approved prompt, or use a bounded BatchMode check if you do not want to attempt interactive credentials. **Checkpoint:** distinguish the local user from the remote login username.

## 6. Safely inspect resolved addresses and route selection

Use your approved SSH hostname:

```bash
getent ahostsv4 APPROVED_HOST
ip -4 route get APPROVED_IP
```

Replace placeholders. `getent` queries name service; `ip route get` reports local route selection. Neither proves SSH is available. If DNS fails, do not change system resolver configuration. If a route is absent, record the exact error and escalate through approved network operations. IPv6 paths may differ; use `getent ahostsv6` and `ip -6 route get` when appropriate. **Checkpoint:** state what each command proves and does not prove.

## 7. Inspect TCP reachability without confusing it with authentication

A bounded SSH connection attempt can distinguish several failure classes:

```bash
ssh -vv -o BatchMode=yes -o ConnectTimeout=5 -p 22 USER@APPROVED_HOST true
```

Only use an explicitly approved host/account. `-vv` increases client diagnostics, `BatchMode=yes` avoids interactive credential prompts, and `true` is a harmless remote command **if authentication succeeds and command execution is authorized**. The command may fail because no key is configured; that is not necessarily a network problem. Verbose output can reveal usernames, hostnames, paths, and fingerprints: sanitize before publication. For a lab host where command execution is not approved, omit `true` and follow the authorized login procedure. **Checkpoint:** identify the last successful stage from the verbose trace.

## 8. Understand first-contact host-key verification

On first contact, OpenSSH may ask whether you trust a host key fingerprint. This is a **security decision**, not a cosmetic prompt. Obtain the expected fingerprint from an administrator or trusted inventory over an independent channel; compare it before accepting. A DNS name alone does not authenticate the remote machine. Do not use `StrictHostKeyChecking=no`, `UserKnownHostsFile=/dev/null`, or a blanket deletion of `known_hosts` to suppress warnings. If you cannot verify the fingerprint, stop and request confirmation.

**Controlled inspection:**

```bash
ls -l "$HOME/.ssh/known_hosts" 2>/dev/null || true
ssh-keygen -F APPROVED_HOST 2>/dev/null || true
```

`ssh-keygen -F` searches known-host entries and can handle hashed hostnames; output is not proof that a currently reachable server has the same key. **Checkpoint:** explain the difference between a key in `known_hosts` and a fingerprint independently verified by your organization.

## 9. Understand a changed-host-key warning

A changed-host-key warning can result from legitimate host replacement, a reinstalled OS, changed DNS/routing, load balancer changes, or an active impersonation attempt. **Do not click through it or remove the old entry without verification.** Record host, address, time, presented fingerprint, and expected fingerprint from approved inventory. Contact the responsible team. Only after independently verifying a legitimate change should an authorized engineer update the exact known-host entry. A changed host key is a trust failure, not an authentication-method problem.

## 10. Learn SSH authentication methods and policy

SSH user authentication can involve public keys, passwords, keyboard-interactive methods, certificates, and organization-specific policies. The server controls which methods are accepted. Passwordless login usually means public-key authentication without typing a remote account password; it does **not** mean authentication is absent. An SSH agent can hold decrypted key material or broker signatures but does not itself grant authorization. MFA and hardware-backed keys may be required by policy. Never enable password authentication globally just to troubleshoot a public-key failure. **Checkpoint:** list three reasons why `Permission denied (publickey)` can occur without any network outage.

## 11. Understand remote identity, command execution, and exit status

After successful authorized login, inspect the remote environment using a bounded command:

```bash
ssh -o ConnectTimeout=5 -p 22 USER@APPROVED_HOST 'id -un; hostname; pwd'
printf 'ssh_exit=%s
' "$?"
```

Use an approved lab account. The quoted string runs on the remote host, not your local machine. A nonzero SSH exit status may reflect connection/authentication failure or the remote command's exit code. Avoid sending sensitive shell commands, editing files, or assuming remote shell permissions. **Checkpoint:** distinguish local command execution from remote execution and identify where the output originated.

## 12. Read verbose logs responsibly

SSH supports `-v`, `-vv`, and `-vvv` for increasing verbosity. Example:

```bash
ssh -vv -o BatchMode=yes -o ConnectTimeout=5 USER@APPROVED_HOST true 2> ssh-client-debug.private.log
```

This is appropriate only for an authorized target and account. **Do not commit the raw log**: it can expose infrastructure and user information. Extract a sanitized stage timeline instead: DNS resolved → TCP connected → server host key checked → offered auth method → accepted/denied → session opened/closed. A message about an offered key does not prove the server accepted it. **Checkpoint:** identify the line indicating actual authentication success, not merely an offered identity.

## 13. Understand host keys, known_hosts, and user keys

The server has host private/public keys; the client checks the server's public host identity against its trust information. Separately, a user may hold a private authentication key and place its corresponding public key in an authorized server-side account configuration. `known_hosts` is **not** the same as `authorized_keys`. A private user key must never be copied to the server or placed in a public repository. A host key warning cannot be fixed by adding a user key. This separation is essential for diagnosing trust and access failures.

## 14. Examine SSH client configuration without editing

Read effective client configuration for an approved target:

```bash
ssh -G -p 22 USER@APPROVED_HOST | grep -E '^(hostname|user|port|proxyjump|identityfile|identitiesonly|stricthostkeychecking|connecttimeout) '
```

`ssh -G` prints evaluated client configuration and exits without connecting. It is useful for detecting unexpected usernames, ports, identity paths, or proxy jumps. The result can include private path names; sanitize it. Do not infer server policy from client settings. **Checkpoint:** explain why an incorrect `Host` alias can send SSH to the wrong destination.

## 15. Understand jump hosts and forwarding without enabling them

In enterprise networks, an SSH bastion or jump host may mediate access. A ProxyJump configuration changes the connection path and can introduce separate DNS, TCP, authentication, and host-key checks for the jump and final hosts. SSH also supports local, remote, and dynamic forwarding, but forwarding may cross trust boundaries or bypass intended access controls. **Do not create tunnels or port forwards in this junior lab.** Document whether an approved jump host is required; distinguish a failure reaching the bastion from a failure reaching the destination through it.

## 16. Understand SSH keepalives and idle disconnects

SSH clients and servers can disconnect idle sessions because of policy, NAT/firewall state, application behavior, or network interruptions. `ServerAliveInterval` and `ServerAliveCountMax` are client-side SSH protocol keepalive controls; `TCPKeepAlive` uses TCP keepalive behavior. These do not fix every network failure and must not be changed globally without approval. If a session drops after 10 minutes, record exact timing, affected networks, and policy context rather than automatically increasing timeouts. **Checkpoint:** explain why an idle timeout differs from `Permission denied` during initial authentication.

## 17. Simulated failure: connection refused

**Symptom:** `ssh: connect to host ... port 22: Connection refused`. **Evidence:** name resolves, route exists, no listener on intended endpoint in an authorized server-side `ss` snapshot. **Hypotheses:** sshd stopped, wrong port, wrong bind address, deliberate rejection by firewall/intermediary. **Tests:** compare expected port with approved inventory, server listener and service logs. **Root cause:** not confirmed until correlated. **Unsafe action:** starting or reconfiguring sshd on a live host without change control. **Verification:** approved login works and host identity is verified.

## 18. Simulated failure: timeout

**Symptom:** connection attempt exceeds `ConnectTimeout`. **Evidence:** client has a route but no completed TCP connection. **Hypotheses:** silent filtering, wrong route, offline host, asymmetric return path, intermediary failure. **Tests:** authorized network telemetry, server listener, route selection, source/destination and port, firewall logs. **Root cause:** unresolved from timeout alone. A ping result is supportive context, not proof of SSH reachability.

## 19. Simulated failure: permission denied (publickey)

**Symptom:** server host identity verifies; authentication is rejected. **Evidence:** verbose log shows public-key method attempted but not accepted. **Hypotheses:** wrong remote user, missing public key on server, incorrect file ownership/permissions, server-side policy, wrong key offered, revoked access, expired certificate. **Tests:** compare approved account, authorized key inventory, effective client identity selection, authorized server logs. **Root cause:** not established from client error alone. Do not weaken server authentication. **Recovery:** successful approved login using intended identity and retained policy controls.

## 20. Simulated failure: changed host identity

**Symptom:** `REMOTE HOST IDENTIFICATION HAS CHANGED!`. **Impact:** trust cannot be established; access should stop. **Hypotheses:** authorized rebuild, wrong DNS target, changed network route, active impersonation. **Tests:** independently verify expected fingerprint and host lifecycle change with responsible team. **Root cause:** unresolved until identity is confirmed. **Remediation:** only authorized, exact known-host update after verification. **Prevention:** inventory host keys and document host replacement procedures.

## 21. Integrated incident: access denied after account migration

**Simulated timeline:** 08:00 account migration; 08:15 engineer attempts SSH; DNS resolves and TCP connects; host key matches approved inventory; server offers publickey; client offers old key; server denies. A new key is listed in approved account provisioning, but the client's effective config still selects the retired identity. **Supported conclusion:** access failure is at user authentication, not DNS/TCP/host identity. **Root cause:** stale client identity selection is strongly supported by approved provisioning and `ssh -G` evidence; server-side logs should confirm. **Authorized fix:** use the provisioned identity through approved client config or account process; do not copy private keys between users. **Recovery:** confirm the intended account, host fingerprint, and successful least-privilege remote command; monitor for unexpected auth failures.

## 22. Cleanup and no-change verification

This lab does not require starting sshd, creating users, changing `known_hosts`, or modifying server configuration. Remove only temporary **sanitized** notes you no longer need; keep an approved report. If you created `ssh-client-debug.private.log`, store it securely under organizational policy or delete it after extracting necessary evidence:

```bash
rm -f ssh-client-debug.private.log
```

Never delete another user's logs or known-host entries. **Checkpoint:** document whether any connection attempt was made, its authorization, and whether the remote server was changed (expected: no).

## Command-and-flag reference — review immediately before the final challenge

| Command/option | Purpose | Caveat |
|---|---|---|
| `ssh -V` | Client version | Often stderr |
| `ssh USER@HOST` | Remote login | Authorized target only |
| `-p PORT` | Destination SSH port | Not necessarily 22 |
| `-o ConnectTimeout=5` | Bound connect phase | Not a total session deadline |
| `-o BatchMode=yes` | Disable interactive auth prompts | May fail if key/passphrase needed |
| `-v` / `-vv` / `-vvv` | Client debugging | Logs may contain sensitive metadata |
| `ssh -G HOST` | Print effective client config | No connection; can reveal paths |
| `ssh-keygen -F HOST` | Find known-host entries | Stored key not independently verified |
| `ss -ltn` | Inspect local listeners | Does not prove remote reachability |
| `systemctl status ssh` | Server unit status | Not proof of full access path |
| `getent ahostsv4` | Resolve hostname | Does not test SSH |
| `ip route get` | Selected route | Does not send a packet |
| `ssh ... 'id -un'` | Authorized remote command | Executes on remote host |
| `StrictHostKeyChecking=no` | Weakens host-key protection | **Do not use to bypass warnings** |

## Troubleshooting decision tree — review immediately before the final challenge

```text
SSH LOGIN FAILS
  |
  +-- Name not resolved? --> DNS/NSS
  +-- No route? --> route/source/VPN
  +-- TCP refused or timed out? --> port/listener/path/filtering
  +-- Host key unknown/changed? --> STOP; verify fingerprint out-of-band
  +-- Permission denied? --> account, identity, key authorization, server policy
  +-- Auth succeeds but session closes? --> shell/session policy, logs, resources
  v
TIMESTAMPED EVIDENCE -> COMPETING HYPOTHESES -> AUTHORIZED FIX -> ORIGINAL LOGIN VERIFIED
```
**Never conflate host-key trust, user authentication, and TCP reachability.**

## Integrated final challenge — evidence-based engineering assessment

**Deliverable:** `report/JUN-055-SSH-Access-Investigation.md`.

1. Inventory SSH client/version and local identity.
2. Document an authorized lab SSH destination (or mark remote steps as simulated if unavailable).
3. Inspect DNS and route to the destination; record actual output.
4. Inspect local SSH listener only on a machine you administer; do not start a service.
5. Use `ssh -G` to record effective hostname, user, port, identity, and jump behavior (sanitize paths).
6. Explain host-key verification and record the **independently verified** expected fingerprint source, if available.
7. Perform a bounded authorized login or BatchMode test, with sanitized verbose stage evidence.
8. Distinguish connection refused, timeout, changed host key, and publickey denial using simulated cases.
9. Investigate the account-migration incident; list at least three competing hypotheses and tests.
10. Recommend only approved remediation; verify original least-privilege login and host identity.
11. Document whether you used an agent, jump host, or client configuration, without publishing secrets.
12. Finish with cleanup and a statement that no production authentication policy was weakened.

**Report structure:** scope → source/destination → DNS/route → TCP → host identity → user authentication → session → evidence → hypotheses → tests → root cause status → authorized remediation → verification → prevention → cleanup. **Pass:** a reader can locate the failed SSH stage without guessing and without bypassing security.

## Knowledge check with worked answers

**1. What does SSH provide?**  
Authenticated encrypted remote access and related channels.

**2. Is TCP 22 mandatory?**  
No; it is conventional.

**3. What is a host key?**  
A server identity key used in SSH host authentication.

**4. What is a user key?**  
A key pair that can be used to authenticate a user account.

**5. Are host and user keys interchangeable?**  
No; they serve different trust purposes.

**6. What is `known_hosts`?**  
Client-side records of server host keys/trust.

**7. What is `authorized_keys`?**  
A server-side user account file commonly listing permitted public keys.

**8. What does `ssh -V` show?**  
SSH client version.

**9. What does `ssh -G` do?**  
Prints effective client configuration without connecting.

**10. What does `-p` do?**  
Selects the remote SSH port.

**11. What does `BatchMode=yes` do?**  
Prevents interactive password/passphrase prompts.

**12. What does `ConnectTimeout` bound?**  
The connection setup phase, not an entire interactive session.

**13. Does successful ping prove SSH works?**  
No; ICMP and SSH/TCP are separate.

**14. Does TCP connect prove authentication?**  
No.

**15. What is host-key verification for?**  
To detect wrong or impersonated servers.

**16. Why not disable host-key checking?**  
It weakens protection against connecting to an impostor.

**17. What can changed host key mean?**  
Legitimate rebuild, wrong target, or attack; verify independently.

**18. What does permission denied (publickey) mean?**  
User public-key authentication was not accepted; causes vary.

**19. What is an SSH agent?**  
A component that can broker cryptographic signing using loaded identities.

**20. Is an agent equivalent to authorization?**  
No; server policy decides access.

**21. What is a jump host?**  
An approved intermediate SSH access point.

**22. Why sanitize `-vv` logs?**  
They expose names, addresses, paths, and key metadata.

**23. What does a remote quoted command do?**  
Runs that command on the remote host after successful authentication.

**24. Why record the original failing command?**  
It provides a concrete recovery verification target.

**25. Should you restart sshd on first failure?**  
No; identify the failing stage and obtain change authorization.

## Interview preparation with worked answers

**Interview 1: How do you triage SSH connection refused?**  
Check destination/port, authorized server listener, and filtering; do not assume sshd is down.

**Interview 2: How do you triage an SSH timeout?**  
Check source, route, target, TCP path, network policy and telemetry; timeout alone is inconclusive.

**Interview 3: How do you handle a changed host key?**  
Stop, verify the new fingerprint through an independent trusted channel, then follow approved update procedure.

**Interview 4: Why can SSH fail after TCP connects?**  
Host identity or user authentication may fail, or the session may be denied.

**Interview 5: What does `ssh -G` help diagnose?**  
Unexpected user, hostname, port, identity file, or jump configuration.

**Interview 6: What is the difference between host authentication and user authentication?**  
Host authentication verifies server identity; user authentication proves account authorization.

**Interview 7: How do you troubleshoot publickey denial?**  
Confirm account, offered identity, authorized public key, permissions/policy, and approved server logs.

**Interview 8: What is the security problem with `StrictHostKeyChecking=no`?**  
It removes an important check against impersonated servers.

**Interview 9: How would you verify a fix?**  
Confirm expected host fingerprint and successful least-privilege login/command from affected client.

**Interview 10: Why avoid changing sshd settings in this lab?**  
A mistake can lock out administrators or weaken production security.

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

**JUN-056 — SSH Keys**. Continue in sequence; carry forward your evidence discipline and the distinction between connectivity, authentication, configuration, and application health.

**End of 055 — SSH Fundamentals.**
