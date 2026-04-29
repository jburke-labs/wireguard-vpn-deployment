# WireGuard VPN, Double NAT Deployment and Troubleshooting

**Project Cerberus Homelab · Secure Remote Access Through a Double-NAT Environment**

---

## Overview

This project covers the full deployment and end-to-end validation of a WireGuard VPN server hosted inside a Proxmox LXC container, placed behind a FortiGate firewall, operating in a double-NAT arrangement with a Virgin Media upstream router holding the public IP.

The objective was not to get a tunnel that looked configured. It was to validate the full connection path from a mobile device on 5G through to the internal lab network, understand exactly where each problem was occurring, and resolve each issue methodically until a successful handshake and live traffic flow were confirmed.

This became a genuinely useful exercise in Linux networking, firewall policy design, NAT, packet tracing, and structured fault isolation across multiple independent layers.

---

## Environment

| Component | Detail |
|---|---|
| WireGuard Host | Proxmox LXC on DEKU |
| LXC LAN IP | 10.0.20.5 |
| WireGuard Port | UDP 51820 |
| Tunnel Subnet | 10.0.40.0/24 |
| Server Tunnel IP | 10.0.40.1 |
| Client Tunnel IP | 10.0.40.2/32 |
| Firewall | FortiGate (WAN behind Virgin Media router) |
| Upstream Router | Virgin Media (holds public IP) |
| Client | Mobile phone running WireGuard app over 5G |
| Dashboard | WGDashboard |

---

## Network Path

The working design required traffic to traverse two separate NAT boundaries before reaching the WireGuard server:

```
Phone (5G)
  → Public Internet
    → Virgin Media Router (public IP, port forward to FortiGate WAN)
      → FortiGate (VIP + inbound firewall policy, forward to WireGuard LXC)
        → WireGuard LXC (10.0.20.5:51820)
          → Internal lab subnet (10.0.20.0/24)
```

This double-NAT arrangement is what made the project more complex than a standard single-firewall VPN deployment. Configuring only the FortiGate was never going to be enough ,inbound traffic had to be forwarded correctly at both layers before the tunnel could ever be established.

---

## What Was Built

### WireGuard Server Configuration

The WireGuard server was deployed inside a Proxmox LXC using the WGDashboard management interface. The clean final configuration included:

- Server tunnel address: `10.0.40.1/24`
- Listening port: UDP 51820
- IP forwarding enabled and persisted via `sysctl.d`
- PostUp and PostDown rules for forwarding and NAT masquerade via `iptables`
- Client peer assigned `10.0.40.2/32`

**wg0.conf (clean final state):**

```ini
[Interface]
Address = 10.0.40.1/24
ListenPort = 51820
PrivateKey = <server-private-key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.0.40.2/32
```

### Client Peer Configuration

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.0.40.2/32
DNS = 10.0.20.53

[Peer]
PublicKey = <server-public-key>
Endpoint = <public-ip>:51820
AllowedIPs = 10.0.20.0/24, 10.0.40.0/24
PersistentKeepalive = 25
```

The `AllowedIPs` design is a split-tunnel ,only traffic destined for the internal lab subnets is routed through the VPN. General internet traffic from the phone stays on the mobile connection.

### FortiGate Configuration

A Virtual IP (VIP) was created to map the FortiGate WAN interface on UDP 51820 to the WireGuard LXC at `10.0.20.5:51820`. An inbound firewall policy allowed WAN to internal traffic using this VIP, with the service tightened specifically to UDP 51820.

### Virgin Media Router

A port forward rule on the Virgin Media router directed inbound UDP 51820 from the public internet to the FortiGate WAN IP. Without this first-stage forward in place, the traffic would never reach the FortiGate at all.

---

## Troubleshooting ,What Broke and How It Was Fixed

The troubleshooting process covered six distinct issue areas, each resolved independently before the next was uncovered.

---

### Issue 1 ,IP Forwarding Disabled

**Finding:**

```bash
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 0
```

IP forwarding was disabled on the WireGuard LXC. With this in place, the server would accept a tunnel connection but would not route traffic onward into the rest of the internal network.

**Fix:**

Enabled immediately as a live test, then persisted via a custom file in `/etc/sysctl.d/`:

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-wireguard.conf
sysctl -p /etc/sysctl.d/99-wireguard.conf
```

**Key lesson:**

> A VPN server that cannot forward packets is an expensive dead end. Verify IP forwarding before anything else.

---

### Issue 2 ,Missing NAT and Forwarding Rules

**Finding:**

The initial `wg0.conf` had no `PostUp` or `PostDown` entries. WireGuard was configured to accept the tunnel connection but had no instructions for what to do with traffic arriving from the VPN client.

Without these rules, the tunnel might have handshaked eventually, but the client would not have been able to reach anything inside the lab.

**Fix:**

Added `iptables` forwarding and NAT masquerade rules via `PostUp` and `PostDown` in the WireGuard config (see configuration above).

**Key lesson:**

> IP forwarding allows the kernel to route packets between interfaces. NAT masquerade rewrites the source address so return traffic knows where to come back to. Both are required. One without the other is not enough.

---

### Issue 3 ,Malformed Configuration File

**Finding:**

The `wg0.conf` contained incomplete entries with blank values:

```ini
SaveConfig =
PreUp =
PreDown =
```

WireGuard failed to parse this and returned an error indicating a value was *"neither true nor false"* ,pointing directly at the empty `SaveConfig` line.

```
wg0: error: SaveConfig: neither true nor false
```

**Fix:**

Removed all empty or incomplete fields from the config. WireGuard does not tolerate blank values for known options ,the field should either have a valid value or not be present at all.

