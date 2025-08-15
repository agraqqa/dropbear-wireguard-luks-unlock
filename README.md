WireGuard in initramfs with Dropbear
====================================

This document explains how to set up a WireGuard VPN inside `initramfs` so you can SSH remotely into a system with Dropbear **before** the root filesystem is mounted.

This is useful for remote unlocking of encrypted disks.

Typical use case - you have a Raspberry Pi with an encrypted disk that is connected to internet w/o static IP address. Blackout happens, the Pi reboots, and you need to unlock the disk remotely via SSH, but you are not in the same network. This setup brings up a WireGuard interface in the `initramfs` environment, allowing you to SSH into the device over encrypted tunnel and unlock the disk.

WireGuard connection procedure isn't blocking, it will retry to establish the connection until it succeeds, while still allowing you to SSH into the device using LAN IP address

# 1. Prerequisites

- Debian/Ubuntu with `initramfs-tools`

- Dropbear for initramfs installed:

```bash
sudo apt install dropbear-initramfs
```

and configured to start in `initramfs` phase. Also, it is highly recommended to set up ssh keys for Dropbear

- WireGuard tools:

```bash
sudo apt install wireguard-tools
```

- A known WireGuard configuration, e.g., `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 192.168.163.14/32
PrivateKey = <your_private_key>
PersistentKeepalive = 25

[Peer]
PublicKey = <server_public_key>
PresharedKey = <preshared_key>
AllowedIPs = 192.168.163.0/24
Endpoint = <server_ip>:51820
```

# 2. Copy WireGuard configuration to initramfs

```bash
sudo mkdir -p /etc/initramfs-tools/etc/wireguard
sudo cp /etc/wireguard/wg0.conf /etc/initramfs-tools/etc/wireguard/wg0.conf
sudo chmod 600 /etc/initramfs-tools/etc/wireguard/wg0.conf
```

# 3. Enable WireGuard module in initramfs

Enable `wireguard`, `udp_tunnedl` and `ip6_udp_tunnedl` in `/etc/initramfs-tools/modules`:

```bash
echo wireguard      | sudo tee -a /etc/initramfs-tools/modules
echo udp_tunnel     | sudo tee -a /etc/initramfs-tools/modules
echo ip6_udp_tunnel | sudo tee -a /etc/initramfs-tools/modules
```

# 4. Create initramfs hook for WireGuard installation

Create /etc/initramfs-tools/hooks/wireguard:

```bash
#!/bin/sh
PREREQ=""

prereqs() {
    echo "$PREREQ"
}

case $1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

# Copy binaries
copy_exec() {
    cp "$1" "${DESTDIR}/usr/bin/"
}

copy_exec /usr/bin/wg
[ -x /usr/bin/wg-quick ] && copy_exec /usr/bin/wg-quick

# Copy your pre-made config into initramfs
mkdir -p "${DESTDIR}/etc/wireguard"
cp /etc/initramfs-tools/etc/wireguard/wg0.conf "${DESTDIR}/etc/wireguard/wg0.conf"
chmod 600 "${DESTDIR}/etc/wireguard/wg0.conf"
```

Make sure the script is executable:

```bash
sudo chmod +x /etc/initramfs-tools/hooks/wireguard
```

# 5. Create initramfs script to bring up WireGuard interface

Create /etc/initramfs-tools/scripts/init-premount/wireguard:

```bash
#!/bin/sh
PREREQ="dropbear"

prereqs() {
    echo "$PREREQ"
}

case "$1" in
    prereqs)
        prereqs
        exit 0
    ;;
esac

LOGFILE="/run/wg-initramfs.log"

log() {
    echo "[$(date '+%H:%M:%S')] $1" >> "$LOGFILE"
}

(
log "Starting WireGuard retry loop in initramfs..."

# Wait until default route exists (network is up)
while ! ip route | grep -q default; do
    log "Waiting for default route..."
    sleep 2
done

log "Default route detected, starting WireGuard..."

# Retry loop
while true; do
    # If handshake is established, stop loop
    if wg show wg0 2>/dev/null | grep -q 'latest handshake'; then
        log "WireGuard is up — stopping retry loop."
        break
    fi

    log "Attempting WireGuard connection..."

    # Remove wg0 if it exists from previous failed attempt
    ip link show wg0 >/dev/null 2>&1 && {
        log "Removing existing wg0 interface from previous attempt..."
        ip link delete wg0
    }

    # Bring up wg0 using wg-quick
    if [ -x /usr/bin/wg-quick ]; then
        /usr/bin/wg-quick up /etc/wireguard/wg0.conf >> "$LOGFILE" 2>&1 || log "wg-quick failed, will retry..."
    else
        log "wg-quick binary not found!"
    fi

    sleep 10
done
) &
```

# 6. Rebuild initramfs

Rebuild the initramfs to apply the changes:

```bash
sudo update-initramfs -u
```

# 7. Reboot and test

After rebooting, you should be able to:
- SSH into the device using the WireGuard interface
- SSH into the device using the local LAN IP address

You have to be connected to the same WireGuard network as the device:

```bash
ssh -i ~/.ssh/keys/dropbear_rsa -p 222 root@192.168.163.14
```

