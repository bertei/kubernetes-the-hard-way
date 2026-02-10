# K8s Hard Way - Network & Infrastructure Setup

**Hardware:** Windows 11, i5-12600KF, RTX 4070 Super, 32GB DDR5  
**Setup:** Vagrant + VirtualBox (Intel-compatible)  
**Network CIDR:** `192.168.56.0/24` (private, host-only)

---

## Network Architecture Overview

Your cluster uses a **hybrid network setup** with two adapters per VM:

### Network Adapters (Per VM)

| Adapter | Name | Purpose | Details |
|---------|------|---------|---------|
| **Adapter 1** | NAT | Internet Access | Outbound only (package downloads) |
| **Adapter 2** | Host-only | Cluster Traffic | Private network `192.168.56.x` for K8s communication |

**Key Point:** All Kubernetes cluster traffic (etcd, API server, kubelet, etc.) flows over **Adapter 2 (host-only network `192.168.56.x`)**. This is isolated and secure.

---

## VM Network Configuration

### IP Address Assignments

| VM Name | Role | Host-only IP | SSH Port (Forwarded) |
|---------|------|--------------|----------------------|
| `controlplane01` | Control Plane 1 | `192.168.56.11` | `2711` |
| `controlplane02` | Control Plane 2 | `192.168.56.12` | `2712` |
| `loadbalancer` | HAProxy LB | `192.168.56.30` | `2730` |
| `node01` | Worker 1 | `192.168.56.21` | `2721` |
| `node02` | Worker 2 | `192.168.56.22` | `2722` |

### SSH Access from Host

Since VMs are on a private network, Vagrant creates port forwarding to your host machine:

```bash
# Example: SSH into controlplane01
ssh -i <path-to-vagrant-key> vagrant@localhost -p 2711

# Or if using Vagrant directly:
vagrant ssh controlplane01
```

**Important:** Each VM's SSH port is unique. If port 2711 is busy, Vagrant assigns a fallback (2222, 2200, etc).

---

## How VMs Communicate

### Internal (Cluster) Communication
- **Network:** `192.168.56.x` (host-only adapter)
- **Protocol:** Direct IP-to-IP
- **Security:** Isolated from your host network (no external access)
- **Example:** `controlplane01` (`.11`) talks to `node01` (`.21`) directly over this private network

### External (Host to VM) Access
- **SSH:** Port forwarding (e.g., port 2711 on host → port 22 on VM)
- **Kubernetes API:** Will be accessed via load balancer (HAProxy at `192.168.56.30`)
- **Security:** You control access via forwarded ports

### Internet Access (Outbound Only)
- **Adapter 1 (NAT):** VMs can download packages but can't receive inbound connections
- **Use case:** `apt-get install`, pulling container images, etc.

---

## Network Summary

```
┌─────────────────────────────────────────────────────────────┐
│                       Your Windows Host                      │
│  (192.168.1.x or DHCP - your home/work network)             │
└──────┬──────────────────────────────────────────────────────┘
       │ Port Forwarding (2711→22, 2712→22, etc.)
       │
┌──────┴──────────────────────────────────────────────────────┐
│                   VirtualBox                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Private Network: 192.168.56.0/24 (host-only)         │ │
│  │                                                        │ │
│  │  controlplane01  controlplane02  loadbalancer         │ │
│  │   192.168.56.11   192.168.56.12   192.168.56.30       │ │
│  │        │               │               │               │ │
│  │        └───────────────┴───────────────┘               │ │
│  │              (Direct cluster traffic)                  │ │
│  │                                                        │ │
│  │  node01              node02                            │ │
│  │ 192.168.56.21      192.168.56.22                       │ │
│  │      │                  │                              │ │
│  │      └──────────────────┘                              │ │
│  │        (Pod networking via CNI)                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  All VMs also have NAT adapter for internet access          │
└──────────────────────────────────────────────────────────────┘
```

---

## Key Networking Facts for K8s Hard Way

### Certificate Validation
When you generate certificates in **Lab 04**, use these IPs/hostnames:

