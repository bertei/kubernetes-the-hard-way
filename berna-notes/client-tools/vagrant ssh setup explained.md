# Vagrant SSH Setup - Deep Dive

**Goal:** Understand SSH key generation, network interfaces, and how Vagrant automates trust

---

## Concept 1: Network Interfaces (`enp0s8`)

You asked: **"What is `enp0s8`? Why not just `eth0`?"**

### What is a Network Interface?

A **network interface** is a virtual or physical "door" on your machine for sending/receiving data.

**In Linux/Unix naming:**
- Old style: `eth0`, `eth1`, `eth2` (Ethernet 0, 1, 2...)
- New style: `enp0s8`, `enp0s3`, `wlan0` (more descriptive)

**Breaking down `enp0s8`:**
```
enp0s8
│  │ └─ 8 = slot/device number
│  └──── s = PCI slot
└─────── enp = Ethernet, PCI, onboard
```

**In your VMs (from Vagrantfile):**
```ruby
node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"
```

This creates TWO network interfaces:
- **enp0s3** = NAT adapter (Adapter 1) → Internet access
- **enp0s8** = Host-only adapter (Adapter 2) → Private cluster network `192.168.56.x`

**Why you check `enp0s8`:**
- That's the one connected to your K8s cluster network
- That's where `192.168.56.11` lives (controlplane01's private IP)

### The `ip addr` Command (Legendary Networking Troubleshooting)

**What it does:** Lists all network interfaces and their IP configurations

````bash
ip addr show enp0s8
# OR
ip addr show
# OR (old-style, still works)
ifconfig enp0s8
````

### Extended: Complete Network Interface Breakdown

Your output shows THREE network interfaces on controlplane01:

#### Interface 1: `lo` (Loopback)

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
```

**What it is:**
- `lo` = "localhost" interface (virtual, not physical)
- IP: `127.0.0.1` (always loopback)
- Purpose: Internal machine communication (like talking to yourself)

**Why you have it:**
- Every Linux machine has this
- Used for internal services (databases, APIs running locally)
- If you run a web server on `localhost:8080`, it uses the `lo` interface

**Example:**
```bash
# Talking to yourself (loopback)
curl http://127.0.0.1:8080
# Uses lo interface internally

# Talking to external IP
curl http://192.168.56.12:8080
# Uses enp0s8 interface (goes out to network)
```

**Key facts:**
- `mtu 65536` = Max packet size (larger than normal because it's virtual)
- `scope host` = Only visible on THIS machine (not on network)
- `state UNKNOWN` = Loopback doesn't care about physical state

---

#### Interface 2: `enp0s3` (NAT - Outbound Internet)

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 02:93:de:ec:7c:d5 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
```

