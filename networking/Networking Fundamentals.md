# Networking Fundamentals

A reference handbook covering networking from the link layer up through application protocols.

---

## Table of Contents

1. The Layered Model — Overview
2. Layer 1 — Physical
3. Layer 2 — Data Link
4. Layer 3 — Network (IP, ICMP, Routing)
5. Layer 4 — Transport (TCP, UDP, QUIC)
6. Layer 5/6/7 — Session, Presentation, Application
7. Network Services (DHCP, NAT, DNS)
8. Network Security
9. System Design Networking
10. Debugging & Tools

---

## 1. The Layered Model — Overview

Networking is organized into layers, each solving one piece of the communication problem. Each layer only communicates with its peer on the other side — lower layers are just the delivery mechanism.

### OSI Model (7 layers)

```
 Layer   Name            Responsibility                        Protocols            PDU
 -----   --------------  ------------------------------------  ------------------   --------
   7     Application     App-level semantics                   HTTP, DNS, SSH       Message
   6     Presentation    Data formatting, encryption           TLS, SSL, JPEG
   5     Session         Connection management                 Sockets, sessions
   4     Transport       End-to-end delivery between processes TCP, UDP, QUIC       Segment
   3     Network         Host-to-host delivery across networks IP, ICMP             Packet
   2     Data Link       Hop-to-hop delivery on a single link  Ethernet, WiFi, ARP  Frame
   1     Physical        Bits on the wire (or air)             Cables, radio        Bits
```

### TCP/IP Model (what the Internet actually uses)

The TCP/IP model collapses the top three OSI layers into a single Application layer, and the bottom two into a Network Access layer. OSI provides the finer-grained shared vocabulary; TCP/IP describes how the stack is actually implemented.

```
  OSI                TCP/IP
  -----------        --------------
  Application  ----
  Presentation ----  Application
  Session      ----
  Transport    ----  Transport
  Network      ----  Internet
  Data Link    ----
  Physical     ----  Network Access
```

### Encapsulation and PDUs

As data moves down the stack, each layer wraps the payload from above with its own control information, forming a new **PDU (Protocol Data Unit)**. This process is called **encapsulation**. On the receiving side, each layer strips its header and passes the payload up — **decapsulation**.

#### PDU terminology

Each layer has its own name for the data unit it works with:

|Layer|PDU name|What gets added|
|---|---|---|
|Application|**Data / Message**|Application headers (e.g., HTTP headers)|
|Transport|**Segment** (TCP) / **Datagram** (UDP)|Transport header (ports, seq numbers, flags)|
|Network|**Packet**|IP header (source/destination IP, TTL)|
|Data Link|**Frame**|Frame header (MAC addresses) + trailer (CRC)|
|Physical|**Bits**|Encoded as electrical/optical/radio signals|

#### How wrapping works

The formal model uses three terms for what happens at each layer:

- **SDU (Service Data Unit):** the payload received from the layer above — what needs to be delivered.
- **PCI (Protocol Control Information):** the header (and sometimes trailer) added by the current layer — addresses, sequence numbers, error checks, flags.
- **PDU (Protocol Data Unit):** the complete unit at this layer = PCI + SDU.

The PDU of one layer becomes the SDU of the layer below:

```
  Layer N+1 produces a PDU
         |
         v
  Layer N receives it as an SDU
         |
         v  adds its own PCI (header / trailer)
         |
         v
  Layer N produces a new PDU  -->  becomes SDU for Layer N-1
```

The complete set of protocols used across all layers is called the **protocol stack**.

#### Encapsulation in action

An HTTP GET request traveling down the stack. Each layer treats everything above it as an opaque payload:

```
  Application    +------------------------------------------+
                 |            HTTP GET /api/data            |  <-- Data (L7 PDU)
                 +------------------------------------------+
                                    |
                         Transport receives this as its SDU
                         and adds a TCP header (PCI)
                                    |
                                    v
  Transport      +----------+-------------------------------+
                 | TCP HDR  |         HTTP payload          |  <-- Segment (L4 PDU)
                 | src:4921 |                               |
                 | dst:443  |                               |
                 +----------+-------------------------------+
                                    |
                         Network receives this as its SDU
                         and adds an IP header (PCI)
                                    |
                                    v
  Network        +----------+-------------------------------+
                 |  IP HDR  |        TCP segment            |  <-- Packet (L3 PDU)
                 | src:10.0 |                               |
                 | dst:93.x |                               |
                 +----------+-------------------------------+
                                    |
                         Data Link receives this as its SDU
                         and adds Ethernet header + CRC trailer (PCI)
                                    |
                                    v
  Data Link      +----------+-------------------------------+------+
                 | ETH HDR  |        IP packet              | CRC  | <-- Frame (L2 PDU)
                 | src:aa:bb|                               |      |
                 | dst:cc:dd|                               |      |
                 +----------+-------------------------------+------+
                                    |
                                    v
  Physical         101010001110010101001...                          <-- Bits (L1)
```

**Key points:**

- Each layer only reads its own header. A router (L3 device) strips the frame header, reads the IP header, makes a routing decision, and re-encapsulates in a new frame for the next hop.
- Headers are added at the front (**headers**), trailers at the end (**trailers**). The CRC (Cyclic Redundancy Check) in Ethernet is a trailer. Most layers only add headers.
- This is exactly what Wireshark displays — nested layers that can be expanded one by one.
- The total overhead of all headers is why a 1500-byte Ethernet MTU (Maximum Transmission Unit) carries only ~1460 bytes of TCP payload: 20 bytes consumed by IP, 20 by TCP.

### What happens when you type a URL

A complete end-to-end walkthrough that exercises most of the stack. The full sequence, layer by layer:

```
  Browser                                              Server
    |                                                    |
    |--- 1. DNS lookup: example.com -------------------> | DNS Server
    |<-- IP: 93.184.216.34 ----------------------------  |
    |                                                    |
    |--- 2. TCP SYN ---------------------------------->  |
    |<-- SYN-ACK --------------------------------------  |  1 RTT
    |--- ACK ----------------------------------------->  |
    |                                                    |
    |--- 3. TLS ClientHello -------------------------->  |
    |<-- ServerHello + Cert + Key ---------------------  |  1 RTT (TLS 1.3)
    |--- Finished ------------------------------------>  |
    |                                                    |
    |--- 4. HTTP GET / --------------------------------> |
    |<-- HTTP 200 OK + HTML ---------------------------  |
    |                                                    |
    |--- 5. Parse HTML, request CSS/JS/images -------->  |
    |<-- Assets ---------------------------------------  |
    |                                                    |
    |    6. Render page                                  |
```

> The walkthrough is naturally organized by layer — DNS resolution, TCP handshake, TLS handshake, HTTP request, response parsing — with ARP, routing, and frame-level delivery underpinning it at the link layer.

---

## 2. Layer 1 — Physical

The physical layer handles transmission of raw bits over a medium: copper (Ethernet cables), fiber optics, or radio (WiFi, cellular). Software engineers rarely interact with this layer directly, but a few aspects matter:

**What to know:**
- **Ethernet cable categories:** Cat5e (1 Gbps), Cat6 (10 Gbps short range), Cat6a (10 Gbps longer range). Datacenter interconnects use fiber.
- **WiFi standards:** 802.11n (WiFi 4), 802.11ac (WiFi 5), 802.11ax (WiFi 6). Higher numbers = more bandwidth, but wireless is always less reliable than wired.
- **Bandwidth vs. latency:** bandwidth is the pipe diameter (data per second), latency is the pipe length (time for one bit to arrive). High bandwidth does not mean low latency.
- **Full-duplex vs. half-duplex:** modern Ethernet is full-duplex (send and receive simultaneously). WiFi is inherently half-duplex (shared medium).

---

## 3. Layer 2 — Data Link

The data link layer handles delivery across a single link (one hop). It deals with MAC addresses, frames, error detection, and media access control.

### MAC (Media Access Control) Addresses

48-bit hardware identifiers burned into NICs (Network Interface Cards), written in hex: `aa:bb:cc:dd:ee:ff`. Globally unique per interface. Used only for **local delivery** within a LAN segment.

The burned-in address is fixed, but the address an interface *presents on the wire* can be overridden in software (MAC spoofing) — the OS simply puts a different value in the frame's source field.

**Key insight:** source and destination MAC addresses are **rewritten at every router hop**, while IP addresses stay the same end-to-end.

### Ethernet Frames

Ethernet is the dominant LAN technology. Modern Ethernet is point-to-point, full-duplex over switches — collisions are a thing of the past (CSMA/CD (Carrier Sense Multiple Access with Collision Detection) is only relevant for legacy shared-media segments).

```
  Ethernet Frame Structure

  +----------+-----+----------+----------+----------+--------------+------+
  | Preamble | SFD | Dst MAC  | Src MAC  | Type/Len |   Payload    | CRC  |
  |  7 bytes | 1B  |  6 bytes |  6 bytes |  2 bytes | 46-1500 B    | 4B   |
  +----------+-----+----------+----------+----------+--------------+------+
   (Start                                              ^
    Frame                                              |
    Delimiter)                              MTU: 1500 bytes (default)
```

**MTU (Maximum Transmission Unit):** Ethernet's default max payload is **1500 bytes**. This is where TCP's common MSS (Maximum Segment Size) of 1460 comes from: 1500 (MTU) - 20 (IP header) - 20 (TCP header). CRC (Cyclic Redundancy Check) is the error-detection trailer appended to each frame. Jumbo frames (9000B) exist in datacenter environments.

### ARP (Address Resolution Protocol)

Resolves IP addresses to MAC addresses within a LAN. Operates with broadcast requests and unicast replies.

```
  Host A (10.0.0.2)                                Host B (10.0.0.5)
    |                                                 |
    |-- ARP Request (broadcast) -------------------->>|
    |   "Who has 10.0.0.5? Tell 10.0.0.2"             |  All hosts on the
    |                                                 |  LAN receive this,
    |<<- ARP Reply (unicast) -------------------------|  only B replies
    |   "10.0.0.5 is at aa:bb:cc:dd:ee:ff"            |
    |                                                 |
    |  (A caches: 10.0.0.5 -> aa:bb:cc:dd:ee:ff)      |
```

