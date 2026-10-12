# 054 — curl and HTTP Testing

**VERIQTA Linux World | Public-Facing Junior Engineer Follow-Along Lab**

**Canonical path:** `Junior/03-Networking/JUN-054-curl-and-HTTP-Testing/README.md`  
**Environment:** Ubuntu Server 24.04 LTS; non-root account; an isolated or explicitly authorized lab machine  
**Level:** Junior Engineer, progressing from fundamentals to production evidence  
**Duration:** 5–8 hours of reading, execution, evidence collection, and reporting (variable by experience)  
**Changes:** No production configuration changes; all temporary lab resources are confined to an authorized local environment  
**Prerequisites:** All preceding JUN networking labs through JUN-053  
**Next:** JUN-055 — SSH Fundamentals

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
mkdir -p "$HOME/veriqta-linux-labs/JUN-054-curl-and-HTTP-Testing"
cd "$HOME/veriqta-linux-labs/JUN-054-curl-and-HTTP-Testing"
date -u '+%Y-%m-%dT%H:%M:%SZ'
```

**Checkpoint:** Identify your host, user, interfaces, lab directory, and whether you are working locally or over SSH. If remotely connected, do not perform an experiment that could interrupt the session.

## 1. Production mission: ICMP works but checkout returns 503

**Scenario (simulated):** `api.lab.example` resolves to an application host. ICMP Echo succeeds, the TCP port accepts connections, but users receive intermittent HTTP `503` responses. An operator suggests restarting the server. Your mission is to identify whether failures occur at name resolution, TCP connect, TLS negotiation, HTTP protocol, redirect handling, or application processing, without changing production state.

| Signal | What it can support | What it cannot prove |
|---|---|---|
| `ping` succeeds | ICMP round trip | TCP/TLS/HTTP health |
| `ss` shows LISTEN | A kernel listener exists | Valid HTTP responses |
| TCP connect succeeds | Transport handshake completed | TLS certificate validity |
| TLS succeeds | Secure session established under client policy | Application request succeeds |
| HTTP 200 | That particular request succeeded | All users, routes, and dependencies are healthy |
| HTTP 503 | Server/proxy reports unavailable | Which backend component caused it |

**Mission deliverable:** an HTTP diagnostic report with stage-by-stage evidence, failure classification, bounded reproducible commands, and a recovery verification plan.

## 2. Understand the complete request path

An HTTP client must often perform several distinct operations: parse URL → resolve hostname → choose address → route packets → establish TCP (or another negotiated transport) → negotiate TLS for HTTPS → send HTTP request → receive status/headers/body → interpret application behavior. HTTP/1.1, HTTP/2, and HTTP/3 differ in transport and negotiation details; this lab focuses on ordinary HTTP/1.1 over TCP and HTTPS with normal `curl` negotiation. `curl` supports more protocols, but support varies by build.

A URL such as `https://api.example.test:8443/health?verbose=1` contains scheme (`https`), hostname (`api.example.test`), port (`8443`), path (`/health`), and query (`verbose=1`). A fragment (`#section`) is client-side and is not sent as part of the HTTP request target. An HTTPS URL defaults to TCP port 443 when no port is specified; HTTP commonly defaults to TCP port 80. An HTTP status code is returned by an HTTP-speaking endpoint, not by the network itself.

**Checkpoint:** Explain how a successful TCP handshake can coexist with an HTTP `503` response and why a failed DNS lookup is different from a TLS certificate failure.

## 3. Inventory curl and establish a safe baseline

Run:

```bash
command -v curl
curl --version
curl --help all | head -n 35
getent ahostsv4 localhost
ss -ltn
```

`curl --version` reports the installed version, SSL/TLS backend, and compiled features. Do not assume HTTP/2, HTTP/3, or a particular TLS backend is available until you inspect it. `curl --help all` displays options; `head` limits display. `getent` queries name resolution, while `ss` shows listeners. They are different evidence sources.

**Record:** executable path, curl version, TLS backend, available HTTP features, UTC timestamp, and current listeners. **Expected:** localhost resolves; there may be no HTTP listener yet. That is not a failure of the lab.

