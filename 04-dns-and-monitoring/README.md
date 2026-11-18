# 04: DNS filtering and monitoring

## Goal

Two services on the relay (part 2), both in Docker:

- **AdGuard Home:** block ads, trackers and telemetry with DNS, on all my devices and from anywhere
- **Uptime Kuma:** monitor my services and get alerts on my phone

Why Uptime Kuma and not Zabbix: Zabbix is much heavier and better suited for a real hypervisor later. Uptime Kuma runs fine on an old laptop and is quick to set up.

## 1. Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# log out and back in
docker run hello-world
```

## 2. Free port 53

Ubuntu's `systemd-resolved` listens on port 53 by default, which blocks AdGuard. Disable its stub listener:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
printf "[Resolve]\nDNSStubListener=no\n" | sudo tee /etc/systemd/resolved.conf.d/adguard.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

## 3. Compose

[`docker-compose.yml`](docker-compose.yml) in `~/homelab` on the relay:

```bash
cd ~/homelab
docker compose up -d
```

Design choices:

- **AdGuard uses host networking**, so it can listen on port 53 on every interface (LAN, Tailscale) and the query log shows the real client IPs instead of a Docker address.
- **Uptime Kuma is pinned to `:2`**, because `:latest` still points to the old 1.x branch.

## 4. AdGuard Home

Setup wizard on `http://<relay>:3000`:

| Setting | Value |
|---|---|
| Admin web interface | all interfaces, port **8080** (keeps port 80 free for a reverse proxy later) |
| DNS server | all interfaces, port **53** |

After setup the dashboard is on `http://<relay>:8080`.

**Upstream DNS** (Settings > DNS settings), encrypted with DNS-over-HTTPS and queried in parallel:

```
https://dns.quad9.net/dns-query
https://cloudflare-dns.com/dns-query
```

## 5. Using AdGuard on all my devices via Tailscale

I did **not** change the DNS on my home router. If the relay goes down, nobody at home would have working DNS.

Instead, Tailscale pushes AdGuard as DNS server to my own devices only:

Tailscale admin console > **DNS** > Nameservers > *Add nameserver* > Custom > the relay's Tailscale IP, with **Override DNS servers** on.

Result: my desktop, laptop and iPhone use AdGuard **everywhere**, also at school and on mobile data. MagicDNS keeps working.

Fallback: if the relay is offline, turn the override off again in the admin console.

### Test

Right after enabling it, from my laptop at school:

- 82 DNS queries in the AdGuard log, 78 of them from the laptop's Tailscale IP
- 22 blocked (27%), mostly Microsoft and NVIDIA telemetry domains

If a site or app breaks: check the **Query log** in AdGuard and unblock the domain.

## 6. Uptime Kuma

`http://<relay>:3001`, database: **SQLite** (lightest option for this hardware).

| Monitor | Type | Target | Why |
|---|---|---|---|
| Internet | HTTP | `https://www.google.com` | Is the relay online at all |
| AdGuard web UI | HTTP | `http://<relay Tailscale IP>:8080` | Is the AdGuard container up |
| AdGuard DNS | DNS | resolve `google.com` via the relay's Tailscale IP | Does DNS resolving actually work |
| Desktop RDP | TCP port | desktop's Tailscale IP, port `3389` | Is the desktop on and reachable |

Notes:

- Uptime Kuma runs in a bridge network, so `localhost` inside the container is the container itself, not the relay. That's why the AdGuard monitors use the relay's Tailscale IP.
- The desktop is monitored with a **TCP port check**, not ping, because Windows blocks ICMP by default. It shows as down whenever the desktop is off, which is expected.

### Phone alerts with ntfy

- Uptime Kuma > Settings > Notifications > **ntfy**, server `https://ntfy.sh`, a long random topic name (anyone who knows the topic can read it)
- *Default enabled* and *Apply on all existing monitors* on
- iPhone: the free **ntfy** app, subscribed to the same topic
