# DarkFlare Setup Guide

This guide walks through every step required to deploy DarkFlare, from initial setup through production operation. It covers the server, client, Cloudflare configuration, SSH integration, UDP tunneling, VPN tunneling, authentication, and troubleshooting.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Installation](#2-installation)
3. [Architecture](#3-architecture)
4. [Server Setup](#4-server-setup)
5. [Cloudflare Configuration](#5-cloudflare-configuration)
6. [Client Setup](#6-client-setup)
7. [Testing the Tunnel](#7-testing-the-tunnel)
8. [SSH Integration](#8-ssh-integration)
9. [Proxy Support](#9-proxy-support)
10. [Authentication](#10-authentication)
11. [Server-Side Destination Override](#11-server-side-destination-override)
12. [Application Mode](#12-application-mode)
13. [OpenVPN / NordVPN Tunneling](#13-openvpn--nordvpn-tunneling)
14. [Windows Fileless Execution](#14-windows-fileless-execution)
15. [Debug Mode](#15-debug-mode)
16. [Command Reference](#16-command-reference)
17. [Troubleshooting](#17-troubleshooting)

---

## 1. Prerequisites

**Server machine** (where darkflare-server runs):
- A public-facing server with a static IP address or a hostname that resolves to it.
- An open port that Cloudflare can reach (typically 8080 for HTTP or 443 for HTTPS).
- The target TCP service must be reachable from this machine. For example, if you are tunneling SSH, an SSH daemon must be running on this machine or on a host accessible from it.

**Client machine** (where darkflare-client runs):
- Any machine that can make outbound HTTPS connections (port 443). This is the machine where you want to initiate tunneled connections from.

**Cloudflare account**:
- A free Cloudflare account is sufficient.
- A domain name added to Cloudflare with DNS managed by Cloudflare.

**Build from source (optional)**:
- Go 1.20 or later.

---

## 2. Installation

### Option A: Prebuilt Binaries

Download the appropriate binaries from the GitHub Releases page. Available binaries:

| Binary | Platform |
|--------|----------|
| `darkflare-client-darwin-arm64` | macOS Apple Silicon |
| `darkflare-client-darwin-amd64` | macOS Intel |
| `darkflare-client-linux-amd64` | Linux x64 |
| `darkflare-client-linux-arm64` | Linux ARM64 |
| `darkflare-client-windows-amd64.exe` | Windows x64 |
| `darkflare-server-darwin-arm64` | macOS Apple Silicon |
| `darkflare-server-darwin-amd64` | macOS Intel |
| `darkflare-server-linux-amd64` | Linux x64 |
| `darkflare-server-linux-arm64` | Linux ARM64 |
| `darkflare-server-windows-amd64.exe` | Windows x64 |

After downloading, verify the checksums:

```bash
sha256sum -c checksums.txt
```

Make the binaries executable on Unix systems:

```bash
chmod +x darkflare-client-* darkflare-server-*
```

### Option B: Build from Source

Clone the repository and build both components:

```bash
# Build the client
go build -o darkflare-client ./client/

# Build the server (from the server directory, which has its own go.mod)
cd server
go build -o darkflare-server .
```

### CA Root Certificates

On some Linux systems, TLS connections may fail if system CA certificates are not installed. Install them if needed:

```bash
# Debian / Ubuntu
sudo apt install ca-certificates

# RHEL / CentOS / Fedora
sudo yum install ca-certificates
```

---

## 3. Architecture

DarkFlare has two components that work together:

```
[Client Machine]                [Cloudflare]                [Server Machine]

  TCP/UDP Application                                         Target Service
  (e.g., ssh, DNS)                                            (e.g., sshd, DNS)
       |                                                           ^
       v                                                           |
  darkflare-client              Cloudflare                    darkflare-server
  Listens on a local     --->   Reverse Proxy    --->         Listens on a port
  TCP or UDP port               (your domain)                 (e.g., 8080)
  (e.g., 2222) or                                             Forwards TCP/UDP
  stdin/stdout                                                to the target service
```

**How data flows (TCP mode, default):**

1. A TCP application (such as an SSH client) connects to darkflare-client on a local port.
2. darkflare-client reads the TCP data, wraps it in an HTTP POST request with browser-like headers, and sends it to your Cloudflare-proxied domain.
3. Cloudflare forwards the request to your origin server where darkflare-server is running.
4. darkflare-server extracts the TCP data from the HTTP request and writes it to a TCP connection to the target service.
5. Response data flows back the same way: darkflare-server buffers data from the target service, and darkflare-client polls for it with HTTP GET requests.

**How data flows (UDP mode, `-proto udp`):**

1. A UDP application sends a datagram to darkflare-client on a local UDP port.
2. darkflare-client sends each datagram as an individual HTTP POST to the server.
3. darkflare-server extracts the datagram and sends it via UDP to the target service.
4. darkflare-server buffers incoming UDP response datagrams. When the client polls with GET, the server returns all buffered datagrams using 4-byte big-endian length-prefix framing, hex-encoded.
5. darkflare-client decodes the response and sends each datagram back to the original UDP client.

The client uses HTTP POST to send data upstream and HTTP GET to poll for downstream data. Each request includes randomized URL paths (e.g., `/a3kf9x.jpg`, `/m2nq.php`) and standard browser headers (User-Agent, Accept, Sec-Ch-Ua, etc.) so the traffic looks like normal web browsing to any network observer.

---

## 4. Server Setup

The server runs on the machine where the target TCP service is accessible. It listens for HTTP or HTTPS connections from Cloudflare and forwards the decapsulated TCP data to the target.

### HTTP Mode (Testing / Development)

For local testing or when TLS termination is handled by Cloudflare:

```bash
./darkflare-server -o http://0.0.0.0:8080
```

This starts the server listening on all interfaces on port 8080. By default, the server rejects connections that do not include Cloudflare headers (`Cf-Connecting-Ip`). For local testing without Cloudflare, add `-allow-direct`:

```bash
./darkflare-server -o http://0.0.0.0:8080 -allow-direct
```

The `-allow-direct` flag disables the Cloudflare header check. Do not use this in production -- it allows anyone who can reach the server to use it directly.

### HTTPS Mode (Production)

For HTTPS, you must provide a TLS certificate and private key. These should be Cloudflare origin certificates (see [Section 5](#5-cloudflare-configuration)):

```bash
./darkflare-server -o https://0.0.0.0:443 -c /path/to/cert.pem -k /path/to/key.pem
```

Both `-c` (certificate) and `-k` (key) are required when using HTTPS. The server will terminate TLS itself using these certificates.

### Binding to a Specific Interface

You can bind to a specific IP instead of all interfaces:

```bash
./darkflare-server -o http://127.0.0.1:8080 -allow-direct
```

The server validates that the bind address is a local IP on the machine.

### Silent Mode

To suppress all non-error log output:

```bash
./darkflare-server -o http://0.0.0.0:8080 -s
```

### Custom Redirect

When the server receives a request that does not contain the expected DarkFlare headers (e.g., someone browsing to the server directly), it returns a 302 redirect. By default, this redirects to the DarkFlare GitHub page. You can customize this to make the server appear to be a normal website:

```bash
./darkflare-server -o http://0.0.0.0:8080 -redirect "https://www.example.com"
```

### Running as a systemd Service

To run darkflare-server persistently on Linux:

Create `/etc/systemd/system/darkflare-server.service`:

```ini
[Unit]
Description=DarkFlare Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/darkflare-server -o http://0.0.0.0:8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable darkflare-server
sudo systemctl start darkflare-server
sudo systemctl status darkflare-server
```

---

## 5. Cloudflare Configuration

Cloudflare acts as the reverse proxy between the client and server. The client sends HTTPS requests to your Cloudflare-proxied domain, and Cloudflare forwards them to your origin server.

### Step 1: Add Your Domain

If you haven't already, add your domain to Cloudflare and point the nameservers to Cloudflare's nameservers. This is done in the Cloudflare dashboard under your domain's DNS settings.

### Step 2: Create a DNS Record

Create an A record (or AAAA for IPv6) pointing a hostname to your server's public IP address. For example:

| Type | Name | Content | Proxy Status |
|------|------|---------|-------------|
| A | tunnel | 203.0.113.50 | Proxied (orange cloud) |

The proxy status must be set to **Proxied** (orange cloud icon). If it is set to DNS Only (gray cloud), traffic will not pass through Cloudflare and the tunnel will not work.

This creates the hostname `tunnel.yourdomain.com` which routes through Cloudflare to your server.

### Step 3: Configure Origin Rules

If your darkflare-server listens on a non-standard port (anything other than 80 or 443), you need an origin rule to tell Cloudflare which port to forward to on your server.

1. In the Cloudflare dashboard, go to **Rules > Origin Rules**.
2. Create a new rule:
   - **If**: Hostname equals `tunnel.yourdomain.com`
   - **Then**: Override destination port to `8080` (or whatever port your server uses)
3. Save and deploy the rule.

If your server listens on port 443 with HTTPS, you can skip this step -- Cloudflare will forward to port 443 by default.

### Step 4: Set SSL/TLS Encryption Mode

Go to **SSL/TLS > Overview** in the Cloudflare dashboard and set the encryption mode to **Full** (or **Full (strict)** if you are using a valid origin certificate).

- **Full**: Cloudflare connects to your origin over HTTPS but does not validate the certificate. Use this with self-signed certificates.
- **Full (strict)**: Cloudflare connects over HTTPS and validates the certificate against Cloudflare's CA. Use this with Cloudflare origin certificates.

Do not use **Flexible** -- this causes Cloudflare to connect to your origin over plain HTTP, which may cause redirect loops or expose traffic between Cloudflare and your server.

### Step 5: Generate Origin Certificates (for HTTPS)

If your darkflare-server runs in HTTPS mode, generate origin certificates:

1. In the Cloudflare dashboard, go to **SSL/TLS > Origin Server**.
2. Click **Create Certificate**.
3. Select the hostnames to cover (e.g., `tunnel.yourdomain.com` or `*.yourdomain.com`).
4. Choose a validity period (Cloudflare offers up to 15 years for origin certs).
5. Click **Create**.
6. Copy the **Origin Certificate** and save it to a file on your server (e.g., `/etc/darkflare/cert.pem`).
7. Copy the **Private Key** and save it to a file on your server (e.g., `/etc/darkflare/key.pem`).

Set restrictive permissions on the private key:

```bash
chmod 600 /etc/darkflare/key.pem
```

Then start the server with these files:

```bash
./darkflare-server -o https://0.0.0.0:443 -c /etc/darkflare/cert.pem -k /etc/darkflare/key.pem
```

---

## 6. Client Setup

The client runs on the machine where you want to initiate tunneled connections. It listens for local TCP or UDP connections and forwards them through Cloudflare to the server.

### Basic TCP Usage

To tunnel SSH (port 22) through Cloudflare:

```bash
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22
```

This command does the following:
- `-l 2222` -- Listen on local TCP port 2222 for incoming connections.
- `-t https://tunnel.yourdomain.com` -- Send the tunneled traffic to this Cloudflare-proxied URL. This is the hostname you configured in Cloudflare DNS (Step 2 above).
- `-d localhost:22` -- Tell the server to forward the decapsulated TCP data to `localhost:22` (the SSH daemon on the server machine).

Once running, you can connect to the tunnel:

```bash
ssh user@localhost -p 2222
```

This SSH connection goes to local port 2222, through DarkFlare, through Cloudflare, to the server, and finally to the SSH daemon -- all wrapped in HTTPS traffic that looks like normal web browsing.

### Understanding the `-t` Flag

The `-t` flag specifies where the client sends its HTTP requests. This should be the Cloudflare-proxied hostname of your server:

```
-t https://tunnel.yourdomain.com       # HTTPS on port 443 (default)
-t https://tunnel.yourdomain.com:443   # Same as above, explicit port
-t http://tunnel.yourdomain.com        # HTTP on port 80
-t http://localhost:8080               # Direct connection (testing only)
```

When connecting through Cloudflare, always use HTTPS. The HTTP option exists for local testing with `-allow-direct` on the server.

### Understanding the `-d` Flag

The `-d` flag specifies the final destination for the TCP traffic. This address is resolved by the server, not the client. The client base64-encodes this value and sends it in the `X-Requested-With` header.

```
-d localhost:22          # SSH on the server machine itself
-d 192.168.1.50:22       # SSH on a different machine in the server's network
-d internal.host:3389    # RDP on an internal host
```

The server validates this destination (DNS resolution, port range 1-65535) before connecting.

### UDP Mode

To tunnel UDP traffic (DNS, WireGuard, game servers, etc.), use `-proto udp`:

```bash
# Tunnel DNS queries to 8.8.8.8
./darkflare-client -l 5353 -t https://tunnel.yourdomain.com -d 8.8.8.8:53 -proto udp

# Query through the tunnel
dig @localhost -p 5353 example.com
```

In UDP mode:
- The client listens on a local UDP socket instead of TCP.
- Each incoming UDP datagram is sent as an individual HTTP POST.
- The client polls with GET requests to receive response datagrams from the server.
- The server automatically detects the protocol from the client's headers. No server-side flag is needed.
- UDP sessions time out after 2 minutes of inactivity (vs 5 minutes for TCP).
- stdin/stdout mode is not supported with UDP.

### Specifying the Scheme and Port

If you omit the scheme, the client defaults to HTTPS:

```bash
# These are equivalent:
./darkflare-client -l 2222 -t tunnel.yourdomain.com -d localhost:22
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com:443 -d localhost:22
```

For HTTP (testing only):

```bash
./darkflare-client -l 2222 -t http://localhost:8080 -d localhost:22
```

---

## 7. Testing the Tunnel

The fastest way to verify the tunnel works is a local end-to-end test that does not require Cloudflare. This is useful for verifying the software works before configuring DNS and origin rules.

### Step 1: Start a Target Service

You need a TCP service to tunnel to. If SSH is running on the server machine:

```bash
# Verify SSH is listening
ss -tlnp | grep :22
```

If SSH is not available, start a simple TCP echo server for testing:

```bash
# Using ncat (from nmap)
ncat -l -k -p 9999 -e /bin/cat

# Or using Python
python3 -c "
import socket, threading
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 9999))
s.listen(1)
while True:
    c, _ = s.accept()
    data = c.recv(4096)
    c.sendall(data)
    c.close()
"
```

### Step 2: Start the Server

```bash
./darkflare-server -o http://0.0.0.0:8080 -allow-direct -debug
```

The `-debug` flag enables verbose logging so you can see connections being established and data flowing. The `-allow-direct` flag is required for local testing since the requests will not contain Cloudflare headers.

You should see:

```
DarkFlare server listening on http://0.0.0.0:8080
DarkFlare server running on http://0.0.0.0:8080
Warning: Direct connections allowed (no Cloudflare required)
```

### Step 3: Start the Client

In a separate terminal:

```bash
./darkflare-client -l 2222 -t http://localhost:8080 -d localhost:22 -debug
```

If you used a test echo server on port 9999 instead of SSH, replace `-d localhost:22` with `-d localhost:9999`.

You should see:

```
Debug mode enabled
DarkFlare client listening on port 2222
Connecting via http://localhost:8080
```

### Step 4: Connect

For SSH:

```bash
ssh user@localhost -p 2222
```

For a generic TCP test:

```bash
echo "hello" | nc -w 3 localhost 2222
```

### Step 5: Verify in Debug Output

In the server's debug output, you should see lines like:

```
Connection: 127.0.0.1 [a1b2c3d4] -> localhost:22
POST: Writing 5 bytes to connection for session a1b2c3d4
Response: Sending 48 bytes (encoded: 96 bytes) for session a1b2c3d4
```

This confirms data is flowing in both directions through the tunnel.

### Step 6: Test UDP Mode

To verify UDP tunneling works locally:

```bash
# Terminal 1: Start the server
./darkflare-server -o http://0.0.0.0:8080 -allow-direct

# Terminal 2: Start the client in UDP mode, tunneling to a public DNS server
./darkflare-client -l 5353 -t http://localhost:8080 -d 8.8.8.8:53 -proto udp

# Terminal 3: Send a DNS query through the tunnel
dig @localhost -p 5353 example.com
```

You should see a DNS response with IP addresses for `example.com`.

### Step 7: Test Through Cloudflare

Once local testing works, switch to testing through Cloudflare:

1. Remove `-allow-direct` from the server.
2. Change the client's `-t` to point to your Cloudflare-proxied hostname:

```bash
./darkflare-server -o https://0.0.0.0:443 -c cert.pem -k key.pem -debug
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22 -debug
```

---

## 8. SSH Integration

### Port Forwarding Mode

The simplest approach: the client listens on a local TCP port and you point SSH at it.

```bash
# Terminal 1: Start the client
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22

# Terminal 2: Connect via SSH
ssh user@localhost -p 2222
```

### stdin/stdout Mode (ProxyCommand)

The client can operate without binding to a local port by using stdin/stdout mode. Instead of listening on a TCP port, the client reads from stdin and writes to stdout. SSH's `ProxyCommand` directive feeds data directly to/from the client process.

This mode is useful when:
- You cannot bind to local ports (no root, firewall restrictions).
- You want a single command to both start the tunnel and connect.
- You want to configure SSH tunneling permanently in your SSH config.

**Command line usage:**

```bash
ssh -o ProxyCommand="./darkflare-client -l stdin:stdout -t https://tunnel.yourdomain.com -d localhost:22" user@server
```

SSH spawns darkflare-client as a subprocess. SSH writes to the subprocess's stdin and reads from its stdout. The client tunnels this data through Cloudflare to the server, which forwards it to the SSH daemon.

**Permanent SSH config:**

Add the following to `~/.ssh/config`:

```
Host myserver
    HostName server.example.com
    User myuser
    ProxyCommand /path/to/darkflare-client -l stdin:stdout -t https://tunnel.yourdomain.com -d localhost:22
```

Then connect with:

```bash
ssh myserver
```

SSH reads the config, spawns the ProxyCommand, and establishes the connection through the tunnel.

**With a proxy (see [Section 9](#9-proxy-support)):**

```
Host myserver
    HostName server.example.com
    User myuser
    ProxyCommand /path/to/darkflare-client -l stdin:stdout \
        -t https://tunnel.yourdomain.com \
        -d localhost:22 \
        -p socks5://corporate-proxy.internal:1080
```

**How stdin/stdout mode differs from port forwarding mode:**

| | Port Forwarding (`-l 2222`) | stdin/stdout (`-l stdin:stdout`) |
|---|---|---|
| Binds a local port | Yes | No |
| Requires root/admin for low ports | Yes (below 1024) | No |
| Supports multiple connections | Yes (one per accept) | No (single session) |
| Works with ProxyCommand | No | Yes |
| Works with non-SSH clients | Yes | Only if the client supports stdin/stdout piping |

---

## 9. Proxy Support

The client can route its outbound HTTPS requests through a proxy. This is useful in corporate environments where direct internet access is blocked and all traffic must go through a proxy server.

### Supported Proxy Types

| Scheme | Description |
|--------|-------------|
| `http://` | HTTP proxy (CONNECT method) |
| `https://` | HTTPS proxy (CONNECT method with TLS to proxy) |
| `socks5://` | SOCKS5 proxy (TCP-level proxying) |
| `socks5h://` | SOCKS5 proxy with remote DNS resolution |

### Usage

```bash
# HTTP proxy
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22 \
    -p http://proxy.corporate.com:3128

# HTTP proxy with authentication
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22 \
    -p http://user:password@proxy.corporate.com:3128

# SOCKS5 proxy
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22 \
    -p socks5://proxy.corporate.com:1080

# SOCKS5 proxy with authentication
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22 \
    -p socks5://user:password@proxy.corporate.com:1080
```

### Traffic Flow with a Proxy

```
[TCP App] --> [darkflare-client] --> [Corporate Proxy] --> [Cloudflare] --> [darkflare-server] --> [Target]
```

The proxy sees HTTPS requests going to your Cloudflare domain. The proxy cannot inspect the content because it is encrypted with TLS. To the proxy, it looks like the user is browsing a normal website.

### SOCKS5 vs SOCKS5H

- `socks5://` -- The client resolves the destination hostname locally, then tells the SOCKS5 proxy to connect to the resolved IP.
- `socks5h://` -- The client tells the SOCKS5 proxy to resolve the hostname. Use this when local DNS is restricted or when you want to hide the destination hostname from local DNS resolvers.

---

## 10. Authentication

DarkFlare supports HTTP Basic Authentication to restrict access to the tunnel. When authentication is enabled, clients that do not provide valid credentials receive a 302 redirect instead of tunnel access.

### Server-Side Configuration

```bash
./darkflare-server -o http://0.0.0.0:8080 -auth admin:secretpassword
```

The `-auth` flag takes credentials in `user:password` format. When set, every request must include a valid `Authorization: Basic ...` header.

Unauthenticated requests receive a 302 redirect to the URL specified by `-redirect` (defaults to the DarkFlare GitHub page). This makes the server appear to be a normal website to unauthorized visitors.

### Client-Side Configuration

Include the credentials in the `-t` URL:

```bash
./darkflare-client -l 2222 -t https://admin:secretpassword@tunnel.yourdomain.com -d localhost:22
```

The client extracts the username and password from the URL and includes them as a Basic Auth header on every request.

### Combined with Custom Redirect

For a more convincing cover:

```bash
./darkflare-server -o http://0.0.0.0:8080 -auth admin:secretpassword -redirect "https://www.example.com"
```

Anyone who visits your server's hostname in a browser (without credentials) gets redirected to `www.example.com`, making the server appear to be a simple redirect.

---

## 11. Server-Side Destination Override

By default, the client specifies the destination address with `-d`, and the server connects to whatever destination the client requests. The `-override-dest` flag on the server forces all connections to a fixed destination, ignoring the client's `-d` value.

### Usage

```bash
./darkflare-server -o http://0.0.0.0:8080 -override-dest localhost:22
```

With this configuration, every client connection is forwarded to `localhost:22` regardless of what the client specifies with `-d`. This is useful for:

- Restricting the server to forward only to a specific service.
- Preventing clients from using the server as an open proxy to arbitrary destinations.
- Simplifying client configuration (the `-d` value is ignored but still required by the client binary).

---

## 12. Application Mode

The server can launch an application process for each incoming request instead of forwarding TCP traffic. This is enabled with the `-a` flag.

### Usage

```bash
./darkflare-server -o http://0.0.0.0:8080 -a "/usr/sbin/sshd -i"
```

In this mode, the server does not forward TCP to a destination. Instead, it spawns the specified command for each request and connects the HTTP request/response to the application's stdin/stdout.

### Considerations

- The application must accept connections on stdin/stdout (like `sshd -i`).
- Running `sshd` this way requires proper host key configuration. Without it, SSH clients will reject the connection due to host key verification failure.
- Some applications (like `pppd`) require root privileges. If the server is not running as root, these will fail.

---

## 13. OpenVPN / NordVPN Tunneling

DarkFlare can tunnel OpenVPN over TCP through Cloudflare. This allows you to use a commercial VPN service (such as NordVPN) even in environments where VPN protocols are blocked.

### Step 1: Obtain a TCP-Based OpenVPN Configuration

Download an `.ovpn` configuration file that uses TCP (not UDP). For NordVPN:

1. Log into your NordVPN account.
2. Go to Manual Setup.
3. Download a TCP `.ovpn` configuration file.
4. Note your NordVPN service credentials (username and password) -- these are different from your NordVPN account login.

### Step 2: Set Up the DarkFlare Tunnel

Start the darkflare-server on a machine that can reach the NordVPN server:

```bash
./darkflare-server -o http://0.0.0.0:8080
```

Start the darkflare-client on your local machine:

```bash
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d us5055.nordvpn.com:443
```

Replace `us5055.nordvpn.com:443` with the hostname and port from your `.ovpn` file's `remote` line.

### Step 3: Modify the .ovpn Configuration

Edit the `.ovpn` file. Change the `remote` line to point to the local darkflare-client port:

```
# Original (comment this out):
# remote us5055.nordvpn.com 443

# New (point to DarkFlare):
remote 127.0.0.1 2222
```

### Step 4: Connect

```bash
openvpn --config your-config.ovpn --script-security 2
```

The `--script-security 2` flag allows OpenVPN to run the up/down scripts that handle routing.

### Routing

OpenVPN modifies the default gateway by default, which can break the DarkFlare tunnel itself (since the tunnel traffic would then try to route through the VPN). There are two approaches to handle this:

**Option A: Disable gateway redirect (testing)**

Add this line to your `.ovpn` file:

```
pull-filter ignore "redirect-gateway"
```

This prevents OpenVPN from changing your default route. Your regular internet traffic will not go through the VPN, but you can verify the tunnel works.

**Option B: Use routing scripts (production)**

The `examples/` directory includes `OpenVPN-up.sh` and `OpenVPN-down.sh` scripts that:

1. Save your current default gateway.
2. Add static routes for all Cloudflare IP ranges through your original gateway (so DarkFlare traffic bypasses the VPN).
3. Set the default route to the VPN tunnel gateway.

On disconnect, the down script reverses these changes.

To use them, add these lines to your `.ovpn` file (update the paths):

```
up /path/to/darkflare/examples/OpenVPN-up.sh
down /path/to/darkflare/examples/OpenVPN-down.sh
```

The scripts require `sudo` access for route manipulation and `curl` to fetch the current Cloudflare IP ranges.

---

## 14. Windows Fileless Execution

DarkFlare provides DLL and PowerShell-based execution methods for Windows environments where writing files to disk is undesirable.

### DLL Variants

Available in `bin/dll/`:

- `darkflare-client-windows-386.dll` (32-bit)
- `darkflare-client-windows-amd64.dll` (64-bit)

These can be loaded into a C# or C++ application's memory space and executed without ever writing the DarkFlare binary to disk.

### PowerShell Memory Execution

The `examples/memory-exec.ps1` script downloads the client binary into memory and executes it without saving to disk:

```powershell
.\memory-exec.ps1 -t tunnel.yourdomain.com -d localhost:22
```

Parameters:
- `-t` (required): Target URL of the darkflare-server.
- `-d` (required): Destination address.
- `-l` (optional): Listen mode. Defaults to `stdin:stdout`.
- `-p` (optional): Proxy URL.

This feature is intended for authorized testing and security assessment scenarios only.

---

## 15. Debug Mode

Both the client and server support a `-debug` flag that enables verbose logging.

### Client Debug Output

```bash
./darkflare-client -l 2222 -t https://tunnel.yourdomain.com -d localhost:22 -debug
```

Debug output includes:
- Full request headers for every HTTP request sent.
- The session ID assigned to each connection.
- Data sizes for each send and receive operation.
- Response status codes from the server.
- Poll results (data received or empty).
- Error details for failed connections.

### Server Debug Output

```bash
./darkflare-server -o http://0.0.0.0:8080 -debug
```

Debug output includes:
- All request headers received.
- DNS resolution results for destination hosts.
- Bytes written and read for each session.
- TLS handshake details (HTTPS mode): client address, server name, supported TLS versions, cipher suites, curves, ALPN protocols.
- Connection state changes.
- Authentication attempts (if `-auth` is set).

### TLS Verification Bypass

For testing with self-signed certificates or direct server connections without valid TLS:

```bash
./darkflare-client -l 2222 -t https://direct.example.com -d localhost:22 -insecure
```

The `-insecure` flag disables TLS certificate verification on the client. Do not use this in production.

---

## 16. Command Reference

### darkflare-client

```
Usage: darkflare-client [options]

Required:
  -l        Local listen address.
            <port>         Listen on a TCP port (e.g., 2222).
            stdin:stdout   Read/write from stdin/stdout (for SSH ProxyCommand).

  -t        Target URL of the darkflare-server.
            Format: [http(s)://][user:pass@]hostname[:port]
            Default scheme: https.
            Default port: 443 (https), 80 (http).
            Credentials in the URL are used for Basic Authentication.

  -d        Destination address for the target service.
            Format: hostname:port
            This value is sent to the server, which connects to it.

Optional:
  -proto    Protocol for local listener and destination.
            tcp (default) or udp.
            In UDP mode, the client listens on a local UDP socket
            and each datagram is tunneled individually.

  -p        Outbound proxy URL.
            Format: scheme://[user:pass@]host:port
            Supported: http, https, socks5, socks5h.

  -debug    Enable verbose debug logging to stderr.

  -insecure Disable TLS certificate verification.
```

### darkflare-server

```
Usage: darkflare-server [options]

Required:
  -o             Listen address.
                 Format: proto://host:port
                 Default: http://0.0.0.0:8080
                 The host must be a local IP address.

Optional:
  -c             Path to TLS certificate file (required for HTTPS).

  -k             Path to TLS private key file (required for HTTPS).

  -allow-direct  Allow connections that do not include Cloudflare headers.
                 Default: false.
                 Required for local testing without Cloudflare.

  -auth          Basic authentication credentials.
                 Format: user:pass
                 Unauthenticated requests receive a 302 redirect.

  -a             Application command to execute per request.
                 The server spawns this command instead of forwarding TCP.
                 Example: "/usr/sbin/sshd -i"

  -override-dest Override the client-specified destination.
                 Format: host:port
                 All traffic is forwarded to this address regardless
                 of the client's -d value.

  -redirect      Custom redirect URL for unauthorized or unrecognized requests.
                 Default: https://github.com/doxx/darkflare

  -debug         Enable verbose debug logging to stderr.

  -s             Silent mode. Suppress all non-error log output.
```

---

## 17. Troubleshooting

### Connection Refused on Local Port

**Symptom:** `ssh user@localhost -p 2222` returns "Connection refused."

**Cause:** The darkflare-client is not running or failed to bind to port 2222.

**Fix:** Verify the client is running and check for errors in its output. If the port is already in use, choose a different port with `-l`. On Unix, ports below 1024 require root.

### 403 Forbidden

**Symptom:** The client logs show HTTP 403 responses.

**Cause:** The server is rejecting the connection because it does not see Cloudflare headers.

**Fix:** Either route traffic through Cloudflare (use the Cloudflare-proxied hostname in `-t`), or add `-allow-direct` to the server for testing.

### 502 Bad Gateway

**Symptom:** The client logs show HTTP 502 responses.

**Cause:** Cloudflare cannot reach your origin server.

**Fix:**
- Verify darkflare-server is running and listening on the expected port.
- Verify the server's firewall allows inbound connections from Cloudflare's IP ranges on the server port.
- Verify the Cloudflare origin rule is forwarding to the correct port.

### 521 Origin Down

**Symptom:** The client logs show "Error 521" in the response body.

**Cause:** Cloudflare cannot establish a TCP connection to your origin server.

**Fix:** Same as 502. Additionally, verify the server process has not crashed. Check `systemctl status darkflare-server` or look for the process with `ps aux | grep darkflare`.

### 522 Connection Timed Out

**Symptom:** The client logs show "Error 522."

**Cause:** Cloudflare established a TCP connection to your origin but the connection timed out before a response was received.

**Fix:** Check that darkflare-server is responding. The server may be overloaded or the network between Cloudflare and your server may have high latency.

### 524 A Timeout Occurred

**Symptom:** The client logs show "Error 524."

**Cause:** Cloudflare's 100-second timeout for HTTP responses was exceeded.

**Fix:** This typically occurs with idle connections. The client polls every 50ms, so this should not happen under normal operation. If it does, check server health and network stability.

### Authentication Failures

**Symptom:** The server redirects the client instead of establishing the tunnel.

**Cause:** The credentials in the client's `-t` URL do not match the server's `-auth` value.

**Fix:** Verify the credentials match exactly. Check the server's debug output, which logs both the received and expected credentials.

### DNS Resolution Failed

**Symptom:** The server logs "DNS resolution failed" for the destination.

**Cause:** The server cannot resolve the hostname specified in the client's `-d` flag.

**Fix:** Verify the destination hostname is resolvable from the server machine. Try `nslookup <hostname>` or `dig <hostname>` on the server.

### TLS Certificate Errors

**Symptom:** The client fails to connect with TLS-related errors.

**Cause:** The client cannot verify the server's TLS certificate.

**Fix:**
- If connecting through Cloudflare, ensure the client is using the Cloudflare-proxied hostname (not the server's direct IP).
- If connecting directly for testing, use `-insecure` on the client.
- Ensure system CA certificates are installed (see [Section 2](#2-installation)).

### Tunnel Connects but No Data Flows

**Symptom:** The client and server both start without errors, the tunnel appears to connect, but no data passes through.

**Cause:** This can occur if the target service is not running or not accepting connections on the specified port.

**Fix:**
- Verify the target service is running: `ss -tlnp | grep :<port>` on the server.
- Test the target service directly from the server: `nc -z localhost <port>`.
- Enable `-debug` on both client and server to trace where data flow stops.

---
