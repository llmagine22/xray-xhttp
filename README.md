[Youtube](https://youtu.be/NziF6Srh-08?si=vJ97qI2N_eZAgl2m)

# 🛡️ Xray (VLESS) + XHTTP + NGINX Stealth Proxy

This repository documents the setup of a stealth proxy that blends into normal web traffic. By using **NGINX** as a reverse proxy, **Xray** (VLESS) as the core, and the **XHTTP** transport protocol, we can disguise VPN traffic as standard HTTPS requests to a web game or API.

To a Deep Packet Inspection (DPI) firewall, this setup looks like:

1. **DNS:** Resolves to a legitimate domain.
2. **TLS:** Handshakes look like a standard Chrome browser visiting a site.
3. **Traffic:** Looks like a user playing a web game or making standard API calls.

## 🏗️ Architecture

1. **Public Interface (Port 443):** NGINX listens for HTTPS traffic.
2. **Traffic Splitting:**
* **Root path (`/`)**: Serves a legitimate service (e.g., a copyparty server).
* **Secret path (`/video/stream-dl`)**: Forwards traffic to the local Xray backend.


3. **Xray Backend (Port 8080):** Accepts decrypted HTTP traffic from NGINX, unwraps the VLESS protocol, and routes it to the internet.

---

## 📋 Prerequisites

* A Linux VPS (Ubuntu/Debian recommended).
* A domain name pointed to your server's IP (via Cloudflare or other DNS).
* Ports **80** and **443** open on your firewall.

---

## 🚀 Server-Side Setup


### 0. Security

1) Change ssh port, ban root
2) :
```bash
useradd -m /bin/bash lupin
passwd lupin
nano /etc/sudoers

```
 add *lupin ALL=(ALL:ALL) ALL*

### 1. Install Dependencies

Update your system and install NGINX and Certbot.
```bash
su lupin

```

```bash
sudo apt update && sudo apt install nginx certbot python3-certbot-nginx -y

```

### 2. Setup the "Camouflage" cpp server

Create cpp user and download cpp fileserver.

```bash
useradd -m /bin/bash cpp
passwd cpp
cd /etc/systemd/system/
wget https://raw.githubusercontent.com/llmagine22/xray-xhttp/refs/heads/main/copyparty.service
mkdir /var/lib/copyparty-jail
sudo systemctl daemon-reload

```

```bash
su cpp
wget https://github.com/9001/copyparty/releases/latest/download/copyparty-sfx.py
wget https://raw.githubusercontent.com/llmagine22/xray-xhttp/refs/heads/main/cpp-conf

```

### 3. Obtain SSL Certificates

Use Certbot to get a free Let's Encrypt certificate. Replace `yourdomain.com` with your actual domain.

```bash
su lupin
cd
certbot certonly --nginx -d cppru.nekos96.xyz

```

### 4. Configure NGINX (Base HTTPS)

Set up NGINX to handle HTTPS and redirect HTTP to HTTPS.

```bash
cd /etc/nginx/sites-available
wget https://raw.githubusercontent.com/llmagine22/xray-xhttp/refs/heads/main/default
cd /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/default default
nginx -t
sudo systemctl enable --now nginx
```

### 5. Install Xray

Install the latest version of Xray using the official script.

```bash
sudo -i
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

```

### 6. Configure Xray

Generate a UUID for authentication:

```bash
xray uuid
# Copy the output, you will need it for the config below
```

```bash
cd /usr/local/etc/xray
rm config.json
wget https://raw.githubusercontent.com/llmagine22/xray-xhttp/refs/heads/main/config.json
```

*Note: Ensure `path` matches what we will configure in NGINX.*

Restart Xray:

```bash
su lupin
systemctl restart xray

```

### 7. Configure NGINX Reverse Proxy

Test and reload NGINX and cpp:

```bash
nginx -t
sudo systemctl reload nginx
sudo systemctl enable --now copyparty

```

---

## 💻 Client Setup

You can use **v2RayN** (Windows), **v2RayNG** (Android), or the raw **Xray CLI**.

### Client Config (config.json)

```json
{
  "inbounds": [
    {
      "port": 1080,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": {
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "cppru.nekos96.xyz",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID_HERE",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "tls",
        "tlsSettings": {
          "serverName": "cppru.nekos96.xyz",
          "fingerprint": "chrome"
        },
        "xhttpSettings": {
          "path": "/video/stream-dl",
          "mode": "packet-up"
        }
      }
    }
  ]
}

```

### v2RayN GUI Settings

If using a GUI, map the settings as follows:

* **Address:** `cppru.nekos96.xyz`
* **Port:** `443`
* **Protocol:** `VLESS`
* **UUID:** (Paste your generated UUID)
* **Flow:** Empty (or packet-up if available)
* **Encryption:** `none`
* **Transport:** `xhttp` or `http` (depending on client version)
* **Path:** `/video/stream-dl`
* **TLS:** `TLS`
* **SNI:** `cppru.nekos96.xyz`
* **Fingerprint (uTLS):** `chrome`

---

## 🔍 Verification

1. **Check NGINX Logs:**
On the server, run:
```bash
tail -f /var/log/nginx/access.log

```


You should see `POST /api/stream` requests appearing as normal HTTP traffic.
2. **Check IP:**
On your client machine, configure your browser to use the SOCKS proxy (`127.0.0.1:1080`) and visit `https://ifconfig.me`. You should see your VPS IP address.

---

## 🧠 Theory: Why this works

* **VLESS:** A lightweight, stateless protocol for tunneling. It handles authentication but leaves encryption to the transport layer.
* **XHTTP:** Wraps VLESS frames into standard HTTP POST (upload) and GET (download) requests. This allows the traffic to pass through strict firewalls and CDNs that only allow valid HTTP.
* **TLS + uTLS:** Encrypts the XHTTP stream. We use `uTLS` with a `chrome` fingerprint to mimic the exact handshake of a Google Chrome browser, preventing DPI from flagging the connection based on TLS ClientHello signatures.