For off-subnet destinations, the frame goes to the **default gateway's MAC**. The router then performs ARP on the next segment. This is why `arp -a` on the local machine shows the gateway's MAC but not the MAC of remote servers.

> **Security:** ARP has zero authentication. Any host can claim any IP, enabling man-in-the-middle attacks on the LAN (ARP spoofing). Mitigations: DAI (Dynamic ARP Inspection), 802.1X port authentication, static ARP entries.

### Switches

Switches operate at Layer 2: they learn MAC addresses by inspecting source addresses of incoming frames, build a forwarding table (`MAC -> port`), and send frames only to the correct port. Each port is a separate collision domain. Tables are built automatically — plug-and-play.

### VLANs (Virtual Local Area Networks)

A VLAN lets one physical switch act as multiple isolated L2 networks. Traffic between VLANs must pass through a router (or L3 switch). Essential for security segmentation and reducing broadcast domains.

```
             +--------------- Switch ---------------+
             |                                       |
        VLAN 10               VLAN 20              VLAN 30
      +-----------+       +-----------+        +-----------+
      | Employees |       |  Guests   |        |  Servers  |
      | 10.0.10.x |       | 10.0.20.x |        | 10.0.30.x |
      +-----------+       +-----------+        +-----------+
             |                                       |
             +----- traffic between VLANs -----------+
                   must go through a router
```

### STP (Spanning Tree Protocol)

When switches have redundant paths (for resilience), loops can form. STP prevents this by electing a root bridge and pruning redundant links. In production, **RSTP (Rapid Spanning Tree Protocol, 802.1w)** replaced classic STP for much faster convergence (~1-2 seconds vs. 30-50 seconds).

---

## 4. Layer 3 — Network (IP, ICMP, Routing)

The network layer handles host-to-host delivery across different networks. This is where IP addressing, routing, and fragmentation live.

### Unicast, Broadcast, and Multicast

Three fundamental modes of IP communication:

**Unicast** — one-to-one. A packet is sent from one source to one specific destination. This is the default for almost all traffic (HTTP requests, SSH sessions, database queries). Each packet has a single source IP and a single destination IP.

```
  Host A  ---------> Host B         One sender, one receiver
```

**Broadcast** — one-to-all. A packet is sent to every host on the local network. The destination address is the subnet's broadcast address (all host bits set to 1). For example, on the `192.168.1.0/24` network, the broadcast address is `192.168.1.255`.

```
  Host A  ---------> All hosts      One sender, all receivers on the subnet
                     on the LAN
```

Broadcasts are confined to the local subnet — routers do not forward them (this boundary is called the **broadcast domain**). Protocols that use broadcast: ARP, DHCP Discover/Request. Excessive broadcasts degrade network performance, which is one reason to use VLANs to keep broadcast domains small.

> **IPv6 has no broadcast.** Its replacement is **multicast** — for example the all-nodes group `ff02::1` does the job broadcast did in IPv4. (**Anycast** — one-to-nearest, where the same address is announced from many locations and routing picks the closest — also exists in IPv6, but it predates IPv6 and is a separate addressing mode, not the broadcast replacement.)

**Multicast** — one-to-many (subscribers only). A packet is sent to a **group address** and delivered only to hosts that have joined that group. Uses the IP range `224.0.0.0/4` (224.0.0.0 – 239.255.255.255) in IPv4.

```
  Host A  ---------> Group members  One sender, only subscribed receivers
                     (not everyone)
```

Hosts join/leave groups using IGMP (Internet Group Management Protocol). Multicast is efficient for streaming video to many viewers, service discovery (mDNS — multicast DNS — uses `224.0.0.251`), and cluster coordination. Multicast-capable routers and switches are required — on the public Internet it's rarely available, but within datacenters and enterprise LANs it's common.

**Anycast** — one-to-nearest. Multiple hosts share the same IP address, and the network routes packets to the **topologically closest** one based on routing metrics. Implemented via BGP (Border Gateway Protocol). Used by CDNs (Content Delivery Networks), DNS root servers, and services like Cloudflare (`1.1.1.1`) and Google DNS (`8.8.8.8`).

```
  Client  ---------> Nearest server   Multiple servers share one IP,
                     with same IP     routing picks the closest
```

|Mode|Destination|Receivers|Scope|Example|
|---|---|---|---|---|
|Unicast|Single host|1|Global|HTTP request|
|Broadcast|All hosts on subnet|All on LAN|Local subnet|ARP, DHCP|
|Multicast|Group address|Subscribed hosts|LAN or routed|Video streaming, mDNS|
|Anycast|Nearest of many|1 (closest)|Global (via BGP)|CDN, DNS root servers|

### IPv4

32-bit address, written in dotted decimal. Every address has a **network prefix** and a **host portion**, separated by the subnet mask.

```
  IP address:    192  .  168  .   1   .  10
  Binary:     11000000.10101000.00000001.00001010
              +------ network (24 bits) ------++host+

  Subnet mask:   255  .  255  .  255  .   0
              11111111.11111111.11111111.00000000

  CIDR notation: 192.168.1.10/24
```

### IPv6

128-bit addresses written in hex: `2001:db8::1`. Solves IPv4 address exhaustion, simplifies the header (no checksum, no router fragmentation). IPv6 now carries a large and growing share of Internet traffic; dual-stack (v4 + v6) is the standard transition strategy.

Key differences from IPv4: no broadcast (uses multicast instead), no NAT needed (enough addresses for every device), recommended (not mandatory) IPsec (IP Security) support — originally required, relaxed to SHOULD by RFC 6434, simplified header (fixed 40 bytes, no options field, no header checksum).

### CIDR (Classless Inter-Domain Routing) and Subnetting

CIDR replaced the old Class A/B/C system. Addresses are `IP/prefix_length`, allowing variable-length prefixes and route aggregation.

**Quick math:** usable hosts = 2^(32 - prefix) - 2. The minus 2 accounts for the network address (all-zeros host) and broadcast (all-ones host).

|CIDR|Mask|Total IPs|Usable hosts|Typical use|
|---|---|---|---|---|
|`/32`|255.255.255.255|1|Single host|Loopback, host route|
|`/30`|255.255.255.252|4|2|Router interconnect|
|`/28`|255.255.255.240|16|14|Small server subnet|
|`/26`|255.255.255.192|64|62|Department LAN|
|`/24`|255.255.255.0|256|254|Standard LAN|
|`/20`|255.255.240.0|4,096|4,094|VPC (Virtual Private Cloud) subnet (AWS default)|
|`/16`|255.255.0.0|65,536|65,534|Large VPC|
|`/8`|255.0.0.0|16.7M|16.7M|Private class A|

**VLSM (Variable Length Subnet Masking)** allows different prefix lengths within the same allocation — standard practice:
```
  10.0.0.0/16 (entire allocation)
  +-- 10.0.0.0/20   -- Production servers    (4094 hosts)
  +-- 10.0.16.0/20  -- Staging               (4094 hosts)
  +-- 10.0.32.0/24  -- Management / bastion   (254 hosts)
  +-- 10.0.33.0/28  -- Load balancers          (14 hosts)
  +-- 10.0.34.0/28  -- Database tier           (14 hosts)
```

### Private Addresses (RFC 1918)

|Range|CIDR|Common use|
|---|---|---|
|10.0.0.0 – 10.255.255.255|`10.0.0.0/8`|Cloud VPCs, enterprise|
|172.16.0.0 – 172.31.255.255|`172.16.0.0/12`|Docker default|
|192.168.0.0 – 192.168.255.255|`192.168.0.0/16`|Home routers|

**Special:** `127.0.0.0/8` (loopback), `169.254.0.0/16` (link-local / DHCP failure), `0.0.0.0` (all interfaces).

### Subnets and Routing Decisions

A **subnet** is a logical division of IP space defined by prefix/mask. It's also a **broadcast domain** boundary—hosts within the same subnet communicate via L2 (ARP), different subnets need L3 routing.

**Routing decision algorithm (on every device):**
1. Destination & my IP → apply my subnet mask → same network prefix?
	- **Yes** → same subnet → ARP for destination MAC → send L2 frame directly
	- **No** → different subnet → ARP for default gateway MAC → send to router.
2. Router receives packet → checks routing table → longest prefix match → forwards to next hop. 

#### Routing scenarios

| My IP/mask     | Destination IP | Same subnet?           | Next hop         | ARP target   |
| -------------- | -------------- | ---------------------- | ---------------- | ------------ |
| 10.0.1.10/24   | 10.0.1.20      | ✓ Yes                  | Direct L2        | 10.0.1.20    |
| 10.0.1.10/24   | 10.0.2.50      | ✗ No                   | Gateway 10.0.1.1 | 10.0.1.1     |
| 10.0.1.10/24   | 8.8.8.8        | ✗ No                   | Gateway 10.0.1.1 | 10.0.1.1     |
| 192.168.1.5/26 | 192.168.1.10   | ✓ Yes (both in .0-.63) | Direct L2        | 192.168.1.10 |
| 192.168.1.5/26 | 192.168.1.70   | ✗ No (.70 in .64-.127) | Gateway          | Gateway MAC  |
#### Example: packet from 10.0.1.10/24 to 8.8.8.8

```
Host 10.0.1.10                Router 10.0.1.1              Internet
      |                             |                          |
      | 1. Check: 8.8.8.8 same subnet?                         |
      |    10.0.1.10 & 255.255.255.0 = 10.0.1.0                |
      |    8.8.8.8   & 255.255.255.0 = 8.8.8.0                 |
      |    → Different! Need gateway.                          |
      |                             |                          |
      | 2. ARP: who has 10.0.1.1?   |                          |
      |<--- MAC: aa:bb:cc:dd:ee:ff--|                          |
      |                             |                          |
      | 3. Send L2 frame:           |                          |
      |    Dst MAC: aa:bb:cc:...    |                          |
      |    Dst IP:  8.8.8.8  ------>|                          |
      |                             |                          |
      |                             | 4. Router checks table:  |
      |                             |    0.0.0.0/0 → ISP       |
      |                             |                          |
      |                             | 5. New L2 frame -------->|
      |                             |    Dst MAC: ISP router   |
      |                             |    Dst IP: 8.8.8.8       |
```

