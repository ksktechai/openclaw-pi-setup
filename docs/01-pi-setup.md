# 1. Raspberry Pi 5 — Headless Setup

This is the groundwork: get a brand-new Pi 5 onto your network, reachable over SSH without a
password, addressable by hostname, and (optionally) usable with a Bluetooth keyboard and mouse
for the few times you sit in front of it. Everything after this — Node, OpenClaw, Ollama — assumes
you can `ssh pi` and land in a shell.

> Placeholders used throughout these docs: `<user>` is your Pi login, `<pi-host>` is the Pi's
> hostname, `<pi-ip>` is the Pi's LAN IP, `<ollama-ip>` is the LAN IP of the machine running Ollama.

New to the Pi? The official [Raspberry Pi getting-started guide](https://www.raspberrypi.com/documentation/computers/getting-started.html)
covers flashing the OS with Raspberry Pi Imager. In the Imager's advanced options, set the
hostname, enable SSH, and configure Wi-Fi up front — it saves you a monitor on first boot.

**Hardware/OS this was done on:**

| Component | Spec |
|-----------|------|
| Board | Raspberry Pi 5 Model B Rev 1.1 |
| Architecture | aarch64 (ARM 64-bit) |
| RAM | 16 GB |
| OS | Debian GNU/Linux 13 (Trixie) |

---

## 1.1 Enable SSH

Fresh Raspberry Pi OS installs often ship with the SSH server **off**, so the first connection
fails:

```bash
ssh <user>@<pi-ip>
# ssh: connect to host <pi-ip> port 22: Connection refused
```

Enable it from the Pi (via monitor/keyboard, or the Imager's advanced options before flashing):

```bash
sudo raspi-config        # Interface Options → SSH → Enable
# or:
sudo systemctl enable --now ssh
```

Then the connection succeeds:

```bash
ssh <user>@<pi-ip>
```

---

## 1.2 Passwordless SSH with a key pair

> **Key direction matters.** Generate the key on the machine you connect **from** (your Mac/laptop),
> not on the Pi. The public key gets copied **to** the Pi.

On your Mac:

```bash
ssh-keygen -t ed25519
# creates ~/.ssh/id_ed25519 (private — never share) and ~/.ssh/id_ed25519.pub (public)
```

Copy the public key to the Pi:

```bash
ssh-copy-id <user>@<pi-ip>

# Or, if ssh-copy-id isn't available:
cat ~/.ssh/id_ed25519.pub | ssh <user>@<pi-ip> \
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Verify — this should log in with **no password prompt**:

```bash
ssh <user>@<pi-ip>
```

---

## 1.3 Reach the Pi by hostname (mDNS)

IP addresses drift when the router hands out a new lease. The hostname doesn't.

On the Pi, confirm its name:

```bash
hostname
# <pi-host>
```

From the Mac, resolve it via mDNS (Bonjour/Avahi, built into Raspberry Pi OS and macOS):

```bash
ping <pi-host>.local
# PING <pi-host>.local (<pi-ip>)
```

A reply confirms mDNS, Bonjour/Avahi, and hostname resolution are all working.

---

## 1.4 Create an SSH alias

So you can type `ssh pi` instead of the full user@host. Edit `~/.ssh/config` on the Mac:

```ssh
# Raspberry Pi 5
Host pi
    HostName <pi-host>.local
    User <user>
```

Now:

```bash
ssh pi
```

Because this resolves through the hostname, it keeps working even after the router reassigns the
Pi a new IP — no config change needed.

---

## 1.5 Fix the "Host key verification failed" warning

After adding the hostname alias, the first connect may complain:

```text
Host key verification failed
```

This happens because SSH already recorded the Pi under its **IP**, but now sees it under its
**hostname** — it treats them as different hosts. Clear both stale entries and reconnect:

```bash
ssh-keygen -R <pi-ip>
ssh-keygen -R <pi-host>.local
ssh pi          # accept the fingerprint with "yes"
```

---

## 1.6 Pair a Bluetooth keyboard and mouse (optional)

Handy for the rare times you want a local console. Done here with a Logitech MX Keys Mini and
MX Master 2S, but any Bluetooth device works the same way.

Start the Bluetooth tool on the Pi:

```bash
bluetoothctl
```

Inside the `bluetoothctl` prompt:

```text
power on
agent on
default-agent
scan on
```

Put the device into pairing mode (hold its Easy-Switch / pairing button ~3s until the LED blinks
rapidly). It shows up in the scan:

```text
[NEW] Device <bt-mac> MX Keys Mini
```

Pair, trust (so it reconnects automatically), and connect — using the MAC shown in your scan:

```text
pair <bt-mac>
trust <bt-mac>
connect <bt-mac>
```

Verify:

```text
info <bt-mac>
# Paired: yes
# Trusted: yes
# Connected: yes
```

Once everything is paired, stop scanning and exit:

```text
scan off
devices        # lists paired devices
quit
```

Because the devices are **trusted**, they auto-reconnect after a reboot — provided Bluetooth is on,
the devices are powered, and they aren't currently connected to another machine.

---

## 1.7 Verify connectivity (bottom-up)

Walk the stack from the local NIC outward so a failure tells you exactly which layer is broken:

```bash
hostname -I                 # 1. Pi has a local IP
ping -c 4 192.168.1.1       # 2. router reachable (use your gateway)
ping -c 4 8.8.8.8           # 3. internet reachable (raw IP, no DNS)
ping -c 4 google.com        # 4. DNS resolving
curl -I https://www.google.com   # 5. HTTPS working (expect HTTP/2 200 or 301)
sudo apt update             # 6. package repos reachable over HTTPS
```

If step 3 works but step 4 doesn't, it's DNS. If step 4 works but step 5 doesn't, it's TLS/proxy.

---

## 1.8 Keep it current

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

At this point the Pi is headless-ready: passwordless `ssh pi`, hostname-addressable, updated, and
online. Next: **[2. Installing OpenClaw](02-openclaw-setup.md)**.
