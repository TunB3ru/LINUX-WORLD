# 057 — SSH Configuration Basics

**VERIQTA Linux World | Public-Facing Junior Engineer Follow-Along Lab**

**Canonical path:** `Junior/03-Networking/JUN-057-SSH-Configuration-Basics/README.md`  
**Environment:** Ubuntu Server 24.04 LTS; non-root account; an isolated or explicitly authorized lab machine  
**Level:** Junior Engineer, progressing from fundamentals to production evidence  
**Duration:** 5–8 hours of reading, execution, evidence collection, and reporting (variable by experience)  
**Changes:** No production configuration changes; all temporary lab resources are confined to an authorized local environment  
**Prerequisites:** All preceding JUN networking labs through JUN-056  
**Next:** JUN-058 — Continue with the next locked Linux World curriculum lab (verify exact title in the repository roadmap)

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
mkdir -p "$HOME/veriqta-linux-labs/JUN-057-SSH-Configuration-Basics"
cd "$HOME/veriqta-linux-labs/JUN-057-SSH-Configuration-Basics"
date -u '+%Y-%m-%dT%H:%M:%SZ'
```

**Checkpoint:** Identify your host, user, interfaces, lab directory, and whether you are working locally or over SSH. If remotely connected, do not perform an experiment that could interrupt the session.

## 1. Production mission: an SSH alias silently targets the wrong account

**Simulated incident:** An engineer uses `ssh production-api`, expecting `deploy@10.20.30.40:22`. After a workstation configuration change, the alias selects a retired username, port 2222, and a stale key path. A colleague recommends changing the server daemon to accept the old port. Your mission is to inspect client configuration precedence, isolate a safe temporary configuration, verify evaluated settings, identify the mismatch, and produce a remediation plan **without changing the live SSH server**. This lab teaches configuration fundamentals, not production hardening or daemon administration.

## 2. Distinguish client and server configuration

OpenSSH clients commonly read system-wide and user-specific configuration such as `/etc/ssh/ssh_config` and `~/.ssh/config`, possibly with included files. The SSH daemon uses server-side configuration, commonly `/etc/ssh/sshd_config` and include directories. **These are different files with different purposes.** Client `Port` changes where the client connects; server `Port` controls where sshd listens if an authorized administrator changes it. Client `IdentityFile` selects an identity to offer; server authorization still decides whether it is accepted. **Do not edit `/etc/ssh/sshd_config` or restart sshd in this lab.**

## 3. Inventory existing configuration without exposing secrets

Run:

```bash
ssh -V
ls -ld "$HOME/.ssh" 2>/dev/null || true
ls -l "$HOME/.ssh/config" /etc/ssh/ssh_config 2>/dev/null || true
ssh -G localhost | grep -E '^(hostname|user|port|connecttimeout|stricthostkeychecking) '
```

`ssh -G` prints the evaluated client settings and exits without opening a network connection. Avoid posting full effective configuration because it may reveal internal hosts, identity paths, and proxy information. If the user config does not exist, that is normal. Do not overwrite an existing `~/.ssh/config`. **Checkpoint:** identify which file belongs to the client and which to the daemon.

## 4. Understand Host blocks and aliases

A client config can define an alias:

```sshconfig
Host lab-api
    HostName 192.0.2.40
    User labengineer
    Port 22
```

`Host lab-api` is a matching pattern for a client invocation; `HostName` is the actual destination; `User` is the remote username; `Port` is the destination port. The documentation address `192.0.2.40` is illustrative and must not be used as a real target. SSH aliases do not create DNS records and do not guarantee that the destination exists. **Checkpoint:** explain why `ssh lab-api` can target a different hostname from the alias text.

## 5. Create an isolated temporary client config

Use a separate file so the exercise cannot overwrite your real configuration:

```bash
mkdir -p "$HOME/veriqta-linux-labs/JUN-057"
cd "$HOME/veriqta-linux-labs/JUN-057"
umask 077
cat > ssh_config.lab <<'SSHCONF'
Host lab-api
    HostName 192.0.2.40
    User labengineer
    Port 2222
    ConnectTimeout 5
    BatchMode yes
    IdentitiesOnly yes

Host lab-bastion
    HostName 192.0.2.10
    User bastionuser
    Port 22

Host *
    ServerAliveInterval 0
