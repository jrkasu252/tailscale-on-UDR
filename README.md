# Tailscale on UniFi Dream Router (UDR) with Exit Node + Auto-Update

This guide helps you install **Tailscale** persistently on a **UniFi Dream Router (UDR)** running UniFi OS, with:

* ✅ Persistent install in `/mnt/data/tailscale`
* 🌐 Exit Node support (`--advertise-exit-node`)
* 🔄 Auto-update to the latest stable Tailscale release
* ↺ Startup script that runs on every reboot

---

## ⚙️ Requirements

* UDR running UniFi OS 4.x+
* SSH access
* Architecture: `aarch64` (ARM64)

---

## 💪 Installation Steps

### 1. Create Persistent Tailscale Folder

```sh
mkdir -p /mnt/data/tailscale
cd /mnt/data/tailscale
```

---

### 2. Create Auto-Update Script

```sh
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

# Move binaries regardless of archive structure
if [ -f tailscale ]; then
  echo "Binaries already in current directory"
else
  mv tailscale*/tailscale .
  mv tailscale*/tailscaled .
  rm -rf tailscale_*
fi

echo "Tailscale updated to v$VERSION"
EOF

chmod +x /mnt/data/tailscale/update-tailscale.sh
```

---

### 3. Create Startup Script for Reboots

```sh
mkdir -p /mnt/data/on_boot.d
cat <<'EOF' > /mnt/data/on_boot.d/99-tailscale.sh
#!/bin/sh
/mnt/data/tailscale/update-tailscale.sh
/mnt/data/tailscale/tailscaled --state=/mnt/data/tailscaled.state --socket=/run/tailscale/tailscaled.sock &
sleep 3
/mnt/data/tailscale/tailscale up --advertise-exit-node --ssh
EOF

chmod +x /mnt/data/on_boot.d/99-tailscale.sh
```

---

## ✅ Testing

* Run manually to test:

  ```sh
  /mnt/data/on_boot.d/99-tailscale.sh
  ```

* Check status:

  ```sh
  /mnt/data/tailscale/tailscale status
  ```

* Reboot and verify it comes back up:

  ```sh
  reboot
  ```

---

## 🧠 Notes

* To include subnet routing, add `--advertise-routes=192.168.1.0/24` to the `tailscale up` command.
* To auto-authenticate without user input, create a [Tailscale auth key](https://login.tailscale.com/admin/settings/authkeys) and use:

  ```sh
  /mnt/data/tailscale/tailscale up --authkey <YOUR_KEY> --advertise-exit-node --ssh
  ```

---

## 💡 Credits

This setup combines:

* [Tailscale static binary install](https://pkgs.tailscale.com/stable/)
* UniFi's `/mnt/data` persistent boot scripts
* Crontab fallback for systems without full `rc.local` support

---

## 🧪 Tested On

* UniFi Dream Router (UDR)
* UniFi OS 4.2.14
* Kernel 5.4.x
* Tailscale v1.84.0 (auto-updated)

---

> PRs and improvements welcome!