## 4. Build an isolated local HTTP server

This exercise creates a loopback-only HTTP endpoint using Python's standard library. **Use your own VM, not a production server.** Choose a free port dynamically; do not assume port 8000 is available.

```bash
cat > local_http_lab.py <<'PYHTTP'
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path.startswith('/health'):
            status, payload = 200, {'status': 'ok', 'service': 'veriqta-http-lab'}
        elif self.path.startswith('/unavailable'):
            status, payload = 503, {'status': 'unavailable', 'reason': 'simulated'}
        elif self.path.startswith('/redirect'):
            self.send_response(302)
            self.send_header('Location', '/health')
            self.end_headers()
            return
        elif self.path.startswith('/missing'):
            status, payload = 404, {'error': 'not found'}
        else:
            status, payload = 200, {'message': 'local test endpoint'}
        body = json.dumps(payload).encode('utf-8')
        self.send_response(status)
        self.send_header('Content-Type', 'application/json')
        self.send_header('Content-Length', str(len(body)))
        self.end_headers()
        self.wfile.write(body)
    def do_HEAD(self):
        self.send_response(200 if self.path == '/health' else 404)
        self.send_header('Content-Length', '0')
        self.end_headers()
    def log_message(self, fmt, *args):
        print('LAB HTTP: ' + fmt % args, flush=True)

server = ThreadingHTTPServer(('127.0.0.1', 0), Handler)
print(f'LAB_URL=http://127.0.0.1:{server.server_port}', flush=True)
try:
    server.serve_forever()
except KeyboardInterrupt:
    pass
finally:
    server.server_close()
PYHTTP
python3 local_http_lab.py
```

Keep Terminal A open. Copy the actual `LAB_URL` (for example `http://127.0.0.1:43817`). In Terminal B:

```bash
BASE='http://127.0.0.1:43817'   # REPLACE with your printed LAB_URL
curl --noproxy '*' --connect-timeout 3 --max-time 10 "$BASE/health"
ss -ltn | grep ':43817'         # REPLACE port
```

`--noproxy '*'` prevents a configured proxy from changing the local experiment. `--connect-timeout` bounds connection establishment; `--max-time` bounds the entire transfer. `curl` prints a JSON response. The listener should be bound to `127.0.0.1`, not `0.0.0.0`. If a command fails, verify that the server is still running and that the port matches. **Checkpoint:** record the actual port, PID (if inspected), bind address, response body, and whether the endpoint is reachable from loopback.

## 5. Make GET, HEAD, and header-only requests

HTTP methods express intended operations; they are not interchangeable. GET retrieves a representation. HEAD requests response metadata without the response body. Use your local server:

```bash
curl --noproxy '*' -i --connect-timeout 3 --max-time 10 "$BASE/health"
curl --noproxy '*' -I --connect-timeout 3 --max-time 10 "$BASE/health"
curl --noproxy '*' -D headers.txt -o body.json --connect-timeout 3 --max-time 10 "$BASE/health"
cat headers.txt
cat body.json
```

`-i` includes response headers with the body; `-I` issues HEAD; `-D` writes headers to a file; `-o` writes body to a file. Do not assume a HEAD request always behaves exactly like GET on all applications. The controlled server deliberately implements both. The HTTP version in the status line depends on the server implementation; Python's handler defaults to HTTP/1.0 responses unless configured otherwise. **Checkpoint:** identify status, content type, content length, and body; explain why the HEAD response has no body.

## 6. Understand status codes through controlled failures

Run each request independently:

```bash
curl --noproxy '*' -i "$BASE/health"
curl --noproxy '*' -i "$BASE/missing"
curl --noproxy '*' -i "$BASE/unavailable"
curl --noproxy '*' -sS -o /dev/null -w 'HTTP=%{http_code}
' "$BASE/unavailable"
```

Expected **illustrative** statuses: `200`, `404`, and `503`. `-sS` suppresses progress while still showing curl errors; `-w` prints a selected metric; `/dev/null` discards body. HTTP classes: 1xx informational, 2xx success, 3xx redirection, 4xx client-error class, 5xx server-error class. These are broad categories; a 4xx can arise from authentication, authorization, request shape, or routing and is not automatically the user's fault.