SSHCONF
chmod 600 ssh_config.lab
```

This file is **configuration data only**. It does not create a network listener or connect to the example addresses. The addresses are reserved for documentation. **Checkpoint:** verify mode with `stat -c '%a %n' ssh_config.lab`.

## 6. Evaluate the isolated config safely with ssh -G

Run:

```bash
ssh -F ./ssh_config.lab -G lab-api   | grep -E '^(hostname|user|port|connecttimeout|batchmode|identitiesonly|serveraliveinterval) '
```

`-F FILE` selects the supplied user configuration file; `-G` evaluates and prints configuration without connecting. Expect `hostname 192.0.2.40`, `user labengineer`, `port 2222`, and the specified options. Record your actual output. **Do not run `ssh lab-api` against the documentation IP.** **Checkpoint:** identify how a configuration alias differs from a real network reachability test.

## 7. Understand precedence and first-obtained-value behavior

OpenSSH client configuration generally applies the **first obtained value** for many parameters. The order of matching `Host` blocks matters, and command-line options commonly take precedence. This is why more specific patterns are usually placed before broad defaults. Not every directive has identical accumulation semantics: for example, multiple `IdentityFile` entries can be accumulated. Avoid memorizing a simplistic rule that every directive is overwritten by the last occurrence.

Create a second file:

```bash
cat > ssh_precedence.lab <<'SSHCONF'
Host lab-api
    User firstuser
    Port 2201
Host lab-*
    User seconduser
    Port 2202
Host *
    User fallback
    Port 22
SSHCONF
ssh -F ./ssh_precedence.lab -G lab-api | grep -E '^(user|port) '
ssh -F ./ssh_precedence.lab -o Port=2299 -G lab-api | grep -E '^(user|port) '
```

Expected: first matching values typically give `firstuser`/`2201`; command-line `-o Port=2299` overrides port. **Checkpoint:** explain why ordering a wildcard block first can cause unexpected defaults.

## 8. Understand Include and Match at a safe conceptual level

`Include` allows configuration to be split across files; included paths can be relative to configuration context and may be globbed. `Match` applies settings conditionally, and some conditions require special care. For this junior lab, inspect only your isolated files; do not modify system include directories. When diagnosing unexpected effective settings, check which files and patterns matched, then use `ssh -G` to verify. **Security:** included files can change identity selection, proxy behavior, or destination. Treat configuration files as trusted code-like operational inputs, not harmless decoration.

## 9. Understand IdentityFile and IdentitiesOnly

`IdentityFile` identifies a private key file to offer; `IdentitiesOnly yes` helps limit identity selection to configured identities (and relevant certificates) instead of indiscriminately offering agent keys. **Never put private key contents inside config.** Add a dummy identity path to your isolated config only for evaluation:

```bash
cat >> ssh_config.lab <<'SSHCONF'

Host lab-key-demo
    HostName 192.0.2.50
    User labengineer
    IdentityFile ~/veriqta-linux-labs/JUN-056/keys/jun056_ed25519
    IdentitiesOnly yes
SSHCONF
ssh -F ./ssh_config.lab -G lab-key-demo | grep -E '^(hostname|user|identityfile|identitiesonly) '
```

This does not load or use the key and does not connect. The referenced file may have been removed during JUN-056 cleanup. **Checkpoint:** explain why effective client identity selection is not proof of remote authorization.

## 10. Understand connection and session timeout options

`ConnectTimeout` bounds connection setup. `ServerAliveInterval` controls client-side encrypted SSH keepalive messages during a session; `ServerAliveCountMax` determines tolerated unanswered intervals. `TCPKeepAlive` is a different TCP-level mechanism. Do not increase values arbitrarily to hide network instability or violate session policies. For your isolated config, inspect defaults:

```bash
ssh -F ./ssh_config.lab -G lab-api   | grep -E '^(connecttimeout|serveraliveinterval|serveralivecountmax|tcpkeepalive) '
```

**Checkpoint:** distinguish an initial connection timeout from an idle session disconnect.

## 11. Understand ProxyJump without making a tunnel

`ProxyJump` routes an SSH connection through an intermediate SSH host, subject to authorization. It can introduce two independent host identities, user-authentication steps, and network paths. For evaluation only, add:

```bash
cat >> ssh_config.lab <<'SSHCONF'

Host lab-through-bastion
    HostName 192.0.2.60
    User appuser
    ProxyJump lab-bastion
