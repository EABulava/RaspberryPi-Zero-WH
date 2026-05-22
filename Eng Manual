# 🕳️ Pi-hole on Raspberry Pi Zero WH (ARMv6)

> Complete installation guide for Pi-hole v6.2.1 on Raspberry Pi Zero WH (ARMv6) with Debian Trixie.  
> Resolves known compatibility issues: broken `dialog` binary on ARMv6, lack of official ARMv6 support, Raspbian repository conflicts.

---

## Table of Contents

1. [Flash the OS](#1-flash-the-os)
2. [Connect via SSH](#2-connect-via-ssh)
3. [Force IPv4 for apt](#3-force-ipv4-for-apt)
4. [Switch to Debian Repositories](#4-switch-to-debian-repositories)
5. [Install Dependencies](#5-install-dependencies)
6. [Replace dialog with a Stub (ARMv6 fix)](#6-replace-dialog-with-a-stub-armv6-fix)
7. [Download Installer and Repositories](#7-download-installer-and-repositories)
8. [Patch the Installer](#8-patch-the-installer)
9. [Run the Installer](#9-run-the-installer)
10. [Configure the Router](#10-configure-the-router)

---

## 1. Flash the OS

Use **Raspberry Pi Imager**:

- Select image: `Raspberry Pi OS (32-bit)` — **Trixie**
- In advanced settings, configure:
  - `hostname`
  - SSH (enable)
  - Username and password
  - Wi-Fi (SSID + password)
- Write to SD card

---

## 2. Connect via SSH

```bash
ssh-keygen -R 192.168.1.88
ssh eadmin@192.168.1.88
```

---

## 3. Force IPv4 for apt

ARMv6 devices often have issues with IPv6 connectivity. Force IPv4 to avoid `apt` failures:

```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

---

## 4. Switch to Debian Repositories

Raspbian repositories don't properly support ARMv6 — replace them with official Debian Trixie.

### Disable Raspbian repositories

```bash
sudo mv /etc/apt/sources.list.d/raspbian.sources /etc/apt/sources.list.d/raspbian.sources.bak
sudo mv /etc/apt/sources.list.d/raspi.sources /etc/apt/sources.list.d/raspi.sources.bak
```

### Add Debian Trixie

```bash
sudo tee /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
EOF
```

### Import Debian GPG keys

```bash
sudo gpg --keyserver keyserver.ubuntu.com --recv-keys \
  6ED0E7B82643E131 78DBA3BC47EF2265 F8D2585B8783D481 \
  54404762BBB6E853 BDE6D2B9216EC7A8 605C66F00D6C9793

sudo gpg --export \
  6ED0E7B82643E131 78DBA3BC47EF2265 F8D2585B8783D481 \
  54404762BBB6E853 BDE6D2B9216EC7A8 605C66F00D6C9793 \
  | sudo tee /etc/apt/trusted.gpg.d/debian.gpg > /dev/null

sudo apt update
```

---

## 5. Install Dependencies

```bash
sudo dpkg --remove --force-remove-reinstreq pihole-meta 2>/dev/null

sudo apt install -y \
  libdialog15 libjq1 libjemalloc2 libprotobuf-c1 \
  liberror-perl libonig5 lshw dialog git jq dnsutils
```

---

## 6. Replace `dialog` with a Stub (ARMv6 fix)

The `dialog` binary from the repository is incompatible with ARMv6 and crashes the installer. Replace it with an empty stub script:

```bash
sudo mv /usr/bin/dialog /usr/bin/dialog.real

sudo tee /usr/bin/dialog << 'EOF'
#!/bin/sh
exit 0
EOF

sudo chmod +x /usr/bin/dialog
```

> **Note:** The original binary is preserved at `/usr/bin/dialog.real`.

---

## 7. Download Installer and Repositories

Download archives instead of using `git clone` to avoid git-related failures on ARMv6:

```bash
# Pi-hole installer
wget -O ~/install.sh \
  https://raw.githubusercontent.com/pi-hole/pi-hole/v6.2.1/automated%20install/basic-install.sh

# Pi-hole core
sudo mkdir -p /etc/.pihole
wget -O /tmp/pihole.tar.gz \
  https://github.com/pi-hole/pi-hole/archive/refs/tags/v6.2.1.tar.gz
sudo tar -xzf /tmp/pihole.tar.gz -C /etc/.pihole --strip-components=1

# Web interface
sudo mkdir -p /var/www/html/admin
wget -O /tmp/pihole-web.tar.gz \
  https://github.com/pi-hole/web/archive/refs/tags/v6.2.1.tar.gz
sudo tar -xzf /tmp/pihole-web.tar.gz -C /var/www/html/admin --strip-components=1
```

---

## 8. Patch the Installer

The installer tries to run `git` commands inside directories that were extracted from archives. Replace all git calls with `true` (no-op):

```bash
sed -i 's/git stash --all --quiet.*$/true # git stash/' ~/install.sh
sed -i 's/git clean --quiet --force -d.*$/true # git clean/' ~/install.sh
sed -i 's/git pull --no-rebase --quiet.*$/true # git pull/' ~/install.sh
sed -i 's/curBranch=\$(git rev-parse.*$/curBranch="master" # git rev-parse/' ~/install.sh
sed -i 's/git reset --hard.*$/true # git reset/' ~/install.sh
sed -i 's/        git status/        true # git status/' ~/install.sh
sed -i '/git clone -q/c\        true # git clone replaced' ~/install.sh
```

---

## 9. Run the Installer

```bash
sudo PIHOLE_SKIP_OS_CHECK=true \
  PIHOLE_INTERFACE=wlan0 \
  IPV4_ADDRESS=192.168.1.88/24 \
  IPV6_ADDRESS="" \
  PIHOLE_DNS_1=8.8.8.8 \
  PIHOLE_DNS_2=8.8.4.4 \
  QUERY_LOGGING=true \
  INSTALL_WEB_SERVER=true \
  INSTALL_WEB_INTERFACE=true \
  LIGHTTPD_ENABLED=true \
  CACHE_SIZE=10000 \
  DNS_BOGUS_PRIV=true \
  DNSMASQ_LISTENING=single \
  WEBPASSWORD="" \
  BLOCKING_ENABLED=true \
  bash ~/install.sh --unattended
```

> Once complete, the web interface is available at: **http://192.168.1.88/admin**

---

## 10. Configure the Router

Example for **Linksys**:

```
Local Network → DHCP → Static DNS 1: 192.168.1.88 → Apply
```

All devices on the network will now automatically use Pi-hole as their DNS resolver.

---

## Troubleshooting Summary

| Problem | Solution |
|---|---|
| `dialog` crashes the installer on ARMv6 | Replace with a stub `#!/bin/sh` script |
| `git clone` fails during installation | Patch the installer + download archives manually |
| Raspbian packages incompatible with ARMv6 | Switch to official Debian Trixie |
| IPv6 blocks `apt update` | Set `Acquire::ForceIPv4 "true"` |

---

## Versions

| Component | Version |
|---|---|
| Pi-hole | v6.2.1 |
| Raspberry Pi OS | Trixie (32-bit) |
| Architecture | ARMv6 |

---

*Tested on Raspberry Pi Zero WH with Pi-hole v6.2.1 and Debian Trixie.*
