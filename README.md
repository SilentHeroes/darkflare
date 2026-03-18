# DarkFlare - TCP-over-CDN Tunnel

DarkFlare tunnels arbitrary TCP traffic through HTTP(S) connections routed via CDN infrastructure (Cloudflare, Akamai, Fastly, CloudFront, etc.). The traffic is encapsulated in requests that conform to typical browser patterns, making it indistinguishable from normal web traffic at the network layer.

## Overview

DarkFlare consists of two components:

- **darkflare-client** -- Accepts local TCP connections (or stdin/stdout), encodes the data into HTTP(S) requests with browser-like headers, and sends them to a CDN-protected endpoint.
- **darkflare-server** -- Receives those requests, decodes the encapsulated data, and forwards it as raw TCP to the target service (e.g., SSH on port 22).

The protocol is transport-agnostic: any TCP-based service can be tunneled. Security depends on the encryption of the underlying transport (TLS to the CDN, and whatever protocol runs inside the tunnel). DarkFlare provides obfuscation, not encryption -- always use end-to-end encryption (SSH, TLS, etc.) for sensitive traffic.

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
2. Wrapped as HTTPS --> Cloudflare CDN (or any CDN)
3. Forwarded to --> darkflare-server
4. Unwrapped back to TCP --> Target Service
```

## Why CDNs

CDN providers such as Cloudflare, Akamai, Fastly, and Amazon CloudFront are deeply embedded in global internet infrastructure. They serve millions of domains across government, healthcare, finance, and other critical sectors. Blocking CDN IP ranges causes significant collateral damage, which makes CDN-routed traffic resistant to network-level censorship. Regional alternatives (CDNetworks in Russia, ArvanCloud in Iran, ChinaCache in China) may provide similar properties in jurisdictions where global CDNs are less prevalent.

## Censorship Circumvention

In countries that employ deep packet inspection and IP-based blocking (China's Great Firewall, Iran's filtering infrastructure, Russia's VPN restrictions), DarkFlare tunnels TCP traffic through HTTPS connections to CDN endpoints that cannot be blocked without disrupting access to large portions of the web. The traffic profile matches normal browser activity, avoiding heuristic detection.

## Features

- Protocol-agnostic TCP tunneling over HTTP/HTTPS
- CDN integration (Cloudflare and others) for traffic obfuscation
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

## Installation

1. Download the latest release from the GitHub Releases page.
   - Binaries are available for:
     - `darkflare-client-darwin-arm64` -- macOS Apple Silicon
     - `darkflare-client-darwin-amd64` -- macOS Intel
     - `darkflare-client-linux-amd64` -- Linux x64
     - `darkflare-client-windows-amd64.exe` -- Windows x64
     - Corresponding `darkflare-server-*` binaries
2. Verify checksums against `checksums.txt`.
3. Make binaries executable (Unix):
```bash
chmod +x darkflare-client-* darkflare-server-*
```

## Quick Start

### Server

```bash
# HTTP (testing/development)
./darkflare-server -o http://0.0.0.0:8080 -allow-direct

# HTTPS (production, with Cloudflare origin certificates)
./darkflare-server -o https://0.0.0.0:443 -c /path/to/cert.pem -k /path/to/key.pem

# With authentication
./darkflare-server -o http://0.0.0.0:8080 -auth user:pass
```

### Client

```bash
# Basic SSH tunnel
./darkflare-client -l 2222 -t https://cdn.example.com -d localhost:22

# Then connect
ssh user@localhost -p 2222

# With proxy
./darkflare-client -l 2222 -t https://cdn.example.com -d localhost:22 -p socks5://proxy:1080

# Skip TLS verification (testing only)
./darkflare-client -l 2222 -t https://direct.example.com -d localhost:22 -insecure
```

## stdin/stdout Mode

The client supports stdin/stdout mode (`-l stdin:stdout`) for direct integration with SSH ProxyCommand. This avoids binding to local ports, which is useful when:

- Local firewall rules prevent port binding
- Non-root execution is required
- Integration with SSH config is preferred

### SSH ProxyCommand

Command line:
```bash
ssh -o ProxyCommand="darkflare-client -l stdin:stdout -t cdn.example.com -d localhost:22" user@remote
```

SSH config (`~/.ssh/config`):
```
Host remote-server
    HostName remote-server.example.com
    User myuser
    ProxyCommand darkflare-client -l stdin:stdout -t cdn.example.com -d localhost:22
```

With proxy:
```
Host remote-server
    HostName remote-server.example.com
    User myuser
    ProxyCommand darkflare-client -l stdin:stdout \
                                 -t cdn.example.com \
                                 -d localhost:22 \
                                 -p socks5://proxy.local:1080
