# Lab 03: Client Tools - Setup & Theory

**Goal:** Install kubectl, cfssl, and cfssljson on Windows host  
**Date Started:** Feb 2026  
**Status:** In Progress

---

## Theory: Understanding the Foundation

Before we install tools, let's understand **why we need them** and how they fit into your cluster.

### Concept 1: What is a "Client Tool"?

A **client tool** is a program that lets you **talk to a server** from your computer.

**Examples:**
- **kubectl** = client tool for Kubernetes (like texting the cluster)
- **curl** = client tool for HTTP (like texting a website)
- **ssh** = client tool for secure shell (like texting a remote machine)

**In your case:**
- Your Windows 11 PC = client (user)
- Your K8s cluster (on VMs) = server
- **kubectl** = the translator between you and the cluster

When you type `kubectl get nodes`, kubectl:
1. Reads your kubeconfig file (location & credentials)
2. Connects to API server at `192.168.56.30:6443` (load balancer)
3. Sends "GET nodes" request
4. Displays response

---

### Concept 2: Virtualization & Vagrant (Quick Review)

You said: "Vagrant spins up like Docker images but for virtual machines"

**More accurate:** Vagrant is a **wrapper** that automates VM creation. Here's the layer cake:

```
┌─────────────────────────────────┐
│   Your Windows 11 PC            │
│   (Host Operating System)       │
└──────────────┬──────────────────┘
               │ Uses
┌──────────────┴──────────────────┐
│   VirtualBox                    │
│   (Hypervisor - creates VMs)    │
│                                 │
│  ┌─────────────────────────────┐│
│  │ VM 1: controlplane01        ││
│  │ - Ubuntu OS (guest)         ││
│  │ - CPU: 2-4 cores (allocated)││
│  │ - RAM: 2GB-8GB (allocated)  ││
│  │ - Disk: ~20GB virtual       ││
│  └─────────────────────────────┘│
│  ┌─────────────────────────────┐│
│  │ VM 2: controlplane02        ││
│  │ - Ubuntu OS (guest)         ││
│  └─────────────────────────────┘│
│  ... (more VMs)                 │
└─────────────────────────────────┘
         ↑
    Vagrant automates this
    (reads Vagrantfile, spins up VMs)
```

**Key point:** Vagrant is just **automating the boring parts** (IP assignment, port forwarding, DNS setup). VirtualBox is what actually creates the virtual computers.

---

### Concept 3: Networking - Host-only Adapter

You asked: **"Why adapter? What devices can be in my local network?"**

#### What is an "Adapter"?

An **adapter** is a **network interface card (NIC)**—a physical or virtual door through which data flows in/out.

**In the real world:**
- Your laptop has WiFi adapter (connects to WiFi router)
- Your laptop has Ethernet adapter (connects via cable)
- A server might have 2-4 NICs (for redundancy, different networks)

**In your VMs:**
- **Adapter 1 (NAT):** Door to the internet (outbound only, for `apt-get`, docker pulls)
- **Adapter 2 (Host-only):** Private door just for your VMs to talk to each other (and to your host)

```
Your Windows Host
  ├─ WiFi Adapter → WiFi Router → Internet
  └─ VirtualBox Host-only Adapter → Private Network 192.168.56.x
                                      ├─ controlplane01 (192.168.56.11)
                                      ├─ controlplane02 (192.168.56.12)
                                      ├─ loadbalancer (192.168.56.30)
                                      ├─ node01 (192.168.56.21)
                                      └─ node02 (192.168.56.22)
```

#### What Devices Can Be on Your Local Network?

**Any device with a NIC:** (Network Interface Card)

- **Physical machines:** Your laptop, desktop, phone (on WiFi)
- **Virtual machines:** Your 5 K8s VMs
- **Network devices:** Router, printer, smart TV
- **Servers:** Database servers, web servers