SSHCONF
ssh -F ./ssh_config.lab -G lab-through-bastion   | grep -E '^(hostname|user|proxyjump|port) '
```

Do not attempt a real connection to the documentation addresses. Do not create forwarding rules or bypass corporate network access policy. **Checkpoint:** draw the conceptual client → bastion → destination path and mark both trust boundaries.

## 12. Understand strict host-key checking and known-host files

`StrictHostKeyChecking` governs how SSH handles new or changed server host keys. The safest behavior depends on organizational trust provisioning, but disabling checks to bypass a warning is not a valid fix. `UserKnownHostsFile` selects the client known-host store; a separate lab file can be useful for an isolated authorized test, but do not redirect host-key storage to `/dev/null` to suppress identity checks. `UpdateHostKeys` can support host-key rotation under specific trust conditions. **Never claim that `ssh -G` verifies a host key**: it prints configuration and does not contact the host.

## 13. Understand permissions and configuration ownership

OpenSSH can reject unsafe user configuration ownership or permissions. Your isolated lab file should be owned by you and not writable by others. Run:

```bash
stat -c '%U %a %n' ssh_config.lab ssh_precedence.lab
```

Use `chmod 600` only on files created in this lab. Do not recursively alter `~/.ssh`, do not edit another user's files, and do not weaken directory ownership controls. **Checkpoint:** explain why a writable-by-everyone SSH config is a security risk.

## 14. Safely test syntax and effective configuration

`ssh -G` is the primary no-connection validation tool in this lab. It validates and evaluates many client options and can surface configuration parsing errors. It does **not** validate network reachability or server policy. To see failure handling, create a disposable malformed file:

```bash
cat > ssh_bad.lab <<'SSHCONF'
Host lab-bad
    DefinitelyNotAnSSHOption yes
SSHCONF
ssh -F ./ssh_bad.lab -G lab-bad >/dev/null
printf 'config_exit=%s
' "$?"
```

You should observe a configuration error and nonzero exit. Do not change your real config to induce failure. **Checkpoint:** explain why a valid config can still produce `Permission denied (publickey)` on connection.

## 15. Diagnose conflicting Host patterns

**Simulated config:**

```sshconfig
Host *
    User generic
    Port 22
Host production-api
    User deploy
    Port 2222