By default, curl can exit successfully after receiving an HTTP 404/503 because an HTTP response was received. Compare:

```bash
curl --noproxy '*' --fail-with-body -sS -o /dev/null "$BASE/unavailable"
printf 'curl_exit=%s
' "$?"
```

`--fail-with-body` returns an error for HTTP status >=400 while retaining the response body if requested. Do not rely on `--fail`/`--fail-with-body` as universal security policy; authentication and proxy cases can have nuances. **Checkpoint:** distinguish a successful transport exchange from an unsuccessful application response.

## 7. Inspect redirects and method behavior

Run:

```bash
curl --noproxy '*' -i "$BASE/redirect"
curl --noproxy '*' -i -L --max-redirs 3 "$BASE/redirect"
curl --noproxy '*' -sS -o /dev/null -w 'redirects=%{num_redirects} final=%{http_code} url=%{url_effective}
' -L --max-redirs 3 "$BASE/redirect"
```

Without `-L`, curl returns the initial 302 and Location header. With `-L`, it follows the redirect. `--max-redirs` bounds redirect chains. In real systems, redirects can cross domains and authentication boundaries; do not blindly follow untrusted redirects with credentials. A 301/302/303 may change a POST to GET under common client behavior; 307/308 are designed to preserve method semantics. **Checkpoint:** record initial status, Location, final status, final URL, and redirect count.

## 8. Separate DNS, TCP, TLS, and HTTP timing

Use curl's write-out variables:

```bash
curl --noproxy '*' -sS -o /dev/null --connect-timeout 3 --max-time 10   -w 'remote_ip=%{remote_ip}
http=%{http_code}
lookup=%{time_namelookup}
connect=%{time_connect}
tls=%{time_appconnect}
first_byte=%{time_starttransfer}
total=%{time_total}
'   "$BASE/health"
```

The reported times are cumulative timestamps measured from the start of the transfer, not independent stage durations. `time_namelookup` is time until name lookup completion; `time_connect` until TCP connection completion; `time_appconnect` until TLS/other application connection negotiation completion (typically zero for plain HTTP); `time_starttransfer` until the first response byte; `time_total` for the entire operation. Subtract relevant cumulative values to estimate stage intervals, while recognizing connection reuse, proxies, HTTP versions, and client behavior can affect interpretation.

**Follow-along calculation:** if lookup=0.005, connect=0.020, TLS=0.090, first-byte=0.140, total=0.180 seconds, then approximate TCP connect increment=0.015, TLS increment=0.070, server/first-byte increment=0.050, and transfer remainder=0.040. These are illustrative and do not uniquely isolate backend compute time. **Checkpoint:** explain why `time_starttransfer` is not pure server processing time.

## 9. Diagnose common curl exit codes without conflating layers

Useful errors include: `6` could not resolve host; `7` failed to connect; `28` operation timed out; `35` TLS handshake error; `51`/`60` certificate verification issues depending on situation/version; `22` HTTP error when `--fail` or `--fail-with-body` is enabled. Always preserve stderr and the actual version-specific message.

Controlled refused-connection exercise (first verify the chosen port is not listening):

```bash
ss -ltn '( sport = :49999 )'
curl --noproxy '*' -sS --connect-timeout 2 --max-time 4 http://127.0.0.1:49999/health
printf 'exit=%s
' "$?"
```

If another service is using 49999, choose another unused high loopback port. You should typically see a connection-refused error, not a DNS or TLS error. The command is safe because it targets your own loopback. **Checkpoint:** record the exact exit code, error, listener observation, and most narrowly supported conclusion.

## 10. Understand HTTPS, certificate verification, and trust

HTTPS normally adds TLS authentication and encryption to HTTP. A TLS certificate is checked against a trusted issuer and hostname, with validity and policy checks. A successful TLS handshake does not guarantee the HTTP application is healthy. Conversely, an HTTP response from an endpoint reached with verification disabled does not establish authentic server identity.