**In YOUR case (host-only `192.168.56.x`):**
- Only your Windows host + the 5 VMs can see each other
- This network is **isolated** from the internet
- This is **by design**—your cluster is private and secure

#### What is an IP Address?

IP = **Internet Protocol address** = unique ID on a network

**Analogy:** Like a home address for devices
- `192.168.56.11` = controlplane01's address
- `192.168.56.12` = controlplane02's address
- When controlplane01 wants to send data to controlplane02, it uses `192.168.56.12` as the destination

---

### Concept 4: CIDR Notation (`/24`)

You said: "I forgot how this is calculated"

**CIDR** = **Classless Inter-Domain Routing** (fancy name for "network math")

**What `/24` means:**
- Network: `192.168.56.0/24`
- `/24` = "the first 24 bits are the network, the last 8 bits are for hosts"
- Binary breakdown:
  ```
  192.168.56.0 = 11000000.10101000.00111000.00000000
  /24 = ^^^^^^^^  ^^^^^^^^  ^^^^^^^^  
        (network part, fixed)     (host part, variable)
  ```

**How many IPs?**
- Last 8 bits = 2^8 = 256 addresses total
- But first and last are reserved (network & broadcast)
- **Usable: 254 IPs** (`.1` to `.254`)

**In your Vagrantfile:**
```ruby
IP_NW = "192.168.56."  # Prefix (network part)

MASTER_IP_START = 10   # Start at .10
# controlplane01: 192.168.56.11 (10+1)
# controlplane02: 192.168.56.12 (10+2)

NODE_IP_START = 20     # Start at .20
# node01: 192.168.56.21 (20+1)
# node02: 192.168.56.22 (20+2)

LB_IP_START = 30       # Fixed
# loadbalancer: 192.168.56.30
```

**Why this matters for Kubernetes:**
- Your cluster needs a private network where all nodes can talk to each other
- The `/24` gives you plenty of room (254 IPs) for growth
- All K8s traffic stays on this private network (secure, no internet exposure)

---

### Concept 5: SSH - Secure Shell (Trust & Keys)

You asked: **"How does SSH key authentication work? How is trust established?"**

#### Problem SSH Solves

**Scenario:** You're on your Windows PC. You want to run commands on controlplane01 (a VM somewhere in VirtualBox).

**Without encryption:**
```
You: "ssh vagrant@controlplane01 -p 2711"
You type password: "password123"
→ Password travels OVER THE NETWORK in plain text
→ Anyone watching could steal it!
```

**This is bad.** SSH fixes it with cryptography.

#### How SSH Key Authentication Works

**The Idea:** Instead of passwords (which travel over network), use **two linked keys**:
- **Public Key** = lock (anyone can have it, safe to share)
- **Private Key** = key (ONLY you have it, never share)

**How it works:**

```
SETUP (one time):
┌──────────────────────────────┐
│ On your Windows PC            │
│                              │
│ Generate key pair:           │
│ ssh-keygen                   │
│ ├─ Private key: id_rsa       │ ← KEEP SECRET (like password)
│ └─ Public key: id_rsa.pub    │ ← Share with servers
└──────────────────────────────┘
                │
                │ Copy public key to VM
                ▼
┌──────────────────────────────┐
│ On controlplane01 VM          │
│                              │
│ ~/.ssh/authorized_keys       │
│ (contains your public key)   │
└──────────────────────────────┘


AUTHENTICATION (every time you SSH):
┌──────────────────────────────┐
│ You on Windows:              │
│ ssh -i id_rsa vagrant@...   │
└──────────────────────────────┘
        │
        │ Sends: "I have private key, trust me"
        │        (doesn't send actual key!)
        ▼
┌──────────────────────────────┐
│ Server (controlplane01):      │
│ "Do you have the key to      │
│  unlock my public key lock?" │
│                              │
│ Checks: Does your proof      │
│ match my public key?         │
│ ✅ Match! You're in!         │
└──────────────────────────────┘
```

