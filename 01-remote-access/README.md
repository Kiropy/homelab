# 01: Remote access

## Goal

Most of my files and VMs were on my laptop, which meant I always had to carry it around. I wanted to:

- reach my desktop from my laptop (also from school), and the other way around
- use the desktop's resources (CPU, RAM, VMs) remotely
- move my VMs from the laptop to the desktop, so the laptop becomes just a client

## Overview

| Direction | Tool | Why |
|---|---|---|
| Laptop/iPhone → Desktop | RDP | Desktop runs Windows 11 Pro, so it can be an RDP host. Light on bandwidth, adapts to the client's resolution |
| Desktop → Laptop | Sunshine + Moonlight | Laptop runs Windows 11 **Home**, which cannot accept RDP connections |
| Everything | Tailscale | Mesh VPN, no port forwarding needed, works behind school/mobile networks |

## 1. Tailscale

1. Installed Tailscale on the desktop and laptop and logged in with the same account.
2. In the admin console (`login.tailscale.com/admin/machines`): **Disable key expiry** for the desktop, so it doesn't drop off the tailnet after a few months while I'm away.
3. Verified connectivity from the laptop:

```powershell
tailscale ping <desktop>
# pong from <desktop> (100.x.x.x) via 192.168.x.x:41641 in 3ms
```

The `via 192.168.x.x` part means a direct LAN connection, not a relay.

## 2. RDP to the desktop

On the desktop:

- Settings > System > Remote Desktop > **On** (Network Level Authentication stays enabled)
- Settings > System > Power > sleep **Never** when plugged in (replaced later by Wake-on-LAN, see part 3)

On the laptop: `mstsc` > `<desktop>` (Tailscale MagicDNS name) or its 100.x.x.x IP.

**Security:** port 3389 is **not** forwarded on the router. RDP is only reachable through Tailscale.

### Troubleshooting: RDP login failed

First I checked whether it was a network problem:

```powershell
tailscale ping <desktop>                       # Tailscale OK
Test-NetConnection 100.x.x.x -Port 3389        # TcpTestSucceeded : True
```

Network and port were fine, so the problem was authentication. The desktop uses a **Microsoft account**, and I always logged in with a PIN.

Testing the credentials locally on the desktop:

```powershell
runas /u:MicrosoftAccount\<email> cmd.exe
# RUNAS ERROR: 1326: The user name or password is incorrect.
```

**Cause:** I had changed my Microsoft account password months earlier, but only ever logged in to the desktop with a PIN since then. Windows still had the **old password cached**, so the new one was rejected.

**Fix:** sign in on the desktop once with the actual password, so Windows refreshes the cache. The password option didn't show on the lock screen, because Windows was enforcing passwordless sign-in (and the toggle was hidden in Settings). Re-enabled it in an **elevated** PowerShell:

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\PasswordLess\Device" /v DevicePasswordLessBuildVersion /t REG_DWORD /d 0 /f
```

(The first attempt gave `Access is denied` because the shell was not elevated.)

After that: lock screen > Sign-in options > key icon > log in with password. RDP worked with `MicrosoftAccount\<email>`.

## 3. Sunshine + Moonlight (desktop → laptop)

Because the laptop runs Windows Home, I used Sunshine (host) on the laptop and Moonlight (client) on the desktop. Both are free and open source.

1. Laptop: installed `Sunshine-Windows-AMD64-installer.msi` from the official GitHub releases, opened `https://localhost:47990` and created a web UI login.
2. Desktop: installed Moonlight, added the laptop manually by its Tailscale IP.
3. Paired with the PIN shown in Moonlight (entered in the Sunshine web UI under **PIN**).

### Troubleshooting: error 409

```
GeForce Experience returned error: A pairing session with this uniqueid already exists (Error 409)
```

**Cause:** an earlier pairing attempt was still open on the Sunshine side.
**Fix:** restart Sunshine and pair again, entering the PIN right away:

```powershell
Restart-Service SunshineService
```

## 4. iPhone

Installed Tailscale and the Windows App (Remote Desktop) on the iPhone. RDP to the desktop works the same way as from the laptop.

Controlling the iPhone *from* the PC is not possible: iOS doesn't allow third-party remote control. For files between PC and iPhone I use **Taildrop** (built into Tailscale).

## 5. Moving the VMs to the desktop

The VMs were stored on the laptop in `OneDrive\Documents\Virtual Machines`. They now live on the desktop's HDD (`E:\VMs`). The HDD is slower than the SSD, but fine for lab VMs.

VMware Workstation Pro is free; the installer comes from the Broadcom support portal (free account, *Free Software Downloads*). It's installed on C: (SSD), only the VMs are on the HDD.

### SMB share on the desktop (elevated PowerShell)

```powershell
New-Item -ItemType Directory -Path E:\VMs
New-SmbShare -Name VMs -Path E:\VMs -FullAccess "$env:COMPUTERNAME\$env:USERNAME"

# Allow SMB only from Tailscale addresses (CGNAT range used by Tailscale)
New-NetFirewallRule -DisplayName "SMB via Tailscale" -Direction Inbound -Protocol TCP -LocalPort 445 -RemoteAddress 100.64.0.0/10 -Action Allow
```

### Copy from the laptop

VMs fully shut down (not suspended) and OneDrive syncing paused first.

```powershell
net use \\100.x.x.x\VMs /user:MicrosoftAccount\<email> *
robocopy "$env:OneDrive\Documents\Virtual Machines" \\100.x.x.x\VMs /E /Z /MT:8 /R:2 /W:5
```

`/Z` makes the copy restartable, `/MT:8` copies with 8 threads.

### Troubleshooting the copy

| Error | Cause | Fix |
|---|---|---|
| `ERROR 2` accessing source | Documents was redirected to OneDrive | Use `$env:OneDrive\Documents\...` |
| `ERROR 53` network path not found | Used a hostname that didn't resolve | Use the Tailscale IP |
| `ERROR 1326` user name or password incorrect | Laptop sent its own credentials | `net use` with explicit credentials first |

### Opening the VMs

VMware > File > Open > `.vmx`, and choose **I Moved It** so the VMs keep their MAC addresses and network config (important for a lab with a domain controller).