**Key lesson:**

> WireGuard config parsing is unforgiving. A blank value for a known option is an error, not a default. When something is not needed, remove the line entirely rather than leaving it empty.

---

### Issue 4 ,Interface Not Actually Up

**Finding:**

At one point the system returned:

```bash
ip addr show wg0
# Device "wg0" does not exist.
```

The config file existed on disk and looked correct, but the WireGuard interface had never been successfully instantiated. WGDashboard was also showing the interface as off.

This was an important distinction ,not "no handshake" but "no interface at all."

**Fix:**

Once the malformed config was corrected, the interface was brought up manually and validated:

```bash
sudo wg-quick up wg0
sudo wg show
ip addr show wg0
sudo ss -lunp | grep 51820
```

All four checks confirmed the interface was up, had the correct address, and was actively listening on UDP 51820.

**Key lesson:**

> A config file existing is not evidence that a service is running. Always verify the actual interface state independently of the config.

---

### Issue 5 ,Double NAT Not Accounted For

**Finding:**

Checking the FortiGate WAN IP revealed it was in the `192.168.x.x` range ,a private address. The FortiGate was not internet-facing directly. It was sitting behind the Virgin Media router, which held the actual home public IP.

This meant two separate port forwards were required:

1. Virgin Media router → FortiGate WAN IP (first NAT boundary)
2. FortiGate VIP → WireGuard LXC (second NAT boundary)

**Verification using FortiGate packet capture:**

Stage 1 ,confirmed traffic arriving on the FortiGate WAN from the phone's public IP:

```bash
# FortiGate sniffer ,confirmed inbound UDP 51820 traffic visible on WAN interface
diagnose sniffer packet <wan-interface> "udp port 51820" 4
```

Stage 2 ,confirmed the FortiGate was forwarding traffic toward the WireGuard LXC:

```bash
# Traffic seen leaving FortiGate toward 10.0.20.5:51820
```

Stage 3 ,confirmed traffic arriving on the WireGuard LXC itself:

```bash
sudo tcpdump -i eth0 udp port 51820
```

At this point the full inbound path had been proven: phone → Virgin router → FortiGate → WireGuard LXC.

**Key lesson:**

> Double NAT changes remote access troubleshooting completely. It is not enough to configure only the downstream firewall. Every NAT boundary in the path must be accounted for and verified independently.

---

### Issue 6 ,Client Endpoint Set to Internal IP

**Finding:**

After every layer of the inbound path had been proven, there was still no successful handshake. The interface was up. The peer was configured. Traffic was reaching the LXC. The peer config was clean.

The final issue was in the mobile client configuration. The WireGuard tunnel endpoint on the phone had been set to an internal IP address:

```
Endpoint = 10.0.x.x:51820   ← incorrect
```

From a phone on 5G outside the home network, `10.0.x.x` is unreachable. The phone must connect to the home public IP, which is then forwarded inward through both NAT boundaries.

**Fix:**

Corrected the endpoint to the home public IP:

```
Endpoint = <public-ip>:51820   ← correct
```

**Result:**

Immediately after this change, the tunnel established a successful WireGuard handshake and traffic flow was confirmed to the internal lab subnet.

**Key lesson:**

> WireGuard is simple in design but unforgiving in practice. A single incorrect value on the client side is enough to prevent the tunnel from establishing, even when everything on the server side is perfectly correct. Validate the client config last, after the server path has been proven end to end.

---

## Final Working State

| Check | Result |
|---|---|
| WireGuard interface up | ✅ |
| Peer configured and visible | ✅ |
| UDP 51820 listening | ✅ |
| Virgin router forwarding | ✅ |
| FortiGate VIP and policy | ✅ |
| Traffic reaching LXC | ✅ |
| Handshake established | ✅ |
| Internal lab subnet reachable | ✅ |

---

## Troubleshooting Sequence Summary

| Stage | Issue | Resolution |
|---|---|---|
| 1 | IP forwarding disabled | Enabled and persisted via sysctl.d |
| 2 | Missing NAT / forwarding rules | Added PostUp/PostDown iptables rules |
| 3 | Malformed config file | Removed empty / invalid fields |
| 4 | Interface not instantiated | Fixed config, brought interface up manually |
| 5 | Double NAT not fully accounted for | Verified Virgin router forward, confirmed with FortiGate packet capture and tcpdump |
| 6 | Client endpoint set to internal IP | Corrected to home public IP |

---

## Skills Demonstrated

- WireGuard server deployment in a Proxmox LXC
- Linux networking ,IP forwarding, iptables, sysctl persistence, interface state
- FortiGate VIP design and inbound firewall policy configuration
- Double-NAT troubleshooting methodology
- Packet capture and traffic flow validation using FortiGate sniffer and tcpdump
- WGDashboard configuration and interface state management
- Structured fault isolation ,each layer proven independently before moving to the next
- Split-tunnel VPN client design
- Secure remote access design and end-to-end validation

---

## Full Documentation

The complete project writeup including extended troubleshooting notes and full environment context is available in the [Project Cerberus Notion workspace](https://www.notion.so/2b4fe4fa7fd38083935bfe52e0a8b899).

---

## Related Projects

- [Project Cerberus ,Infrastructure Overview](https://jburke-labs.github.io/project-cerberus)
- [NGINX Proxy Manager + AdGuard + Wildcard Certificate](https://github.com/jburke-labs/nginx-proxy-wildcard-cert)
- [Snipe-IT Replatform ,Docker Host Migration](https://github.com/jburke-labs/snipe-it-replatform)
- [Wazuh SOC Automation Pipeline](https://github.com/jburke-labs/wazuh-soc-automation)
