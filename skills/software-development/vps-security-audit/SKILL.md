---
name: vps-security-audit
description: Use when auditing or hardening a Linux VPS or server.
---

# VPS & Server Security Audit

A systematic procedure for auditing Linux VPS, self-hosted home servers, and cloud instances for security vulnerabilities, resource bottlenecks, and exposure to attacks.

## Procedure

### 1. Resource & OOM Bottleneck Inspection
Examine memory, swap, and long-running background processes before diagnosing network issues. A server that goes down under light load is frequently crashing from swap thrashing or OOM exhaustion rather than raw bandwidth saturation.

```bash
# Check memory, swap pressure, uptime, and load averages
free -m
uptime
vmstat 1 5

# Identify top resource consumers (CPU and RSS memory)
ps aux --sort=-%mem | head -n 15
ps aux --sort=-%cpu | head -n 10

# Detect orphaned CLI/agent processes holding deleted pseudo-terminals or headless browsers
ls -l /proc/*/fd/* 2>/dev/null | grep '(deleted)' | grep pts
ps aux | grep -iE 'chromium|chrome|headless' | grep -v grep

# Check kernel dmesg for OOM-killer invocations or TCP SYN drops
dmesg -T | grep -iE 'oom|killed process|syn flood' | tail -n 20

# Check for crash-looping systemd services saturating disk and syslog
systemctl --failed
grep -E 'Scheduled restart job|Failed to start' /var/log/syslog 2>/dev/null | tail -n 15
```

- **Rule**: Cap memory limits on container runtimes. For already-running containers, apply limits live without downtime via `docker update --memory 768m --memory-swap 1g <container>`. If swap usage is high and free memory is < 200MB, incoming request bursts trigger disk I/O freezes.
- **Rule**: Inspect and terminate orphaned CLI agent processes (`opencode`, detached subshells) holding deleted pseudo-terminals (`ls -l /proc/*/fd/* 2>/dev/null | grep '(deleted)' | grep pts`). These detached background processes can silently consume hundreds of megabytes of RSS memory and keep the host permanently near OOM failure.
- **Pitfall**: Detached interactive CLI agents or subshells whose parent terminal was closed (`pts (deleted)`) can remain active in background loops, consuming memory and keeping the server permanently near OOM. Verify process trees with `pstree -p <PID>` and inspect `/proc/<PID>/cmdline` before terminating.
- **Pitfall**: Headless browser automation (Chromium, Chrome, Playwright) can leave orphaned zombie processes after command timeouts or unhandled exits. Orphaned browser daemons consume 200MB–400MB RSS each and loop in CPU busy-wait states, driving system load averages above 10. Sweep with `pkill -9 -f "chromium|chrome" 2>/dev/null || true` and clean `/tmp/snap-private-tmp` or `/tmp/agent-browser-chrome-*`.
- **Pitfall**: Crash-looping background services (e.g. DNS daemons repeatedly failing to bind ports due to conflicts) can restart hundreds of thousands of times unnoticed, bloating `/var/log/syslog` by hundreds of megabytes and causing CPU churn. Check `systemctl --failed` and disable non-functional services.

### 2. Network Exposure & Listening Ports Audit
Identify all interfaces, listening services, and unintended external bindings.

```bash
# List all listening TCP and UDP sockets with owning processes
ss -tulpn

# Verify public IP and routing context
ip a
curl -s https://ipinfo.io/json
```

- **Rule**: Never expose databases (MySQL 3306, PostgreSQL 5432, Redis 6379) or internal web panels to `0.0.0.0`. Bind strictly to `127.0.0.1` or LAN interfaces.
- **Pitfall**: Docker publishes ports by prepending `iptables` NAT rules that bypass UFW entirely. Binding `-p 8080:80` exposes the port to the public internet even if UFW has default deny. Always specify `-p 127.0.0.1:8080:80` unless public exposure is intended, or deploy `ufw-docker`.

### 3. Reverse Proxy & CDN Edge Configuration
Verify that reverse proxies (Nginx, Caddy, Traefik) and edge tunnels (Cloudflare Tunnel, Tailscale) correctly reconstruct client IP headers.

```bash
# Check Nginx error and access logs for client IP resolution
tail -n 30 /var/log/nginx/error.log /www/wwwlogs/*.log 2>/dev/null
```