**Key insight:** IP addresses stay constant end-to-end. MAC addresses change at each L3 hop.

#### Multi-subnet enterprise example

```
VPC: 10.0.0.0/16

Subnet A: 10.0.1.0/24 (web tier)     Router R1
Subnet B: 10.0.2.0/24 (app tier)     Router R1
Subnet C: 10.0.3.0/24 (db tier)      Router R1

Host in Subnet A (10.0.1.50) → Host in Subnet B (10.0.2.100):
  1. Different subnets → send to gateway
  2. Gateway R1 receives → longest match: 10.0.2.0/24 → connected interface
  3. R1 ARPs for 10.0.2.100 on Subnet B → delivers frame
```

**Routing table on R1:**
```
Destination       Next Hop        Interface
10.0.1.0/24       Connected       eth0
10.0.2.0/24       Connected       eth1
10.0.3.0/24       Connected       eth2
0.0.0.0/0         203.0.113.1     eth3 (default route to Internet)
``` 

**Longest prefix match examples:**

| Destination | Matches                             | Winner                          |
| ----------- | ----------------------------------- | ------------------------------- |
| 10.0.1.25   | 10.0.1.0/24, 10.0.0.0/16, 0.0.0.0/0 | 10.0.1.0/24 (/24 most specific) |
| 10.0.99.10  | 10.0.0.0/16, 0.0.0.0/0              | 10.0.0.0/16 (/16 beats /0)      |
| 8.8.8.8     | 0.0.0.0/0                           | 0.0.0.0/0 (default route)       |
### IPv4 Packet Header

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |Version|  IHL  |    DSCP   |ECN|         Total Length          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |         Identification        |Flags|    Fragment Offset      |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  TTL  |    Protocol           |       Header Checksum         |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                       Source Address                          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                    Destination Address                        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                    Options (if IHL > 5)                       |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key fields explained:**
- **Version (4 bits):** 4 for IPv4, 6 for IPv6.
- **IHL (Internet Header Length, 4 bits):** header length in 32-bit words. Minimum 5 (= 20 bytes), maximum 15 (= 60 bytes with options).
- **DSCP (Differentiated Services Code Point) / ECN (Explicit Congestion Notification):** used for QoS (Quality of Service) marking and congestion signaling.
- **Total Length (16 bits):** entire packet size in bytes. Max 65,535 bytes.
- **Identification, Flags, Fragment Offset:** used for fragmentation/reassembly.
    - **DF (Don't Fragment) flag:** when set, routers drop oversized packets instead of fragmenting, replying with ICMP "fragmentation needed." Basis of **PMTUD (Path MTU Discovery)**.
    - **MF (More Fragments) flag:** 0 = last fragment, 1 = more fragments follow.
    - In practice, avoid fragmentation — it's inefficient, duplicates effort if any fragment is lost, and many firewalls drop fragments.
- **TTL (Time To Live, 8 bits):** decremented at every router. Packet dropped at 0, triggering an ICMP Time Exceeded message. Prevents infinite routing loops. Typical initial values: 64 (Linux), 128 (Windows), 255 (network gear). Basis of `traceroute`.
- **Protocol (8 bits):** identifies the upper-layer protocol. TCP=6, UDP=17, ICMP=1, OSPF=89.
- **Header Checksum (16 bits):** error check on the header only (not payload). Recalculated at every hop because TTL changes. Removed in IPv6 (redundant with L2 and L4 checksums).
- **Source/Destination Address (32 bits each):** unchanged end-to-end (except when NAT rewrites source).
- **Options:** rarely used. Can carry source routing, record route, timestamps. Most networks drop packets with options for security reasons.

### ICMP (Internet Control Message Protocol)

ICMP is the network layer's diagnostic and error-reporting protocol. ICMP messages are encapsulated inside IP packets (protocol field = 1) but are considered part of Layer 3, not Layer 4.

**ICMP is not used for data transfer** — it exists to communicate problems back to the source and to perform network diagnostics.

#### ICMP message structure

Every ICMP message has: **Type** (what kind of message), **Code** (specific condition within that type), **Checksum**, and type-specific **Data**.

#### Key ICMP message types

|Type|Name|Purpose|Used by|
|---|---|---|---|
|0|Echo Reply|Response to a ping|`ping`|
|3|Destination Unreachable|Host/network/port not reachable|Routers, firewalls|
|4|Source Quench|Congestion notification (deprecated)|—|
|5|Redirect|Tells source to use a better route|Routers|
|8|Echo Request|"Are you there?"|`ping`|
|11|Time Exceeded|TTL reached 0|`traceroute`|
|12|Parameter Problem|Malformed header|Routers|

#### ICMP Destination Unreachable codes

When a router or host can't deliver a packet, the code field explains why:

|Code|Meaning|Common cause|
|---|---|---|
|0|Network unreachable|No route to destination network|
|1|Host unreachable|Route exists but specific host down|
|3|Port unreachable|UDP packet to a port with no listener|
|4|Fragmentation needed, DF set|Packet too big, can't fragment (PMTUD)|
|13|Communication administratively filtered|Firewall blocking traffic|

#### How `ping` works

1. Source sends ICMP Echo Request (type 8) to destination.
2. Destination replies with ICMP Echo Reply (type 0).
3. Source measures the round-trip time.

If no reply comes back, the destination might be down, ICMP might be blocked by a firewall, or the route might be broken. No reply does not always mean the host is unreachable.

#### How `traceroute` works

1. Source sends packets with TTL=1. The first router decrements TTL to 0, drops the packet, and sends back ICMP Time Exceeded (type 11).
2. Source records the router's IP and RTT, then sends with TTL=2.
3. Repeats, incrementing TTL, until the destination is reached (responds with ICMP Echo Reply or ICMP Port Unreachable depending on the implementation).

Linux `traceroute` uses UDP by default (to high ports); Windows `tracert` uses ICMP Echo Request. `mtr` combines ping and traceroute in a continuous display — best for diagnosing intermittent issues.

> **Important rules:** an ICMP error about another ICMP error is never generated (prevents storms). ICMP errors are never sent for broadcast/multicast packets. ICMP errors always include the original packet's IP header + first 8 bytes of payload so the source can identify which flow caused the error.

### Routing

Every router has a **routing table** mapping destination prefixes to next-hop addresses and output interfaces. The core mechanism:

**Longest prefix match:** when multiple entries match a destination, the most specific one (longest mask) wins. This is how route aggregation and CIDR work.

```
  Packet journey: Host A (10.0.1.5) -> Host B (203.0.113.10)

  Host A           Router 1           Router 2           Host B
  10.0.1.5         10.0.1.1           203.0.113.1        203.0.113.10
    |                  |                  |                  |
    |-- frame ------>> |                  |                  |
    |  dst MAC: R1     |-- frame ------>> |                  |
    |  dst IP:  B      |  dst MAC: R2     |-- frame ------>> |
    |                  |  dst IP:  B      |  dst MAC: B      |
    |                  |                  |  dst IP:  B      |

  IP addresses stay the same end-to-end.
  MAC addresses change at every hop.
```

### Autonomous Systems and Routing Protocols

#### Interior vs. Exterior Routing

The Internet is a network of networks. Each organization controls an **Autonomous System (AS)** identified by an ASN (AS Number, 16 or 32-bit).
- **IGP (Interior Gateway Protocol):** routes **within** an AS. Goal: find shortest/best paths. Administrative control is unified.
- **EGP (Exterior Gateway Protocol):** routes **between** ASes. Goal: honor business relationships and policy, not just shortest path.
#### OSPF (Open Shortest Path First) — Link-State IGP

**How it works:**
1. Each router floods **LSAs (Link-State Advertisements)** describing its directly connected links (neighbors, cost/metric).
2. Every router builds identical **topology database** of the entire network.
3. Each router runs **Dijkstra's algorithm** (SPF = Shortest Path First) to compute best paths to every destination.
4. Updates triggered by topology changes → fast convergence (seconds).

**Key concepts:**
- **Areas:** OSPF supports hierarchy. Area 0 (backbone) required; other areas connect through it. Reduces LSA flooding and routing table size.
- **Metric:** typically based on interface bandwidth. Cost = reference bandwidth / interface bandwidth. Lower cost = better.
- **DR/BDR (Designated Router / Backup):** on broadcast networks (Ethernet), one router is elected DR to reduce LSA flooding overhead.
  
**OSPF packet types:**

| Type | Name                       | Purpose                                  |
| ---- | -------------------------- | ---------------------------------------- |
| 1    | Hello                      | Discover neighbors, maintain adjacencies |
| 2    | DBD (Database Description) | Summarize LSA database                   |
| 3    | LSR (Link-State Request)   | Request specific LSAs                    |
| 4    | LSU (Link-State Update)    | Send LSAs (topology updates)             |
| 5    | LSAck                      | Acknowledge LSAs                         |

**Advantages:**
- Fast convergence (subsecond to a few seconds).
- Scales well with areas.
- Loop-free (Dijkstra guarantees shortest paths).
- Standard protocol (RFC 2328 for OSPFv2, RFC 5340 for OSPFv3/IPv6).

**When used:** Enterprise networks, ISP internal networks, datacenter fabric (sometimes).  

#### BGP (Border Gateway Protocol) — Path-Vector EGP

**The protocol that holds the Internet together.** BGP is policy-driven, not metric-driven. Routes are chosen based on business relationships, not just hop count or bandwidth.

**How it works:**
1. BGP speakers establish **TCP connections (port 179)** with peers.
2. Routers exchange **full routing tables** initially, then incremental updates.
3. Each route advertisement includes the **AS path** — the list of ASes the route traversed.
4. Loop prevention: if a router sees its own AS in the path, it rejects the route.

**BGP session types:**

| Type                    | Relationship           | Use case                                    |
| ----------------------- | ---------------------- | ------------------------------------------- |
| **eBGP** (External BGP) | Between different ASes | Internet peering, customer-provider         |
| **iBGP** (Internal BGP) | Within the same AS     | Distribute external routes learned via eBGP |

**Route selection algorithm (simplified):**

BGP chooses routes based on **attributes** in this order:
1. **Highest LOCAL_PREF** (local preference, set by policy within AS).
2. **Shortest AS_PATH** (fewer AS hops).
3. **Lowest ORIGIN** (IGP < EGP < incomplete).
4. **Lowest MED** (Multi-Exit Discriminator, hint from neighbor).
5. **eBGP over iBGP** (prefer external routes).
6. **Lowest IGP cost to BGP next hop**.
7. **Lowest router ID** (tiebreaker).

**AS relationships:**

```
         ISP Tier 1 (AS 100)
              |
    +---------+---------+
    |                   |
Customer (AS 200)   Peer (AS 300)
```
- **Customer → Provider:** Customer pays provider for transit. Provider announces customer's routes to the world.
- **Peer ↔ Peer:** Mutual agreement to exchange traffic for free (settlement-free peering). Only announce own customers, not transit routes.
- **Provider → Customer:** Provider gives full routing table (default route or partial).

**Why BGP matters for engineers:**

1. **Anycast:** Same IP announced from multiple locations. BGP routing ensures clients reach the nearest POP (Point of Presence). Used by:  
   - CDNs (Cloudflare, Akamai)
   - DNS root servers, Google Public DNS (8.8.8.8), Cloudflare DNS (1.1.1.1)
2. **Multi-region failover:** Cloud services use BGP to shift traffic during outages. If AWS us-east-1 fails, BGP withdraws routes, traffic goes to us-west-2.
3. **BGP hijacks:** Malicious or accidental route announcements can redirect traffic. Examples:
   - 2008: Pakistan Telecom hijacked YouTube's IP space, took YouTube offline globally.
   - 2018: Misconfigured route leaked via BGP, sent traffic through China.
4. **Traffic engineering:** By prepending AS numbers (making path look longer) or adjusting LOCAL_PREF, operators control inbound/outbound traffic flows.

**BGP convergence:** Slow compared to OSPF (minutes). Updates propagate hop-by-hop across the Internet. Route flapping (unstable routes) is dampened to prevent instability.

**BGP message types:**

| Type         | Purpose                                       |
| ------------ | --------------------------------------------- |
| OPEN         | Establish BGP session, negotiate capabilities |
| UPDATE       | Advertise/withdraw routes                     |
| KEEPALIVE    | Maintain session (sent every 60s by default)  |
| NOTIFICATION | Error, close session                          |
#### Comparison Table
| Aspect              | OSPF                     | BGP                                    |
| ------------------- | ------------------------ | -------------------------------------- |
| **Type**            | Link-state IGP           | Path-vector EGP                        |
| **Scope**           | Within AS                | Between ASes                           |
| **Metric**          | Cost (bandwidth-based)   | Policy (AS_PATH, LOCAL_PREF, etc.)     |
| **Algorithm**       | Dijkstra (SPF)           | Best path selection (policy-driven)    |
| **Convergence**     | Fast (seconds)           | Slow (minutes)                         |
| **Transport**       | IP protocol 89           | TCP port 179                           |
| **Scalability**     | Medium (areas help)      | Massive (Internet-scale, hundreds of thousands of routes) |
| **Loop prevention** | Topology knowledge       | AS_PATH loop detection                 |
| **Primary use**     | Enterprise, ISP internal | Internet routing, multi-AS             |

---

## 5. Layer 4 — Transport (TCP, UDP, QUIC)

The transport layer provides end-to-end communication between processes on different hosts, identified by **port numbers**.

### Port Numbers

Assigned by IANA (Internet Assigned Numbers Authority). Well-known ports (0-1023) require root/admin privileges. Ephemeral ports (>1023, typically 49152-65535) are used by clients.

|Port|Service|Port|Service|
|---|---|---|---|
|22|SSH (Secure Shell)|443|HTTPS|
|25|SMTP (Simple Mail Transfer Protocol)|3306|MySQL|
|53|DNS|5432|PostgreSQL|
|80|HTTP|6379|Redis|
|110|POP3 (Post Office Protocol)|8080|HTTP (alternate)|
|143|IMAP (Internet Message Access Protocol)|27017|MongoDB|

### Sockets — Deep Dive

A **socket** is one endpoint of a network connection — an abstraction provided by the OS that lets applications send and receive data over the network. Think of it as a file descriptor for a network connection.

#### What identifies a socket

- **UDP socket:** identified by the tuple `(local IP, local port)`. Any host can send datagrams to it. There's no "connection" — each datagram is independent.
- **TCP socket:** a listening socket is identified by `(local IP, local port)`. Once a connection is accepted, the connected socket is identified by the full **5-tuple:** `(protocol, local IP, local port, remote IP, remote port)`.

This is how a web server handles thousands of connections on port 443 — each client creates a unique 5-tuple because they have different (source IP, source port) combinations.

#### Socket lifecycle (TCP)

```
  Server                                Client
    |                                      |
    |  socket()     -- create socket       |
    |  bind()       -- assign address      |
    |  listen()     -- mark as passive     |
    |                                      |  socket()
    |                                      |  connect()  --> SYN
    |  accept()     <-- SYN received       |
    |               --> SYN-ACK            |
    |               <-- ACK                |  connect() returns
    |  accept() returns new socket         |
    |                                      |
    |  read()/write()  <--- data --->  read()/write()
    |                                      |
    |  close()      --> FIN                |
    |               <-- ACK, FIN           |  close()
    |               --> ACK                |
```

#### Socket types in practice

|Type|C constant|Protocol|Behavior|
|---|---|---|---|
|Stream|`SOCK_STREAM`|TCP|Reliable, ordered byte stream|
|Datagram|`SOCK_DGRAM`|UDP|Unreliable, independent messages|
|Raw|`SOCK_RAW`|Any|Direct access to IP layer (requires root)|

#### Practical socket concepts

- **Backlog queue:** `listen(fd, backlog)` sets how many pending connections the OS will queue before refusing new ones. If the server is slow to `accept()`, clients are refused connections once the backlog fills.
- **`SO_REUSEADDR`:** allows binding to an address that is in TIME_WAIT. Essential for servers that restart frequently — without it, "address already in use" errors occur.
- **`SO_REUSEPORT`:** allows multiple sockets to bind to the same address/port. The kernel distributes incoming connections across them. Used for multi-process/multi-thread server architectures (Nginx, Go).
- **Non-blocking I/O and multiplexing:** production servers do not use one thread per connection. They use `epoll` (Linux), `kqueue` (macOS/BSD), or `io_uring` (modern Linux) to handle thousands of connections in one thread via event loops. This is the mechanism underlying Node.js, Nginx, and Go's runtime.
- **`TCP_NODELAY`:** disables Nagle's algorithm, which batches small writes into larger segments. For latency-sensitive applications (real-time, interactive), enable `TCP_NODELAY` to send data immediately.
- **Keep-alive:** `SO_KEEPALIVE` sends periodic probes on idle connections to detect dead peers. Configurable timers: how long before first probe, interval between probes, how many probes before giving up.

### TCP (Transmission Control Protocol)

Reliable, ordered, byte-stream delivery. Turns unreliable IP into a pipe applications can trust.

#### TCP Header (20 bytes minimum, up to 60 bytes with options)

```
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |          Source Port          |       Destination Port        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        Sequence Number                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Acknowledgment Number                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Data |       |U|A|P|R|S|F|                                   |
   |Offset |Reserv.|R|C|S|S|Y|I|            Window                 |
   |       |       |G|K|H|T|N|N|                                   |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Checksum            |         Urgent Pointer        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options (if Data Offset > 5)               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                            Data                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```

#### Key fields

| Field                 |     Size | Meaning                             |
| --------------------- | -------: | ----------------------------------- |
| Source Port           |  16 bits | Sending application port            |
| Destination Port      |  16 bits | Receiving application port          |
| Sequence Number       |  32 bits | Byte position in the stream         |
| Acknowledgment Number |  32 bits | Next expected byte                  |
| Data Offset           |   4 bits | TCP header length in 32-bit words   |
| Reserved              |   3 bits | Reserved for future use             |
| Flags                 |   6 bits | URG, ACK, PSH, RST, SYN, FIN        |
| Window                |  16 bits | Receiver-advertised buffer space    |
| Checksum              |  16 bits | Error check for header and data     |
| Urgent Pointer        |  16 bits | End of urgent data, if URG is set   |
| Options               | Variable | MSS, Window Scale, SACK, Timestamps |

#### TCP flags

- **SYN** — start connection, synchronize sequence numbers
- **ACK** — acknowledgment field is valid
- **FIN** — graceful close
- **RST** — reset connection immediately
- **PSH** — push data to application promptly
- **URG** — urgent pointer is valid

The handshake and teardown combinations (`SYN`, `SYN-ACK`, `ACK`, `FIN-ACK`) are shown in the diagrams below. Two other common combinations: `PSH-ACK` is an ordinary data-carrying segment, and a bare `RST` aborts or refuses a connection.

#### Three-Way Handshake

```
  Client                            Server
    |                                 |
    |--- SYN (seq=x) --------------->>|  Client picks ISN
    |                                 |  (Initial Sequence Number) x
    |<<- SYN-ACK (seq=y, ack=x+1) ----|  Server picks ISN y,
    |                                 |  acknowledges x+1
    |--- ACK (seq=x+1, ack=y+1) ---->>|  Connection open.
    |                                 |  Can carry data.
    |         <--- 1 RTT --->         |
```

Costs **1 RTT (Round-Trip Time)** before data flows. Add TLS 1.3 (another RTT) = **2 RTTs** before the first byte of application data on a new HTTPS connection.

#### Connection Teardown

```
  Side A                            Side B
    |--- FIN ---------------------->>|
    |<<- ACK ------------------------|
    |<<- FIN ------------------------|
    |--- ACK ---------------------->>|
    |                                |
    |  A enters TIME_WAIT (~60s)     |
```

**TIME_WAIT** is entered by the side that sends the first FIN (the active closer) and lasts **2×MSL** (Maximum Segment Lifetime, ~60s on Linux), for two reasons: (1) let delayed duplicate segments from this connection expire before the 4-tuple is reused, and (2) allow the final ACK to be retransmitted if the peer re-sends its FIN. On busy servers thousands of TIME_WAIT sockets can exhaust ephemeral ports; tunable via `net.ipv4.tcp_tw_reuse` on Linux (safe for outbound connections only).

#### Reliability

TCP retransmits lost segments, detected by **timeout** or **3 duplicate ACKs** (fast retransmit). The RTO (Retransmission Timeout) adapts dynamically to measured RTT.

**SACK (Selective Acknowledgment):** the receiver reports exactly which byte ranges it has, so the sender retransmits only the truly missing ranges. Without SACK the sender has less information: fast retransmit still resends just the one segment the duplicate ACKs point to, but a full **retransmission timeout** forces it to resume from the last cumulatively-acknowledged byte and resend everything after it (the Go-Back-N-style behavior), since it cannot tell which later segments actually arrived.

#### Flow Control

The receiver advertises available buffer space in the **window** field. The sender limits in-flight data to this value. Window = 0 means "stop sending" — the sender probes periodically until the window opens.

#### Congestion Control

TCP regulates sending rate to avoid collapsing the network. The sender maintains a **cwnd (congestion window)** — the actual window is the minimum of cwnd and the receiver's advertised window.

```
  cwnd
   ^
   |        /\      /\      /\
   |       /  \    /  \    /  \       <-- multiplicative decrease (halve)
   |      /    \  /    \  /    \
   |     /      \/      \/      \     <-- additive increase (linear)
   |    /
   |   / <-- slow start (exponential)
   |  /
   | /
   +--------------------------------> time
```

1. **Slow start:** cwnd starts at the initial window (classically 1 MSS; modern stacks use ~10 MSS, RFC 6928) and doubles every RTT (exponential) until ssthresh or loss.
2. **Congestion avoidance:** cwnd grows by 1 MSS per RTT (linear).
3. **On timeout:** ssthresh = cwnd/2, cwnd = 1, back to slow start.
4. **On 3 dup ACKs (fast recovery):** ssthresh = cwnd/2, cwnd = ssthresh, stay linear.

**Modern algorithms:** **CUBIC** (Linux default, more aggressive recovery), **BBR** (Google, estimates bandwidth + RTT instead of relying on loss signals, much better on lossy/long-distance links).

#### Why TCP Matters for Application Code

- **New connections start slow.** Short-lived connections pay full handshake + slow start cost every time. This is why connection pooling and keep-alive exist.
- **Head-of-line blocking:** one lost packet stalls all data on that connection, even if it belongs to independent streams. This is HTTP/2's problem over TCP.
- **Loss = congestion assumption.** On lossy wireless links, this is wrong and throughput collapses.

#### Key TCP Options

|Option|Purpose|
|---|---|
|MSS (Maximum Segment Size)|Negotiated at handshake (typically 1460)|
|Window Scaling|Allows windows > 65535 bytes (essential for high-bandwidth links)|
|Timestamps|RTT measurement, duplicate detection via PAWS (Protection Against Wrapped Sequences)|
|SACK (Selective Acknowledgment)|Reports received byte ranges so only lost segments are retransmitted|

### UDP (User Datagram Protocol)

Minimal transport: adds port numbers and an optional checksum on top of IP. No connection, no reliability, no ordering, no congestion control. **8 bytes of header** vs. 20+ for TCP.

```
  UDP Header
   0               16              32
   +---------------+---------------+
   |  Source Port   |   Dest Port   |
   +---------------+---------------+
   |    Length      |   Checksum    |
   +---------------+---------------+
   |           Payload              |
   +-------------------------------+
```

**When to use UDP:** DNS queries, real-time media (VoIP, video), gaming, DHCP, NTP (Network Time Protocol), and as the substrate for QUIC.

**Pitfalls:** no congestion control (buggy senders can flood networks), NAT mappings expire fast (~30s, need keepalives), some firewalls block/rate-limit UDP.

### QUIC

Transport protocol designed by Google, running over UDP. Implements reliability, multiplexing, and congestion control in userspace. HTTP/3 runs over QUIC.

Key advantages over TCP+TLS:

- **Per-stream loss recovery:** eliminates head-of-line blocking.
- **Built-in TLS 1.3:** encryption is mandatory, not optional.
- **Faster setup:** 1 RTT (0-RTT on resumption) vs. 2-3 RTTs for TCP+TLS.
- **Connection migration:** survives IP changes (e.g., WiFi to cellular).
- **QPACK header compression:** HTTP/3's equivalent of HTTP/2's HPACK, redesigned so that header-table updates cannot stall when QUIC delivers streams out of order.

```
  TCP + TLS 1.3 (2 RTTs):           QUIC (1 RTT, 0-RTT on resumption):

  Client      Server                 Client      Server
    |-- SYN ---->|                     |-- Initial -->|  crypto + transport
    |<- SYNACK --|  1 RTT              |<- Handshk ---|  combined: 1 RTT
    |-- ACK ---->|                     |-- Data ----->|
    |-- TLS ---->|                     |              |
    |<- TLS -----|  1 RTT
    |-- Data --->|
```

---

## 6. Layer 5/6/7 — Session, Presentation, Application

In the TCP/IP model these are merged into the Application layer. This is where most application development occurs.

### HTTP

The protocol of the web and the dominant API transport.

|Version|Transport|Key feature|Limitation|
|---|---|---|---|
|HTTP/1.1|TCP|Keep-alive, chunked encoding|1 request at a time per connection (HOL blocking)|
|HTTP/2|TCP|Binary framing, multiplexed streams, HPACK header compression|TCP-level HOL blocking on packet loss|
|HTTP/3|QUIC (UDP)|Per-stream loss recovery, 0-RTT, built-in TLS 1.3, QPACK header compression|UDP sometimes blocked by networks|

**HTTP request/response structure:**

```
  Request:                         Response:
  GET /api/users HTTP/1.1          HTTP/1.1 200 OK
  Host: example.com                Content-Type: application/json
  Authorization: Bearer xyz        Content-Length: 128

                                   {"users": [...]}
```

**HTTP methods:** GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE (remove), HEAD (metadata only), OPTIONS (CORS — Cross-Origin Resource Sharing — preflight).

**Status codes to know:**

- 2xx: success (200 OK, 201 Created, 204 No Content)
- 3xx: redirect (301 Moved Permanently, 302 Found, 304 Not Modified)
- 4xx: client error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- 5xx: server error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout)

