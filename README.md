# network-debug

> An [OpenCode](https://opencode.ai) skill that teaches AI agents to intercept and debug iOS app network traffic using mitmproxy.

Works with physical iPhones and iOS Simulators. Wraps [mitmproxy-mcp](https://github.com/nicholasgasior/mitmproxy-mcp) with iOS-specific configuration, Tailscale setup, and the gotchas you'd otherwise spend hours discovering.

---

## Quick start

```bash
# 1. Clone and link the skill
git clone https://github.com/kristianvast/opencode-skill-network-debug.git
ln -s "$(pwd)/opencode-skill-network-debug" ~/.config/opencode/skills/network-debug

# 2. Add mitmproxy-mcp to ~/.config/opencode/opencode.json
# (see full config below)

# 3. Trust the CA cert, configure your device, start debugging
```

Then ask your agent: *"Debug the network traffic from my iPhone app."* It knows what to do.

---

## How it works

```
iPhone / Simulator
      |
      | HTTP proxy (port 8888)
      v
  mitmproxy (Mac)  <-->  mitmproxy-mcp  <-->  OpenCode agent
      |                  (localhost)           (reads this skill)
      v
  Internet
```

The MCP server gives the agent tools to inspect captured traffic. This skill gives the agent the knowledge to configure everything correctly for iOS.

---

## What the MCP gives you vs. what this skill adds

The [mitmproxy-mcp](https://github.com/nicholasgasior/mitmproxy-mcp) server exposes tools: `start_proxy()`, `inspect_flow()`, `search_traffic()`, `replay_flow()`, and others.

That's the raw capability. This skill adds the knowledge to use it correctly on iOS:

| Without this skill | With this skill |
|---|---|
| Agent starts proxy on port 8080 | Knows to use port 8888 |
| `--ignore-hosts` regex silently fails | Correct `host:port` format that actually matches |
| Response bodies missing from logs | `flow_detail=4` enabled by default |
| Physical device proxy breaks on network change | Tailscale IP stays stable across networks |
| MCP can't reach physical device | Standalone mitmdump on `0.0.0.0` bridges the gap |
| Xcode provisioning breaks while proxied | Apple domains passed through correctly |
| SSE traffic appears empty | Knows bodies only log on disconnect |
| Simulator proxy left on after session | Teardown commands included |

---

## Prerequisites

- [mitmproxy](https://mitmproxy.org/) via `uvx` (no global install needed)
- [Tailscale](https://tailscale.com/) on both Mac and iPhone, for a stable proxy IP
- [OpenCode](https://opencode.ai) with mitmproxy-mcp configured

---

## Setup

### 1. Install the skill

```bash
git clone https://github.com/kristianvast/opencode-skill-network-debug.git
ln -s "$(pwd)/opencode-skill-network-debug" ~/.config/opencode/skills/network-debug
```

### 2. Configure mitmproxy-mcp

Add to your global `~/.config/opencode/opencode.json`. Keep it disabled globally so it only runs when you need it:

```json
{
  "mcp": {
    "mitmproxy": {
      "type": "local",
      "command": ["uvx", "--python", "3.13", "mitmproxy-mcp"],
      "enabled": false
    }
  }
}
```

Enable it per-project in the project's `opencode.json`:

```json
{
  "mcp": {
    "mitmproxy": {
      "enabled": true
    }
  }
}
```

### 3. Trust the CA certificate

On first run, mitmproxy generates a CA cert at `~/.mitmproxy/mitmproxy-ca-cert.pem`. Trust it on each target:

| Target | Command / Steps |
|---|---|
| macOS | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.mitmproxy/mitmproxy-ca-cert.pem` |
| iOS Simulator | `xcrun simctl keychain <UDID> add-root-cert ~/.mitmproxy/mitmproxy-ca-cert.pem` |
| Physical iPhone | With proxy active, open Safari and go to `http://mitm.it`. Install the profile, then: Settings > General > About > Certificate Trust Settings > enable the mitmproxy cert. |

### 4. Start mitmdump for physical device debugging

The MCP server binds to localhost only, so physical devices need a separate mitmdump process listening on all interfaces.

First, find your Tailscale IP (stable across networks, unlike your DHCP Wi-Fi address):

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale ip -4
```

Then start mitmdump:

```bash
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

On iPhone: **Settings > Wi-Fi > (i) > HTTP Proxy > Manual**
- Server: your Tailscale IP
- Port: `8888`

When you're done, turn the proxy off on your iPhone. Leaving it on while switching networks breaks all connectivity.

### 5. iOS Simulator proxy

The Simulator inherits macOS system proxy settings:

```bash
# Enable
networksetup -setwebproxy "Wi-Fi" 127.0.0.1 8888
networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 8888

# Disable when done
networksetup -setwebproxystate "Wi-Fi" off
networksetup -setsecurewebproxystate "Wi-Fi" off
```

Always disable when you're finished. Leaving this on routes all Mac traffic through mitmproxy.

---

## Lessons learned the hard way

### The `--ignore-hosts` regex matches `host:port`, not just hostname

mitmproxy matches the pattern against the full `host:port` string. A pattern without `:443` will never match HTTPS traffic:

```bash
# Wrong — the actual string is "gateway.icloud.com:443", so this never matches
--ignore-hosts '.*\.icloud\.com$'

# Correct
--ignore-hosts '^(.+\.)?icloud\.com:443$'
```

Without the correct passthrough rules, Xcode provisioning, App Store, and iCloud all break while the proxy is active.

### `flow_detail=4` is required to see response bodies

Without this flag, mitmdump logs only status codes and byte counts. You won't see error messages, JSON responses, or anything useful for debugging. Always include `--set flow_detail=4`.

### SSE stream bodies only appear after disconnect

SSE (Server-Sent Events) connections are long-lived. mitmdump buffers the body and only writes it to the log when the connection closes. If you're debugging SSE traffic, the client must disconnect before you'll see the body in the captured flow.

---

## License

MIT