- **Pitfall**: When traffic enters via Cloudflare Tunnel (`cloudflared`) or reverse proxies without configuring trusted proxy IPs (`set_real_ip_from` and `real_ip_header CF-Connecting-IP;`), Nginx logs the client as `127.0.0.1`. Any downstream rate-limiting zone (e.g. `limit_req_zone $binary_remote_addr`) pools all users into `127.0.0.1`. A single attacker will trigger rate limits that block every legitimate user globally.
- **Rule**: Configure CDN edge security (Cloudflare WAF, Rate Limiting Rules, Bot Fight Mode) rather than local rate limiting. For home servers, traffic must be mitigated before reaching the local home router, because home broadband connections and consumer NAT tables saturate at low packet volumes.
- **Rule**: When routing sites via Cloudflare Tunnel (`cloudflared`), bind Nginx virtual hosts strictly to loopback (`listen 127.0.0.1:<port>;`). Never bind to `0.0.0.0` or open router ports; traffic flows via the outbound tunnel daemon, keeping local networks completely shielded from external port scans.
- **Rule**: Autonomous DNS Provisioning for Cloudflare Tunnels: If `cloudflared tunnel route dns` fails with `code: 10000, reason: Authentication error` (due to expired or missing `cert.pem` origin cert token), manage DNS directly via Cloudflare REST API using a scoped API token (`Zone.DNS Edit` + `Zone.Zone Read`):
  1. Retrieve Zone ID: `GET https://api.cloudflare.com/client/v4/zones?name=<domain>`
  2. Create/Update CNAME: `POST/PUT https://api.cloudflare.com/client/v4/zones/<zone_id>/dns_records` pointing `<subdomain>` to `<tunnel_id>.cfargotunnel.com` with `"proxied": true`.
  3. Store token securely in `/root/.cloudflare_token` (`chmod 600`) to enable fully autonomous future deployments without prompting the user.
- **Rule**: Return HTTP `444` (Connection Closed Without Response) in Nginx for malicious scanner probe patterns (`.env`, `.git`, `.bak`, `.sql`). Unlike 404, status 444 closes the TCP connection immediately without transmitting response headers or bodies, cutting CPU cycles and conserving upload bandwidth.
- **Pitfall**: Hard-restarting `cloudflared` (`systemctl restart cloudflared`) tears down all active QUIC/HTTP2 tunnel connections to Cloudflare edge nodes. During the 5–15 second reconnect and TLS handshake window, Cloudflare Edge cannot reach the origin host and immediately returns HTTP **Error 1033 (Argo Tunnel error)** to end users. Always validate ingress syntax first (`cloudflared tunnel --config <path> ingress validate`) and prefer reload/SIGHUP over hard restarts during active traffic.
- **Rule**: Set `net.ipv4.ping_group_range = 0 2147483647` when using `cloudflared` ICMP proxying on Linux hosts so GID 0 (root) and container users can send ICMP packets without `permission denied` warnings in syslog.
- **Rule**: Configure explicit `originRequest` timeout parameters (`connectTimeout: 30s`, `keepAliveTimeout: 90s`) in `/etc/cloudflared/config.yml` for long-polling, streaming, or AI gateway endpoints (`/v1/chat/completions`) to prevent edge stream cancellations (`error="stream canceled by remote with error code 0"`).
- **Rule**: Prevent Agent-Gateway Hairpinning Deadlocks. When an autonomous AI agent or daemon runs on the same host as its LLM backend or API router (e.g. `9router`, vLLM, LiteLLM, Ollama), never configure the agent's `base_url` to traverse public CDN or Cloudflare Tunnel domains (e.g. `https://gateway.domain.com/v1`). Point the agent strictly to local loopback (e.g. `http://127.0.0.1:<port>/v1`). External routing introduces hairpinning fragility: any tunnel reload, ISP hiccup, or transient Cloudflare Error 1033 immediately severs the agent's connection to its own LLM mid-execution, crashing active tasks. Local loopback provides sub-millisecond latency and complete immunity to ingress tunnel restarts.
- **Rule**: Multi-Domain Custom Hostnames in Cloudflare Tunnel Architectures. In multi-domain setups (e.g. URL shorteners, multi-tenant web apps allowing custom domains), registering a domain in the application's local database does not automatically route internet traffic to the host. Two independent routing layers must be synchronized:
  1. Edge DNS: The custom domain's DNS must point a CNAME to `<tunnel_id>.cfargotunnel.com` (with Cloudflare proxy enabled) in its authoritative DNS zone. If the custom domain resides in a separate Cloudflare account, cross-zone CNAME routing requires Cloudflare for SaaS (Custom Hostnames) or mutual zone ownership to prevent Cloudflare Error 1014 (CNAME Cross-User Banned).
  2. Local Tunnel Ingress: To achieve zero-configuration multi-domain routing where domains can be added/removed dynamically without modifying `/etc/cloudflared/config.yml` or restarting the daemon, set the tunnel's terminal ingress rule to point directly to the application backend (`- service: http://127.0.0.1:<port>`) instead of `- service: http_status:404`. Any incoming domain whose DNS hits the tunnel will be caught and routed to the multi-tenant application immediately.
