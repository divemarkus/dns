
# DNS Configuration Guide

## Table of Contents
1. [Introduction](#introduction)
2. [DNS Over TLS (DoT) & DNS Over HTTPS (DoH)](#dns-over-tls-dot--dns-over-https-doh)
3. [Third-Party Recursive Public DNS Services](#third-party-recursive-public-dns-services)
4. [Self-Hosted DNS with Pi-hole](#self-hosted-dns-with-pi-hole)
5. [Failback Design](#failback-design)
6. [Additional Considerations](#additional-considerations)
7. [Resources & Documentation](#resources--documentation)

## Introduction

The Domain Name System (DNS) is a critical component of the internet, translating human-readable domain names into machine-readable IP addresses. This guide provides detailed instructions on setting up DNS for enhanced privacy and security.

## DNS Over TLS (DoT) & DNS Over HTTPS (DoH)

### Overview
- **DNS over TLS (DoT)**: Encrypts DNS queries using TLS, providing privacy at the transport layer.
  - Uses port 853.
  - Traffic is visible but encrypted.
  - Easier to block by port.

- **DNS over HTTPS (DoH)**: Encapsulates DNS queries within HTTPS requests.
  - Shares the same port as web traffic (port 443).
  - Hides DNS traffic within regular HTTPS traffic.
  - Harder to block without restricting all web traffic.

### Use Cases
- **DoT**: Ideal for networks requiring monitoring while maintaining user privacy.
- **DoH**: Preferred for maximum privacy and circumventing censorship.

## Third-Party Recursive Public DNS Services

### Recommended Providers
1. **Google DNS**
   - Primary: `8.8.8.8`
   - Secondary: `8.8.4.4`

2. **Cloudflare DNS**
   - Primary: `1.1.1.1`
   - Secondary: `1.0.0.1`

3. **OpenDNS (Cisco)**
   - Primary: `208.67.222.222`
   - Secondary: `208.67.220.220`

4. **Quad9**
   - Primary: `9.9.9.9`

### DoT and DoH Hostnames
| Provider       | DoT Host                     | DoH Host                           |
|----------------|------------------------------|------------------------------------|
| Google         | `dns.google`                 | `https://dns.google/dns-query`    |
| Cloudflare     | `one.one.one.one`            | `https://cloudflare-dns.com/dns-query` |
| OpenDNS (Cisco)| `dns.quad9.net`              | `https://dns.quad9.net/dns-query`  |
| Quad9          | `dns.quad9.net`              | `https://dns.quad9.net/dns-query`  |

## Self-Hosted DNS with Pi-hole

### Benefits
- **Privacy**: No third-party DNS sees your queries.
- **Control**: Full control over filtering and caching.
- **Ad Blocking**: Blocks ads and trackers.
- **Local Cache**: Fast repeat lookups.
- **IoT Lockdown**: Define and group client.

### Diagram - Pi-hole with Unbound Full Recursive Lookup
```
Client Device
    │
    ▼
Pi-hole (DNS filter, blocklists, local cache)
    │
    ▼
Unbound (local recursive resolver)
    │
    ├── Query Root Servers (.)
    │
    ├── Query TLD Servers (.com, .net, etc.)
    │
    ├── Query Authoritative Server (example.com)
    │
    ▼
Receives Final Answer
    │
    ▼
Pi-hole caches it
    │
    ▼
Client receives DNS response
```

### Key Characteristics - Self-hosted
- No third-party DNS sees your queries
- From your local network, you query straight to root servers, not Cisco, Google, or worst your ISP
- No logging outside your LAN
- Local caching = very fast repeat lookups
- Full control over filtering
- LAN hostnames resolved locally
- Cannot be bypassed if DoT/DoH is blocked
- This is the most private and most deterministic DNS architecture

### Disadvantage - Self-hosted
- If you require failback, you must use two pi-hole or enterprise-grade firewall
- DO NOT DO THIS: primary dns "pi-hole", secondary dns "google" = nope!

## Failback Design

### Design with failback - Self-hosted + any enterprise grade Firewall (FW)
- Comparison with local FW vs third-party recursive DNS servers -- FW is inside your LAN — OpenDNS or Google is outside
- Path using FW as secondary DNS (below):
```
Client → Pi-hole → (if Pi-hole down) → FW → Upstream DNS
```
- FW configured to use OpenDNS as upstream or its own full resursive like 'unbound'
- FW will only answer if pi-hole is unresponsive
- Path using pi-hole + OpenDNS or Google or ISP or Cloudflare (below):
```
Client → Pi-hole → (blocked) → OpenDNS → Bypass
```
- Local Firewall enforces firewall rules — OpenDNS, Google, ISP and the likes cannot

## Additional Considerations

### Monitoring & Maintenance
- **Logs**: Monitor Pi-hole logs for unusual activity to ensure network security.
- **Updates**: Keep Pi-hole and related software updated to benefit from the latest features and security patches.

### IoT Devices
- You can lock-down some of your IoT's if they only require very specific DNS queries

## Resources & Documentation

- **Pi-hole Official Docs**: [https://pi-hole.net/documentation/](https://pi-hole.net/documentation/)
- **OpenDNS Consumer**: [https://www.opendns.com/home-internet-security/](https://www.opendns.com/home-internet-security/)
- **Unbound DNS Documentation**: [https://unbounddns.github.io/unbound/](https://unbounddns.github.io/unbound/)

## Conclusion

Setting up a secure and private DNS infrastructure enhances your network's security and privacy. Whether you choose third-party services like Google or Cloudflare, or opt for self-hosted solutions like Pi-hole, the guide above provides comprehensive instructions to get you started.

