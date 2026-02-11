# DNS
DNS or Domain Name System is the system that automatically translates human-readable domain names (the name) into machine-readable IP addresses (the phone number) so your browser knows which server to connect to.


#### DNS over TLS (DoT) is a security protocol that addresses the privacy flaw of traditional DNS by adding encryption at the transport layer

#### DNS over HTTPS (DoH) is another security protocol that also encrypts DNS queries, but it does so by wrapping them inside a standard HTTPS web request


### Key differences

```
# Transport Protocol
DoT - DNS encapsulated directly in TLS
DoH - DNS encapsulated in HTTPS

# Port Used
DoT - 853
DoH - 443

# Network Visibility
DoT - DNS traffic is visible (as port 853 is used), but the content is encrypted
DoH - DNS traffic is hidden within regular HTTPS traffic

# Ease of Blocking
DoT - Just block port 853
DoH - Difficult to block without blocking all web traffic

# Use case
DoT - Preferred for network-level control and monitoring while maintaining user privacy
DoH - Preferred for maximum user privacy and bypassing network filtering/censorship
```


### Third-party recursive public DNS
These are the entity that will provide you with free recursive DNS service. The likes of Google, Cloudflare, Cisco, and of course your ISP. It is always advisable not to use your ISP's DNS, as they already have access to you network. Also, in my experience ISP's DNS are not as good as the ones I listed above. As of today's date, there's limited number of ISP's providing encrypted DNS service.

Cisco acquired OpenDNS, but still provides free home account. [Sign-up here](https://www.opendns.com/) - [or here](https://www.opendns.com/home-internet-security/)
- With free account you have access what you block: country code domain ending with 'xx' or the likes (xx is any country of your choice)
- Domain blocking and other features

### Public recursive DNS hosts
Here are the corresponding hosts and its DoT/DoH equivalents:
- Cisco: "208.67.222.222"; DoTHost = "dns.umbrella.com"; DoHHost = "https://dns.umbrella.com/dns-query"
- Cisco: "208.67.220.220"; DoTHost = "dns.umbrella.com"; DoHHost = "https://dns.umbrella.com/dns-query"
- Google: "8.8.8.8"; DoTHost = "dns.google"; DoHHost = "https://dns.google/dns-query"
- Google: "8.8.4.4"; DoTHost = "dns.google"; DoHHost = "https://dns.google/dns-query"
- Cloudflare: "1.1.1.1"; DoTHost = "one.one.one.one"; DoHHost = "https://cloudflare-dns.com/dns-query"
- Quad9: "9.9.9.9"; DoTHost = "dns.quad9.net"; DoHHost = "https://dns.quad9.net/dns-query"

### Alternative - Self-hosted
Use Pi‑hole as your primary DNS if you want:
- Privacy
- Control
- Ad blocking
- Local overrides
- LAN‑first resolution
- IoT lockdown
- Homelab flexibility

### Diagram - Pi-hole Full Recursive Lookup
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

### Key Characteristics - Self-hosted
- No third-party DNS sees your queries
- No logging outside your LAN
- Local caching = very fast repeat lookups
- Full control over filtering
- LAN hostnames resolved locally
- Cannot be bypassed if DoT/DoH is blocked
- This is the most private and most deterministic DNS architecture

### Disadvantage - Self-hosted
- DHCP handout for DNS is only the pi-hole server
- If you require failback, you must use two pi-hole or enterprise-grade firewall

### About...
[Check it out...](https://en.wikipedia.org/wiki/DNS_over_TLS/)