### TLS (Transport Layer Security) — Deep Dive

This section treats TLS from the transport angle: the handshake as a sequence of round trips, the version differences that drive connection-setup latency, and how TLS is deployed and terminated in production. The complementary view — how the underlying primitives (AEAD ciphers, ECDHE, the HKDF key schedule) compose to provide the guarantees below — belongs to applied cryptography and is developed as a case study in `Cryptography.md`.

TLS encrypts the communication channel between two parties, providing three guarantees: **confidentiality** (the data cannot be read by third parties), **integrity** (the data cannot be modified undetected), and **authentication** (the remote party is who it claims to be).

#### TLS versions

|Version|Status|Handshake RTTs|Key difference|
|---|---|---|---|
|TLS 1.0 / 1.1|Deprecated|2|Vulnerable, disabled in modern browsers|
|TLS 1.2|Supported|2|Still widely used, many cipher suites|
|TLS 1.3|Current standard|1 (0-RTT on resumption)|Faster, simpler, removed weak ciphers|

#### TLS 1.3 handshake in detail

Each side sends a **Diffie-Hellman (DH) key share** in its first message; the two shares combine into a shared secret that neither could derive alone, from which the session keys are derived.

```
  Client                                     Server
    |                                          |
    |-- ClientHello ------------------------->>|
    |   - supported cipher suites              |
    |   - key share (client's DH public key)   |
    |   - supported TLS versions               |
    |                                          |
    |<<- ServerHello --------------------------|
    |   - chosen cipher suite                  |
    |   - key share (server's DH public key)   |
    |   --- everything below is encrypted ---  |
    |   - EncryptedExtensions                  |
    |   - server certificate                   |
    |   - certificate verify (signature)       |
    |   - finished (MAC of handshake)          |
    |                                          |
    |   (both sides derive session keys        |
    |    from the DH key exchange)             |
    |                                          |
    |-- Finished ---------------------------->>|
    |   (MAC of handshake)                     |
    |                                          |
    |<<========== encrypted data ============>>|
```