On an **explicitly approved** HTTPS lab endpoint, use:

```bash
curl -v --connect-timeout 5 --max-time 15 https://APPROVED_HTTPS_HOST/health -o /dev/null
curl -I --connect-timeout 5 --max-time 15 https://APPROVED_HTTPS_HOST/health
```

`-v` prints connection and TLS diagnostics to stderr; **it can expose headers and secrets**, so do not paste unredacted output into public issues. Do not use `-k`/`--insecure` as a normal fix: it disables certificate verification and undermines endpoint authentication. If your lab lacks an approved HTTPS host, analyze the following illustrative failure instead: `curl: (60) SSL certificate problem: certificate verify failed`. Investigate hostname, trust chain, expiry, and system trust policy; do not install unknown root certificates. **Checkpoint:** distinguish TLS trust failure from HTTP 500.

## 11. Control HTTP versions and interpret negotiation

Use your local Python server to request HTTP/1.1 explicitly:

```bash
curl --noproxy '*' --http1.1 -v --max-time 10 "$BASE/health" -o /dev/null
```

`--http1.1` requests HTTP/1.1 from the client; the simple Python server may respond with an HTTP/1.0 status line. Inspect actual output rather than assuming the server matches the requested version. For HTTPS servers, `--http2` is available only when the curl build supports HTTP/2; check `curl --version` first. HTTP/3 requires additional support and is out of scope. **Checkpoint:** explain the difference between client preference, negotiated/used protocol, and server response version.

## 12. Inspect request headers safely and avoid secret leakage

On the local server:

```bash
curl --noproxy '*' -v -H 'X-Lab-Trace: jun-054-local' --max-time 10 "$BASE/health"
curl --noproxy '*' -A 'VERIQTA-Linux-Lab/1.0' --max-time 10 "$BASE/health"
```

`-H` adds a header and `-A` changes User-Agent. These commands are safe because the values are not secrets. In production, Authorization headers, cookies, tokens, session IDs, and signed URLs can leak through shell history, process lists, trace logs, and saved outputs. Never include live credentials in a command shared publicly. Avoid `--trace`/`--trace-ascii` with sensitive traffic unless there is an approved secure evidence process. **Checkpoint:** identify which headers are request headers versus response headers.

## 13. Understand proxies and the no-proxy lab boundary

Curl may honor environment variables such as `http_proxy`, `https_proxy`, `ALL_PROXY`, and `NO_PROXY`. Inspect names without printing secret values:

```bash
for name in http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY; do
  if printenv "$name" >/dev/null; then printf '%s is set
' "$name"; fi
done
```

A proxy can change the network path and the meaning of a failure. Use `--noproxy '*'` for your loopback tests so they stay local. For corporate proxy testing, follow approved policy; do not bypass mandatory security controls. **Checkpoint:** explain why `curl http://127.0.0.1` might behave differently when a proxy is configured.

## 14. Interpret connection reuse, headers, and response body correctly

HTTP persistent connections and pooling can change the meaning of timings. One curl process can perform multiple transfers; separate shell invocations usually do not share the same in-memory connection pool. A `200` from `/health` may be shallow, while `/checkout` can fail due to a dependency. Response headers such as `Content-Type`, `Content-Length`, `Location`, `Cache-Control`, `Retry-After`, and server-specific trace IDs are useful but not automatically trustworthy. Never infer a backend architecture solely from `Server` headers. Record response body only if it contains no secrets or personal data.

**Exercise:** compare `/health` and `/unavailable` status, body, and timing; write three things each result proves and three it does not.

## 15. Build a repeatable, bounded HTTP evidence collector

Use the existing local `BASE` variable. This collects only status/timing metadata, not response bodies or credentials:

```bash
mkdir -p evidence
for path in health missing unavailable redirect; do
  printf 'timestamp=%s path=%s ' "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" "$path"
  curl --noproxy '*' -sS -o /dev/null --connect-timeout 3 --max-time 10     -w 'http=%{http_code} connect=%{time_connect} first_byte=%{time_starttransfer} total=%{time_total}
'     "$BASE/$path"
done | tee evidence/http-summary.txt
```

