---
name: network-debug
description: Debug HTTP/HTTPS traffic from the iOS app, browser, or backend using mitmproxy-mcp. Use when investigating API errors, inspecting request/response bodies, debugging auth issues, or capturing traffic from physical iOS devices.
---

# Network Debugging with mitmproxy-mcp

Intercept and inspect HTTP/HTTPS traffic from iOS apps, web clients, or backends using mitmproxy.

## When to Use This Skill

- Investigating API errors (4xx, 5xx responses)
- Inspecting request/response bodies and headers
- Debugging authentication/session issues
- Capturing traffic from the physical iPhone or iOS Simulator
- Verifying SSE streaming (AI chat endpoint)
- Replaying or modifying captured requests
- Generating OpenAPI specs from live traffic

## MCP Tools Available

The `mitmproxy` MCP server exposes these tools:

| Tool | Purpose |
|---|---|
| `start_proxy(port)` | Start proxy (default 8080) |
| `stop_proxy()` | Stop proxy |
| `set_scope(allowed_domains)` | Whitelist domains — **always call immediately** |
| `get_traffic_summary(limit)` | List recent flows |
| `inspect_flow(flow_id)` | Full request/response + curl command |
| `search_traffic(query, domain, method)` | Filter flows |
| `replay_flow(flow_id, ...)` | Re-send with modifications |
| `add_interception_rule(...)` | Inject headers, replace bodies, block |
| `extract_from_flow(flow_id, ...)` | Extract via JSONPath or CSS selector |
| `export_openapi_spec(domain)` | Generate OpenAPI v3 from traffic |
| `detect_auth_pattern(flow_ids)` | Infer auth mechanism |
| `fuzz_endpoint(flow_id, ...)` | Lightweight DAST fuzzing |
| `clear_traffic()` | Wipe flow history |

## Standard Debugging Workflow

### Step 1: Start and Scope

```
start_proxy(port=8888)
set_scope(["localhost", "your-api-domain.com"])
```

**CRITICAL**: Always call `set_scope()` immediately after starting. Without it, Apple telemetry, iCloud, and OS background noise flood the capture within seconds.

**Port selection**:
- Do NOT use `8080` — that's the Vite dev server
- Use `8888` or `9090`

### Step 2: Capture Traffic

Ask the user to reproduce the issue, then:

```
get_traffic_summary(limit=30)
```

### Step 3: Inspect Errors

Look for non-200 status codes, then inspect:

```
inspect_flow("<flow_id>")
```

This returns the full request (method, URL, headers, body) and response (status, headers, body, size) plus an equivalent curl command.

### Step 4: Deep Analysis (if needed)

```
# Extract specific values from JSON responses
extract_from_flow("<flow_id>", json_path="$.error")

# Search for patterns across all captured traffic
search_traffic(query="error", domain="liv.agrointel.no")

# Replay a request with modifications
replay_flow("<flow_id>", headers_json='{"Authorization": "Bearer new-token"}')
```

### Step 5: Cleanup

```
stop_proxy()
```

If the user has a physical device proxied, remind them:
> Turn off the proxy on your iPhone: Settings → Wi-Fi → (i) → Proxy → Off

## Physical Device Setup

The MCP `start_proxy()` binds to **localhost only** — physical devices can't reach it.

### For physical iPhone debugging, run standalone mitmdump:

```bash
# Kill any existing proxy
kill $(lsof -t -i :8888) 2>/dev/null

# Start on all interfaces with full body logging + Apple passthrough
nohup uvx --python 3.13 --from mitmproxy mitmdump \
  --listen-host 0.0.0.0 --listen-port 8888 \
  --set confdir=$HOME/.mitmproxy \
  --set flow_detail=4 \
  --ignore-hosts '^(.+\.)?apple\.com:443$' \
  --ignore-hosts '^(.+\.)?icloud\.com:443$' \
  --ignore-hosts '^(.+\.)?mzstatic\.com:443$' \
  --ignore-hosts '^(.+\.)?cdn-apple\.com:443$' \
  --ignore-hosts '^(.+\.)?push\.apple\.com:443$' > /tmp/mitmproxy.log 2>&1 &
```

The `--ignore-hosts` regex matches against `host:port` (e.g. `gateway.icloud.com:443`), NOT just the hostname. The `:443$` suffix is required — without it the pattern won't match and Apple services will break. These flags pass Apple's certificate-pinned domains straight through without intercepting.

Then tell the user to configure their iPhone:
1. Settings → Wi-Fi → tap (i) on network → Configure Proxy → Manual
2. Server: `100.86.4.25` (Tailscale IP — both devices must have Tailscale running)
3. Port: `8888`

**IMPORTANT**: Use the Tailscale IP (`100.86.4.25`), NOT the local Wi-Fi IP. DHCP-assigned IPs change between sessions and will silently break the proxy (the phone sends traffic to a dead IP with no error — just no connectivity). The Tailscale IP is stable across all networks.

With `--ignore-hosts`, the proxy can stay on permanently — no need to toggle it off for Xcode builds or App Store access.

### Read traffic from standalone mitmdump:

With `flow_detail=4`, the log includes full request/response headers and bodies. Parse it:

```bash
# All requests with responses
grep -A1 "HTTP/2.0" /tmp/mitmproxy.log | tail -40

# Only errors (non-200 responses)
grep "<< HTTP" /tmp/mitmproxy.log | grep -v "200 OK"

# Full error details — find a 4xx/5xx, then read the body after it
grep -A30 "400\|401\|403\|404\|500" /tmp/mitmproxy.log | tail -40

# Filter app traffic (exclude OS noise)
grep -v "apple\|icloud\|google\|snapchat\|sentry\|paypal\|facebook\|instagram" /tmp/mitmproxy.log | grep -E "GET |POST |PUT |DELETE |<< HTTP" | tail -30

# Only phone traffic (filter by Tailscale subnet or local IP)
grep "100\.\|192.168." /tmp/mitmproxy.log | grep -v "apple\|icloud\|google\|snapchat" | tail -20
```

**Key**: `flow_detail=4` is critical. Without it, response bodies are not logged and you cannot see error messages from the server.

## iOS Simulator Setup

The simulator inherits macOS system proxy settings:

```bash
networksetup -setwebproxy "Wi-Fi" 127.0.0.1 8888
networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 8888
```

MCP tools work normally for simulator traffic (localhost is reachable).

Disable when done:
```bash
networksetup -setwebproxystate "Wi-Fi" off
networksetup -setsecurewebproxystate "Wi-Fi" off
```

## Certificate Trust

CA cert: `~/.mitmproxy/mitmproxy-ca-cert.pem`

| Target | Trust Command |
|---|---|
| macOS Keychain | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.mitmproxy/mitmproxy-ca-cert.pem` |
| iOS Simulator | `xcrun simctl keychain <UDID> add-root-cert ~/.mitmproxy/mitmproxy-ca-cert.pem` |
| Physical iPhone | Safari → `http://mitm.it` → install profile → Settings → General → About → Certificate Trust Settings → enable |

**Prerequisite for physical device**: The phone must already be proxied through mitmproxy for `mitm.it` to resolve.
