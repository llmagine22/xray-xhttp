[Youtube](https://youtu.be/NziF6Srh-08?si=vJ97qI2N_eZAgl2m)

# 🛡️ Xray (VLESS) + XHTTP + NGINX Stealth Proxy

This repository documents the setup of a stealth proxy that blends into normal web traffic. By using **NGINX** as a reverse proxy, **Xray** (VLESS) as the core, and the **XHTTP** transport protocol, we can disguise VPN traffic as standard HTTPS requests to a web game or API.

To a Deep Packet Inspection (DPI) firewall, this setup looks like:

1. **DNS:** Resolves to a legitimate domain.
2. **TLS:** Handshakes look like a standard Chrome browser visiting a site.
3. **Traffic:** Looks like a user uses a WebDav folder.

## 🏗️ Architecture

1. **Public Interface (Port 443):** NGINX listens for HTTPS traffic.
2. **Traffic Splitting:**
* **Root path (`/`)**: Serves a legitimate service (e.g., a copyparty server).
* **Secret path (`/video/stream-dl`)**: Forwards traffic to the local Xray backend.


3. **Xray Backend (Port 8080):** Accepts decrypted HTTP traffic from NGINX, unwraps the VLESS protocol, and routes it to the internet.

---

## 📋 Prerequisites

* A Linux VPS (Ubuntu/Debian recommended).
* A domain name pointed to your server's IP (via Cloudflare, Afraid.org or other DNS).
* Ports **80** and **443** open on your firewall.

---

## 🚀 Server-Side Setup

### 0. Security

1) Change ssh port, ban root
2) :
```bash
useradd -m -s /bin/bash lupin
passwd lupin
nano /etc/sudoers

```
and add "*lupin ALL=(ALL:ALL) ALL*" below root user

### 1. Install Dependencies

Update your system and install NGINX and Certbot.
```bash
su lupin

```

```bash
sudo apt update && sudo apt install nginx certbot python3-certbot-nginx -y

```

`If that user can do whatever you want fuck it and do` ```
```bash
sudo -i
```
### 2. Setup the "Camouflage" cpp server

Create cpp user and download cpp fileserver.

```bash
useradd -m -s /bin/bash cpp
passwd cpp
nano /etc/systemd/system/copyparty.service
```

paste into that file:
``` /etc/systemd/system/copyparty.service
[Unit]
Description=copyparty file server
#After=network.target
Requires=network-online.target

[Service]
User=cpp
Type=notify
SyslogIdentifier=copyparty
Environment=PYTHONUNBUFFERED=x
WorkingDirectory=/var/lib/copyparty-jail
ExecReload=/bin/kill -s USR1 $MAINPID
PermissionsStartOnly=true
Environment=XDG_CONFIG_HOME=/var/lib/copyparty/.config
ExecStart=/usr/bin/python3 /home/cpp/copyparty-sfx.py -c /home/cpp/conf
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

then:
```bash
mkdir /var/lib/copyparty-jail
sudo systemctl daemon-reload
```

Download copyparty binary `as user cpp!`:
```bash
su cpp
cd
mkdir /home/cpp/fileserver
wget https://github.com/9001/copyparty/releases/latest/download/copyparty-sfx.py

```

Create copyparty config in /home/cpp/

```conf
nano /home/cpp/conf
```

paste into that file:
```/home/cpp/conf
[global]
  i: 127.0.0.1
  p: 3923
  dedup
  ansi
  xff-hdr: x-forwarded-for
  xff-src: lan
  rproxy: 1
  no-robots
  shr: /shr

[accounts]
  user: passwd

[/]
  /home/cpp/fileserver/
  accs:
    rwmda: user
```
### 3. Obtain SSL Certificates

Use Certbot to get a free Let's Encrypt certificate. Replace `cppru.nekos96.xyz` with your actual domain.

```bash
sudo -i
cd
certbot certonly --nginx -d cppru.nekos96.xyz

```

### 4. Configure NGINX

Set up NGINX to handle HTTPS and split traffic to fileserver and xray backend:

```bash
nano /etc/nginx/sites-available/xray-shadow
```

paste into that file:
```/etc/nginx/sites-available/xray-shadow
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 404;
}

server {
    listen 80;
    listen [::]:80;
    server_name cppru.nekos96.xyz;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;

    ssl_reject_handshake on;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name cppru.nekos96.xyz;
    http2 on;

    ssl_certificate /etc/letsencrypt/live/cppru.nekos96.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cppru.nekos96.xyz/privkey.pem;

    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ecdh_curve X25519:prime256v1:secp384r1;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    server_tokens off;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        proxy_pass https://127.0.0.1:3923;

        client_max_body_size 0;
        proxy_buffering off;
        proxy_request_buffering off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;

        proxy_set_header Origin $scheme://$host;
        proxy_buffers 32 8k;
        proxy_buffer_size 16k;
        proxy_busy_buffers_size 24k;
    }

    location /video/stream-dl {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        proxy_set_header Connection "";

        proxy_read_timeout 10m;
        proxy_send_timeout 10m;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_hide_header X-Powered-By;
        proxy_hide_header Server;

        access_log off;
        error_log off;

        limit_except POST GET {
            deny all;
        }
    }  # ← Закрыли /video/stream-dl
}  # ← Закрыли server

```

