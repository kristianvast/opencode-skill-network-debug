# network-debug

An [OpenCode](https://opencode.ai) skill for debugging iOS app network traffic using [mitmproxy-mcp](https://github.com/nicholasgasior/mitmproxy-mcp).

## What this is

An instruction set that teaches AI agents how to intercept and debug HTTP/HTTPS traffic from physical iPhones and iOS Simulators. It wraps the mitmproxy-mcp server with iOS-specific configuration, workflows, and hard-won lessons.

## What the MCP gives you

Tools — `start_proxy()`, `inspect_flow()`, `search_traffic()`, `replay_flow()`, etc.

## What this skill adds

Knowledge the MCP doesn't have:

- **Physical device setup** via Tailscale (stable IP across networks)
- **Apple domain passthrough** — `--ignore-hosts` with correct `host:port` regex so Xcode provisioning, App Store, and iCloud work while proxied
- **Full body logging** — `flow_detail=4` so you can actually read error responses
- **iOS Simulator proxy** setup and teardown
- **CA certificate trust** steps for macOS Keychain, Simulator, and physical devices
- **Standalone mitmdump** config for physical devices (MCP binds to localhost only)
- **Traffic filtering** — grep patterns to cut through Apple/Google/system noise
- **Gotchas** — SSE bodies only log on disconnect, port 8888 not 8080, always disable proxy when done

## Prerequisites

- [mitmproxy](https://mitmproxy.org/) — installed via `uvx` (no global install needed)
- [Tailscale](https://tailscale.com/) — on both Mac and iPhone for stable proxy IP
- [OpenCode](https://opencode.ai) — with mitmproxy-mcp configured

## Setup

### 1. Install the skill

Copy `SKILL.md` to your OpenCode skills directory:

```bash
# Clone
git clone https://github.com/kristianvast/opencode-skill-network-debug.git

# Symlink into OpenCode skills
ln -s "$(pwd)/opencode-skill-network-debug" ~/.config/opencode/skills/network-debug
```

### 2. Configure mitmproxy-mcp

Add to your global `~/.config/opencode/opencode.json`:

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

Enable per-project in `opencode.json`:

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

On first run, mitmproxy generates a CA cert at `~/.mitmproxy/mitmproxy-ca-cert.pem`.

| Target | How |
|---|---|
| macOS | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.mitmproxy/mitmproxy-ca-cert.pem` |
| iOS Simulator | `xcrun simctl keychain <UDID> add-root-cert ~/.mitmproxy/mitmproxy-ca-cert.pem` |
| Physical iPhone | Proxy the phone first, then Safari → `http://mitm.it` → install profile → Settings → General → About → Certificate Trust Settings → enable |

### 4. Physical device proxy (Tailscale)

Use your Mac's Tailscale IP — it never changes, unlike DHCP-assigned Wi-Fi IPs.

```bash
# Find your Tailscale IP
/Applications/Tailscale.app/Contents/MacOS/Tailscale ip -4
```

Start mitmdump for physical device debugging:

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

On iPhone: **Settings → Wi-Fi → (i) → HTTP Proxy → Manual**
- Server: your Tailscale IP
- Port: `8888`

### 5. iOS Simulator proxy

```bash
# Enable
networksetup -setwebproxy "Wi-Fi" 127.0.0.1 8888
networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 8888

# Disable (always do this when done!)
networksetup -setwebproxystate "Wi-Fi" off
networksetup -setsecurewebproxystate "Wi-Fi" off
```

## Key details

### `--ignore-hosts` regex format

mitmproxy matches against `host:port`, not just hostname:

```
# WRONG — won't match (string is "gateway.icloud.com:443")
--ignore-hosts '.*\.icloud\.com$'

# CORRECT
--ignore-hosts '^(.+\.)?icloud\.com:443$'
```

### `flow_detail=4`

Without this flag, mitmdump only logs status codes and sizes — not response bodies. You need `flow_detail=4` to see error messages from the server.

### SSE streams

SSE (Server-Sent Events) responses are long-lived connections. mitmdump only logs the body when the connection closes. To see SSE traffic, the client must disconnect first.

## License

MIT
