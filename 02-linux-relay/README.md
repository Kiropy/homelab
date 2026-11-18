# 02: Linux relay (old HP laptop)

## Goal

Wake-on-LAN needs a device on the home network that is always on and can send the magic packet. Instead of buying a Raspberry Pi, I reused an old HP laptop:

- built-in "UPS" (the battery)
- low power usage
- can grow into a small Linux server later

The catch: **half of the screen is broken**, and the BIOS has an **administrator password I don't know**. So the goal was to get it running headless and manage it only over SSH.

## Install

- **OS:** Ubuntu Server 26.04.1 LTS (amd64). Server instead of Desktop: no GUI, much lighter for old hardware.
- **USB:** written with Rufus.
- **Installer choices:** Ubuntu Server (not minimized), use entire disk with LVM, **OpenSSH server** enabled, no extra snaps.

## Problem 1: it wouldn't boot after the install

After rebooting, the laptop tried PXE network boot and then dropped into HP PC Hardware Diagnostics. I couldn't change boot settings because the BIOS is password protected, and the broken screen made the menus hard to read.

I booted the USB again and opened a shell from the installer (**Help > Enter shell**):

```
sda       298.1G disk
├─sda1        1M part
├─sda2        2G part
└─sda3    296.1G part
  └─ubuntu--vg-ubuntu--lv  100G lvm
```

A **1 MB partition** (`bios_grub`) and no EFI partition: the USB had booted in **legacy BIOS mode**, so Ubuntu was installed for legacy boot on a GPT disk.

**Cause:** many older BIOSes (HP included) refuse to legacy-boot a GPT disk unless the protective MBR is marked bootable.

**Fix:** set the flag and reinstall GRUB from a chroot:

```bash
parted /dev/sda disk_set pmbr_boot on

vgchange -ay
mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt
mount /dev/sda2 /mnt/boot
for d in dev proc sys; do mount --bind /$d /mnt/$d; done
chroot /mnt grub-install --target=i386-pc /dev/sda
chroot /mnt update-grub
reboot
```

(`parted` prints "You may need to update /etc/fstab". That's just an info message, no partitions were changed.)

After this, the laptop booted into Ubuntu.

## Problem 2: no network

The Ethernet port had no link (`cat /sys/class/net/en*/carrier` returned `0`), so I switched to Wi-Fi. That's fine for this role: only the desktop needs a cable for Wake-on-LAN; the relay can send the packet over Wi-Fi.

Config: [`configs/02-wifi.yaml`](configs/02-wifi.yaml) in `/etc/netplan/`.

```bash
sudo chmod 600 /etc/netplan/02-wifi.yaml
sudo netplan apply
```

### Errors I hit

| Error | Cause | Fix |
|---|---|---|
| `Invalid YAML: inconsistent indentation` | `password:` had one space too few | Every level is indented exactly 2 spaces deeper |
| `networkd backend does not support wifi with match:, only by interface name` | Used `match: name: "wl*"` like for Ethernet | Use the real interface name from `ip -br link` (e.g. `wlo1`) |

## Post-install

```bash
# Updates
sudo apt update && sudo apt upgrade -y

# The installer only used 100 GB of the LVM volume group: use all of it
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Keep running with the lid closed
sudo sed -i 's/^#\?HandleLidSwitch=.*/HandleLidSwitch=ignore/; s/^#\?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind

# Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Wake-on-LAN tool
sudo apt install -y wakeonlan
```

Key expiry is disabled for the relay in the Tailscale admin console.

## SSH: keys only

At first I used Tailscale SSH (`tailscale up --ssh`), which authenticates with your Tailscale identity. I switched to regular OpenSSH with keys, because it's the standard way and works better with the iPhone Shortcut (see part 3).

```bash
sudo tailscale set --ssh=false
```

### Keys

Desktop (PowerShell):

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@<relay> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

iPhone:

- **Termius:** generated an ED25519 key in the Keychain and used *Export to host*.
- **Shortcuts:** the *Run Script Over SSH* action has its own key; copied its public key into `~/.ssh/authorized_keys`.

### Disable password login

Only after key login worked from every device (with one SSH session kept open as a fallback):

Config: [`configs/00-keys-only.conf`](configs/00-keys-only.conf) in `/etc/ssh/sshd_config.d/`. The `00-` prefix makes it load before Ubuntu's own drop-in files, and for sshd the first value wins.

```bash
sudo systemctl restart ssh
```

Check from the desktop:

```powershell
ssh -o PubkeyAuthentication=no <user>@<relay>
# Permission denied (publickey).
```

The Linux password is still used for `sudo`; only SSH password login is disabled.