**Why this is secure:**
- Private key NEVER travels over network (stays on your PC)
- Server ONLY stores public key (even if hacked, doesn't compromise you)
- Cryptography proves you have the private key without revealing it
- **No password needed!** Just the key pair.

#### SSH in Your Vagrant Setup

From `Vagrantfile`:
```ruby
node.vm.network "forwarded_port", guest: 22, host: "#{2710 + i}"
```

This means:
- VM's SSH port (22) → Your host port (2711, 2712, etc.)
- Port forwarding = "translate my request through this tunnel"

**How you SSH in:**
```bash
# Vagrant auto-generates key pair and puts it in .vagrant/machines/
vagrant ssh controlplane01
# Internally does: ssh -i .vagrant/machines/controlplane01/.../private_key vagrant@localhost -p 2711
```

**Or manually:**
```bash
ssh -i .vagrant/machines/controlplane01/virtualbox/private_key vagrant@localhost -p 2711
```

**Key files Vagrant creates:**
```
.vagrant/machines/controlplane01/virtualbox/
├── private_key    ← Your secret key (chmod 600)
└── (public key is on VM at ~/.ssh/authorized_keys)
```

---

### Concept 6: Why We Need Client Tools

**kubectl, cfssl, cfssljson** = command-line tools to manage Kubernetes

| Tool | Purpose | When You'll Use It |
|------|---------|-------------------|
| **kubectl** | Talk to K8s cluster | Every day: `kubectl get nodes`, `kubectl apply -f` |
| **cfssl** | Create certificates (CA, server certs) | Lab 04: Certificate Authority |
| **cfssljson** | Convert cert outputs to JSON | Lab 04: Cert generation |

**Why on your HOST (Windows), not on VMs?**
- Easier to work with (familiar terminal)
- Generate certs once, copy to VMs
- Manage cluster from your PC (like a DevOps engineer)

---

// ...existing code...

### Concept 7: Port Forwarding (Bridge Between Networks)

You asked: **"Port forwarding in K8s is like tunneling a pod to my local machine, right?"**

**YES! Exactly right.** Port forwarding works the same way in three scenarios:
1. **Vagrant/VirtualBox** (what you're doing now)
2. **Kubernetes** (what you'll do in Phase 3)
3. **SSH tunnels** (advanced, but same principle)

Let me show you the pattern.

#### The Core Concept: Bridging Isolated Networks

**Problem:** You have two networks that can't see each other (or can't easily see each other):
- Network A: Your Windows host (`192.168.1.x` or DHCP)
- Network B: Your VirtualBox private network (`192.168.56.x`)

**Devices on Network B are invisible to Network A** (no route between them).

**Solution:** Use a **bridge/tunnel** that listens on Network A and forwards traffic to Network B.

```
Your Windows Host              VirtualBox Private Network
(Network A)                    (Network B)
192.168.1.100                  192.168.56.11 (controlplane01)
    │                                   ▲
    │ Port Forwarding Bridge            │
    │ localhost:2711 ──────────────────▶ SSH port 22
    │                                   │
    └─ You type: ssh localhost -p 2711  │
       Bridge translates:               │
       "Oh, that's controlplane01:22"  │
       and forwards it ───────────────▶│
```

**In plain English:**
- You say: "Connect to **localhost:2711**" (Network A)
- Bridge says: "Oh, that maps to **192.168.56.11:22**" (Network B)
- Bridge forwards your traffic through the tunnel
- You're now talking to controlplane01

#### Scenario 1: Vagrant Port Forwarding (What You're Doing Now)

From your `Vagrantfile`:
```ruby
node.vm.network "forwarded_port", guest: 22, host: "#{2710 + i}"
```

**What this does:**

```
Your Windows Host                VirtualBox VM (controlplane01)
localhost:2711                   192.168.56.11:22 (SSH)
    │                                    ▲
    │ PORT FORWARDING                    │
    │ "Anything on host port 2711"       │
    │ gets tunneled to guest port 22"    │
    │                                    │
    └──────────────────────────────────▶│

When you type:
$ ssh vagrant@localhost -p 2711

It's translated to:
$ ssh vagrant@192.168.56.11:22
```

**Why?** VirtualBox creates the private network (`192.168.56.x`) which is ISOLATED. You can't directly reach it from Windows. Port forwarding is the bridge.

**Multiple VMs = Multiple Ports:**
```
Host Port → Guest Port & IP
2711 → controlplane01:22
2712 → controlplane02:22
2721 → node01:22
2722 → node02:22
2730 → loadbalancer:22
```

Each VM needs a unique host port because **one port on your Windows machine can only forward to one place at a time**.

---

#### Scenario 2: Kubernetes Port-Forward (What You'll Do in Phase 3)

**You have a pod running in K8s cluster** (on `192.168.56.x` network):
```
Pod: my-web-app
├─ Namespace: default
├─ Running on: node01 (192.168.56.21)
└─ Pod IP: 10.244.1.5 (pod network, even MORE isolated!)
└─ Port: 8080 (inside the pod)
```

**Problem:** You want to access this pod from your Windows browser at `http://localhost:8080`

**But the pod IP `10.244.1.5` is invisible to you!** (It's on the pod network, which is only visible inside the cluster)

**Solution: kubectl port-forward**

```powershell
kubectl port-forward pod/my-web-app 8080:8080
```

**What happens:**
```
Your Windows Host                Kubernetes Cluster Network
localhost:8080                   Pod Network: 10.244.1.5:8080
    │                                         ▲
    │ KUBECTL PORT-FORWARD                    │
    │ "Anything on my localhost:8080"         │
    │ gets tunneled to pod 10.244.1.5:8080"   │
    │                                         │
    └─────────────────────────────────────────│

kubectl creates a tunnel from your machine → API server → pod
(kubernetes handles routing through the cluster)

When you open browser:
http://localhost:8080 → connects to pod at 10.244.1.5:8080
```

**Comparison to Vagrant:**

| Setup | What's Isolated | Bridge | How You Access |
|-------|-----------------|--------|-----------------|
| **Vagrant** | Private network `192.168.56.x` | VirtualBox port forward | `localhost:2711` → VM:22 |
| **Kubernetes** | Pod network `10.244.x.x` | `kubectl port-forward` | `localhost:8080` → Pod:8080 |

---

#### Scenario 3: SSH Tunnel (Advanced, Same Principle)

Even **SSH itself can do port forwarding**! (This is how professionals tunnel through firewalls)

```bash
# Forward local port 3306 to a database running on controlplane01
ssh -L 3306:192.168.56.11:3306 vagrant@localhost -p 2711

# Now you can:
mysql -h localhost -P 3306  # Connects to database on controlplane01!
```

**How it works:**
- SSH creates encrypted tunnel
- You connect to `localhost:3306`
- SSH forwards it to `192.168.56.11:3306` (over encrypted tunnel)
- You're now talking to the remote database

---

#### The Universal Pattern

**Every port forward follows this pattern:**

```
Local Network              Bridge/Tunnel            Remote Network
(Your Machine)            (Port Forward)           (Target)

localhost:PORT_A  ────────────────────────▶  remote-host:PORT_B
   ▲
   │ You connect here
   │
```

**Three examples of the same idea:**

1. **Vagrant:** `localhost:2711` ──▶ `192.168.56.11:22`
2. **Kubernetes:** `localhost:8080` ──▶ `10.244.1.5:8080`
3. **SSH Tunnel:** `localhost:3306` ──▶ `192.168.56.11:3306`

---

#### Why This Matters for Your Labs

**Right now (Lab 03):**
- You use Vagrant port-forward to SSH into VMs
- `localhost:2711` ──▶ `controlplane01:22`

**In Phase 2 (Lab 04+):**
- You'll still use SSH port-forward to copy certs to VMs
- `scp -P 2711 cert.pem vagrant@localhost:/tmp/`

**In Phase 3 (March):**
- You'll use `kubectl port-forward` to access your Message Board app from your browser
- `kubectl port-forward svc/message-board 8080:8080`
- Then: `http://localhost:8080` in your browser

---

#### Common Port-Forward Mistakes

**Q: "Why can't I just connect to 192.168.56.11:22 directly?"**

**Answer:** Try it:
```powershell
ssh vagrant@192.168.56.11 -p 22
# Result: Connection timeout or refused
# Because 192.168.56.x is ONLY visible inside VirtualBox
# Your Windows host can't route there directly
```

**The port forward creates the route for you** (that's its job).

---

**Q: "Can I use two port forwards on the same host port?"**

**Answer:** No!
```ruby
# This would conflict:
node.vm.network "forwarded_port", guest: 22, host: 2711
node2.vm.network "forwarded_port", guest: 22, host: 2711  # ERROR!
# Port 2711 can only map to ONE VM
```

**That's why Vagrant uses different ports for each VM** (2711, 2712, 2721, 2722, 2730).

---

**Q: "What if I want to expose the VM to the internet?"**

**Answer:** Port forwarding only works from YOUR machine. To expose to internet, use:
- **Bridged network adapter** (VM gets real IP on your home network)
- **NAT with open firewall** (risky, not recommended)

**For Kubernetes The Hard Way:** Don't do this! Keep it isolated. That's the point.

---

#### Visual Summary

```
┌────────────────────────────────────────────────────────────────┐
│                   YOUR WINDOWS HOST                            │
│  192.168.1.x (DHCP, home/work network)                         │
│                                                                │
│  You want to SSH into VMs on isolated network                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ PORT FORWARDING BRIDGE (VirtualBox)                      │ │
│  │                                                          │ │
│  │ localhost:2711 ←─→ 192.168.56.11:22 (controlplane01)   │ │
│  │ localhost:2712 ←─→ 192.168.56.12:22 (controlplane02)   │ │
│  │ localhost:2721 ←─→ 192.168.56.21:22 (node01)           │ │
│  │ localhost:2722 ←─→ 192.168.56.22:22 (node02)           │ │
│  │ localhost:2730 ←─→ 192.168.56.30:22 (loadbalancer)     │ │
│  │                                                          │ │
│  │ (These translations happen inside VirtualBox)           │ │
│  └──────────────────────────────────────────────────────────┘ │
│         │                           ▲                         │
│         │ "Connect to localhost:2711"                         │
│         │                           │                         │
│         └───────────────────────────┘                         │
│                                                                │
│  ┌────────────────────────────────────────────────────────────┐
│  │            VIRTUALBOX PRIVATE NETWORK                     │
│  │           192.168.56.0/24 (Host-only)                     │
│  │                                                            │
│  │  controlplane01 ─────────────────── controlplane02        │
│  │  192.168.56.11                     192.168.56.12          │
│  │       │                                    │               │
│  │       └─────────────────┬──────────────────┘               │
│  │                         │                                   │
│  │                    loadbalancer                            │
│  │                    192.168.56.30                           │
│  │                                                            │
│  │  node01 ─────────────────────────── node02                │
│  │  192.168.56.21                     192.168.56.22          │
│  │                                                            │
│  └────────────────────────────────────────────────────────────┘
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## Personal Notes

*Add your own reflections here as you work through the lab:*

- What confused me: _______________
- What I found interesting: _______________
- Gotchas I hit: _______________
- Time spent: ___ hours

## Practical: Install Client Tools on Windows 11

### Step 1: Install kubectl

**What it does:** Let's you talk to Kubernetes API server

**Installation:**

Option A: **Using Chocolatey** (if you have it):
```powershell
choco install kubernetes-cli
kubectl version --client
```

Option B: **Manual download:**
1. Go to https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
2. Download kubectl.exe (v1.x.x matching your cluster version)
3. Add to PATH or remember where it is
4. Test: `kubectl version --client`

Option C: **Using Windows Package Manager:**
```powershell
winget install Kubernetes.kubectl
```

**Verification:**
```powershell
kubectl version --client
# Output: version.BuildInfo{Major:"1", Minor:"28", ...}
```

**Document in notes:**
- [ ] kubectl installed at: `C:\Program Files\Kubernetes\kubectl.exe` (or wherever)
- [ ] Version: `1.x.x` (match to your cluster!)
- [ ] Test: `kubectl version --client` ✅

---

### Step 2: Install cfssl & cfssljson

**What they do:** Create X.509 certificates (for Kubernetes API server, kubelet, etc.)

**Installation:**

```powershell
# Navigate to a folder where you want tools (e.g., C:\k8s-tools)
cd C:\k8s-tools

# Download cfssl (certificate bundler)
Invoke-WebRequest -Uri https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_windows_amd64.exe -OutFile cfssl.exe

# Download cfssljson (JSON tool for certs)
Invoke-WebRequest -Uri https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_windows_amd64.exe -OutFile cfssljson.exe

# Add folder to PATH (so you can call them anywhere)
# This is optional but recommended
```

**Or if you prefer curl:**
```bash
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_windows_amd64.exe -o cfssl.exe
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_windows_amd64.exe -o cfssljson.exe
```

**Verification:**
```powershell
.\cfssl version
.\cfssljson --help
```

**Document in notes:**
- [ ] cfssl installed at: `C:\k8s-tools\cfssl.exe`
- [ ] cfssljson installed at: `C:\k8s-tools\cfssljson.exe`
- [ ] Added to PATH: YES/NO
- [ ] Test: `cfssl version` ✅, `cfssljson --help` ✅

---

### Step 3: Verify All Tools Work Together

Create a test folder to verify everything:

```powershell
# Create workspace
mkdir C:\k8s-workspace
cd C:\k8s-workspace

# Test 1: kubectl
kubectl version --client
# Output: Shows client version

# Test 2: cfssl
cfssl version
# Output: Shows cfssl version and hash

# Test 3: cfssljson
echo '{"cert": "test"}' | cfssljson -bare test
# Output: Creates test.csr, test-key.pem files (expected for cert workflow)
```

---

## Understanding Lab 03 Context

### Why These Tools Matter for K8s The Hard Way

In **Lab 03**, you're preparing your **host machine** to:

1. **Generate certificates** (Lab 04)
   - `cfssl` creates Kubernetes PKI infrastructure
   - One CA (Certificate Authority) to sign all certs
   - Server certs for API server, etcd, kubelet
   - Client certs for kube-proxy, kube-scheduler, etc.

2. **Create kubeconfigs** (Lab 05)
   - `kubectl` uses these to connect to cluster
   - Contains: server URL, cert, key, context name

3. **Manage the cluster** (Lab 12+)
   - `kubectl` talks to API server on `192.168.56.30:6443`
   - Deploy apps, scale pods, troubleshoot

### Flow Visualization

```
Lab 03 (Now)          Lab 04          Lab 05         Lab 12+
┌─────────────┐   ┌──────────────┐  ┌────────────┐  ┌──────────┐
│ Install     │   │ Generate     │  │ Create     │  │ Use      │
│ Tools       │──▶│ Certs        │─▶│ Kubeconfig │─▶│ kubectl  │
│             │   │ (cfssl)      │  │ files      │  │          │
│ ✓ kubectl   │   │              │  │            │  │ $ kubectl│
│ ✓ cfssl     │   │ ✓ CA cert    │  │ ✓ admin.   │  │  get     │
│ ✓ cfssljson │   │ ✓ API cert   │  │   conf     │  │  nodes   │
└─────────────┘   │ ✓ Kubelet    │  │ ✓ kubectl. │  └──────────┘
                  │   cert       │  │   conf     │
                  │ ... (10 more)│  │            │
                  └──────────────┘  └────────────┘
```

---

## Doubts & Answers (Common Q&A for This Lab)

### Q1: "Do I need to install all three tools? What if I skip cfssl?"

**Answer:** No, you need all three.

- **cfssl** = generates certs (no kubectl substitute)
- **cfssljson** = converts cert output to JSON (no substitute)
- **kubectl** = talks to cluster (could skip temporarily, but you'll need it by Lab 12)

**If you skip:** You won't be able to complete Lab 04 (certificate generation), which is blocking for everything else.

---

### Q2: "Can I install these on the VMs instead of my host?"

**Answer:** Technically yes, but don't. Here's why:

**Why on HOST is better:**
- Faster (no SSH tunnel for every cert generation)
- Easier to manage (you know your Windows terminal)
- Standard practice (DevOps engineers work from their laptops)
- You can commit certs to git (Lab 04 outputs)

**If on VM only:**
- Slow: Generate cert → SCP to host → SCP to other VMs
- Annoying: You'd be SSHing constantly
- Not portable: Can't use certs if you nuke the VM

**Best practice:** HOST is your "jumpbox" (control point)

---

### Q3: "What if kubectl version doesn't match cluster version?"

**Answer:** K8s is backward-compatible (usually).

- Client v1.28 can talk to Server v1.27 ✅
- Client v1.27 can talk to Server v1.28 ✅ (might warn)
- Client v1.26 talking to v1.28 might have issues ⚠️

**For this lab:**
- Your cluster will be v1.28 (Kelsey's version)
- Install kubectl v1.28 to match

**Check version in docs:**
- See `/docs/03-client-tools.md` (Kelsey's original)

---

### Q4: "Do I need to add tools to Windows PATH?"

**Answer:** Not required, but highly recommended.

**If NOT in PATH:**
```powershell
C:\k8s-tools\cfssl version    # Must use full path every time
```

**If in PATH:**
```powershell
cfssl version    # Call from anywhere
```

**How to add to PATH (Windows):**
1. Settings → Environment Variables → Edit System variables
2. New variable: `C:\k8s-tools` → Add to PATH
3. Restart PowerShell/Terminal
4. Test: `cfssl version` (no path needed)

---

## Checklist: Lab 03 Complete

- [ ] **kubectl** installed and verified
  - Command: `kubectl version --client` shows version
  - Document version number in notes
  
- [ ] **cfssl** installed and verified
  - Command: `cfssl version` shows hash/version
  
- [ ] **cfssljson** installed and verified
  - Command: `cfssljson --help` shows usage
  
- [ ] **Added to PATH** (optional but recommended)
  - Can call tools from any folder in PowerShell
  
- [ ] **Workspace ready**
  - Folder: `C:\k8s-workspace\` (or wherever you prefer)
  - Will use this for cert generation (Lab 04)

- [ ] **Notes documented**
  - Where each tool is installed
  - Which PATH used
  - Versions installed
  - What each tool does (in your own words)

---

## Next: Lab 04 (Certificate Authority)

Once Lab 03 is done:

1. You'll use **cfssl** to create:
   - **Certificate Authority** (the "trust root")
   - **All K8s certificates** (API server, kubelet, etcd, etc.)

2. **cert_verify.sh** will validate each cert:
   - Check that cert is signed by CA
   - Check that cert has correct SANs (Subject Alternative Names)
   - Check expiration dates

3. Copy certs to VMs via SCP or shared `/vagrant` folder

**Theory you'll need for Lab 04:**
- What is X.509 certificates?
- Why does K8s need so many certs?
- What are "Subject Alternative Names" (SANs)?
- How does TLS handshake work?

I'll explain those when you're ready for Lab 04!

---

## Personal Notes

*Add your own reflections here as you work through the lab:*

- What confused me: _______________
- What I found interesting: _______________
- Gotchas I hit: _______________
- Time spent: ___ hours