```

For first-obtained-value directives, the earlier wildcard may prevent the later intended values from applying. **Diagnosis:** reproduce in an isolated file, run `ssh -G production-api`, compare with approved inventory. **Remediation:** reorder patterns in an authorized user config after review; avoid editing the production daemon. **Verification:** effective user/port match approved values and a separately authorized connection succeeds.

## 16. Diagnose wrong identity path

**Simulated symptom:** effective config points to a retired private key; server denies public-key authentication. **Evidence:** `ssh -G` reports stale `identityfile`; `ssh -vv` shows the identity being offered (if a connection is authorized). **Hypotheses:** wrong config alias, key removed, agent interference, remote authorization changed. **Tests:** approved key inventory, effective settings, permissions, server logs. **Fix:** use correct approved identity path, not a copied private key from a colleague. **Verification:** host identity verified and intended account logs in.

## 17. Diagnose unexpected ProxyJump behavior

**Simulated symptom:** direct route works, but SSH alias fails through an obsolete bastion. `ssh -G` shows `proxyjump old-bastion`. **Possible cause:** inherited configuration or stale alias. **Tests:** compare approved architecture, evaluate matching blocks/includes, determine which hop fails. **Root cause:** confirmed only after configuration and route evidence align. **Fix:** update approved client alias through change process; do not bypass mandatory bastions or security controls.

## 18. Distinguish client configuration from server configuration

**Simulated symptom:** `ssh -G` shows port 2222, while server `ss -ltn` shows only port 22. This is a destination mismatch, but the appropriate correction depends on approved architecture. Changing sshd to listen on 2222 may create unnecessary exposure and lockout risk. First confirm the intended port in inventory and whether a proxy/forwarder is involved. **Client-side changes can be tested without altering the server**; server-side changes require a separate authorized runbook with configuration validation, rollback, and session preservation.

## 19. Understand server-side configuration safely (read-only)

On a lab VM you administer, you may inspect the presence of server files without editing:

```bash
ls -l /etc/ssh/sshd_config /etc/ssh/sshd_config.d 2>/dev/null || true
systemctl status ssh --no-pager 2>&1 | head -n 30
```

If the daemon exists, an authorized administrator may use `sshd -T` for effective server configuration and `sshd -t` for syntax validation, but these can require privileges and context; they are **not required** in this junior lab. Do not run server configuration changes, `systemctl restart ssh`, or firewall changes. **Checkpoint:** explain why editing the server's listener is much riskier than evaluating a disposable client config.

## 20. Integrated incident: wrong port, user, and identity after workstation migration

**Simulated approved inventory:** alias `production-api` should select host `10.20.30.40`, user `deploy`, port 22, identity `~/.ssh/deploy_ed25519`, and approved bastion `corp-bastion`. **Observed effective client config:** host correct, user `legacydeploy`, port 2222, identity `~/.ssh/retired_key`, proxyjump none. **Impact:** access attempts fail, and the client may attempt an unapproved path. **Hypotheses:** wildcard precedence, stale include, wrong alias, command-line override. **Discriminating tests:** `ssh -G`, inspect matching config blocks/includes, compare inventory, test an isolated corrected config with `-F` and `-G` (no connection). **Supported cause:** configuration mismatch confirmed; exact origin of each wrong value remains open until matching source is identified. **Authorized correction:** review and update user client config under policy; preserve mandatory bastion; never change sshd to accommodate stale client settings. **Verification:** effective settings match inventory, host key verified, least-privilege login succeeds, logs show expected access path. **Prevention:** config review, documented aliases, safe onboarding, change audit.

## 21. Cleanup and final verification

Remove only the disposable config files created by this lab after recording sanitized findings:

```bash
rm -f ssh_config.lab ssh_precedence.lab ssh_bad.lab
```

You did not need to connect to the documentation addresses or alter `~/.ssh/config`. Check your report for identity paths, usernames, jump hosts, and internal addresses before publication. **Checkpoint:** confirm `sshd_config`, systemd units, firewall, routes, user keys, and real `known_hosts` were not modified.

## Command-and-flag reference — review immediately before the final challenge

| Directive/command | Purpose | Caution |
|---|---|---|
| `Host PATTERN` | Match alias/pattern | Order matters |
| `HostName NAME` | Actual destination | Alias is not DNS |
| `User NAME` | Remote account | Does not grant authorization |
| `Port N` | Client destination port | Does not reconfigure sshd |
| `IdentityFile PATH` | Candidate private identity | Do not embed private key data |
| `IdentitiesOnly yes` | Restrict identity selection | Server still controls access |
| `ConnectTimeout N` | Connection setup timeout | Not total session duration |
| `ServerAliveInterval N` | SSH keepalive interval | Must respect session policy |
| `ServerAliveCountMax N` | Unanswered keepalive tolerance | Not a universal network fix |
| `ProxyJump HOST` | Intermediate SSH hop | Must be authorized |
| `Include PATH` | Read more client config | Included files affect trust/path |
| `Match` | Conditional config | Requires careful evaluation |
| `StrictHostKeyChecking` | Host identity behavior | Never bypass a changed-key warning |
| `ssh -F FILE -G HOST` | Evaluate isolated client config | No network connection |
| `ssh -o Port=N` | CLI override | Verify actual precedence |
| `ssh -V` | Client version | Often stderr |
| `/etc/ssh/sshd_config` | Server daemon configuration | **Do not edit in this lab** |

## Troubleshooting decision tree — review immediately before the final challenge

```text
SSH ALIAS BEHAVES UNEXPECTEDLY
  |
  +-- Evaluate `ssh -F FILE -G ALIAS` (or approved real config)
  +-- Wrong hostname? --> HostName / matching alias / Include
  +-- Wrong user? --> User / wildcard precedence / command line
  +-- Wrong port? --> Port / override / intended proxy architecture
  +-- Wrong identity? --> IdentityFile / IdentitiesOnly / agent
  +-- Unexpected hop? --> ProxyJump / Include / policy
  +-- Host key warning? --> STOP and verify identity independently
  v
COMPARE WITH APPROVED INVENTORY -> FIX CLIENT CONFIG SAFELY
  -> RE-EVALUATE -> AUTHORIZED LOGIN -> VERIFY ORIGINAL OPERATION