Unlike TLS 1.2, everything after the ServerHello — including the server's certificate — is encrypted under keys derived during this handshake, so an on-path observer cannot see which certificate the server presents.

**Key exchange:** TLS 1.3 exclusively uses ephemeral Diffie-Hellman (ECDHE — Elliptic Curve Diffie-Hellman Ephemeral). Every connection generates fresh keys, providing **forward secrecy** — if the server's private key is compromised later, past sessions can't be decrypted.

**0-RTT resumption:** if the client has connected before, it can send data in the first message using a PSK (Pre-Shared Key) from a previous session. Faster, but the 0-RTT data is not protected against replay attacks — only use it for idempotent requests.

#### Certificates and trust

- The server presents a **certificate** signed by a CA (Certificate Authority).
- The client verifies the chain: server cert -> intermediate CA -> root CA (trusted by the OS/browser).
- The certificate contains the server's public key and the domain(s) it's valid for (CN (Common Name) or SAN (Subject Alternative Name) field).
- **Let's Encrypt** provides free, automated certificates and has become the dominant CA.

#### TLS for engineers — practical knowledge

- **TLS termination:** where TLS is decrypted. May be at the load balancer (AWS ALB, Nginx), a reverse proxy, or the application itself. Traffic between the termination point and the backend may be unencrypted.
- **mTLS (mutual TLS):** both sides present certificates. The server verifies the client's certificate too. Standard in service mesh (Istio, Linkerd) and zero-trust architectures. Used to authenticate service-to-service communication.
- **Certificate pinning:** the client only trusts specific certificates (not any CA-signed cert). Used in mobile apps for extra security. Makes rotation harder — use with care.
- **Certificate transparency (CT):** public logs of all issued certificates. Enables detection of unauthorized certificates issued for a domain.
- **SNI (Server Name Indication):** TLS extension where the client specifies which hostname it's connecting to. Required for virtual hosting (multiple HTTPS sites on one IP). The SNI rides in the ClientHello and is sent in plaintext in **both TLS 1.2 and TLS 1.3** — an on-path observer still sees the destination hostname. **ECH (Encrypted Client Hello)** is a separate, optional extension that encrypts the whole ClientHello including SNI; it needs support on both ends and is not yet widely deployed.
- **HSTS (HTTP Strict Transport Security):** response header telling browsers to always use HTTPS for the domain. Prevents SSL-stripping MITM attacks.

