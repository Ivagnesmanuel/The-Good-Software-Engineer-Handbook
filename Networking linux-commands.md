## Linux Networking Commands Reference

Organized by layer and use case. Commands used in practice for debugging.

---

### Layer 2 — Data Link

**Show interfaces and MAC addresses:**
```bash
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
```

**ARP table (IP → MAC mappings):**
```bash
$ ip neigh show
# or legacy:
$ arp -a

# Output:
10.0.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

**Flush ARP cache (force re-learning):**
```bash
$ sudo ip neigh flush dev eth0
```

**Bridge and VLAN info (advanced):**
```bash
$ bridge link show           # Show bridge ports
$ bridge fdb show            # Forwarding database (MAC table)
$ bridge vlan show           # VLAN assignments
```

---

### Layer 3 — Network (IP, Routing)

**Show IP addresses:**
```bash
$ ip addr show
# or legacy:
$ ifconfig

# Output shows IP, subnet, broadcast for each interface
```

**Routing table:**
```bash
$ ip route show
# or legacy:
$ route -n

# Output:
default via 10.0.1.1 dev eth0 proto dhcp metric 100
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.10
```

**Add/delete routes:**
```bash
$ sudo ip route add 192.168.2.0/24 via 10.0.1.254
$ sudo ip route del 192.168.2.0/24
```

**Show all routing tables (including policy routing):**
```bash
$ ip route show table all
$ ip rule list              # Show routing policy rules
```

**Ping (ICMP Echo Request/Reply):**
```bash
$ ping -c 4 8.8.8.8         # Send 4 packets
$ ping -I eth0 8.8.8.8      # Use specific interface
```

**Traceroute (map packet path):**
```bash
$ traceroute 8.8.8.8        # UDP probes (Linux default)
$ traceroute -I 8.8.8.8     # ICMP probes (like Windows)
$ traceroute -T 8.8.8.8     # TCP probes (port 80)

# Better alternative (continuous):
$ mtr 8.8.8.8               # Live, updates like top
```

**Path MTU discovery test:**
```bash
$ ping -M do -s 1472 8.8.8.8   # Don't fragment, test MTU
# 1472 data + 28 ICMP/IP header = 1500 bytes
```

---

### Layer 4 — Transport (TCP, UDP)

**Show all sockets (TCP, UDP, listening, established):**
```bash
$ ss -tuln                   # TCP + UDP, listening, numeric
$ ss -tunap                  # Add processes (requires root)

# Legacy equivalent:
$ netstat -tuln
```

**TCP connection details with metrics:**
```bash
$ ss -ti                     # TCP with detailed info
# Shows: RTT, congestion window, retransmits, MSS, etc.

# Output example:
ESTAB   0  0   10.0.1.10:54321   8.8.8.8:443
	 cubic wscale:7,7 rto:204 rtt:3.5/1.2 cwnd:10 ssthresh:7 send 31.4Mbps
```

**Watch TCP states in real-time:**
```bash
$ watch -n 1 'ss -tan | grep ESTAB | wc -l'   # Count ESTABLISHED
$ ss -tan state time-wait | wc -l             # Count TIME-WAIT
```

**TCP/UDP statistics:**
```bash
$ ss -s                      # Summary statistics
# Shows total sockets, TCP states breakdown, UDP, etc.

$ netstat -s                 # Detailed protocol statistics
# Shows segments sent/received, retransmits, errors, etc.
```

**Check which process is using a port:**
```bash
$ sudo ss -tlnp | grep :80
$ sudo lsof -i :80           # Alternative
$ sudo fuser 80/tcp          # Another option
```

---

### DNS Resolution

**Query DNS (dig - most powerful):**
```bash
$ dig example.com            # Basic A record query
$ dig example.com AAAA       # IPv6 address
$ dig example.com MX         # Mail servers
$ dig example.com NS         # Nameservers
$ dig example.com ANY        # All records (often blocked now)

$ dig +short example.com     # Just the answer
$ dig +trace example.com     # Full resolution path
$ dig @8.8.8.8 example.com   # Use specific nameserver
```

**Check current DNS resolver:**
```bash
$ cat /etc/resolv.conf       # Legacy

# systemd-resolved:
$ resolvectl status
$ resolvectl query example.com
```

**Flush DNS cache:**
```bash
$ sudo resolvectl flush-caches          # systemd-resolved
$ sudo systemd-resolve --flush-caches   # older systemd
```

**Alternative DNS tools:**
```bash
$ host example.com           # Simple output
$ nslookup example.com       # Interactive/legacy
```

---

### Packet Capture and Analysis

**Capture packets (tcpdump):**
```bash
$ sudo tcpdump -i eth0                    # Capture on eth0
$ sudo tcpdump -i any                     # All interfaces
$ sudo tcpdump -i eth0 -c 100             # Capture 100 packets
$ sudo tcpdump -i eth0 -w capture.pcap    # Save to file

