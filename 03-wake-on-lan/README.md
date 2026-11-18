# 03: Wake-on-LAN

## Goal

Not having to leave the desktop on all day, but still being able to start it from anywhere.

```
iPhone / laptop ──Tailscale──▶ relay (Ubuntu) ──magic packet on LAN──▶ desktop
```

A magic packet is a broadcast on the local network, so it can't be sent directly over Tailscale. The always-on relay (part 2) sits on the home network and sends it for me.

## Desktop

The desktop must be connected **by cable**. Wake-on-LAN over Wi-Fi almost never works.

### BIOS (ASUS ROG Strix B550-F)

Del > F7 (Advanced Mode) > **Advanced > APM Configuration**:

| Setting | Value | Why |
|---|---|---|
| ErP Ready | Disabled | With ErP on, the network card loses power when the PC is off |
| Power On By PCI-E | Enabled | Lets the onboard NIC wake the system |

### Windows

Disable Fast Startup (elevated PowerShell). With Fast Startup, Windows doesn't fully shut down and the NIC often ignores magic packets:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Power" /v HiberbootEnabled /t REG_DWORD /d 0 /f
```

Device Manager > Network adapters > **Intel(R) Ethernet Controller I225-V** > Properties:

- **Power Management:** *Allow this device to wake the computer* and *Only allow a magic packet to wake the computer*
- **Advanced:** *Wake on Magic Packet* = Enabled

MAC address of the Ethernet adapter:

```powershell
Get-NetAdapter | Select-Object Name, MacAddress, Status
```

## Relay

```bash
wakeonlan AA:BB:CC:DD:EE:FF
```

Shortcut so I don't have to remember the MAC:

```bash
echo "alias wakepc='wakeonlan AA:BB:CC:DD:EE:FF'" >> ~/.bashrc && source ~/.bashrc
wakepc
```

## iPhone: one-tap "Wake PC"

Built with the **Shortcuts** app, no extra apps needed:

1. New shortcut > action **Run Script Over SSH**
2. Host: `<relay>`, port `22`, user `<user>`, authentication **SSH Key** (public key added to the relay's `authorized_keys`)
3. Script: `wakeonlan AA:BB:CC:DD:EE:FF` (the full command; `.bashrc` aliases aren't loaded in non-interactive sessions)
4. Added to the home screen as **Wake PC**

Flow from anywhere: tap **Wake PC**, wait a minute, connect with Remote Desktop.

## Test

1. Shut down the desktop normally (Start > Shut down).
2. Run `wakepc` on the relay (or tap the Shortcut).
3. The desktop powers on within a few seconds.
