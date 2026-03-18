# DarkFlare - TCP-over-CDN Tunnel

DarkFlare tunnels arbitrary TCP traffic through HTTP(S) connections routed via Cloudflare's CDN infrastructure. The traffic is encapsulated in requests that conform to typical browser patterns, making it indistinguishable from normal web traffic at the network layer.

## Overview

DarkFlare consists of two components:

- **darkflare-client** -- Accepts local TCP connections (or stdin/stdout), encodes the data into HTTP(S) requests with browser-like headers, and sends them to a Cloudflare-protected endpoint.
- **darkflare-server** -- Receives those requests, decodes the encapsulated data, and forwards it as raw TCP to the target service (e.g., SSH on port 22).

The protocol is transport-agnostic: any TCP-based service can be tunneled. Security depends on the encryption of the underlying transport (TLS to Cloudflare, and whatever protocol runs inside the tunnel). DarkFlare provides obfuscation, not encryption -- always use end-to-end encryption (SSH, TLS, etc.) for sensitive traffic.

```
                            FIREWALL/CENSORSHIP
                            |     |     |     |
                            v     v     v     v

[Client]------+            +------------------+            +-----[Target Service]
              |            |                  |            |    (e.g., SSH Server)
              |            |   CLOUDFLARE     |            |tcp   localhost:22
              |tcp         |     NETWORK      |            |
[darkflare    |            |                  |            | [darkflare
 client]------+---HTTPS--->| (looks like      |---HTTPS--->|  server]
localhost:2222|            |  normal traffic) |            | :8080
or stdin/out  |            |                  |            |
              +------------+------------------+------------+
                           |                  |
                           +------------------+

Flow:
1. TCP traffic --> darkflare-client
2. Wrapped as HTTPS --> Cloudflare CDN
3. Forwarded to --> darkflare-server
4. Unwrapped back to TCP --> Target Service
```

## Why Cloudflare

Cloudflare is deeply embedded in global internet infrastructure, serving millions of domains across government, healthcare, finance, and other critical sectors. Blocking Cloudflare's IP ranges causes significant collateral damage, which makes Cloudflare-routed traffic resistant to network-level censorship.

## Censorship Circumvention

In networks that employ deep packet inspection and IP-based blocking, DarkFlare tunnels TCP traffic through HTTPS connections to Cloudflare endpoints that cannot be blocked without disrupting access to large portions of the web. The traffic profile matches normal browser activity, avoiding heuristic detection.

## Features

- Protocol-agnostic TCP tunneling over HTTP/HTTPS
- Cloudflare integration for traffic obfuscation
- Client-controlled destination addressing via base64-encoded headers
- Server-side destination override (`-override-dest`)
- SOCKS5 and HTTP(S) proxy support (`-p`)
- stdin/stdout mode for SSH ProxyCommand integration
- Basic authentication support
- Session management with automatic cleanup
- TLS 1.2+ with configurable cipher suites
- Custom redirect for unauthorized requests
- Request obfuscation (randomized URL paths, browser-like headers)
- Windows DLL variants for fileless execution
- Debug logging (`-debug`) for connection diagnostics

## Quick Start

```bash
# Server (on your public-facing machine)
./darkflare-server -o http://0.0.0.0:8080 -allow-direct

# Client (on your local machine)
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22

# Connect
ssh user@localhost -p 2222
```

For detailed setup instructions covering installation, Cloudflare configuration, SSH integration, proxy support, authentication, VPN tunneling, and troubleshooting, see **[GUIDE.md](GUIDE.md)**.

## Disclaimer

This tool is provided for educational and authorized use only. Users are responsible for compliance with applicable laws and organizational policies.

## Contributing

Bug reports and pull requests are welcome.

## License

MIT License

---