### WebSocket

WebSocket is a protocol for **persistent, full-duplex communication over a single TCP connection**. Unlike HTTP's request/response model, both client and server can send messages **at any time** after the connection is established.

Defined in **RFC 6455**. Typically runs over:
- `ws://` → WebSocket over TCP
- `wss://` → WebSocket over TLS (same port as HTTPS, 443)

Most real deployments use **wss://** because browsers require secure contexts for many features.

#### HTTP Upgrade Handshake

WebSocket starts as an **HTTP/1.1 request** and upgrades the connection.

```
Client                                    Server
  |                                         |
  |-- HTTP GET /chat --------------------->|
  |   Host: example.com                    |
  |   Upgrade: websocket                   |
  |   Connection: Upgrade                  |
  |   Sec-WebSocket-Key: x3JJHMbDL1Ez...   |
  |   Sec-WebSocket-Version: 13             |
  |                                         |
  |<<- HTTP/1.1 101 Switching Protocols ----|
  |   Upgrade: websocket                    |
  |   Connection: Upgrade                   |
  |   Sec-WebSocket-Accept: HSmrc0sMl...    |
  |                                         |
  |<<=========== WebSocket frames =========>>|
```

Key points:
- **`Upgrade: websocket`** tells the server to switch protocols.
- **`Sec-WebSocket-Key`** is a random value generated by the client.
- The server returns **`Sec-WebSocket-Accept`**, which is a SHA-1 hash of the key + a magic GUID.
- The `101 Switching Protocols` response confirms the upgrade.

After this, the connection is **no longer HTTP** — it becomes raw WebSocket frames.

#### WebSocket Frame Format

Communication happens through **frames**.

```
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5   <- bit, within the first 2 bytes
+-+-+-+-+-------+-+-------------+
|F|R|R|R|opcode |M| payload len |   F=FIN, R=RSV1-3, M=MASK
|I|S|S|S| (4b)  |A|    (7b)     |
|N|V|V|V|       |S|             |
| |1|2|3|       |K|             |
+-+-+-+-+-------+-+-------------+
| extended payload length (if len == 126 or 127)  |
+--------------------------------------------------+
| masking key, 4 bytes (only if MASK = 1)          |
+--------------------------------------------------+
| payload data                                     |
+--------------------------------------------------+
```

Important fields:

|Field|Meaning|
|---|---|
|FIN|Whether this is the final frame of a message|
|Opcode|Frame type (text, binary, ping, pong, close)|
|Mask|Client frames must be masked|
|Payload|Actual message data|

Common opcodes:

|Opcode|Type|
|---|---|
|0x1|Text frame|
|0x2|Binary frame|
|0x8|Connection close|
|0x9|Ping|
|0xA|Pong|

#### Connection Lifecycle

```
1. HTTP Upgrade handshake
2. Persistent full-duplex messaging
3. Heartbeats (ping/pong)
4. Close handshake
```

Closing sequence:

```
Client -> Close frame
Server -> Close frame
TCP connection closes
```

#### Scaling WebSocket Systems

WebSocket introduces **stateful connections**, which complicates scaling.

Key challenges:

**1. Long-lived connections**

Unlike HTTP requests, connections stay open for minutes or hours.

Implications:
- File descriptor limits
- Memory per connection
- Event loop scalability

Large deployments often use:
- **event-driven servers** (Node.js, Netty, Go, Rust async)
- **epoll / kqueue**


**2. Load balancing**

WebSocket requires **connection affinity**.

Common approaches:

|Strategy|Description|
|---|---|
|Sticky sessions|Load balancer pins client to one backend|
|Connection router|Dedicated gateway handles connections|
|Pub/Sub backend|Servers share events via Redis/Kafka|

Example architecture:

```
Clients
   |
   v
Load Balancer
   |
   +---- WS Server A ----+
   |                     |
   +---- WS Server B ----+---- Redis / Kafka PubSub
   |                     |
   +---- WS Server C ----+
```

When a message arrives:
1. Client sends message to connected server
2. Server publishes to message broker
3. Other servers forward to their clients

#### WebSocket vs SSE

**Server-Sent Events (SSE)** is a simpler alternative for **one-way streaming**.

|Feature|WebSocket|SSE|
|---|---|---|
|Direction|Bidirectional|Server → Client|
|Transport|Custom protocol|HTTP|
|Binary support|Yes|No|
|Reconnection|Manual|Built-in|
|Proxy compatibility|Sometimes tricky|Excellent|

SSE works well for:
- notifications
- activity feeds
- dashboards

WebSocket is preferred for:
- chat
- multiplayer games
- collaborative editing
- real-time trading

#### WebSocket in Production

Common issues engineers encounter:

**Idle timeouts**

Many proxies terminate idle connections:

|Component|Typical timeout|
|---|---|
|Nginx|60s default|
|AWS ALB|60s|
|Cloudflare|100s|

Solution: **heartbeat ping/pong frames**.

**Backpressure**

If the client cannot process messages fast enough:
- buffers grow
- memory spikes
- connection may be dropped

Production servers implement:
- send queues
- rate limits
- slow consumer detection

**Authentication**

Options include:

|Method|Notes|
|---|---|
|Cookie session|works for browser apps|
|JWT (JSON Web Token) in query param|common but visible in logs|
|JWT in `Authorization` header|preferred|
|Auth during HTTP upgrade|most secure|

After authentication, servers often attach a **user ID to the connection state**.

### gRPC (Google Remote Procedure Call)

gRPC is a **high-performance RPC framework** developed by Google. It uses:
- **HTTP/2** for transport
- **Protocol Buffers (Protobuf)** for serialization
- **generated client/server code**

Instead of designing REST endpoints, **methods are defined in a service interface**.

#### Basic gRPC Service Definition

Services are defined in `.proto` files.

```
syntax = "proto3";

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}

message UserRequest {
  int64 user_id = 1;
}

message UserResponse {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

From this file, the gRPC toolchain generates:
- client stubs
- server interfaces
- serialization code

Supported languages include Go, Java, Python, C++, Rust, Node, etc.

#### How a gRPC Request Works

gRPC runs over **HTTP/2 streams**.

```
Client                             Server
  |                                  |
  |-- HEADERS (method, metadata) --->|
  |-- DATA (protobuf message) ------>|
  |                                  |
  |<<- HEADERS (status) -------------|
  |<<- DATA (protobuf response) -----|
  |<<- TRAILERS (grpc-status) -------|
```

Important HTTP/2 features used:
- **multiplexed streams**
- **flow control**
- **header compression**
- **persistent connections**

#### Serialization: Protocol Buffers

Protobuf is a **binary serialization format**.

Example message:
```
UserResponse {
  id: 42
  name: "Alice"
}
```

Encoded as compact binary fields:
```
08 2A 12 05 41 6C 69 63 65
```

Advantages:

|Benefit|Explanation|
|---|---|
|Compact|Much smaller than JSON|
|Fast|Binary parsing|
|Schema evolution|Backward compatible field additions|
|Strong typing|Generated code|

#### gRPC Communication Patterns

gRPC supports **four types of RPC**.

|Type|Description|
|---|---|
|Unary|Single request → single response|
|Server streaming|Client request → server stream|
|Client streaming|Client stream → server response|
|Bidirectional streaming|Both sides stream|

**Unary RPC**
```
Client ---- request ----> Server
Client <--- response ---- Server
```
Equivalent to typical REST calls.

**Server Streaming**
```
Client ---- request ----> Server
Client <--- msg1 -------- Server
Client <--- msg2 -------- Server
Client <--- msg3 -------- Server
```
Used for:
- logs
- analytics
- live feeds

**Client Streaming**
```
Client ---- msg1 ------> Server
Client ---- msg2 ------> Server
Client ---- msg3 ------> Server
Client <--- response --- Server
```
Useful for:
- file uploads
- batch ingestion

**Bidirectional Streaming**
```
Client ---- msg1 ------> Server
Client <--- msgA ------ Server
Client ---- msg2 ------> Server
Client <--- msgB ------ Server
```
Used for:
- chat systems
- telemetry
- collaborative apps

#### gRPC Metadata

gRPC supports **metadata headers**, similar to HTTP headers.

Examples:
```
authorization: Bearer <token>
x-request-id: abc123
grpc-timeout: 500m
```

Metadata is used for:
- authentication
- tracing
- deadlines
- routing

#### Deadlines and Cancellation

Clients can set **deadlines** to avoid hanging requests.
```go
ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
defer cancel()
```

If the server doesn't respond in time:
- the client cancels the request
- server receives cancellation signal

This is important for **microservice resilience**.

#### gRPC Load Balancing

gRPC supports multiple balancing strategies:

|Strategy|Description|
|---|---|
|Pick-first|First healthy backend|
|Round-robin|Simple rotation|
|xDS|Advanced service mesh config (the Envoy discovery-service API family)|

In Kubernetes, common setups are:
```
Client
  |
  v
Envoy / Service Mesh
  |
  v
gRPC service pods
```

#### gRPC vs REST — Performance

Typical differences:

|Aspect|REST (JSON)|gRPC (Protobuf)|
|---|---|---|
|Payload size|Larger|Smaller|
|Serialization|Slow|Fast|
|Connection|Often short-lived|Persistent|
|Latency|Higher|Lower|

In internal microservices, gRPC can reduce:
- latency
- CPU usage
- bandwidth

#### gRPC in Browsers

Browsers **cannot directly use full gRPC** because they lack raw HTTP/2 framing control.

Solutions:

|Approach|Description|
|---|---|
|grpc-web|Browser-compatible gRPC variant|
|Envoy proxy|Translates grpc-web → gRPC|
|REST gateway|Expose REST alongside gRPC|

Architecture example:
```
Browser
   |
   v
grpc-web
   |
   v
Envoy Proxy
   |
   v