```
# Kubernetes API Server Certificate SANs (Subject Alt Names):
- localhost
- 127.0.0.1
- kubernetes
- kubernetes.default
- kubernetes.default.svc
- kubernetes.default.svc.cluster
- kubernetes.svc.cluster.local
- 192.168.56.11 (controlplane01)
- 192.168.56.12 (controlplane02)
- 192.168.56.30 (load balancer IP - important!)
```

**Why?** When you use `kubectl` from your host, it connects through the load balancer at `192.168.56.30`, so the API server cert MUST include that IP.

### Load Balancer Role
- **HAProxy at 192.168.56.30** sits in front of control planes
- Distributes traffic to `controlplane01` and `controlplane02`
- This gives you **high availability** (if one control plane dies, traffic goes to the other)
- All kubeconfig files (Lab 05) point to `192.168.56.30:6443`

### etcd Clustering
- **etcd runs on both control planes** (`192.168.56.11`, `192.168.56.12`)
- They discover each other via hostname + IP
- Use the private network IPs in etcd configuration

### Worker Node Registration
- **Workers** (`192.168.56.21`, `192.168.56.22`) find the API server at `192.168.56.30:6443`
- Kubelet on each worker communicates with API server over the private network
- Pod-to-pod traffic uses the **CNI (Container Network Interface)** overlay (Lab 13 - Weave or Flannel)

---

## Port Reference

### SSH Access (Host → VM)
```
Host Port → VM Port 22
2711 → controlplane01:22
2712 → controlplane02:22
2730 → loadbalancer:22
2721 → node01:22
2722 → node02:22
```

### Kubernetes Services (Private Network Only)
```
192.168.56.30:6443  → API Server (via HAProxy load balancer)
192.168.56.11:6443  → API Server (controlplane01 directly)
192.168.56.12:6443  → API Server (controlplane02 directly)
192.168.56.11:2379  → etcd (controlplane01)
192.168.56.12:2379  → etcd (controlplane02)
```

---

## When You Get Stuck

### "Can't reach API server from kubectl"
- Check: Is HAProxy running? (`vagrant ssh loadbalancer` → `sudo systemctl status haproxy`)
- Check: Is the cert valid for `192.168.56.30`?
- Check: Are both control planes up? (`kubectl get componentstatuses`)

### "Nodes won't join cluster"
- They must be able to reach `192.168.56.30:6443`
- Check network: `vagrant ssh node01` → `curl https://192.168.56.30:6443 -k`

### "etcd cluster won't bootstrap"
- Both control planes must see each other: `vagrant ssh controlplane01` → `ping 192.168.56.12`
- etcd certs must include both IPs

---

## Next Steps

1. **Lab 03 (Client Tools):** Install kubectl, cfssl, cfssljson on your **host machine** (Windows 11)
2. **Lab 04 (Certificates):** Generate all K8s certs - use `cert_verify.sh` after each cert!
3. **Lab 05 (kubeconfigs):** Create kubectl config files pointing to `192.168.56.30:6443`
4. **Lab 07 onwards:** Deploy to the VMs over this private network

---

## Reference: How IP Assignments Work

From your Vagrantfile (or Vagrant docs):

**Control Planes** (start at `.10`):
```
controlplane01: 192.168.56.11  (MASTER_IP_START = 10, so 10+1 = 11)
controlplane02: 192.168.56.12  (10+2 = 12)
```

**Workers** (start at `.20`):
```
node01: 192.168.56.21  (NODE_IP_START = 20, so 20+1 = 21)
node02: 192.168.56.22  (20+2 = 22)
```

**Load Balancer** (fixed):
```
loadbalancer: 192.168.56.30  (LB_IP_START = 30)
```

---

## Verification Checklist

Before moving to Lab 03, confirm:

- [ ] VirtualBox is installed
- [ ] Vagrant is installed
- [ ] You can list VMs: `vagrant global-status`
- [ ] All 5 VMs boot successfully: `vagrant up`
- [ ] You can SSH into each: `vagrant ssh controlplane01` (and others)
- [ ] VMs can ping each other over `192.168.56.x` network
- [ ] Each VM has internet access for package downloads

---

**Created:** Jan 2025 (Updated Feb 2026)  
**Reference:** Kelsey Hightower's Kubernetes The Hard Way (Apple Silicon Fork)