```
**Never alter production sshd to compensate for an unverified client alias.**

## Integrated final challenge — evidence-based engineering assessment

**Deliverable:** `report/JUN-057-SSH-Client-Configuration-Investigation.md`.

1. Inventory client version and existing configuration file metadata without changing them.
2. Explain client versus server configuration, including risks of restarting sshd.
3. Create the isolated `ssh_config.lab` file exactly as shown; verify restrictive permissions.
4. Use `ssh -F ... -G` to document effective HostName, User, Port, timeout, and identity selection.
5. Reproduce first-obtained-value behavior using `ssh_precedence.lab` and a command-line override.
6. Evaluate a dummy IdentityFile and IdentitiesOnly without attempting a real connection.
7. Evaluate a ProxyJump alias without opening a tunnel; diagram both trust boundaries.
8. Produce and diagnose a disposable invalid configuration; record error and exit status.
9. Analyze three simulated failures: wildcard conflict, wrong identity, unexpected jump host.
10. Analyze the integrated workstation-migration incident; compare approved versus observed settings.
11. Write an authorized correction and rollback plan for **client** configuration only; explain why server changes are out of scope.
12. Verify cleanup and that real SSH configuration/keys/known_hosts are unchanged.

**Report structure:** authorization → baseline → isolated config → effective-value table → precedence experiment → identity and jump behavior → syntax validation → competing hypotheses → discriminating tests → root cause status → authorized client correction → original login verification → prevention → cleanup. **Pass:** effective settings are demonstrated, not guessed; no real production config or daemon was modified.

## Knowledge check with worked answers

**1. What is ~/.ssh/config?**  
A per-user OpenSSH client configuration file.

**2. What is sshd_config?**  
The SSH server daemon configuration, separate from the client config.

**3. What does Host do?**  
Matches an alias or pattern for applying client settings.

**4. What does HostName do?**  
Specifies the actual destination hostname/address.

**5. What does User do?**  
Chooses the remote login username.

**6. What does Port do?**  
Chooses the destination SSH port.

**7. Does client Port change sshd listener?**  
No.

**8. What does ssh -G do?**  
Prints evaluated client configuration without connecting.

**9. What does -F do?**  
Selects an alternative client configuration file.

**10. Why use a disposable config?**  
To test safely without changing working SSH access.

**11. How are many config values selected?**  
The first obtained value generally wins; exceptions exist.

**12. Does every directive have identical precedence behavior?**  
No; some such as IdentityFile can accumulate.

**13. Why order specific Host blocks before wildcard?**  
To prevent broad defaults from taking precedence.

**14. What is Include?**  
A directive that reads additional configuration files.

**15. What is Match?**  
A directive for conditional configuration.

**16. What is IdentityFile?**  
A path to an identity used for client authentication.

**17. Does IdentityFile authorize a server account?**  
No; server-side policy determines authorization.

**18. What is ProxyJump?**  
A directive for an approved intermediate SSH host.

**19. Why verify two host keys with a bastion?**  
Both bastion and destination are separate SSH trust boundaries.

**20. What is ConnectTimeout?**  
A connection setup timeout.

**21. What is ServerAliveInterval?**  
A client-side SSH protocol keepalive interval.

**22. What does StrictHostKeyChecking affect?**  
Server host-key trust behavior.

**23. Can ssh -G validate network reachability?**  
No; it makes no connection.

**24. Why protect SSH config permissions?**  
Untrusted changes could redirect hosts, identities, or proxies.

**25. Should you restart sshd to fix wrong client port?**  
No; confirm approved architecture and correct client settings.

## Interview preparation with worked answers

**Interview 1: How would you debug an SSH alias?**  
Run `ssh -G ALIAS`, compare effective hostname/user/port/identity/proxy to approved inventory, then inspect matching blocks.

**Interview 2: What is a precedence pitfall?**  
An early Host * block can set values before a later specific Host block.

**Interview 3: How do you test config without affecting real access?**  
Create a separate restricted file and use `ssh -F file -G alias`.

**Interview 4: How do you validate a config syntax error?**  
Run `ssh -F badfile -G alias` and inspect nonzero exit/stderr.

**Interview 5: Why is a client port different from server Port?**  
Client port selects the destination; server Port controls listener only when daemon config is changed.

**Interview 6: How do you diagnose a wrong key selection?**  
Inspect `identityfile` and `identitiesonly` with ssh -G; correlate authorized logs if connecting.

**Interview 7: What risks come with ProxyJump?**  
Separate trust and auth steps; bypassing mandatory bastions can violate policy.

**Interview 8: How do you handle changed host key during testing?**  
Stop and verify independently; do not disable host-key checking.

**Interview 9: Why not change sshd_config in this lab?**  
It can disrupt remote access and security and requires change control and rollback.

**Interview 10: What proves an SSH alias fix?**  
Correct evaluated config plus authorized host-key-verified login using intended account and path.

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

**JUN-058 — Continue with the next locked Linux World curriculum lab (verify exact title in the repository roadmap)**. Continue in sequence; carry forward your evidence discipline and the distinction between connectivity, authentication, configuration, and application health.

**End of 057 — SSH Configuration Basics.**