gRPC Services
```

#### When to Use gRPC

Good fit:
- microservice architectures
- high-performance internal APIs
- strongly typed service contracts
- streaming workloads

Less ideal for:
- public APIs
- browser-only clients
- simple CRUD APIs

### REST vs. GraphQL vs. gRPC

|Aspect|REST|GraphQL|gRPC|
|---|---|---|---|
|Transport|HTTP/1.1 or HTTP/2|HTTP|HTTP/2|
|Data format|JSON (text)|JSON (text)|Protobuf (binary)|
|Schema|OpenAPI/Swagger (optional)|Required (SDL — Schema Definition Language)|Required (.proto files)|
|Streaming|Limited (SSE, WebSocket)|Subscriptions (via WS)|Native bidirectional streaming|
|Overfetching|Common (fixed responses)|Solved (client picks fields)|N/A (defined per method)|
|Browser support|Native|Native|Needs grpc-web proxy|
|Best for|Public APIs, web apps|Flexible queries, BFF (Backend-for-Frontend)|Internal microservices, perf-critical|

---

## 7. Network Services (DHCP, NAT, DNS)

### DHCP (Dynamic Host Configuration Protocol)

DHCP automatically configures hosts to join a network. Without it, every device would need manual IP configuration. It assigns: IP address, subnet mask, default gateway, DNS servers, and a lease duration.

Runs over UDP: client uses port 68, server uses port 67. Works via broadcast because the client doesn't have an IP yet.

#### DORA Process

```
  Client (no IP yet)              DHCP Server (10.0.0.1)
    |                                |
    |-- DISCOVER (broadcast) ------>>|  "I need an IP address"
    |   src: 0.0.0.0:68             |   (may reach multiple DHCP servers)
    |   dst: 255.255.255.255:67     |
    |                                |
    |<<- OFFER ----------------------|  "Here, you can use 10.0.0.15"
    |   offered IP: 10.0.0.15       |   (server reserves this IP)
    |   lease: 24 hours             |
    |   gateway: 10.0.0.1           |
    |   DNS: 8.8.8.8               |
    |                                |
    |-- REQUEST (broadcast) ------->>|  "I'll take 10.0.0.15"
    |   (broadcast so other DHCP     |   (if multiple servers offered,
    |    servers know to release     |    this tells them which was chosen)
    |    their offers)               |
    |                                |
    |<<- ACK -----------------------|  "Confirmed. Lease starts now."
    |                                |
    |  Client configures interface   |
    |  with assigned parameters      |
```

#### DHCP lease lifecycle

- **Lease duration:** configurable (1 hour to 30 days typical). Short leases in guest WiFi, long leases for static infrastructure.
- **Renewal:** at 50% of lease time (T1), client sends REQUEST directly to the server (unicast). If no response by 87.5% (T2), client broadcasts REQUEST to any DHCP server.
- **Release:** client sends RELEASE when disconnecting (voluntary, not guaranteed).
- **Reservation:** DHCP server can assign a fixed IP to a specific MAC address. Combines the convenience of DHCP with the predictability of static assignment. Commonly used for printers, servers, and network gear.

#### DHCP relay

DHCP uses broadcast, which doesn't cross routers. In multi-subnet networks, a **DHCP relay agent** (usually configured on the router) forwards DHCP broadcasts from the local subnet to the DHCP server on another subnet as unicast.

> **Debugging tip:** if a host has `169.254.x.x` (link-local), DHCP failed. Check: is the DHCP server running? Is the relay agent configured? Is the scope exhausted (no IPs left)?

### NAT (Network Address Translation)

NAT allows devices with private IP addresses to communicate with the public Internet by rewriting IP addresses (and ports) in packet headers as they pass through the NAT device.

#### Types of NAT

|Type|How it works|Use case|
|---|---|---|
|**SNAT (Source NAT)**|Rewrites source IP on outbound packets|Most common: private hosts accessing the Internet|
|**DNAT (Destination NAT)**|Rewrites destination IP on inbound packets|Port forwarding, load balancing|
|**PAT (Port Address Translation)**|Maps many private IPs to one public IP using different ports|Home routers, cloud NAT gateways|
|**1:1 NAT**|Maps one private IP to one public IP|Servers that need a dedicated public IP|

Most consumer and cloud NAT is **PAT** (also called "NAT overload" or "NAPT" — Network Address Port Translation).

#### How PAT works

```
  Private network               NAT Router                  Internet

  +------------+
  | 10.0.0.2   |- src:10.0.0.2:4921 ->+-----------+-- src:82.1.2.3:5001 --> Server
  | 10.0.0.3   |- src:10.0.0.3:8080 ->| NAT table |-- src:82.1.2.3:5002 --> 203.0.113.10
  +------------+                      | 5001 <-> 10.0.0.2:4921 |
                                      | 5002 <-> 10.0.0.3:8080 |
                                      +-----------+
                                      Public IP: 82.1.2.3

  Return traffic: server sends to 82.1.2.3:5001 -->
  NAT router looks up table, rewrites dst to 10.0.0.2:4921, forwards to private network.
```

#### NAT implications for software engineers

- **Inbound connections:** NAT only creates mappings for outbound traffic. Inbound connections to private hosts require port forwarding or NAT traversal:
    - **STUN (Session Traversal Utilities for NAT):** discovers the public IP/port mapping. Works when both sides are behind "simple" NATs.
    - **TURN (Traversal Using Relays around NAT):** relays traffic through a public server. Works always but adds latency and costs.
    - **ICE (Interactive Connectivity Establishment):** tries STUN first, falls back to TURN. Used by WebRTC.
- **UDP mapping timeout:** NAT entries for UDP expire fast (~30 seconds). Long-lived UDP flows (VoIP, gaming, VPN) need periodic keepalive packets.
- **TCP mapping timeout:** longer (~minutes), but idle connections can still be dropped. TCP keepalive helps.
- **Protocols with embedded IPs:** SIP (Session Initiation Protocol), FTP active mode, and some gaming protocols embed IP addresses in the payload. NAT doesn't rewrite payloads, so these break without an ALG (Application Level Gateway).
- **Cloud NAT layers:** in AWS/GCP/Azure, the path is often: instance private IP -> VPC subnet -> NAT gateway -> public IP. Each layer has its own mapping table and timeouts.
- **Port exhaustion:** the 16-bit port field gives ~65K source ports. Because PAT keys each mapping on the full tuple (public IP, destination IP, destination port, protocol), the ~65K ceiling is *per distinct destination*, not a flat per-public-IP cap — but many clients hitting the same popular destination (e.g. one busy server) still contend for that one bucket. High-traffic NAT gateways can exhaust it, causing connection failures. Solution: add more public IPs.
- **Hairpin NAT:** when a host behind NAT tries to access another host behind the same NAT using the public IP. Not all NAT implementations support this — can cause confusing connectivity issues.

### DNS (Domain Name System) 

DNS translates human-readable names (example.com) to IP addresses (93.184.216.34). It's a distributed, hierarchical database with caching at every level.  

#### DNS Record Types

| Type      | Purpose                                  | Example                                      |
| --------- | ---------------------------------------- | -------------------------------------------- |
| **A**     | Name → IPv4 address                      | `example.com. → 93.184.216.34`               |
| **AAAA**  | Name → IPv6 address                      | `example.com. → 2606:2800:220:1:...`         |
| **CNAME** | Alias to another name                    | `www.example.com. → example.com.`            |
| **MX**    | Mail server for domain                   | `example.com. → 10 mail.example.com.`        |
| **NS**    | Authoritative nameserver                 | `example.com. → ns1.example.com.`            |
| **TXT**   | Arbitrary text (email-auth records SPF and DKIM, domain verification) | `example.com. → "v=spf1 include:..."`        |
| **SRV**   | Service discovery (host + port)          | `_sip._tcp.example.com. → server:5060`       |
| **PTR**   | Reverse lookup (IP → name)               | `34.216.184.93.in-addr.arpa. → example.com.` |
| **SOA**   | Zone authority, serial, timers           | Metadata about the zone                      |
| **CAA**   | Certificate authority authorization      | Which CAs can issue certs for domain         |
**TTL (Time To Live):** How long a record can be cached (in seconds). Short TTL (60-300s) for records that change often, long TTL (3600-86400s) for stable ones.

#### DNS Resolution Process

**Stub resolver** (on the local host) → **Recursive resolver** (ISP or 8.8.8.8) → **Root servers** → **TLD servers** → **Authoritative nameservers**

```
  Your app              Recursive Resolver        Root       .com TLD      Authoritative
  (stub resolver)       (8.8.8.8, 1.1.1.1)        Server      Server       for example.com
      |                         |                   |            |                |
      |-- example.com? -------->|                   |            |                |
      |                         |                   |            |                |
      |                         |-- Where is .com?->|            |                |
      |                         |<-- Try 192.5.5.241 ------------|                |
      |                         |                                |                |
      |                         |-- Where is example.com? ------>|                |
      |                         |<-- Try ns1.example.com -----------------------  |
      |                         |                                                 |
      |                         |-- A record for example.com? ------------------->|
      |                         |<-- 93.184.216.34, TTL 3600 ---------------------|
      |                         |                                                 |
      |                         | (caches for 3600s)                              |
      |<-- 93.184.216.34 -------|                                                 |
      |                         |                                                 |
```

**Iterative vs. Recursive:**
- **Recursive query:** client asks recursive resolver, which does all the work and returns final answer.
- **Iterative query:** recursive resolver asks root → root says "ask .com" → resolver asks .com → .com says "ask ns1.example.com" → resolver asks ns1 → ns1 returns answer.

Most clients use recursive resolvers (configured via DHCP or manually set to 8.8.8.8, 1.1.1.1, etc.). Recursive resolvers do iterative queries to authoritative servers.

#### Caching and TTL

DNS caching happens at **multiple levels:**
1. **Browser cache:** Chrome, Firefox cache DNS for a few minutes.
2. **OS cache:** `systemd-resolved` (Linux), `mDNSResponder` (macOS), DNS Client service (Windows).
3. **Recursive resolver cache:** 8.8.8.8, 1.1.1.1 cache heavily.
4. **Authoritative server cache:** sometimes caches responses from upstream. 

**Negative caching:** NXDOMAIN (non-existent domain) responses are cached too, controlled by the SOA record's negative TTL. Prevents repeated lookups for typos.
**Cache poisoning:** Attackers inject fake DNS responses into resolver caches. Mitigations: DNSSEC (Domain Name System Security Extensions — cryptographic signatures on records), randomized query IDs, source port randomization.

#### Common DNS Tools and Examples

The command-line invocations and output parsing for these tools are covered in more depth in `Networking linux-commands.md`; this section focuses on what the queries reveal about DNS itself.

**`dig` (domain information groper) — the standard DNS query tool**

Basic query:
```bash
$ dig example.com

