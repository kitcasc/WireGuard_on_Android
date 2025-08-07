# 🛡️ Build Your Own WireGuard VPN Server Using an Android Phone (No Root!)

## 📌 Overview
This guide walks you through setting up a personal WireGuard VPN server using an Android phone — no root required. By the end, you'll be able to connect securely to your home network from anywhere in the world using your own domain name and dynamic IP updating via Cloudflare.

## 🧰 Requirements

- An Android phone (Android 7.0 or newer) — this will act as your VPN server
- A domain name (e.g., `yourname.com`)
- A free [Cloudflare](https://cloudflare.com) account
- Access to your home router's admin panel (for port forwarding)
- This guide will only work if your network has a public IP address and allows port forwarding; it will fail if you are behind a Double NAT or CGNAT.
---

## 🛠️ Step 1: Setup Base Environment on the Android Phone

### 1.1 Install Termux
- **Do NOT** use the Google Play version — it’s outdated.
- Download and install the F-Droid app from [https://f-droid.org](https://f-droid.org)
- Open F-Droid → search for `Termux` → install it

### 1.2 Initialize Termux
```bash
termux-setup-storage
```
Grant storage permissions when prompted.

### 1.3 Install Required Tools
```bash
pkg update && pkg upgrade
pkg install git golang nano curl jq tar
```

---

## 🌐 Step 2: Setup Cloudflare Dynamic DNS

### 2.1 Add Your Domain to Cloudflare
- Add your domain as a site in Cloudflare
- Change your domain's nameservers to Cloudflare’s (via your domain registrar)

### 2.2 Create DNS A Record for the VPN
- Go to DNS settings → Add a record:
  - **Type**: A
  - **Name**: `vpn` (creates `vpn.yourdomain.com`)
  - **IPv4 Address**: Use a placeholder like `1.1.1.1`
  - **Proxy Status**: Must be **DNS only** (grey cloud!)

### 2.3 Create an API Token
- Go to Profile → API Tokens → Create Token
- Use **"Edit Zone DNS"** template
- Permissions:
  - Zone → DNS → Edit
- Zone Resources:
  - Include → Specific zone → `yourdomain.com`
- Copy and save the token securely — it won’t be shown again!

### 2.4 Find Zone ID and Record ID
```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/dns_records" \
     -H "Authorization: Bearer YOUR_API_TOKEN" \
     -H "Content-Type: application/json" | jq
```
Or execute in Windows cmd:
```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/zones/ZONE_ID_HERE/dns_records" -H "Authorization: Bearer CF_API_TOKEN_HERE"
```
Find the record ID corresponding to your `vpn` A record.

### 2.5 Create and Setup DDNS Update Script
```bash
nano ~/cloudflare-ddns.sh
```
Paste this and fill in your details:
```bash
#!/bin/bash
CF_API_TOKEN="YOUR_API_TOKEN"
ZONE_ID="YOUR_ZONE_ID"
RECORD_ID="YOUR_RECORD_ID"

LOG_FILE="$HOME/cloudflare-ddns.log"
IP_FILE="$HOME/current_ip.txt"
CURRENT_IP=$(curl -s https://ipv4.icanhazip.com/)

[ ! -f "$IP_FILE" ] && echo "0.0.0.0" > "$IP_FILE"
LAST_IP=$(cat "$IP_FILE")

if [ "$CURRENT_IP" == "$LAST_IP" ]; then
    echo "$(date): IP $CURRENT_IP has not changed." >> "$LOG_FILE"
    exit 0
fi

echo "$(date): IP changed from $LAST_IP to $CURRENT_IP. Updating..." >> "$LOG_FILE"
RESPONSE=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
     -H "Authorization: Bearer $CF_API_TOKEN" \
     -H "Content-Type: application/json" \
     --data "{\"type\":\"A\",\"name\":\"vpn\",\"content\":\"$CURRENT_IP\",\"ttl\":1,\"proxied\":false}")

if echo "$RESPONSE" | grep -q '"success":true'; then
    echo "$CURRENT_IP" > "$IP_FILE"
    echo "$(date): Cloudflare update successful." >> "$LOG_FILE"
else
    echo "$(date): Cloudflare update failed. Response: $RESPONSE" >> "$LOG_FILE"
fi
```
Save and make executable:
```bash
chmod +x ~/cloudflare-ddns.sh
```

### 2.6 Setup Cron to Run the Script Automatically
```bash
pkg install cronie
crond
crontab -e
```
Add to run the script every 10 minutes:
```
*/10 * * * * /data/data/com.termux/files/home/cloudflare-ddns.sh
```

---

## 🔐 Step 3: Install & Configure WireGuard

### 3.1 Compile `wireguard-go`
```bash
cd ~
git clone https://git.zx2c4.com/wireguard-go
cd wireguard-go
go build -o wireguard-go .
mv wireguard-go $PREFIX/bin/
```

### 3.2 Generate Keys
```bash
mkdir -p ~/.wireguard
cd ~/.wireguard
wg genkey | tee server_privatekey | wg pubkey > server_publickey
wg genkey | tee client_privatekey | wg pubkey > client_publickey
```
```bash
cat ~/.wireguard/server_privatekey
cat ~/.wireguard/server_publickey
cat ~/.wireguard/client_privatekey
cat ~/.wireguard/client_publickey
```
Save the keys!

### 3.3 Create Server Config
```bash
nano ~/.wireguard/wg0.conf
```
Example:
```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = YOUR_SERVER_PRIVATE_KEY

[Peer]
PublicKey = YOUR_CLIENT_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32
```

### 3.4 Create Client Config
```bash
nano ~/.wireguard/client.conf
```
```ini
[Interface]
PrivateKey = YOUR_CLIENT_PRIVATE_KEY
Address = 10.0.0.2/32
DNS = 1.1.1.1  # Use Cloudflare DNS to avoid DNS leaks

[Peer]
PublicKey = YOUR_SERVER_PUBLIC_KEY
Endpoint = vpn.yourdomain.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

✅ Optional: Install `qrencode` to generate QR code for client config:
```bash
pkg install qrencode
qrencode -t ansiutf8 < client.conf
```

---

## 🌐 Step 4: Configure Router and Launch Server

### 4.1 Home Router Settings
- **Reserve DHCP IP** for the Android device (via MAC address)
- **Port Forward** UDP 51820 to the reserved internal IP

### 4.2 Install WireGuard App
- Install [WireGuard Android App](https://play.google.com/store/apps/details?id=com.wireguard.android)
- Copy `wg0.conf` to `~/storage/downloads/`
- Open the app → Import → From File → Select `wg0.conf`
- Enable VPN

---

## 📲 Step 5: Client Setup

- Transfer `client.conf` to your remote device
- Install WireGuard on remote device (Windows/macOS/Linux/Android/iOS)
- Import config → Activate
- Check your IP on [https://whatismyipaddress.com](https://whatismyipaddress.com) — it should be your home IP!

---

## 🛡️ Stability Tips: Prevent Android Killing Your Server

### 1. Enable Wake Lock in Termux
```bash
termux-wake-lock
```
Leave that Termux session running in the background.

### 2. Disable Battery Optimization
- For both **Termux** and **WireGuard** apps:
  - Long-press app icon → App Info
  - Battery → Set to "Unrestricted" or "Don't optimize"

---

## ✅ Final Tips

- Test for DNS leaks: [https://dnsleaktest.com](https://dnsleaktest.com)
- Make sure your router doesn’t block UDP 51820
- Consider setting up alerts/logging for DDNS updates

---

## 🎉 Done!
Congrats — you’ve now got a 24/7 personal VPN server running off a spare Android phone, with your own domain and dynamic IP updates.


Let me know if you found this helpful, or suggest improvements.
