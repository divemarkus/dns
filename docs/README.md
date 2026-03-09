
# DNS Stack v1
Pi-hole + Unbound Docker stack. Below are deployment strategies

## Table of Contents
1. [Pi-hole deployment](#pi-hole-deployment)
2. [Conclusions](#conclusions)

Here's a `docker-compose.yml` for a **Pi-hole + Unbound** setup with:

### ✅ IPVLAN:  
**Yes — `ipvlan` (specifically in `l3s` mode) is generally *more stable* than macvlan for Pi-hole on modern kernels (5.x+), especially when the container must coexist with the host’s networking stack.**  
It avoids two known pain points of macvlan:

| Issue | Macvlan | `ipvlan l3s` |
|-------|---------|--------------|
| **ARP Conflicts / Neighbor Table Pollution** | ❌ Yes — each container claims a MAC, confuses switch/host ARP cache → *flaky connectivity* on some NICs/drivers | ✅ No — uses host’s MAC; no duplicate L2 identity |
| **Host ↔ Container Communication** | ❌ Impossible without extra routing (host can’t reach `.50` directly) | ✅ Yes — `l3s` mode allows host↔container via same subnet! |
| **Router Compatibility** | Requires router to accept unknown MACs (some enterprise switches drop them) | Works seamlessly with existing ARP/DHCP |

> 🔍 Confirmed in multiple Docker community reports & Ubuntu bug trackers (e.g., [Docker #39647](https://github.com/moby/moby/issues/39647), [Ubuntu #1980212](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1980212)).  
> Kernel 5.15+ (Ubuntu 22.04 LTS default) has mature `ipvlan l3s` support.

---

## ✅ Optimized Plan: **Pi-hole with `ipvlan l3s`** on Ubuntu 22.04

We’ll use:
- **Driver**: `ipvlan`
- **Mode**: `l3s` — allows host/container to share same subnet *without* needing a router for intra-subnet routing
- **IPAM**: Static IP reservation (like before), but now the *host can also reach it* 🎯

---

### 1️⃣ Confirm Your Network Stack

```bash
# Find your primary interface and subnet
ip -4 route | grep default
# Example: default via 192.168.10.1 dev enp3s0 src 192.168.10.30 metric 100

ip addr show enp3s0
# Look for: inet 192.168.10.30/24 scope global ...
```

Assume:
- Interface: `enp3s0`
- Subnet: `192.168.10.0/24`
- Gateway: `192.168.10.1`
- **Static IP for Pi-hole**: `.50` (outside DHCP range!)

> ⚠️ Ensure `.50` isn’t in your router’s DHCP pool.

---

### 2️⃣ Create the `ipvlan l3s` Network

```bash
# Remove old macvlan net if exists
sudo docker network rm pi-hole-net 2>/dev/null || true

# Create ipvlan network in l3s mode (host-aware)
sudo docker network create -d ipvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=enp3s0 \
  -o ipvlan_mode=l3s \
  pi-hole-l3s
```

> ✅ Key: `-o ipvlan_mode=l3s`  
> This gives us:
> - Pi-hole gets IP `192.168.10.50`
> - Host (`10.30`) can reach `.50` directly (no NAT, no routing table hacks)
> - Clients on LAN see it as a normal host

> 📝 Why not `l2`? Because in `l2` mode the container *shares* the master interface’s IP — impossible for Pi-hole. `l3s` is the sweet spot.

---

### 3️⃣ UPi-hole + Unbound  `docker-compose.yml`
- Unfortunately my compose file is legacy macvlan. 
- From macvlan to ipvlan is future upgrade plan.

```yaml
version: "3.8"

services:

  pihole-v6-unbound:
    container_name: pihole-v6-unbound
    image: mpgirro/docker-pihole-unbound
    hostname: piholev6unbound
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    networks:
      br0:
        ipv4_address: 192.168.0.2

    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "53:53/tcp"
      - "53:53/udp"
      - "5335:5335/tcp"
      - "67:67/udp"
      - "123:123/udp"

    environment:
      TZ: "168"
      FTLCONF_webserver_api_password: "xxxxxxxxxxxxxxxx"
      FTLCONF_webserver_interface_theme: "default-auto"
      FTLCONF_dns_upstreams: "127.0.0.1#5335"
      FTLCONF_dns_dnssec: "true"
      FTLCONF_dns_listeningMode: "single"
      FTLCONF_misc_etc_dnsmasq_d: "false"
      FTLCONF_dns_revServers: "false"

    volumes:
      - /mnt/user/appdata/pihole-v6-unbound:/config
      - /mnt/user/appdata/pihole-unbound/etc_pihole-unbound:/etc/pihole
      - /mnt/user/appdata/pihole-unbound/etc_pihole_dnsmasq-unbound:/etc/dnsmasq.d

  pihole:
    container_name: pihole
    image: diginc/pi-hole:alpine
    restart: unless-stopped
    cap_add:
      - NET_ADMIN

    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "63880:80/tcp"

    environment:
      ServerIP: 192.168.0.13
      DNS1: 8.8.8.8
      DNS2: 8.8.4.4
      IPv6: "False"

    volumes:
      - /mnt/user/appdata/pihole/pihole.log:/var/log/pihole.log
      - /mnt/user/appdata/pihole/blacklist.txt:/etc/pihole/blacklist.txt
      - /mnt/user/appdata/pihole/whitelist.txt:/etc/pihole/whitelist.txt


networks:
  br0:
    driver: macvlan
    driver_opts:
      parent: br0
    ipam:
      config:
        - subnet: 192.168.0.0/24

```

### 3️⃣ UUpdated docker-compose `docker-compose.yml`

---

> ✅ **Critical additions**:
> - `ports:` — since host can reach `.50`, we *still* bind ports to `0.0.0.0` (not localhost) for host access  
> - `DNSMASQ_LISTENING: "all"` — ensures it binds on the interface IP
> - `UPSTREAM_DNS: remove any reference to public free DNS resolvers like google and cloudflare

---

### 4️⃣ Start & Verify (Host-Integrated!)

```bash
cd pi-hole
docker-compose up -d

# Check container is UP and reachable FROM HOST
curl http://192.168.10.50/admin
# Should return Pi-hole web UI HTML

dig @192.168.10.50 google.com +short
```

✅ Now verify **host ↔ Pi-hole** works:
```bash
# From host terminal:
ping 192.168.10.50   # Should succeed!
telnet 192.168.10.50 80  # Should open HTTP connection
```

> 💡 This is the *key advantage* over macvlan: **no extra routing**, no `iptables` hacks.

---

### 5️⃣ Configure Router & Clients

Same as before — but now放心 your router won’t get confused:

| Device | Action |
|--------|--------|
| **Router DHCP** | Set DNS server = `192.168.10.50` |
| **Ubuntu host (you)** | Set primary DNS = `192.168.10.50` in Netplan/Wi-Fi settings |

#### Example: Ubuntu host DNS config (`/etc/netplan/01-netcfg.yaml`)
```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: true
      nameservers:
        addresses: [192.168.10.50, 1.1.1.1]  # ← Pi-hole FIRST!
```
Then apply:  
```bash
sudo netplan apply && sudo systemctl restart systemd-resolved
```

Test host DNS resolution:
```bash
systemd-resolve --status | grep "DNS Servers" -A2
# Should show: 192.168.10.50
```

---

### 🧪 Troubleshooting for `ipvlan l3s` on Ubuntu

| Symptom | Diagnosis & Fix |
|---------|-----------------|
| ❌ Host can’t reach Pi-hole at `.50` | 1. Confirm `ip route` has a direct route to `.50/32`<br>2. Check: `sudo ip link show dev enp3s0` — ensure interface is UP<br>3. Temporarily disable firewall: `sudo ufw disable`, test again |
| ⚠️ Container starts but FTL fails | Increase `FTLCONF_MAX_LOG_SIZE=10` in env vars (common OOM issue)<br>Add to compose: `MEMLOCK: "no"` (if kernel enforces) |
| 📡 DNS queries not reaching Pi-hole from LAN clients | 1. Ensure router’s DHCP option #6 = `.50`<br>2. Run on client: `tcpdump -i eth0 port 53 and host pihole_ip` to see if queries arrive |

---

### Bonus: Keep It Clean (Automated Health Check)

Add this to your compose (optional but usable!):
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/admin/api.php?status"]
  interval: 1m
  timeout: 5s
  retries: 3
```

---

You now have a **stable, host-aware Pi-hole** that won’t break when your switch misbehaves or kernel updates. We’ll add the dashboard UI in the next iteration — but this foundation ensures reliability from day one.

Let me know if you want:
- IPv6 support added (trivial with ipvlan)
- DNS-over-TLS (DoT) config via Pi-hole
- A script to auto-detect your network interface/subnet

I’ll tailor the config! 🛠️

- Add **Unbound root hints** update script  
- Enable **DNS-over-TLS (DoT)** in Unbound (requires extra config in `/etc/unbound/conf.d/`)  
- Block lists via `adlists.list` volume mount  

Happy blocking & resolving! 🌐🔒