The redirect request intentionally does not use `-L`; it should record 302. If you add `-L`, explicitly record that behavior. Review the file before publishing. **Checkpoint:** explain why bounded timeouts, timestamps, and exact paths improve reproducibility.

## 16. Production failure case: DNS resolution regression

**Simulated symptom:** `curl: (6) Could not resolve host: api.lab.example`. `getent ahostsv4 api.lab.example` also returns no result. The application listener may still be healthy. Hypotheses: resolver unavailable, record missing, wrong DNS view, typo, stale configuration. Discriminating tests: compare known-good resolution, local resolver status, expected record and approved DNS configuration. Root cause **unconfirmed** until one hypothesis is supported. Do not change `/etc/resolv.conf` or production DNS as a shortcut. Recovery verification: resolve the intended name and perform the original HTTPS request.

## 17. Production failure case: connection refused versus timeout

**Simulated case A:** curl returns connection refused immediately. That suggests an active rejection at the target/path, often no listener at the destination endpoint, but can also be generated by a firewall or intermediary. Compare `ss -ltn` on the authorized server and expected bind address. **Case B:** curl reaches its connection timeout. That indicates no completed connection before the deadline; possible causes include filtering, path failure, silent drop, unavailable peer, or asymmetric routing. A timeout is not proof of a firewall block. Preserve exact target, timestamp, source, and exit code. No restart is justified by either error alone.

## 18. Production failure case: TLS identity mismatch

**Simulated symptom:** TCP connects, TLS verification fails with hostname mismatch. The likely failure stage is server authentication, not HTTP application routing. Check intended DNS name, SNI, presented certificate identities, reverse proxy/ingress configuration, and trust chain using approved methods. Do not use `-k` to claim success. An approved certificate deployment or hostname correction may be appropriate after validation. Recovery: HTTPS request succeeds with verification **enabled**, correct hostname, and expected application status.

## 19. Production failure case: 503 with healthy transport

**Simulated timeline:** 10:00 health=200; 10:02 checkout=503; 10:04 checkout=503; 10:06 health=200. DNS, TCP, and TLS succeed for both endpoints. Competing hypotheses: unhealthy upstream pool, dependency failure, overload, deployment regression, maintenance mode. Discriminating evidence: gateway logs, application logs, trace IDs, dependency metrics, request rate, recent changes. The 503 confirms an HTTP-level unavailable response but does **not** identify its origin. Root cause remains open until correlated. Recovery must test checkout, not only health.

## 20. Integrated incident: request-stage classification and safe handoff

**Simulated incident:** `api.production.example` resolves; TCP connect ~20 ms; TLS handshake ~90 ms; `/health` returns 200; `/checkout` returns 503 with a request ID. An engineer recommends disabling certificate verification and restarting the host. Reject both: certificate verification succeeded and the problem is path-specific. Record affected endpoint, time, request ID (sanitized), status, stage timings, and whether the error reproduces. Prioritize application/upstream investigation. Hypotheses include dependency outage and routing misconfiguration; neither is proven. Approved fix depends on backend evidence. Verification: run the original checkout workflow, inspect 5xx rate and latency trends, and ensure no new TLS or routing regression.

## 21. Cleanup and checkpoint

Stop **your own** Python server in Terminal A with Ctrl+C. Verify the actual temporary port is no longer listening using `ss -ltn '( sport = :PORT )'` (replace PORT with the real number). Delete only temporary files you created if you do not wish to retain them:

```bash
rm -f local_http_lab.py headers.txt body.json
```

Keep your sanitized report and evidence. **Final checkpoint:** no firewall, DNS, route, TLS trust, or production service configuration was changed; the loopback listener is stopped; all real observations are labeled accurately.

## Command-and-flag reference — review immediately before the final challenge