# Filters:
$ sudo tcpdump port 443                   # HTTPS traffic
$ sudo tcpdump host 8.8.8.8               # Specific host
$ sudo tcpdump tcp and port 80            # HTTP
$ sudo tcpdump 'icmp[icmptype]=icmp-echo' # Ping requests
$ sudo tcpdump -n net 10.0.1.0/24         # Subnet
```

**Read pcap file:**
```bash
$ tcpdump -r capture.pcap
$ tcpdump -r capture.pcap -A             # ASCII payload
$ tcpdump -r capture.pcap -X             # Hex + ASCII
```

**Wireshark (GUI):**
```bash
$ wireshark capture.pcap &               # Open file
$ sudo wireshark &                        # Live capture (requires root)
```

**Display filters (Wireshark syntax in tshark):**
```bash
$ tshark -r capture.pcap -Y "tcp.port == 443"
$ tshark -r capture.pcap -Y "http.request.method == GET"
$ tshark -r capture.pcap -Y "dns.qry.name contains google"
```

---

### Network Performance and Diagnostics

**Bandwidth testing (iperf3):**
```bash
# On server:
$ iperf3 -s

# On client:
$ iperf3 -c server_ip                    # TCP test
$ iperf3 -c server_ip -u -b 100M         # UDP test, 100 Mbps
$ iperf3 -c server_ip -R                 # Reverse (download)
```

**Network interface statistics:**
```bash
$ ip -s link show eth0                   # Packets, bytes, errors
$ ethtool eth0                           # Speed, duplex, link status
$ ethtool -S eth0                        # Driver statistics
```

**Connection tracking (conntrack, for NAT/firewall):**
```bash
$ sudo conntrack -L                      # List tracked connections
$ sudo conntrack -L -p tcp --state ESTABLISHED
```

---

### Firewall and NAT

**View iptables rules:**
```bash
$ sudo iptables -L -n -v                 # List all rules
$ sudo iptables -t nat -L -n -v          # NAT table
$ sudo iptables -t mangle -L -n -v       # Mangle table
```

**Modern nftables (replaces iptables):**
```bash
$ sudo nft list ruleset                  # Show all rules
$ sudo nft list table inet filter
```

**Check NAT translations (on router/NAT gateway):**
```bash
$ sudo conntrack -L -p tcp --state ESTABLISHED
# Shows: original src/dst → translated src/dst
```

---

### System Network Configuration

**Reload network config (systemd-networkd):**
```bash
$ sudo networkctl reload
$ sudo networkctl status
```

**Reload network (NetworkManager):**
```bash
$ sudo nmcli connection reload
$ sudo nmcli connection up eth0
```

**Show active connections:**
```bash
$ nmcli connection show --active
$ nmcli device status
```

**Edit network config files:**
```bash
# systemd-networkd:
/etc/systemd/network/*.network

# NetworkManager:
/etc/NetworkManager/system-connections/

# Legacy (Debian/Ubuntu):
/etc/network/interfaces
```

---

### Advanced / Kernel Tuning

**Kernel network parameters (sysctl):**
```bash
$ sysctl net.ipv4.ip_forward              # IP forwarding enabled?
$ sysctl net.ipv4.tcp_congestion_control  # Which algorithm?

# Set temporarily:
$ sudo sysctl -w net.ipv4.ip_forward=1

# Set permanently:
$ echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
$ sudo sysctl -p
```

**Common tuning parameters:**
```bash
net.ipv4.tcp_tw_reuse=1               # Reuse TIME_WAIT sockets
net.ipv4.tcp_fin_timeout=30           # Faster FIN timeout
net.core.somaxconn=4096               # Backlog queue size
net.ipv4.tcp_max_syn_backlog=8192     # SYN queue size
net.core.netdev_max_backlog=5000      # NIC receive queue
```

**Socket buffer sizes:**
```bash
net.core.rmem_max=16777216            # Max receive buffer
net.core.wmem_max=16777216            # Max send buffer
net.ipv4.tcp_rmem=4096 87380 16777216 # TCP read buffer (min/default/max)
net.ipv4.tcp_wmem=4096 65536 16777216 # TCP write buffer
```

---

### Quick Debugging Workflow

**Problem: "Can't reach server X"**

1. **Check local interface:**
   ```bash
   $ ip addr show
   $ ip link show
   ```

2. **Check routing:**
   ```bash
   $ ip route show
   $ ip route get 8.8.8.8   # Show route for specific dest
   ```

3. **Check gateway reachable:**
   ```bash
   $ ping -c 2 10.0.1.1     # Gateway
   ```

4. **Check external connectivity:**
   ```bash
   $ ping -c 2 8.8.8.8
   ```

5. **Check DNS:**
   ```bash
   $ dig example.com
   $ cat /etc/resolv.conf
   ```

6. **Trace path:**
   ```bash
   $ mtr example.com
   ```

7. **Check specific port open:**
   ```bash
   $ nc -zv example.com 443          # Test TCP port
   $ telnet example.com 443          # Alternative
   $ timeout 2 bash -c '</dev/tcp/example.com/443' && echo "Open"
   ```

8. **Check listening services:**
   ```bash
   $ sudo ss -tlnp | grep :80
   ```

9. **Capture traffic to see what's happening:**
   ```bash
   $ sudo tcpdump -i eth0 host example.com -n
   ```

---

**WHERE TO PASTE:** Add this as a new section **"11. Linux Networking Commands"** at the very end of your `Networking-Fundamentals.md`, after "10. Debugging & Tools" (or replace that section if you want this as the practical reference).