then:
```bash
systemctl disable --now nginx
cd /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/xray-shadow default 
nginx -t
systemctl enable --now nginx
```

### 5. Install 3x-ui

Install the latest version of 3x-ui using the official script.

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)

```

#### `IMPORTANT 1 !!!

Installing you need to choose 3) option - Use existing cert and enter paths to Nginx cert:
`/etc/letsencrypt/live/cppru.nekos96.xyz/fullchain.pem`
`/etc/letsencrypt/live/cppru.nekos96.xyz/privkey.pem`

#### `IMPORTANT 2 !!!
You need to change default panel and subscription ports add paths!

### 6. Configure Inbound

Goto Inbounds, press General Actions -> Import an Inbound:

then paste it in popup:
```bash
{
  "id": 1,
  "userId": 0,
  "up": 1342460,
  "down": 305681653,
  "total": 0,
  "allTime": 307024113,
  "remark": "XHTTP-Nginx",
  "enable": true,
  "expiryTime": 0,
  "trafficReset": "never",
  "lastTrafficResetTime": 0,
  "listen": "127.0.0.1",
  "port": 8080,
  "protocol": "vless",
  "settings": "{\n  \"clients\": [\n    {\n      \"comment\": \"\",\n      \"created_at\": 1775931958000,\n      \"email\": \"user1\",\n      \"enable\": true,\n      \"expiryTime\": 0,\n      \"flow\": \"\",\n      \"id\": \"71be44b6-020f-4b33-9db1-77de3420df22\",\n      \"limitIp\": 0,\n      \"reset\": 0,\n      \"subId\": \"igw07lwlp2csk4jq\",\n      \"tgId\": 0,\n      \"totalGB\": 0,\n      \"updated_at\": 1776102103000\n    }\n  ],\n  \"decryption\": \"none\",\n  \"encryption\": \"none\"\n}",
  "streamSettings": "{\n  \"network\": \"xhttp\",\n  \"security\": \"none\",\n  \"externalProxy\": [\n    {\n      \"forceTls\": \"tls\",\n      \"dest\": \"cppru.nekos96.xyz\",\n      \"port\": 443,\n      \"remark\": \"\"\n    }\n  ],\n  \"xhttpSettings\": {\n    \"path\": \"/video/stream-dl\",\n    \"host\": \"\",\n    \"headers\": {},\n    \"scMaxBufferedPosts\": 30,\n    \"scMaxEachPostBytes\": \"1000000\",\n    \"scStreamUpServerSecs\": \"20-80\",\n    \"noSSEHeader\": false,\n    \"xPaddingBytes\": \"100-1000\",\n    \"mode\": \"packet-up\",\n    \"xPaddingObfsMode\": false,\n    \"xPaddingKey\": \"\",\n    \"xPaddingHeader\": \"\",\n    \"xPaddingPlacement\": \"\",\n    \"xPaddingMethod\": \"\",\n    \"uplinkHTTPMethod\": \"\",\n    \"sessionPlacement\": \"\",\n    \"sessionKey\": \"\",\n    \"seqPlacement\": \"\",\n    \"seqKey\": \"\",\n    \"uplinkDataPlacement\": \"\",\n    \"uplinkDataKey\": \"\",\n    \"uplinkChunkSize\": 0\n  }\n}",
  "tag": "inbound-127.0.0.1:8080",
  "sniffing": "{\n  \"enabled\": false,\n  \"destOverride\": [\n    \"http\",\n    \"tls\",\n    \"quic\",\n    \"fakedns\"\n  ],\n  \"metadataOnly\": false,\n  \"routeOnly\": true\n}",
  "clientStats": [
    {
      "id": 1,
      "inboundId": 1,
      "enable": true,
      "email": "user1",
      "uuid": "71be44b6-020f-4b33-9db1-77de3420df22",
      "subId": "igw07lwlp2csk4jq",
      "up": 1326816,
      "down": 305680812,
      "allTime": 307007628,
      "expiryTime": 0,
      "total": 0,
      "reset": 0,
      "lastOnline": 1775943853006
    }
  ]
}
```

*Note: Ensure `path` matches what we configure in NGINX.*


---

## 💻 Client Setup

You can use **v2RayN** (Windows), **v2RayNG** (Android), or the raw **Xray CLI**.

*`Easy way`*
Just copy subscription URL and paste in client app.

*`Hard way`*
Manually configure connection like this:
#### Client Config (config.json)

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


You should see `POST /video/stream-dl` requests appearing as normal HTTP traffic.
2. **Check IP:**
On your client machine, configure your browser to use the SOCKS proxy (`127.0.0.1:1080`) and visit `https://ifconfig.me`. You should see your VPS IP address.

---

## 🧠 Theory: Why this works

* **VLESS:** A lightweight, stateless protocol for tunneling. It handles authentication but leaves encryption to the transport layer.
* **XHTTP:** Wraps VLESS frames into standard HTTP POST (upload) and GET (download) requests. This allows the traffic to pass through strict firewalls and CDNs that only allow valid HTTP.
* **TLS + uTLS:** Encrypts the XHTTP stream. We use `uTLS` with a `chrome` fingerprint to mimic the exact handshake of a Google Chrome browser, preventing DPI from flagging the connection based on TLS ClientHello signatures.
