# VPS Anti-Censorship Setup Guide (Iran GFW 2026)

> **Server**: Ubuntu 22.04 LTS | **IP**: `YOUR_SERVER_IP` | **Domain**: `your-domain.com`
>
> Complete step-by-step reproduction guide — every command from a fresh VPS to a fully working stealth proxy.

---

## Table of Contents

1. [Phase 1: Analysis & Cleanup](#phase-1-analysis--cleanup)
2. [Phase 2: Enable TCP BBR](#phase-2-enable-tcp-bbr)
3. [Phase 3: Install 3X-UI Panel](#phase-3-install-3x-ui-panel)
4. [Phase 4: Configure Panel (Port, Credentials, WebBasePath)](#phase-4-configure-panel)
5. [Phase 5: Create VLESS + Reality + XTLS-Vision Inbound (Port 443)](#phase-5-create-vless--reality--xtls-vision-inbound)
6. [Phase 6: Create VMess + WebSocket + TLS Inbound (Port 10032)](#phase-6-create-vmess--websocket--tls-inbound)
7. [Phase 7: Domain SSL Certificate](#phase-7-domain-ssl-certificate)
8. [Phase 8: Verification](#phase-8-verification)
9. [Architecture Overview](#architecture-overview)
10. [Client Setup Guide](#client-setup-guide)
11. [Maintenance & Troubleshooting](#maintenance--troubleshooting)

---

## Phase 1: Analysis & Cleanup

### 1.1 — Check for existing VPN services

```bash
systemctl list-units --type=service --state=active | grep -iE 'openvpn|wireguard|shadowsocks|v2ray|xray|trojan|hysteria|x-ui|3x-ui|nginx|apache|caddy'
```

```bash
ps aux | grep -iE 'openvpn|wireguard|shadowsocks|v2ray|xray|trojan|hysteria|x-ui' | grep -v grep
```

### 1.2 — Check who is using ports 80, 443, 2053

```bash
ss -tlnp | grep -E ':80\b|:443\b|:2053\b'
```

### 1.3 — Check all listening ports

```bash
ss -tlnp
```

### 1.4 — Check installed packages

```bash
dpkg -l | grep -iE 'openvpn|wireguard|shadowsocks|v2ray|xray|trojan|hysteria|x-ui|nginx'
```

### 1.5 — Stop and purge any existing Xray / VPN services

If an old Xray or VPN service is found, remove it completely:

```bash
systemctl stop xray
systemctl disable xray
rm -f /usr/local/bin/xray
rm -rf /usr/local/etc/xray/
rm -f /etc/systemd/system/xray.service
rm -rf /usr/local/share/xray/
systemctl daemon-reload
```

### 1.6 — Verify ports are now free

```bash
ss -tlnp | grep -E ':80|:443|:2053'
```

**Why cleanup matters:** Iran's GFW fingerprints VPS IPs that run multiple detectable protocols. A previously flagged IP running VMess without TLS is a dead giveaway. Starting clean is essential.

---

## Phase 2: Enable TCP BBR

BBR (Bottleneck Bandwidth and Round-trip propagation time) is Google's congestion control algorithm. It significantly improves throughput under the artificial packet loss that Iran's GFW uses for throttling.

### 2.1 — Enable BBR permanently

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

### 2.2 — Verify BBR is active

```bash
sysctl net.ipv4.tcp_congestion_control
# Expected output: net.ipv4.tcp_congestion_control = bbr

sysctl net.core.default_qdisc
# Expected output: net.core.default_qdisc = fq

lsmod | grep bbr
# Expected output: tcp_bbr  20480  1 (or similar)
```

---

## Phase 3: Install 3X-UI Panel

3X-UI (MHSanaei fork) is a web-based management panel for Xray-core. It handles inbound/outbound configuration, user management, traffic monitoring, and certificate management.

### 3.1 — Run the installer

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

The installer will:
- Install dependencies (`socat`, `ca-certificates`, `curl`, `tar`)
- Download the latest 3X-UI release and Xray-core binary
- Set up a systemd service (`x-ui.service`)
- Optionally configure an SSL certificate (it will offer Let's Encrypt for IP)
- Generate random credentials and display them

**Save the credentials shown at the end of installation.**

### 3.2 — Verify installation

```bash
systemctl status x-ui
```

You should see `active (running)` with both `x-ui` and `xray-linux-amd64` processes.

---

## Phase 4: Configure Panel

### 4.1 — View current settings

```bash
/usr/local/x-ui/x-ui setting -show
```

### 4.2 — Set panel port to 2083 (non-standard)

Using a non-standard port avoids basic port scanning that looks for common panel ports.

```bash
/usr/local/x-ui/x-ui setting -port 2083
```

### 4.3 — Set secure username and password

```bash
/usr/local/x-ui/x-ui setting -username YOUR_USERNAME -password YOUR_PASSWORD
```

Example with auto-generated values:

```bash
PANEL_USER="admin_$(openssl rand -hex 3)"
PANEL_PASS="$(openssl rand -base64 16 | tr -d '/+=' | head -c 16)"
/usr/local/x-ui/x-ui setting -username "$PANEL_USER" -password "$PANEL_PASS"
echo "Username: $PANEL_USER"
echo "Password: $PANEL_PASS"
```

### 4.4 — Set a random web base path (hides the panel URL)

```bash
WEB_BASE="/$(openssl rand -hex 8)"
/usr/local/x-ui/x-ui setting -webBasePath "$WEB_BASE"
echo "Web Base Path: $WEB_BASE"
```

### 4.5 — Restart to apply changes

```bash
systemctl restart x-ui
```

### 4.6 — Your panel URL

```
https://YOUR_IP:2083/YOUR_WEB_BASE_PATH/
```

---

## Phase 5: Create VLESS + Reality + XTLS-Vision Inbound

This is the primary stealth protocol. Reality eliminates TLS fingerprinting by borrowing a real certificate from a whitelisted domain (Google). When Iran's GFW probes your server on port 443, it sees a legitimate Google TLS handshake.

### 5.1 — Generate Reality x25519 keypair

```bash
/usr/local/x-ui/bin/xray-linux-amd64 x25519
```

Output:
```
PrivateKey: <PRIVATE_KEY>
Password: <PUBLIC_KEY>       ← This is the public key clients will use
```

**Save both keys.** The private key stays on the server. The public key goes into client configs.

### 5.2 — Generate UUIDs for users

```bash
/usr/local/x-ui/bin/xray-linux-amd64 uuid   # User A
/usr/local/x-ui/bin/xray-linux-amd64 uuid   # User B
/usr/local/x-ui/bin/xray-linux-amd64 uuid   # User C
```

### 5.3 — Generate Short IDs

```bash
openssl rand -hex 8   # Primary short ID (16 chars)
openssl rand -hex 4   # Secondary short ID (8 chars)
```

### 5.4 — Create the inbound via API

First, login to get a session cookie:

```bash
curl -sk -c /tmp/cookies.txt -X POST "https://YOUR_IP:2083/YOUR_BASE_PATH/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=YOUR_USER&password=YOUR_PASS"
```

Then create the VLESS Reality inbound:

```bash
curl -sk -b /tmp/cookies.txt -X POST "https://YOUR_IP:2083/YOUR_BASE_PATH/panel/api/inbounds/add" \
  -H "Content-Type: application/json" \
  -d '{
    "up": 0,
    "down": 0,
    "total": 0,
    "remark": "VLESS-Reality-443",
    "enable": true,
    "expiryTime": 0,
    "listen": "",
    "port": 443,
    "protocol": "vless",
    "settings": "{\"clients\":[{\"id\":\"UUID_A\",\"flow\":\"xtls-rprx-vision\",\"email\":\"User_A\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"ua1\",\"reset\":0},{\"id\":\"UUID_B\",\"flow\":\"xtls-rprx-vision\",\"email\":\"User_B\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"ub2\",\"reset\":0},{\"id\":\"UUID_C\",\"flow\":\"xtls-rprx-vision\",\"email\":\"User_C\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"uc3\",\"reset\":0}],\"decryption\":\"none\",\"fallbacks\":[]}",
    "streamSettings": "{\"network\":\"tcp\",\"security\":\"reality\",\"realitySettings\":{\"show\":false,\"xver\":0,\"dest\":\"www.google.com:443\",\"serverNames\":[\"www.google.com\",\"google.com\"],\"privateKey\":\"YOUR_PRIVATE_KEY\",\"minClient\":\"\",\"maxClient\":\"\",\"maxTimediff\":0,\"shortIds\":[\"YOUR_SHORT_ID_1\",\"YOUR_SHORT_ID_2\",\"\"],\"settings\":{\"publicKey\":\"YOUR_PUBLIC_KEY\",\"fingerprint\":\"chrome\",\"serverName\":\"\",\"spiderX\":\"/\"}},\"tcpSettings\":{\"acceptProxyProtocol\":false,\"header\":{\"type\":\"none\"}}}",
    "sniffing": "{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\",\"fakedns\"],\"metadataOnly\":false,\"routeOnly\":false}"
  }'
```

> **Alternatively**, you can create this inbound through the web panel UI at `https://YOUR_DOMAIN:2083/YOUR_BASE_PATH/` — go to Inbounds → Add Inbound and fill in the same values.

### 5.5 — VLESS link format for clients

```
vless://UUID@SERVER:443?type=tcp&security=reality&pbk=PUBLIC_KEY&fp=chrome&sni=www.google.com&sid=SHORT_ID&spx=%2F&flow=xtls-rprx-vision#REMARK
```

---

## Phase 6: Create VMess + WebSocket + TLS Inbound

This is a secondary protocol matching a known working config pattern. VMess + WebSocket is useful because it can also work behind CDNs like Cloudflare.

### 6.1 — Generate a UUID

```bash
/usr/local/x-ui/bin/xray-linux-amd64 uuid
```

### 6.2 — Create the inbound via API

```bash
curl -sk -b /tmp/cookies.txt -X POST "https://YOUR_IP:2083/YOUR_BASE_PATH/panel/api/inbounds/add" \
  -H "Content-Type: application/json" \
  -d '{
    "up": 0,
    "down": 0,
    "total": 0,
    "remark": "VMess-WS-TLS-10032",
    "enable": true,
    "expiryTime": 0,
    "listen": "",
    "port": 10032,
    "protocol": "vmess",
    "settings": "{\"clients\":[{\"id\":\"YOUR_UUID\",\"security\":\"auto\",\"email\":\"user_name\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"alterId\":0}]}",
    "streamSettings": "{\"network\":\"ws\",\"security\":\"tls\",\"tlsSettings\":{\"serverName\":\"YOUR_DOMAIN\",\"minVersion\":\"1.2\",\"maxVersion\":\"1.3\",\"certificates\":[{\"certificateFile\":\"/root/cert/YOUR_DOMAIN/fullchain.pem\",\"keyFile\":\"/root/cert/YOUR_DOMAIN/privkey.pem\",\"ocspStapling\":3600}],\"alpn\":[\"h2\",\"http/1.1\"],\"settings\":{\"allowInsecure\":false,\"fingerprint\":\"chrome\"}},\"wsSettings\":{\"path\":\"/\",\"headers\":{\"Host\":\"YOUR_DOMAIN\"}}}",
    "sniffing": "{\"enabled\":true,\"destOverride\":[\"http\",\"tls\"],\"metadataOnly\":false,\"routeOnly\":false}"
  }'
```

### 6.3 — VMess link format for clients

VMess links use base64-encoded JSON:

```bash
echo -n '{"v":"2","ps":"REMARK","add":"YOUR_DOMAIN","port":"10032","id":"YOUR_UUID","aid":"0","scy":"auto","net":"ws","type":"none","host":"YOUR_DOMAIN","path":"/","tls":"tls","sni":"YOUR_DOMAIN","alpn":"h2,http/1.1","fp":"chrome"}' | base64 -w 0
```

Prefix the output with `vmess://` to create the final link.

---

## Phase 7: Domain SSL Certificate

### 7.1 — Prerequisites

Point your domain's DNS A record to your VPS IP:

| Type | Name | Content |
|------|------|---------|
| A | @ | YOUR_SERVER_IP |
| CNAME | www | your-domain.com |

Ensure your CAA records allow `letsencrypt.org`.

### 7.2 — Verify DNS resolution

```bash
dig +short your-domain.com A
# Expected: YOUR_SERVER_IP
```

### 7.3 — Issue the certificate

acme.sh is already installed by 3X-UI. Issue a cert using standalone mode (port 80 must be free):

```bash
/root/.acme.sh/acme.sh --issue -d your-domain.com -d www.your-domain.com --standalone --httpport 80 --force
```

### 7.4 — Install the certificate to a stable location

```bash
mkdir -p /root/cert/your-domain.com

/root/.acme.sh/acme.sh --install-cert -d your-domain.com \
  --key-file /root/cert/your-domain.com/privkey.pem \
  --fullchain-file /root/cert/your-domain.com/fullchain.pem \
  --reloadcmd "systemctl restart x-ui"
```

This also sets up automatic renewal — acme.sh will renew the cert and restart x-ui automatically.

### 7.5 — Update the panel to use the domain certificate

```bash
/usr/local/x-ui/x-ui setting -webCert /root/cert/your-domain.com/fullchain.pem -webCertKey /root/cert/your-domain.com/privkey.pem
systemctl restart x-ui
```

### 7.6 — Update the VMess inbound to use TLS with the domain cert

Use the panel API or web UI to update the VMess inbound's `streamSettings` to include TLS with the certificate paths at `/root/cert/your-domain.com/fullchain.pem` and `/root/cert/your-domain.com/privkey.pem`, and set the `Host` header and `serverName` to your domain.

---

## Phase 8: Verification

### 8.1 — Verify all ports are listening

```bash
ss -tlnp | grep -E ':443|:10032|:2083'
```

Expected:
```
LISTEN  *:443    xray-linux-amd64
LISTEN  *:10032  xray-linux-amd64
LISTEN  *:2083   x-ui
```

### 8.2 — Verify Reality shows Google's certificate on port 443

```bash
echo | openssl s_client -connect your-domain.com:443 -servername www.google.com 2>/dev/null | grep 'subject='
# Expected: subject=CN = www.google.com
```

### 8.3 — Verify VMess TLS shows your domain cert on port 10032

```bash
echo | openssl s_client -connect your-domain.com:10032 -servername your-domain.com 2>/dev/null | grep 'subject='
# Expected: subject=CN = your-domain.com
```

### 8.4 — Verify panel cert on port 2083

```bash
echo | openssl s_client -connect your-domain.com:2083 -servername your-domain.com 2>/dev/null | grep 'subject='
# Expected: subject=CN = your-domain.com
```

### 8.5 — Verify BBR

```bash
sysctl net.ipv4.tcp_congestion_control
# Expected: bbr
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        VPS: YOUR_SERVER_IP                         │
│                        Domain: your-domain.com                        │
├─────────────┬───────────────────────┬───────────────────────────┤
│  Port 443   │     Port 10032        │      Port 2083            │
│  VLESS      │     VMess             │      3X-UI Panel          │
│  + Reality  │     + WebSocket       │      (Management)         │
│  + Vision   │     + TLS             │                           │
│             │                       │                           │
│  Looks like │  Looks like HTTPS     │  Admin access only        │
│  google.com │  to your-domain.com         │  Hidden base path         │
│  to DPI     │                       │                           │
├─────────────┴───────────────────────┴───────────────────────────┤
│  TCP BBR enabled  |  acme.sh auto-renewal  |  Xray-core latest │
└─────────────────────────────────────────────────────────────────┘
```

### Why this setup beats Iran's GFW (2026):

| GFW Technique | Our Counter-Measure |
|---|---|
| **DPI (Deep Packet Inspection)** | Reality makes traffic indistinguishable from a real Google TLS session |
| **TLS Fingerprinting** | `xtls-rprx-vision` eliminates TLS-in-TLS patterns; uTLS uses Chrome fingerprint |
| **Active Probing** | When GFW probes port 443, it gets a valid Google certificate back |
| **Throttling / Packet Loss** | TCP BBR maintains throughput under artificial congestion |
| **Port Scanning** | Panel on non-standard port 2083 behind a random base path |
| **Protocol Detection** | VMess+WS+TLS on 10032 looks like standard HTTPS traffic |

---

## Client Setup Guide

### Android — v2rayNG

1. Install **v2rayNG** from [Google Play](https://play.google.com/store/apps/details?id=com.v2ray.ang) or [GitHub Releases](https://github.com/2dust/v2rayNG/releases)
2. Make sure it uses **Xray-core** (default in recent versions)
3. Tap the **+** button → **Import config from clipboard**
4. Paste your VLESS or VMess link
5. Tap the play button to connect
6. No extra settings needed — Reality handles certificates automatically

### iOS — V2Box or Streisand

1. Install **V2Box** from the [App Store](https://apps.apple.com/app/v2box-v2ray-client/id6446814690) or **Streisand** from the [App Store](https://apps.apple.com/app/streisand/id6450534064)
2. Go to **Settings → Add Server → Import from clipboard**
3. Paste your VLESS or VMess link
4. Connect
5. "Allow Insecure" is **not needed** for Reality (it uses Google's real cert)

### Windows — v2rayN

1. Download **v2rayN** from [GitHub Releases](https://github.com/2dust/v2rayN/releases) (get the version with Xray-core included)
2. Extract and run `v2rayN.exe`
3. Click **Servers → Import bulk URL from clipboard**
4. Paste your VLESS or VMess link
5. Right-click the server → **Set as active server**
6. Click the system proxy button (bottom left) to route traffic
7. Make sure Xray-core version is **1.8+** for Reality support

### macOS — V2RayXS or V2Box

1. Install **V2RayXS** from [GitHub](https://github.com/tzmax/V2RayXS/releases) or **V2Box** from the Mac App Store
2. Import config from clipboard
3. Connect

### Tips for All Platforms

- If connection drops, try changing `fingerprint` from `chrome` to `firefox` or `safari`
- If one protocol is blocked, try the other (switch between VLESS Reality on 443 and VMess+WS+TLS on 10032)
- Never share the links over unencrypted messaging — use Signal, or share as a QR code in person
- For QR code generation on the server: `qrencode -t ANSIUTF8 "your_vless_or_vmess_link"`

---

## Maintenance & Troubleshooting

### Panel Management Commands

```bash
x-ui status          # Check service status
x-ui start           # Start the service
x-ui stop            # Stop the service
x-ui restart         # Restart the service
x-ui log             # View logs
x-ui settings        # View current settings
x-ui enable          # Enable autostart on boot
```

### Check Xray Logs for Errors

```bash
x-ui log
# or directly:
journalctl -u x-ui -f --no-pager -n 50
```

### Renew SSL Certificate Manually

```bash
/root/.acme.sh/acme.sh --renew -d your-domain.com --force
```

Normally this happens automatically via cron. Check with:

```bash
crontab -l | grep acme
```

### If Port 443 Gets Blocked

Iran sometimes blocks specific IP+port combos. Options:

1. Change the VLESS Reality port to another common port (8443, 2053, 2087)
2. Switch to the VMess+WS+TLS on port 10032
3. If the IP itself is blocked, you need a new IP or put traffic behind a CDN

### If Connection is Slow (Throttling)

1. Verify BBR is active: `sysctl net.ipv4.tcp_congestion_control`
2. Try changing the Reality `dest` to another high-traffic domain: `www.microsoft.com:443`, `www.apple.com:443`, `dash.cloudflare.com:443`
3. Try different `fingerprint` values: `chrome`, `firefox`, `safari`, `random`

### Check Active Connections

```bash
ss -tn | grep ':443' | wc -l     # Reality connections
ss -tn | grep ':10032' | wc -l   # VMess connections
```

### Firewall Rules (if using ufw)

```bash
ufw allow 22/tcp     # SSH
ufw allow 443/tcp    # VLESS Reality
ufw allow 10032/tcp  # VMess WS TLS
ufw allow 2083/tcp   # Panel
ufw allow 80/tcp     # SSL renewal (can close after renewal)
ufw enable
```

---

## File Locations

| File | Purpose |
|---|---|
| `/usr/local/x-ui/x-ui` | 3X-UI binary |
| `/usr/local/x-ui/bin/xray-linux-amd64` | Xray-core binary |
| `/usr/local/x-ui/bin/config.json` | Active Xray config (auto-generated by panel) |
| `/etc/x-ui/x-ui.db` | Panel database (users, inbounds, settings) |
| `/root/cert/your-domain.com/fullchain.pem` | Domain SSL certificate |
| `/root/cert/your-domain.com/privkey.pem` | Domain SSL private key |
| `/root/.acme.sh/your-domain.com_ecc/` | acme.sh certificate store |
| `/root/vpn/iran_bypass_configs.txt` | All credentials and links |
| `/etc/systemd/system/x-ui.service` | Systemd service file |