| Option/command | Meaning | Engineering caveat |
|---|---|---|
| `curl URL` | Perform request | Default behavior depends on scheme and curl build |
| `-i` / `-I` | Include headers / issue HEAD | HEAD may differ from GET |
| `-D FILE` / `-o FILE` | Save headers / body | Protect sensitive content |
| `-sS` | Quiet progress, show errors | Does not hide response body |
| `-w FORMAT` | Print selected metrics | Stage times are cumulative |
| `-L --max-redirs N` | Follow bounded redirects | Consider cross-host credential risk |
| `--connect-timeout N` | Bound connection establishment | Not overall deadline |
| `--max-time N` | Bound entire operation | May terminate mid-transfer |
| `--fail-with-body` | Nonzero for HTTP >=400 | HTTP status still received |
| `-v` | Verbose protocol details | Can leak sensitive headers |
| `-H` / `-A` | Custom header / User-Agent | Avoid credentials in CLI |
| `--http1.1` | Request HTTP/1.1 | Server response may differ |
| `--noproxy '*'` | Bypass proxies for local lab | Do not bypass enterprise policy |
| `--insecure` | Disable TLS verification | **Never** use as a normal remedy |
| `getent ahostsv4` | Resolve IPv4 host | Does not test HTTP |
| `ss -ltn` | Inspect listeners | Does not prove application health |
| `curl` exit 6/7/28/60 | DNS/connect/timeout/cert examples | Confirm actual stderr and version |

## Troubleshooting decision tree — review immediately before the final challenge

```text
USER REQUEST FAILS
   |
   +-- Name resolution fails? --> inspect DNS/NSS; no HTTP request was sent
   |
   +-- TCP connection fails? --> inspect route, listener, bind address, filtering
   |
   +-- TLS verification fails? --> inspect hostname, certificate, trust; never use -k as fix
   |
   +-- HTTP 3xx? --> inspect Location, redirect count, method, destination
   |
   +-- HTTP 4xx? --> inspect request/auth/authorization/route with approved evidence
   |
   +-- HTTP 5xx? --> correlate proxy, app, upstream, dependency, capacity
   |
   +-- HTTP 2xx but user still fails? --> test original workflow, not only /health
   v
TIMESTAMP + COMPETING HYPOTHESES + DISCRIMINATING TESTS + AUTHORIZED FIX + REVERIFY
```
**Rule:** An HTTP response is not a transport failure; a TCP connection is not proof of application health.

## Integrated final challenge — evidence-based engineering assessment

**Deliverable:** `report/JUN-054-HTTP-Request-Stage-Investigation.md`. Complete the following in order:

1. **Baseline:** record Ubuntu, curl version/features, hostname, UTC timestamp, network interfaces, and listening sockets.
2. **Local endpoint:** run the supplied Python HTTP server on an ephemeral loopback port; record actual URL, listener, and process.
3. **Method comparison:** perform GET, HEAD, header/body capture; explain differences.
4. **Status matrix:** capture 200, 302, 404, and 503, including body and status interpretation.
5. **Redirect test:** compare no-follow with bounded `-L`; record final URL and count.
6. **Transport error:** choose an unused loopback port, capture connection failure and curl exit code.
7. **Timing:** capture lookup, connect, first-byte, total; calculate and interpret stage increments.
8. **HTTPS:** on an approved endpoint, capture TLS verification result; otherwise analyze the supplied simulated certificate failure.
9. **Proxy awareness:** document whether relevant proxy environment variables are set without revealing their values.
10. **Incident analysis:** explain why `/health=200` with `/checkout=503` does not prove full health; provide three competing hypotheses and discriminating tests.
11. **Remediation:** propose authorized next steps, verification of the **original** failing checkout, and prevention.
12. **Cleanup:** stop local server, verify listener removal, sanitize evidence.

**Report template:** Scope → expected behavior → environment → command/evidence table → protocol-stage classification → status and redirect analysis → timing analysis → TLS trust status → competing hypotheses → discriminating tests → root cause (confirmed or unresolved) → approved correction → recovery verification → prevention → cleanup. **Pass condition:** every test is reproducible, claims do not exceed evidence, and no credential or private topology is published.

## Knowledge check with worked answers

**1. Does ping success prove HTTP success?**  
No; ICMP Echo does not establish TCP, TLS, or application response.