```

### Proxy Chain

When using both stdin/stdout mode and an outbound proxy:

```
[SSH Client]         [Optional Proxy]        [Cloudflare]         [Target]
    |                     |                      |                   |
    |   stdin/stdout      |       HTTPS          |      TCP          |
    | =================>  |  =================>  | =================>|
    |  darkflare-client   |    CDN Traffic       |  darkflare-server |
```

## Use Cases

- SSH, RDP, or any TCP service through restrictive firewalls or state-controlled networks
- Tunneling TCP-based VPN protocols (OpenVPN over TCP, PPP)
- Application launching via darkflare-server (`-a` flag) for sshd, pppd, etc.
- Accessing blocked services through CDN infrastructure

### OpenVPN / NordVPN

1. Download the OpenVPN client.
2. Obtain the `.ovpn` TCP configuration file from your provider (e.g., NordVPN Manual Setup).
3. Edit the `.ovpn` file: change the remote IP and port to your darkflare client's local address and port.
4. Configure darkflare-server to forward to the VPN endpoint.
5. Import the `.ovpn` file into OpenVPN.

Example using the CLI client:
```bash
openvpn --config 127.0.0.1.tcp2222.ovpn --script-security 2
```

Note: OpenVPN modifies the default gateway by default. For testing, add `pull-filter ignore "redirect-gateway"` to the `.ovpn` file to prevent route changes. See the `examples/` directory for sample configurations and routing scripts (`OpenVPN-up.sh`, `OpenVPN-down.sh`).

## Cloudflare Configuration

1. Add your proxy hostname to a Cloudflare account.
2. Configure origin rules to route traffic for that hostname to your darkflare-server's address and port.
3. Under SSL/TLS, set encryption mode to **Full**.

### Origin Certificates

For HTTPS mode:

1. In the Cloudflare dashboard, go to SSL/TLS > Origin Server.
2. Create or download an origin certificate and private key.
3. Pass both files to darkflare-server with `-c` and `-k`.

Keep the private key secure. Origin certificates are only valid for traffic between Cloudflare and your server.

### CA Root Certificates

If direct connections (without CDN) fail TLS verification, ensure system CA certificates are installed:

```bash
# Debian/Ubuntu
sudo apt install ca-certificates

# RHEL/CentOS/Fedora
sudo yum install ca-certificates
```

## Windows Fileless Execution

DLL variants are available in `bin/dll/` for memory-only execution:

- `darkflare-client-windows-386.dll` (32-bit)
- `darkflare-client-windows-amd64.dll` (64-bit)

These can be loaded from C# or C++ applications without writing to disk.

This feature is intended for authorized testing scenarios only.

## Command Reference

### Client

```
darkflare-client [options]

Options:
  -l        Local listen address
            Format: <port> or stdin:stdout
            Examples: 2222, stdin:stdout

  -t        Target URL of the darkflare-server
            Format: [http(s)://]hostname[:port]
            Default scheme: https, default ports: 443 (https), 80 (http)

  -d        Destination address for the target service
            Format: hostname:port

  -p        Outbound proxy URL
            Format: scheme://[user:pass@]host:port
            Supported schemes: http, https, socks5, socks5h

  -debug    Enable debug logging

  -insecure Disable TLS certificate verification
```

### Server

```
darkflare-server [options]

Options:
  -o             Listen address
                 Format: proto://[host]:port
                 Default: http://0.0.0.0:8080

  -allow-direct  Allow connections without Cloudflare headers
                 Default: false

  -c             TLS certificate file path

  -k             TLS private key file path

  -a             Application command to execute per request

  -auth          Basic authentication credentials (user:pass)

  -debug         Enable debug logging

  -s             Silent mode (suppress non-error output)

  -redirect      Redirect URL for unauthorized requests
                 Default: https://github.com/doxx/darkflare

  -override-dest Override client-specified destination (host:port)
```

## Security Considerations

- DarkFlare provides traffic obfuscation, not encryption. Always use end-to-end encryption (SSH, TLS, etc.) for the tunneled protocol.
- The `-allow-direct` flag bypasses Cloudflare header validation. Do not use in production.
- The `-insecure` flag disables TLS certificate verification. Use only for testing.
- Monitor Cloudflare analytics and server logs for unexpected traffic patterns.
- Keep both client and server components updated.

## Disclaimer

This tool is provided for educational and authorized use only. Users are responsible for compliance with applicable laws and organizational policies.

## Contributing

Bug reports and pull requests are welcome.

## License

MIT License

---