; <<>> DiG 9.18.24 <<>> example.com
;; ANSWER SECTION:
example.com.    3600  IN  A 93.184.216.34
```  

Query specific record type:
```bash
$ dig example.com AAAA
$ dig example.com MX
$ dig example.com TXT
```

**Trace full resolution path:**
```bash
$ dig +trace example.com

.     86400 IN  NS  a.root-servers.net.
.     86400 IN  NS  b.root-servers.net.
[... root servers ...]

com.      172800  IN  NS  a.gtld-servers.net.
[... .com TLD servers ...]

example.com.    172800  IN  NS  ns1.example.com.
example.com.    172800  IN  NS  ns2.example.com.  

example.com.    3600  IN  A 93.184.216.34
```

This shows the exact path: root → .com TLD → authoritative.

**Query specific nameserver:**
```bash
$ dig @8.8.8.8 example.com         # Use Google DNS
$ dig @1.1.1.1 example.com         # Use Cloudflare DNS
$ dig @ns1.example.com example.com # Query authoritative directly

```

**Short answer only:**
```bash
$ dig +short example.com
93.184.216.34
```

**Reverse DNS lookup (PTR):**
```bash
$ dig -x 93.184.216.34
# Returns: 34.216.184.93.in-addr.arpa. → (reverse pointer if set)
```

#### Operational Gotchas and Best Practices

**CNAME at apex:** You **cannot** have a CNAME at the zone apex (bare domain). This breaks DNS:
```
Invalid:  example.com.      CNAME  other.com.     # breaks MX, NS, SOA at the apex
Valid:    www.example.com.  CNAME  example.com.
```
**Solution:** Use A/AAAA records at apex, or services like Cloudflare's "CNAME flattening" or AWS Route 53's Alias records.

**Long TTLs during migrations:** With TTL = 86400 (24h), DNS changes take a full day to propagate. Before migrations:
1. Lower TTL to 60-300s a day in advance.
2. Make the change.
3. Wait for old TTL to expire.
4. Raise TTL back to normal.

**DNS propagation myths:** There's no global "propagation delay." It's just caching:
- If TTL = 3600s, old value can persist up to 1 hour in caches.
- Some resolvers ignore TTL and cache longer (bad behavior, but happens).
- `dig +trace` bypasses caches and queries authoritative servers directly.

**Split-horizon DNS:** Different answers based on client location:
- Internal clients (10.0.0.0/8) see `internal.example.com → 10.0.1.50`
- External clients see `internal.example.com → NXDOMAIN` or public IP.

Used for internal services, cloud VPCs, CDNs (geo-based routing).

**Anycast DNS:** Servers at multiple locations share the same IP (e.g., 8.8.8.8). BGP routing sends queries to the nearest POP. Provides low latency and DDoS resilience.

#### DNS and Performance

- **Latency impact:** First request to a domain pays DNS lookup cost (~20-100ms). Subsequent requests use cache.
- **DNS prefetching:** Browsers can prefetch DNS for links on a page: `<link rel="dns-prefetch" href="//cdn.example.com">`
- **HTTP/2 and HTTP/3 connection coalescing:** Multiple domains can share one connection if they resolve to the same IP and have valid certs.

**DNS over HTTPS (DoH) and DNS over TLS (DoT):**
- Traditional DNS queries are **plaintext UDP/53** — ISPs can see and intercept.
- **DoT (port 853):** Encrypted DNS over TLS.
- **DoH (HTTPS):** Encrypted DNS over HTTPS (looks like web traffic).
- Privacy benefit: the ISP cannot see which domains are resolved.
- Supported by browsers (Firefox, Chrome) and resolvers (Cloudflare, Google).

#### Linux DNS Commands

**Check current DNS resolver config:**
```bash
$ cat /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1

# On systemd systems:
$ resolvectl status
```

**Flush DNS cache:**
```bash
# systemd-resolved:
$ sudo resolvectl flush-caches

# macOS:
$ sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# Windows:
> ipconfig /flushdns
```

**Test DNS with `nslookup` (legacy but still common):**
```bash
$ nslookup example.com
Server:   8.8.8.8
Address:  8.8.8.8#53

Name: example.com
Address: 93.184.216.34
```

**Test DNS with `host` (simpler output):**
```bash
$ host example.com
example.com has address 93.184.216.34
example.com has IPv6 address 2606:2800:220:1:...
```

---
## 8. Network Security

### Firewalls

|Type|Layer|What it does|
|---|---|---|
|Packet filter|L3/L4|Allow/deny by IP, port, protocol. Stateless.|
|Stateful firewall|L3/L4|Tracks connections. Only allows return traffic for established sessions.|
|WAF (Web Application Firewall)|L7|Inspects HTTP content. Blocks SQLi (SQL Injection), XSS (Cross-Site Scripting), etc.|

In cloud: **security groups** (stateful, per-instance) and **NACLs (Network Access Control Lists)** (stateless, per-subnet).

### Common Attack Types

|Attack|Layer|How it works|Mitigation|
|---|---|---|---|
|ARP spoofing|L2|Fake ARP replies redirect traffic|DAI (Dynamic ARP Inspection), 802.1X|
|MAC flooding|L2|Overflow switch table -> broadcast mode|Port security|
|DNS spoofing|L7|False DNS responses|DNSSEC, DoH/DoT|
|Man-in-the-Middle|Any|Intercept/modify traffic|TLS, cert pinning|
|DDoS (Distributed Denial of Service)|L3/L4/L7|Overwhelm with traffic|Rate limiting, CDN, scrubbing|
|SYN flood|L4|Exhaust server connection state|SYN cookies|
|BGP hijack|L3|Announce someone else's prefixes|RPKI (Resource Public Key Infrastructure), monitoring|

### Security Concepts

- **Zero Trust:** never trust based on network location. Authenticate and authorize every request, even inside the perimeter.
- **mTLS (mutual TLS):** both client and server present certificates. Standard in service mesh (Istio, Linkerd).
- **VPN (Virtual Private Network):** encrypted tunnel. IPsec (L3) or WireGuard (modern, simpler). Connects remote users or sites securely.
- **Network segmentation:** VLANs, subnets, security groups. Limit blast radius of a compromise.
- **IDS/IPS (Intrusion Detection System / Intrusion Prevention System):** IDS monitors and alerts, IPS monitors and blocks. Can be network-based or host-based.

---

## 9. System Design Networking

### Latency vs. Bandwidth

**Bandwidth** = data per second (pipe diameter). **Latency** = time for one bit to arrive (pipe length). High bandwidth doesn't help latency-sensitive workloads (many sequential small requests).

**BDP (Bandwidth-Delay Product)** = bandwidth x RTT. The amount of data "in flight" to fully utilize a link. TCP's window must be at least this large.

### Speed of Light — The Hard Limit

Light in fiber travels at ~200,000 km/s. NY to London (~5,500 km) = ~28ms one-way, ~56ms RTT minimum. No protocol beats physics. This is why CDNs (Content Delivery Networks), edge computing, and geo-distributed architectures exist.

### Connection Pooling

TCP+TLS = 2-3 RTTs per new connection. **Always reuse connections:** HTTP keep-alive, database connection pools, gRPC persistent channels. New connections also start in TCP slow start, so throughput ramps up gradually.

### Head-of-Line (HOL) Blocking

A pattern that repeats across protocols:

- **HTTP/1.1:** one request blocks the connection.
- **HTTP/2 over TCP:** one lost packet stalls all multiplexed streams.
- **HTTP/3 / QUIC:** per-stream loss recovery. Solved.

### Load Balancing

|Type|Layer|Routes by|Examples|
|---|---|---|---|
|L4|Transport|IP/port (no payload inspection)|AWS NLB, HAProxy TCP mode|
|L7|Application|HTTP headers, paths, cookies|AWS ALB, Nginx, Envoy|

### Timeouts

Every network boundary needs them: **connection timeout** (handshake), **read timeout** (waiting for data), **idle timeout** (unused connection). Missing or misconfigured timeouts are one of the most common causes of cascading failures in distributed systems.

### The Fallacies of Distributed Computing

Eight false assumptions developers make:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every one of these fails in production. Design for failure: retries with exponential backoff, circuit breakers, idempotent operations.

---

## 10. Debugging & Tools

### Essential Toolkit

|Tool|What it does|Quick example|
|---|---|---|
|`ping`|ICMP echo — reachability + RTT|`ping -c 4 example.com`|
|`traceroute` / `mtr`|Path and per-hop latency|`mtr example.com`|
|`dig`|DNS queries|`dig +short example.com`|
|`curl -v`|HTTP with headers + TLS info|`curl -v https://example.com`|
|`ss` / `netstat`|Socket state, listening ports|`ss -tlnp`|
|`tcpdump`|Packet capture|`tcpdump -i eth0 port 443 -w cap.pcap`|
|`openssl s_client`|TLS connection test|`openssl s_client -connect host:443`|
|`nmap`|Port scanning|`nmap -sT -p 80,443 host`|

### Debugging Checklist: "It Doesn't Connect"

```
  1. DNS      ->  does the name resolve? right IP?        dig example.com
  2. REACH    ->  can you reach the IP?                   ping / mtr
  3. PORT     ->  is the port open? service listening?    nc -zv host 443 / ss -tlnp
  4. TLS      ->  valid cert? right hostname? expired?    openssl s_client
  5. APP      ->  is the app returning errors?            curl -v
  6. FIREWALL ->  security groups / iptables / NACLs?     check cloud console
```

### Metrics to Monitor

- **RTT distribution** (p50, p95, p99) by destination.
- **Packet loss** — even 0.1% significantly impacts TCP throughput.
- **TCP retransmissions** — high rate = network or buffer problem.
- **Connection errors** — refused / timed out / reset.
- **DNS resolution time** — slow DNS = slow everything.
- **TIME_WAIT count** — excessive = ephemeral port exhaustion risk.