**What it is:**
- NAT = Network Address Translation
- Purpose: Access to OUTSIDE internet
- Connected to: Virtual NAT router inside VirtualBox
- IP: `10.0.2.15` (inside VirtualBox's NAT network, not reachable from your host)

**Breaking it down:**

| Field | Meaning |
|-------|---------|
| `link/ether 02:93:de:ec:7c:d5` | MAC address (physical layer ID, like device serial #) |
| `inet 10.0.2.15/24` | Your IP on NAT network (assigned by VirtualBox DHCP) |
| `metric 100` | Route priority (lower = preferred) |
| `dynamic` | Got this IP via DHCP (not static) |
| `scope global` | Reachable globally (within its network) |

**What can it reach?**
```bash
# From inside VM:
ping 8.8.8.8          # ✅ Works (goes through NAT)
apt-get update        # ✅ Works (downloads from internet)

# From your Windows host:
ping 10.0.2.15        # ❌ Doesn't work (NAT network not visible to host)
```

**Why `10.0.2.x` specifically?**
- VirtualBox reserves `10.0.2.0/24` for NAT networks
- `10.0.2.1` = Virtual router (VirtualBox magic)
- `10.0.2.2` = Your Windows host (from VM's perspective)
- `10.0.2.15` = This VM (DHCP assigned)

**Use case:** When you do `apt-get install`, it uses enp0s3 to reach Ubuntu repositories on the internet.

---

#### Interface 3: `enp0s8` (Host-only - Cluster Network)

```
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 08:00:27:6a:95:47 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.11/24 brd 192.168.56.255 scope global enp0s8
```

**What it is:**
- Host-only = Private network between your Windows host + all VMs
- Purpose: K8s cluster communication
- IP: `192.168.56.11` (static, defined in Vagrantfile)

**Breaking it down:**

| Field | Meaning |
|-------|---------|
| `link/ether 08:00:27:6a:95:47` | MAC address (unique per VM) |
| `inet 192.168.56.11/24` | Your IP on cluster network |
| `brd 192.168.56.255` | Broadcast address (reach ALL on network) |
| `scope global` | Reachable globally (on this network) |
| `forever` | IP is permanent (static, not DHCP) |

**What can it reach?**
```bash
# From inside controlplane01:
ping 192.168.56.12    # ✅ Works (controlplane02, same network)
ping 192.168.56.21    # ✅ Works (node01, same network)
ping 192.168.56.30    # ✅ Works (loadbalancer, same network)

# From your Windows host:
ping 192.168.56.11    # ❌ Doesn't work (port forwarding required)
# But via port forward:
ssh localhost -p 2711 # ✅ Works (tunnels to 192.168.56.11:22)
```

**Why static IP?**
- K8s needs stable IPs (can't have controlplane01 change IP randomly)
- Defined in Vagrantfile: `IP_NW + "#{MASTER_IP_START + i}"`

**Use case:** All Kubernetes cluster traffic (etcd, API server, kubelet, pod-to-pod) flows through this network.

---

### Concept 2: IP Routing (`ip route`)

You asked: "What does this mean?"

```
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
10.0.2.3 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.11
```

**What is routing?**

Routing = "How do I get my packets to the destination?"

When you type `ping 8.8.8.8`, your machine asks: "Which interface should I use?"

The routing table is the ANSWER KEY.

**Breaking down each route:**

#### Route 1: Default Route (Everything else)
```
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
```

**Plain English:**
- **default** = "If you don't match any other route, use THIS one"
- **via 10.0.2.2** = "Send packets to VirtualBox's NAT router"
- **dev enp0s3** = "Use the NAT interface"
- **src 10.0.2.15** = "Mark packets as coming from me"
- **metric 100** = "Priority: 100 (lower is better)"

**Use case:** Internet traffic (8.8.8.8, google.com, etc.)

```bash
# You type:
ping 8.8.8.8

# Kernel looks at routing table:
# "8.8.8.8 doesn't match 10.0.2.0/24 or 192.168.56.0/24"
# "Use DEFAULT route!"
# "Send via 10.0.2.2 (NAT router) on enp0s3"
```

#### Route 2: Local NAT Network
```
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
```

**Plain English:**
- **10.0.2.0/24** = "Packets for anything in 10.0.2.x"
- **dev enp0s3** = "Use NAT interface"
- **proto kernel** = "Kernel added this (not manually configured)"

**Use case:** Reaching other VMs on NAT network (rarely used in K8s setup)

#### Route 3: Cluster Network (MOST IMPORTANT FOR YOU)
```
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.11
```

**Plain English:**
- **192.168.56.0/24** = "Packets for anything in 192.168.56.x"
- **dev enp0s8** = "Use the cluster interface"
- **src 192.168.56.11** = "Mark packets as from me"

**Use case:** All K8s cluster traffic!

```bash
# You type (from controlplane01):
ping 192.168.56.12

# Kernel looks at routing table:
# "192.168.56.12 matches 192.168.56.0/24"
# "Send via enp0s8"
# "Direct connection (no router needed, same network)"
```

---

### Visualization: How Data Flows

```
controlplane01 (192.168.56.11)

Internet Traffic (ping 8.8.8.8):
┌─────────────────────────────────────────────────┐
│ 1. Kernel checks routing table                  │
│ 2. "8.8.8.8 not in my networks"                │
│ 3. Use DEFAULT route: via 10.0.2.2 dev enp0s3 │
│ 4. Packet goes out enp0s3 (NAT)                │
│ 5. VirtualBox NAT router forwards to internet  │
└─────────────────────────────────────────────────┘

Cluster Traffic (ping 192.168.56.12):
┌─────────────────────────────────────────────────┐
│ 1. Kernel checks routing table                  │
│ 2. "192.168.56.12 matches 192.168.56.0/24"   │
│ 3. Use: dev enp0s8                             │
│ 4. Packet goes out enp0s8 (Host-only)         │
│ 5. Direct connection to controlplane02         │
└─────────────────────────────────────────────────┘
```

---

### Key Networking Facts

**Summary Table:**

| Interface | Purpose | Network | Reachable From |
|-----------|---------|---------|-----------------|
| **lo** | Loopback (internal) | 127.0.0.1 | Only this machine |
| **enp0s3** | Internet (NAT) | 10.0.2.0/24 | Other VMs (rarely), Internet |
| **enp0s8** | Cluster (Host-only) | 192.168.56.0/24 | All VMs, Your Windows host (via port forward) |

**Troubleshooting Sequence:**

```bash
# Step 1: Check interfaces exist
ip addr show

# Step 2: Check routing table
ip route

# Step 3: Test connectivity
ping 192.168.56.12         # Cluster network
ping 8.8.8.8               # Internet

# Step 4: See which interface is used
ip route get 192.168.56.12 # Shows which interface for this destination

# Step 5: Monitor actual traffic
tcpdump -i enp0s8          # Watch cluster network traffic
```

---

### Why Three Interfaces?

**You might ask:** "Why not just one interface?"

**Answer:** Different purposes need different networks:

1. **Loopback** = Internal services (database on localhost:5432)
2. **NAT** = Internet access (apt-get, docker pulls)
3. **Host-only** = Secure cluster communication (K8s nodes talking to each other)

**Analogy:** Your house has multiple doors
- **Back door** = You to yourself (loopback)
- **Front door** = You to the street/internet (NAT)
- **Private hallway door** = You to family rooms (Host-only cluster network)

---