**2. What is the difference between a URL scheme and a port?**  
The scheme selects protocol behavior; the port identifies the destination transport endpoint. Defaults depend on scheme.

**3. What does `-I` do?**  
It requests headers using HEAD, which may behave differently from GET.

**4. What does `-i` do?**  
It includes response headers in curl output while retaining the response body.

**5. Does curl exit zero imply HTTP 200?**  
No. Without fail options, HTTP 404/503 can still result in a successful curl transfer exit.

**6. What does `--fail-with-body` do?**  
It makes HTTP error statuses produce a nonzero exit while allowing body output.

**7. What does `-L` do?**  
It follows redirects; bound them with `--max-redirs`.

**8. What is HTTP 302?**  
A redirect response, commonly with a Location header.

**9. What is HTTP 404?**  
The requested resource was not found at that endpoint under the request context.

**10. What is HTTP 503?**  
A server-side unavailable response; it does not identify which upstream component failed.

**11. What does curl exit 6 often mean?**  
Hostname resolution failed.

**12. What does curl exit 7 often mean?**  
Connection to the target could not be established.

**13. What does curl exit 28 mean?**  
An operation timed out.

**14. Why is `-k` unsafe as a fix?**  
It disables TLS certificate verification and can hide impersonation risks.

**15. What is `time_connect`?**  
Cumulative time until TCP connection completion, not just the connect phase alone.

**16. What is `time_starttransfer`?**  
Time until first response byte; includes more than backend processing.

**17. What is a proxy?**  
An intermediary that can change the request path, authentication, and failure location.

**18. Why test `/checkout` after `/health` passes?**  
Health endpoints may not exercise the failing business path or dependencies.

**19. What does `--noproxy` do?**  
Excludes matching destinations from proxy use; used here to keep loopback tests local.

**20. Why timestamp curl evidence?**  
To correlate requests with server logs, traces, deployments, and incidents.

**21. Why should verbose output be redacted?**  
It can include authorization headers, cookies, and sensitive endpoints.

**22. What does a TCP listener prove?**  
A listening socket exists, not that HTTP works.

**23. What is an HTTP method?**  
A request semantic such as GET or HEAD.

**24. Why are repeated samples valuable?**  
They distinguish intermittent symptoms from one-off observations but still need context.

**25. What is the final recovery test?**  
Repeat the original user-visible request from the affected environment.

## Interview preparation with worked answers

**Interview 1: How do you separate DNS from HTTP failure?**  
Run approved name resolution and inspect curl stderr/exit status; a DNS failure occurs before the HTTP request.

**Interview 2: A port is open but curl fails—what next?**  
Inspect protocol scheme, TLS verification, HTTP status, headers, and application logs; an open port proves little about request success.

**Interview 3: How do you capture only HTTP status?**  
Use `curl -sS -o /dev/null -w "%{http_code}\n" URL` with bounded timeouts.

**Interview 4: Why does curl return exit zero on 404?**  
It successfully completed an HTTP transfer; use `--fail-with-body` when HTTP error status should affect exit status.

**Interview 5: How would you investigate 503?**  
Correlate request ID, time, proxy logs, backend health, dependencies, capacity, and recent changes; do not guess a root cause.

**Interview 6: How do you diagnose certificate mismatch?**  
Check expected hostname, SNI, presented certificate names, trust chain, and endpoint routing without disabling verification.

**Interview 7: What is the risk of verbose output?**  
Credentials and private headers can leak; redact and store securely.

**Interview 8: How do you avoid hanging tests?**  
Use `--connect-timeout`, `--max-time`, and bounded redirect count.

**Interview 9: What is wrong with treating `/health=200` as universal health?**  
It may be shallow and bypass the failing user journey.

**Interview 10: How do you interpret curl timings?**  
They are cumulative; compare increments cautiously, with connection reuse and proxy behavior in mind.

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

**JUN-055 — SSH Fundamentals**. Continue in sequence; carry forward your evidence discipline and the distinction between connectivity, authentication, configuration, and application health.

**End of 054 — curl and HTTP Testing.**
