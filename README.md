# Tailscale on UniFi Dream Router (UDR)

> This guide explains how to install and persistently run the latest version of [Tailscale](https://tailscale.com) on a UniFi Dream Router (UDR) running UbiOS. It includes steps for enabling exit node, SSH access, and automatic updates.

---

## ✅ Prerequisites

* A UniFi Dream Router (UDR) running **UbiOS** (tested on v4.2.14)
* SSH access enabled on the UDR (via UniFi Network web UI)
* A Tailscale account

---

## 🚪 Step 1: SSH into your UDR

Enable SSH in the UniFi web UI (Settings > Advanced > SSH), then connect:

```bash
ssh root@<UDR-IP>
```

---

## 📁 Step 2: Set Up Persistent Folders

```bash
mkdir -p /mnt/data/tailscale
mkdir -p /mnt/data/on_boot.d
```

---

## ⬇️ Step 3: Install Latest Tailscale (for ARM64)

Create the update script:

```bash
cat <<'EOF' > /mnt/data/tailscale/update-tailscale.sh
#!/bin/sh

ARCH=arm64
INSTALL_DIR=/mnt/data/tailscale

LATEST=$(curl -fsSL https://pkgs.tailscale.com/stable/ | grep -oE "tailscale_[0-9.]+_${ARCH}.tgz" | sort -V | tail -1)
VERSION=$(echo "$LATEST" | cut -d_ -f2)

CURRENT=$($INSTALL_DIR/tailscale version 2>/dev/null | head -n1)

if echo "$CURRENT" | grep -q "$VERSION"; then
  echo "Tailscale is already up to date (v$VERSION)"
  exit 0
fi

echo "Updating to Tailscale v$VERSION..."

cd "$INSTALL_DIR"
curl -fsSL "https://pkgs.tailscale.com/stable/$LATEST" | tar xz
mv tailscale*/tailscale .
mv tailscale*/tailscaled .
rm -rf tailscale*

echo "Tailscale updated to v$VERSION"
EOF

chmod +x /mnt/data/tailscale/update-tailscale.sh
```

Run it once to install:

```bash
/mnt/data/tailscale/update-tailscale.sh
```

---

## ⚙️ Step 4: Create Persistent Boot Script

```bash
cat <<'EOF' > /mnt/data/on_boot.d/99-tailscale.sh
#!/bin/sh

echo "[BOOT] Tailscale starting..." >> /mnt/data/tailscale/boot.log

# Cleanup from previous boots
ip link delete tailscale0 2>/dev/null
pkill -f tailscaled
rm -rf /run/tailscale
mkdir -p /run/tailscale

# Update to latest
/mnt/data/tailscale/update-tailscale.sh >> /mnt/data/tailscale/boot.log 2>&1

# Start daemon
/mnt/data/tailscale/tailscaled \
  --state=/mnt/data/tailscaled.state \
  --socket=/run/tailscale/tailscaled.sock >> /mnt/data/tailscale/boot.log 2>&1 &

sleep 3

# Bring up Tailscale with exit node and SSH
/mnt/data/tailscale/tailscale up --advertise-exit-node --ssh >> /mnt/data/tailscale/boot.log 2>&1

echo "[BOOT] Tailscale startup complete" >> /mnt/data/tailscale/boot.log
EOF

chmod +x /mnt/data/on_boot.d/99-tailscale.sh
```

---

## 🔁 Step 5: Auto-Run on Boot

Create a master boot script if needed:

```bash
cat <<'EOF' > /mnt/data/on_boot.sh
#!/bin/sh
/mnt/data/on_boot.d/99-tailscale.sh &
EOF

chmod +x /mnt/data/on_boot.sh
```

✅ UbiOS automatically runs `/mnt/data/on_boot.sh` on boot.

---

## 🔍 Step 6: Verify Functionality

Check status:

```bash
/mnt/data/tailscale/tailscale --socket=/run/tailscale/tailscaled.sock status
```

Check logs:

```bash
cat /mnt/data/tailscale/boot.log
```

---

## 🔁 Step 7: Reboot to Test Persistence

```bash
reboot
```

After reboot:

```bash
ls -l /run/tailscale
/mnt/data/tailscale/tailscale --socket=/run/tailscale/tailscaled.sock status
```

---

## ✅ Optional: Add Alias for Easier CLI Use

```bash
echo "alias tailscale='/mnt/data/tailscale/tailscale --socket=/run/tailscale/tailscaled.sock'" >> ~/.profile
. ~/.profile
```

Now just run:

```bash
tailscale status
```

---

## 🚀 You're Done!

Your UDR now:

* Runs the latest Tailscale
* Advertises itself as an exit node
* Restarts automatically on boot
* Auto-updates on reboot

---

## 🛠 Troubleshooting

* `tailscale status` fails: check if `tailscaled` is running and the socket exists
* Exit node not advertised? Re-run `tailscale up --advertise-exit-node --ssh`
* Stuck `tailscale0`? Try: `ip link delete tailscale0`

---

## ❤️ Credits

* [Tailscale](https://tailscale.com)
* [SierraSoftworks/tailscale-udm](https://github.com/SierraSoftworks/tailscale-udm)
* Community threads on Reddit & GitHub for persistence tips

---

## 🧠 Improvements

* Add `accept-routes` or subnet routing
* Use Tailscale Auth Keys for headless provisioning
* Backup state file `/mnt/data/tailscaled.state`