- **Rule**: Application-Level Security Hardening for Tunnel-Bypassed Services. When backend applications are routed directly from `cloudflared` (e.g. Node.js/Hono, Go, Python containers) bypassing Nginx L7 inspection, the application layer must enforce its own defense-in-depth:
  1. Mandatory Security Headers: HSTS (`max-age=31536000; includeSubDomains; preload`), `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy: strict-origin-when-cross-origin`.
  2. Anti-SSRF URL Filtering: Reject loopback (`127.0.0.1`, `::1`, `localhost`), RFC 1918 private subnets (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), and cloud metadata IP `169.254.169.254`.
  3. Active Anti-Brute-Force & Account Lockout: Enforce in-memory attempt tracking per client IP (parsed via `CF-Connecting-IP`). Lock IPs for 15 minutes (HTTP 429) after 5 consecutive failed authorization attempts.
  4. Immediate Scanner Probe Dropping: Intercept and drop known vulnerability scanners (`/.git`, `/.env`, `wp-login.php`, `phpinfo`) with immediate 404/400 without querying backend databases.
  5. Payload Size Caps: Limit request body parsing (e.g. 64KB max) to guard against memory exhaustion DoS.
- **Rule**: Stealth Admin UI & Client-Side Credential Verification. In web applications featuring stealth admin controls or hidden management toolbars, never reveal administrative controls based on local input existence (`if (key)`) or unverified localStorage keys. Admin DOM nodes must be rendered strictly hidden by default without conflicting display classes (`hidden` without `flex`). Reveal controls ONLY after an asynchronous server verification probe to an authenticated endpoint returns HTTP 200. On invalid keys, empty values, or HTTP 401/429 errors, immediately re-enforce the hidden DOM state, purge stored credentials, and prevent interface disclosure to unauthorized visitors.
- **Rule**: Integrating Hermes Agent in Coolify with Upstream AI Routers (9router / OpenAI-Compatible Endpoints). When deploying the official Coolify service template `Hermes Agent With Webui` (`hermes-agent` + `hermes-webui`), the default compose template preconfigures standard commercial API keys (`OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) and launches `hermes-agent` with default auto-provider resolution. Connecting to an upstream AI router (e.g. `9router`) can be done in two ways:
  1. Standard Universal Approach (Recommended for 1-Click Templates): Set standard OpenAI-compatible environment variables in Coolify's Environment Variables tab:
     ```bash
     OPENAI_BASE_URL=https://<gateway_domain>/v1
     OPENAI_API_KEY=<gateway_api_key>
     HERMES_MODEL=<model_name>
     ```
     Or use `AI_GATEWAY_BASE_URL` and `AI_GATEWAY_API_KEY` which are native first-class environment variables recognized by Hermes Agent runtime resolution for custom endpoint routing.
  2. Custom Provider via Docker Compose: If configuring via `HERMES_MODEL_PROVIDER=custom`:
     - Set `HERMES_MODEL_PROVIDER=custom`, `HERMES_MODEL_DEFAULT=<model_name>`, `HERMES_MODEL_BASE_URL=https://<gateway_domain>/v1`.
     - Strict Custom API Key Variable Resolution: Hermes derives the required environment variable name for custom providers strictly from the base_url hostname via `custom_endpoint_key_env()` (`re.sub(r"[^A-Z0-9]+", "_", host.upper())`). Generic variable names like `HERMES_CUSTOM_API_KEY` or `CUSTOM_API_KEY` are ignored by Hermes runtime resolution and trigger HTTP 401 Unauthorized errors. The environment variable MUST match the exact host slug: `HERMES_CUSTOM_<ESCAPED_HOST>_API_KEY` (e.g., `HERMES_CUSTOM_9ROUTER_WN_CDN_NET_API_KEY` for `https://9router.wn-cdn.net/v1`).
  3. Upstream Tunnel Timeout Tuning: Ensure the upstream gateway tunnel specifies generous client timeouts (`connectTimeout: 30s`, `keepAliveTimeout: 90s`) in `/etc/cloudflared/config.yml` to sustain long-polling and SSE streaming tokens without intermediate dropped connections.
  4. WebUI Stale-Session Model Persistence: In `hermes-webui`, an existing chat session retains its initial model metadata even after environment variables change. When troubleshooting "No LLM provider configured", always start a new conversation via `+` (New Chat) or set the target model via the bottom input bar model picker (`CUSTOM MODEL ID` -> add `<model_name>`) rather than re-sending into a bricked failed turn.
- **Rule**: Global NPM Package Upgrades on Linux Hosts. When upgrading CLI packages globally with strict network requirements (e.g. `npm i -g <pkg>@latest --prefer-online`), check both the system-wide path and multi-version Node runtime managers (e.g. aaPanel `/www/server/nodejs/v<ver>/bin/` or NVM). After installation, verify the active binary version with `<binary> --version` and inspect whether containerized services mount or wrap that runtime.
- **Pitfall**: Seeing 'no available server' (HTTP 503) when browsing a newly registered custom domain indicates that requests are hitting a legacy origin or reverse proxy (e.g. Traefik/Coolify with no active backend) rather than the intended tunnel server. Check DNS resolution and inspect `/etc/cloudflared/config.yml` before debugging the local application container.
- **Pitfall**: Seeing 'Sorry, the website has been stopped' or default control panel placeholder pages (e.g. aaPanel `0.default.conf`) when accessing a virtual host through Cloudflare Tunnel indicates a port misalignment between `/etc/cloudflared/config.yml` and the local web server. When web server vhosts bind to custom loopback ports (e.g. `listen 127.0.0.1:9696;` or `9697`) to isolate vhosts, configuring tunnel ingress to `http://127.0.0.1:80` bypasses the target vhost and lands on the server's default catch-all vhost. Always check `ss -tulpn` and the vhost's `listen` directive to confirm the exact loopback port before mapping tunnel ingress.
- **Rule**: Secure Reverse Proxy for Client-Side SPAs & Upstream AI Gateways: When deploying Single Page Applications (React/Vue/Vite) that query internal AI gateways (e.g. `9router`, vLLM, LiteLLM) or upstream APIs, never bake API keys into `.env` or client bundles (`VITE_*` variables land in plaintext inside `dist/assets/*.js`). Instead, create an Nginx internal reverse proxy:
  ```nginx
  location /api/ai-proxy/ {
      proxy_pass http://127.0.0.1:<port>/v1/;
      proxy_set_header Host <gateway_host>;
      proxy_set_header Authorization "Bearer <server_key>";
      proxy_set_header Content-Type "application/json";
      proxy_buffering off;
      proxy_read_timeout 90s;
      proxy_connect_timeout 15s;
  }
  ```
  The frontend fetches `/api/ai-proxy/chat/completions` relative to the current origin. This keeps API tokens strictly on the server, eliminates CORS preflight errors, and allows edge-level rate-limiting and prompt inspection.

### 4. Firewall & Intrusion Detection Check
Audit packet filtering, Fail2ban jails, and automated ban scripts.

```bash
# Check firewall status and active rules
ufw status verbose
iptables -S | head -n 30

# Check Fail2ban jail status
fail2ban-client status
```

- **Rule**: Dual-layer IP Banning for Cloudflare Tunnels. Because Cloudflare Tunnel enters locally as `127.0.0.1`, a local UFW deny rule alone cannot drop tunnel packets. Combine: (1) `ufw insert 1 deny from <ip> to any` to block direct connection bypass attempts; and (2) dynamic Nginx L7 blocklist (`include /etc/nginx/conf.d/banned_ips.conf;` containing `deny <ip>;`). When `real_ip_header CF-Connecting-IP;` is configured, Nginx drops tunnel requests with 403/444 before passing to upstream applications.
- **Rule**: Out-of-Band Routing for Sensitive Telemetry and Login Monitoring: Never expose internal access logs, session monitors, or telemetry dashboards via public web endpoints/URLs (even if protected with tokens or hidden query parameters). Public endpoints invite crawling, brute-forcing, and credential discovery. Instead, route sensitive login events, real IP tracking, and security alerts directly to a private out-of-band communication channel (e.g. administrator Telegram bot). Keep high-frequency background operations (data syncing, auto-save) silent in local storage, and dispatch instant notifications only for authentication events (`login`, `credential/PIN change`) with a short debounce window (e.g. 15s) to eliminate duplicate alerts.
- **Rule**: When executing post-incident file integrity sweeps (`find`), use directory pruning (`\( -name .git -o -name node_modules -o -name .cache -o -name storage \) -prune -o ... -print`) rather than post-traversal path exclusions (`! -path`). Descending into massive directory trees like `.git` or `node_modules` causes `find` to time out on disk I/O under concurrent host load.
- **Rule**: Register newly deployed virtual host access logs with active threat monitoring daemons. Omitting newly created vhost access logs (e.g. `/www/wwwlogs/<domain>.log`) from watchdog scripts blinds automated L4/L7 threat mitigation to attacks targeting newly published subdomains.
- **Rule**: Project-Aware Attack Attribution in Security Monitoring & Alarms: In multi-project or multi-tenant VPS environments (hosting multiple vhosts, containers, and services), alerts reporting only raw IP and URI (`Target: /.env`) leave operators blind to which application is being targeted. Intrusion detection and alarm bots must explicitly attribute each attack to its specific project context:
  1. *Web Log Watchers*: Dynamically discover and map access logs (`/www/wwwlogs/*.log`) to application names (`check.wn-cdn.net.log` -> Check Ceklok, `hbd.wn-cdn.net.log` -> Web HBD).
  2. *Container Watchers*: Tail Docker container logs (`docker logs -f --tail 0 <container>`) for structured application security signals (e.g. `[SECURITY_PROBE] IP: <ip> | Path: <path>`) to cover tunnel-routed services that bypass Nginx.
  3. *L4/DDoS Watchers*: In socket flood analysis (`ss -tnp`), correlate the attacked local port (e.g. `9696`, `9697`, `8788`, `20128`, `22`) to the target service/project.
  4. *Alarm Notification Formatting*: Position the target project prominently at the top (`Target Proyek: <project_name>`) alongside attack type and IP, providing instant situational awareness without manual log searches.
- **Rule**: When conducting post-incident integrity sweeps on web roots, ignore framework runtime caches and session directories (e.g. `*/storage/framework/cache/*`, `*/storage/framework/sessions/*`, `*/storage/framework/views/*`, `*/storage/logs/*` for Laravel, or `*/node_modules/*`). Routine framework session/cache writes generate false-positive integrity alarms during normal traffic.
- **Pitfall**: Whitelists in automated intrusion detection/mitigation engines must explicitly include the host server's own public WAN IP, not just private RFC1918 subnets and loopback. When local health checks or test probes query public CDN/Tunnel endpoints (e.g. `https://subdomain.domain.com`), Cloudflare forwards requests with `CF-Connecting-IP` set to the server's public WAN IP. If omitted from the whitelist, internal tests or health probes trigger self-bans, locking out the server from its own hosted services.
- **Pitfall**: Appending rules via `ufw deny from <ip> to any` puts the rule at the bottom of the chain. If earlier `ALLOW` rules exist for ports 80/443/2222, the appended deny rule is bypassed. Always use `ufw insert 1 deny from <ip> to any`.
- **Pitfall**: When writing custom socket monitoring scripts using `ss -tnp`, parse column 5 (Peer Address) rather than column 4 (Local Address). Inverting the source and peer sockets causes automated firewalls to attempt banning localhost or own server interfaces.

### 5. SSH & User Privilege Hardening
Audit user accounts, sudo rights, and active SSH runtime parameters.

```bash
# Verify EFFECTIVE OpenSSH configuration (resolves include overrides)
sshd -T | grep -iE 'passwordauthentication|permitrootlogin|port'

# Audit users with UID >= 1000 and login shells
awk -F: '$3 >= 1000 && $7 !~ /nologin|false/ {print $1, $3, $6, $7}' /etc/passwd

# Check password status and expiration for privileged users
passwd -S root
grep -Po '^sudo:.*:\K.*' /etc/group | tr ',' '\n' | xargs -n1 passwd -S 2>/dev/null

# Identify phantom sudo accounts (active password but no home directory or zero login history)
for u in $(grep -Po '^sudo:.*:\K.*' /etc/group | tr ',' ' '); do [ ! -d "/home/$u" ] && echo "Phantom sudoer: $u"; done
```

- **Pitfall**: In Ubuntu/Debian systems, `/etc/ssh/sshd_config.d/*.conf` (e.g. `50-cloud-init.conf`) overrides `/etc/ssh/sshd_config`. Setting `PasswordAuthentication no` in the main config has NO effect if a cloud-init drop-in sets `yes`. Always inspect effective settings using `sshd -T`.
- **Rule**: Lock or remove unused user accounts that have sudo membership. Sudoers without home directories or SSH keys are frequent remnants of templates or backdoors.
- **Rule**: Disable legacy/unneeded daemons. Headless servers frequently run snap printer daemons (`cupsd`, `cups-browsed` on port 631) or unneeded FTP servers (`pure-ftpd` on port 21). Disable and stop them immediately to eliminate critical RCE vectors and recover memory.

### 6. Kernel & TCP Stack Hardening
Audit kernel sysctl parameters to ensure the OS network stack resists connection exhaustion and SYN flood attacks.

```bash
# Check current SYN flood and connection backlog parameters
sysctl net.ipv4.tcp_syncookies net.ipv4.tcp_max_syn_backlog net.core.somaxconn net.netfilter.nf_conntrack_max
```

- **Rule**: Enable `net.ipv4.tcp_syncookies = 1` and increase `net.ipv4.tcp_max_syn_backlog = 65536` with `net.core.somaxconn = 65535` to prevent connection queue drop under burst traffic.
- **Rule**: Reduce `net.ipv4.tcp_fin_timeout` (e.g. 15s) and enable `net.ipv4.tcp_tw_reuse = 1` to accelerate socket recycling during rapid HTTP connection cycling.

### 7. Disk Hygiene & Storage Maintenance Sweeps
Audit and reclaim disk space from accumulated packaging, container, and daemon runtime layers.

```bash
# Inspect disk consumption across Docker, Snap, journals, and caches
docker system df
du -sh /var/lib/snapd/snaps /var/cache/apt/archives ~/.cache /var/log 2>/dev/null
journalctl --disk-usage
```

- **Rule**: Cap Snap retention to 2 revisions (`snap set system refresh.retain=2`) and purge disabled historical snapshots (`snap list --all | awk '/disabled/{print $1, $3}' | while read s r; do snap remove "$s" --revision="$r"; done`) to reclaim gigabytes in `/var/lib/snapd/snaps`.
- **Rule**: Prune unreferenced Docker builder caches and untagged intermediate layers (`docker builder prune --all -f && docker image prune -f`) after local container rebuilds to prevent unmonitored disk exhaustion.
- **Rule**: Vacuum systemd journals to bounded windows (`journalctl --vacuum-time=3d` or `--vacuum-size=100M`) and clean package installer caches (`apt-get autoremove --purge -y && apt-get clean`, purge `~/.cache/uv`, `~/.cache/pip`, `~/.cache/ms-playwright`).

## Workflow Rules
- **Phased Execution**: Always structure VPS audit and remediation into explicit phases using a task tracker (`todo_list`) before making system changes. Execute and verify each phase sequentially with real command output before proceeding to the next.
- **Service Verification**: After modifying daemon configs (Nginx, MySQL, systemd services), immediately test configuration syntax (`nginx -t`) and confirm the bound ports (`ss -tulpn`) before concluding the task.
