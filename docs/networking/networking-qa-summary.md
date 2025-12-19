# Networking Q&A Summary

## Table of Contents

1. [VM DHCP from Virtual Router](#1-vm-dhcp-from-virtual-router)
2. [Host Bridge and Trunk Port](#2-host-bridge-and-trunk-port)
3. [Virtual Router as a VM](#3-virtual-router-as-a-vm)
4. [What Acts as a Bridge](#4-what-acts-as-a-bridge)
5. [One Bridge Per Network](#5-one-bridge-per-network)
6. [DHCP Flow](#6-dhcp-flow)
7. [VLANs on Host vs Virtual Router](#7-vlans-on-host-vs-virtual-router)
8. [Adding New VLANs](#8-adding-new-vlans)
9. [Bridges are Local to Host](#9-bridges-are-local-to-host)
10. [Multiple Hosts Same VLAN](#10-multiple-hosts-same-vlan)
11. [VLANs on Physical Switch](#11-vlans-on-physical-switch)
12. [VLANs on Physical Router](#12-vlans-on-physical-router)
13. [Why Bridge is Unavoidable](#13-why-bridge-is-unavoidable)
14. [SR-IOV Bypass](#14-sr-iov-bypass)
15. [VF and Trunk Mode](#15-vf-and-trunk-mode)
16. [Multiple Ports Same VLAN](#16-multiple-ports-same-vlan)
17. [Kubernetes Pod Networking](#17-kubernetes-pod-networking)
18. [veth Pairs vs tap Devices](#18-veth-pairs-vs-tap-devices)
19. [Multi-Pod Multi-Container Communication](#19-multi-pod-multi-container-communication)

---

## 1. VM DHCP from Virtual Router

**Q: How does a VM on a hypervisor get DHCP from a virtual router on a VLAN?**

The flow:

1. **VM Network Interface** → VM is attached to a logical network (with VLAN tag)
2. **Host Bridge** → The hypervisor host creates a bridge with VLAN tagging
3. **Physical NIC** → Traffic exits the host on a trunk port
4. **Switch** → Routes VLAN-tagged traffic
5. **Router VLAN Interface** → Router has an interface on that VLAN
6. **Router DHCP Server** → Responds to DHCP requests

---

## 2. Host Bridge and Trunk Port

**Q: What is a host bridge and trunk port?**

### Bridge

A bridge is a virtual network switch inside the hypervisor host that connects VMs to the physical network.

```text
┌─────────────────────────────────────────────┐
│            Hypervisor Host                  │
│   ┌───────┐  ┌───────┐  ┌───────┐          │
│   │ VM 1  │  │ VM 2  │  │ VM 3  │          │
│   └───┬───┘  └───┬───┘  └───┬───┘          │
│       └──────────┼──────────┘               │
│          ┌───────┴───────┐                  │
│          │    BRIDGE     │ ← Virtual Switch │
│          └───────┬───────┘                  │
│          ┌───────┴───────┐                  │
│          │  Physical NIC │                  │
│          └───────────────┘                  │
└─────────────────────────────────────────────┘
```

### Trunk Port

A trunk port carries traffic for multiple VLANs on a single physical cable using 802.1Q tagging.

| Access Port              | Trunk Port                         |
|--------------------------|------------------------------------|
| Carries ONE VLAN only    | Carries MULTIPLE VLANs             |
| No VLAN tags on frames   | Frames are tagged with VLAN ID     |
| For end devices (PCs)    | For switches, routers, hypervisors |

---

## 3. Virtual Router as a VM

**Q: How does it work when the router/firewall is also a VM?**

When the router is a VM on the same hypervisor, traffic can stay entirely inside the bridge (if on same host):

```text
┌─────────────────────────────────────────────────────────┐
│                     Hypervisor Host                     │
│   ┌─────────────┐                  ┌─────────────────┐  │
│   │   Your VM   │                  │   Router VM     │  │
│   │  VLAN 100   │                  │  vNIC: VLAN100  │  │
│   └──────┬──────┘                  └────────┬────────┘  │
│          │ DHCP Discover                    │           │
│          ▼                                  ▼           │
│   ┌──────────────────────────────────────────────────┐  │
│   │               BRIDGE (VLAN 100)                  │  │
│   │   Traffic stays HERE if both VMs on same host   │  │
│   └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

The router VM typically has multiple vNICs, each attached to a different logical network.

---

## 4. What Acts as a Bridge

**Q: In a hypervisor setup, what acts as a bridge?**

Linux Bridges created by the hypervisor (libvirt, VDSM, etc.). When you create a VLAN network and attach it to a host:

| Logical Network | VLAN Interface | Linux Bridge       |
|-----------------|----------------|--------------------|
| Mgmt-Net        | eth0.10        | bridge for VLAN 10 |
| Dev-Net         | eth0.20        | bridge for VLAN 20 |
| Prod-Net        | eth0.30        | bridge for VLAN 30 |

The bridge connects all VMs on the same logical network, including virtual routers.

---

## 5. One Bridge Per Network

**Q: Does each network have its own bridge?**

Yes. Each logical network gets its own Linux bridge on the host.

```text
┌─────────────────────────────────────────────────────────┐
│                    Hypervisor Host                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  Bridge 1   │  │  Bridge 2   │  │  Bridge 3   │     │
│  │  Mgmt-Net   │  │  Dev-Net    │  │  Prod-Net   │     │
│  │ (no VLAN)   │  │ (VLAN 10)   │  │ (VLAN 20)   │     │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
│         ▼                ▼                ▼             │
│       eth0            eth0.10         eth0.20          │
└─────────────────────────────────────────────────────────┘
```

Completely isolated at Layer 2. Traffic crosses between them only through a router (Layer 3).

---

## 6. DHCP Flow

**Q: vNIC to bridge, bridge forwards to router, router responds DHCP?**

Yes, and traffic stays inside the bridge if the router VM is on same host:

```text
1. VM sends DHCP Discover (broadcast)
              │
              ▼
2. Bridge floods broadcast to ALL connected interfaces
              │
   ┌──────────┼──────────┐
   ▼          ▼          ▼
Other VMs   Router    eth0.20
(ignore)    vNIC     (to switch)
              │
              ▼
3. Router has DHCP server enabled on that interface
              │
              ▼
4. Router responds with DHCP Offer
```

Three requirements:

- VM and router on same VLAN
- Broadcast reaches router
- DHCP server enabled on router interface

---

## 7. VLANs on Host vs Virtual Router

**Q: Are VLANs on the router or on the hypervisor host?**

With separate vNICs per network, VLANs are on the **host**, not the router:

| Component  | VLANs?                                           |
|------------|--------------------------------------------------|
| Host       | Yes - eth0.10, eth0.20, etc.                     |
| Router VM  | No - sees vNIC0, vNIC1, etc. as plain interfaces |

The hypervisor separates traffic before it reaches the router. The router just sees clean, separate interfaces.

---

## 8. Adding New VLANs

**Q: How do I create VLANs and launch VMs?**

For hypervisor-managed VLANs:

1. Create logical network in hypervisor (with VLAN tag)
2. Assign to cluster and host
3. Add vNIC to router VM
4. Configure router interface (IP + DHCP)
5. Launch VMs attached to that network

Cannot just create VLANs in the router and have VMs join - the bridge connection is required.

---

## 9. Bridges are Local to Host

**Q: Is a bridge local to a host?**

Yes. A Linux bridge is local to one host only. It doesn't span across multiple hosts.

```text
┌────────────────────────┐      ┌────────────────────────┐
│        Host A          │      │        Host B          │
│   Bridge A (VLAN 20)   │      │   Bridge B (VLAN 20)   │
│          │             │      │          │             │
│       eth0.20          │      │       eth0.20          │
└──────────┬─────────────┘      └──────────┬─────────────┘
           │     Physical Switch           │
           └───────────┬───────────────────┘
                       │
                 VLAN 20 trunk
```

---

## 10. Multiple Hosts Same VLAN

**Q: How do VMs on different hosts communicate on the same VLAN?**

They're NOT the same bridge - they're separate bridges with the same VLAN tag.

The hypervisor creates a bridge on each host when you attach the network:

- Attach "Dev-Net" to Host A's eth0 → Creates Bridge A + eth0.20
- Attach "Dev-Net" to Host B's eth0 → Creates Bridge B + eth0.20

The physical switch connects them:

| Source            | Destination       | Path                            |
|-------------------|-------------------|---------------------------------|
| VM1 (A, VLAN20)   | VM3 (B, VLAN20)   | Bridge A → Switch → Bridge B    |
| VM3 (B, VLAN20)   | VM4 (B, VLAN20)   | Bridge B only (local)           |

---

## 11. VLANs on Physical Switch

**Q: VLANs are always on the host? What's the point of VLANs on a switch?**

In traditional networks (no virtualization), end devices don't tag - the switch does:

```text
Traditional:                      Virtualized:

  PC (dumb)                         VM
     │ untagged                      │ untagged
     ▼                               ▼
  Switch ← adds tag              Host ← adds tag
                                     │ tagged
                                     ▼
                                  Switch ← just forwards
```

In virtualization, the host replaces the switch's VLAN job for VMs.

Physical switch VLANs still matter for:

- Physical devices (PCs, servers) that can't tag
- Connecting multiple hypervisor hosts
- Mixed environments (VMs + physical)

---

## 12. VLANs on Physical Router

**Q: How do VLANs work with a physical router/firewall?**

With a physical router, it needs VLANs to decode the trunk:

```text
┌──────────────────────────────────────────────────────────┐
│                   Hypervisor Host                        │
│   VM1 → Bridge → eth0.20 → ADDS TAG → eth0               │
└────────────────────────────┬─────────────────────────────┘
                             │ tagged
                             ▼
                        Physical Switch
                             │ tagged
                             ▼
┌──────────────────────────────────────────────────────────┐
│                   Physical Router                        │
│   em1 (trunk) → em1.20 STRIPS TAG → DHCP server          │
└──────────────────────────────────────────────────────────┘
```

|                | Physical Router          | Virtual Router             |
|----------------|--------------------------|----------------------------|
| Router VLANs   | Required                 | Not needed                 |
| Host VLANs     | Required                 | Required                   |
| Traffic path   | Host → Switch → Router   | Host → Bridge → Router     |

---

## 13. Why Bridge is Unavoidable

**Q: Can we avoid the hypervisor bridge?**

No. Bridge is required for VMs to connect to the network.

```text
VM needs something to plug into
         │
         ▼
    ┌─────────┐
    │ Bridge  │ ← Virtual switch port for VM
    └─────────┘
```

Only ways to skip bridge:

- **SR-IOV** - VM gets direct hardware NIC slice
- **PCI Passthrough** - VM owns entire NIC
- **Macvtap** - direct NIC access

These are specialized. For normal networking, bridge is always there.

---

## 14. SR-IOV Bypass

**Q: How does SR-IOV work with VLANs?**

With SR-IOV, the NIC hardware or VM handles VLAN tagging - bypasses the host:

```text
┌─────────────────────────────────────────────────────────┐
│                   Hypervisor Host                       │
│   VM1         VM2         Router                        │
│    │           │            │                           │
│   VF0         VF1          VF2                          │
│  VLAN10      VLAN20     (trunk)                         │
│    │           │            │                           │
│  ─────────────────────────────  NO BRIDGE               │
│    │                                                    │
│  Physical NIC (SR-IOV)                                  │
│   - VF0: VLAN 10 enforced by hardware                   │
│   - VF1: VLAN 20 enforced by hardware                   │
│   - VF2: trunk (router tags itself)                     │
└─────────────────────────────────────────────────────────┘
```

---

## 15. VF and Trunk Mode

**Q: What is VF and trunk mode?**

### VF (Virtual Function)

A slice of a physical NIC that acts like a real NIC to a VM:

```text
Physical NIC (SR-IOV)
  │
  ├── PF (Physical Function) - host uses
  ├── VF0 - assigned to VM1
  ├── VF1 - assigned to VM2
  └── VF2 - assigned to router
```

### Trunk Mode

- **Access mode**: VF tags everything with one VLAN (hardware enforced)
- **Trunk mode**: All VLANs pass through, VM handles tagging

```bash
# Configure VFs
ip link set eth0 vf 0 vlan 10   # VF0 = access VLAN 10
ip link set eth0 vf 1 vlan 20   # VF1 = access VLAN 20
ip link set eth0 vf 2 vlan 0    # VF2 = trunk (0 = all)
```

A router VM needs trunk mode to see all VLANs and route between them.

---

## 16. Multiple Ports Same VLAN

**Q: Can two different switch ports be assigned to same VLAN?**

Yes, that's the whole point of VLANs:

```text
┌─────────────────────────────────────────────┐
│                    Switch                   │
│   Port 1      Port 2      Port 3    Port 4  │
│   VLAN 10     VLAN 10     VLAN 10   VLAN 20 │
│      │           │           │         │    │
└──────┼───────────┼───────────┼─────────┼────┘
       ▼           ▼           ▼         ▼
      PC1         PC2       Printer   Server

   └─────────────────────────┘          │
        Same VLAN 10                Different
      Can talk directly               VLAN 20
```

VLANs = group ports together logically, regardless of physical location.

---

## 17. Kubernetes Pod Networking

**Q: What does a K8s pod use for networking?**

K8s uses CNI (Container Network Interface) plugins - same concepts as VM virtualization:

```text
┌────────────────────────────────────────────────────────────┐
│                      K8s Node                              │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐                 │
│   │  Pod A  │   │  Pod B  │   │  Pod C  │                 │
│   └────┬────┘   └────┬────┘   └────┬────┘                 │
│        │ veth        │ veth        │ veth                  │
│        ▼             ▼             ▼                       │
│   ┌────────────────────────────────────────────────────┐  │
│   │              Bridge (cni0)                         │  │
│   └─────────────────────────┬──────────────────────────┘  │
│                           eth0                             │
└─────────────────────────────┼──────────────────────────────┘
```

| VM Virtualization | K8s Pod           |
|-------------------|-------------------|
| VM                | Pod (container)   |
| vNIC              | veth pair         |
| Linux Bridge      | cni0 bridge       |
| VLAN              | Overlay / routing |

CNI options:

- **Flannel**: VXLAN overlay
- **Calico**: BGP routing
- **Cilium**: eBPF

Same building blocks - just lighter weight.

---

## 18. veth Pairs vs tap Devices

**Q: Why is a veth pair needed? Does a VM use it?**

No. **VMs do NOT use veth pairs** - that's for containers. VMs use **tap devices**.

| Type           | Used By                  | How It Works                      |
|----------------|--------------------------|-----------------------------------|
| **veth pair**  | Containers (Docker, K8s) | Virtual cable with two ends       |
| **tap device** | VMs (KVM, libvirt)       | Virtual NIC that bridges to host  |

### Containers: veth Pair

A veth pair is a virtual ethernet cable - two ends, packets in one come out the other:

```text
┌─────────────────────────────────────────────────────────────┐
│                         Host                                │
│                                                             │
│   ┌─────────────────────┐                                  │
│   │     Container       │                                  │
│   │   (own namespace)   │                                  │
│   │                     │                                  │
│   │      eth0           │ ← Container sees this as its NIC │
│   └────────┬────────────┘                                  │
│            │                                                │
│         veth pair                                           │
│       (virtual cable)                                       │
│            │                                                │
│            ▼                                                │
│   ┌─────────────────────┐                                  │
│   │    vethXXXX         │ ← Other end on host              │
│   └────────┬────────────┘                                  │
│            │                                                │
│            ▼                                                │
│   ┌─────────────────────┐                                  │
│   │      Bridge (cni0)  │                                  │
│   └─────────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

**Why veth?** Containers share the host kernel but have isolated network namespaces. The veth pair connects two namespaces.

### VMs: tap Device

VMs don't share the kernel - they're fully isolated. They use tap devices:

```text
┌─────────────────────────────────────────────────────────────┐
│                         Host                                │
│                                                             │
│   ┌─────────────────────┐                                  │
│   │         VM          │                                  │
│   │    (own kernel)     │                                  │
│   │                     │                                  │
│   │    eth0 / ens3      │ ← VM sees this as real NIC       │
│   └────────┬────────────┘                                  │
│            │                                                │
│         virtio                                              │
│       (emulated NIC)                                        │
│            │                                                │
│            ▼                                                │
│   ┌─────────────────────┐                                  │
│   │     tap device      │ ← vnet0, vnet1, etc.             │
│   │      (vnet0)        │                                  │
│   └────────┬────────────┘                                  │
│            │                                                │
│            ▼                                                │
│   ┌─────────────────────┐                                  │
│   │      Bridge         │                                  │
│   └─────────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

### Comparison

|                | veth pair              | tap device                  |
|----------------|------------------------|-----------------------------|
| **Endpoints**  | Two veth interfaces    | VM virtio + host tap        |
| **Isolation**  | Network namespace      | Full VM                     |
| **Kernel**     | Shared with host       | Separate guest kernel       |
| **Use case**   | Containers             | Virtual machines            |
| **Created by** | Docker/K8s CNI         | libvirt/QEMU                |

---

## 19. Multi-Pod Multi-Container Communication

**Q: How do veth pairs work with multiple containers in multiple pods?**

### Containers in Same Pod

Containers in the **same pod share one network namespace** - they share the same veth pair and IP:

```text
┌──────────────────────────────────────────────────────────────┐
│                           Pod A                              │
│                     (one network namespace)                  │
│                                                              │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│   │ Container 1 │    │ Container 2 │    │ Container 3 │     │
│   │   (nginx)   │    │  (php-fpm)  │    │   (redis)   │     │
│   │  port 80    │    │  port 9000  │    │  port 6379  │     │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘     │
│          │                  │                  │             │
│          └──────────────────┼──────────────────┘             │
│                             │                                │
│                    ┌────────┴────────┐                       │
│                    │      eth0       │ ← Shared by all       │
│                    │   10.244.1.5    │   containers          │
│                    └────────┬────────┘                       │
└─────────────────────────────┼────────────────────────────────┘
                              │
                         veth pair
                              │
                              ▼
                     ┌────────────────┐
                     │    vethXXX     │
                     └────────────────┘
```

Container 1 reaches Container 2 via `localhost:9000` - no network hop, same namespace.

### Multiple Pods on Same Node

Each pod gets its **own veth pair**:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                              Node 1                                     │
│                                                                         │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    │
│   │     Pod A       │    │     Pod B       │    │     Pod C       │    │
│   │   10.244.1.5    │    │   10.244.1.6    │    │   10.244.1.7    │    │
│   └───────┬─────────┘    └───────┬─────────┘    └───────┬─────────┘    │
│           │                      │                      │              │
│        veth pair              veth pair              veth pair         │
│           │                      │                      │              │
│           ▼                      ▼                      ▼              │
│   ┌─────────────┐        ┌─────────────┐        ┌─────────────┐        │
│   │  veth1234   │        │  veth5678   │        │  veth9abc   │        │
│   └──────┬──────┘        └──────┬──────┘        └──────┬──────┘        │
│          │                      │                      │                │
│          └──────────────────────┼──────────────────────┘                │
│                                 │                                       │
│                        ┌────────┴────────┐                              │
│                        │   Bridge (cni0) │                              │
│                        │   10.244.1.1    │                              │
│                        └─────────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
```

**Pod A → Pod B (same node):** Traffic stays on the bridge - never leaves the node.

### Multiple Pods on Different Nodes (VXLAN Overlay)

```text
┌──────────────────────────────────┐    ┌──────────────────────────────────┐
│            Node 1                │    │            Node 2                │
│                                  │    │                                  │
│   ┌───────────┐  ┌───────────┐   │    │   ┌───────────┐  ┌───────────┐  │
│   │   Pod A   │  │   Pod B   │   │    │   │   Pod C   │  │   Pod D   │  │
│   │10.244.1.5 │  │10.244.1.6 │   │    │   │10.244.2.5 │  │10.244.2.6 │  │
│   └─────┬─────┘  └─────┬─────┘   │    │   └─────┬─────┘  └─────┬─────┘  │
│         │              │         │    │         │              │        │
│      veth pair      veth pair    │    │      veth pair      veth pair   │
│         │              │         │    │         │              │        │
│         ▼              ▼         │    │         ▼              ▼        │
│   ┌─────────────────────────┐    │    │   ┌─────────────────────────┐   │
│   │     Bridge (cni0)       │    │    │   │     Bridge (cni0)       │   │
│   └───────────┬─────────────┘    │    │   └───────────┬─────────────┘   │
│               │                  │    │               │                 │
│               ▼                  │    │               ▼                 │
│   ┌─────────────────────────┐    │    │   ┌─────────────────────────┐   │
│   │   flannel.1 (VXLAN)     │    │    │   │   flannel.1 (VXLAN)     │   │
│   │   Encapsulates packet   │    │    │   │   Decapsulates packet   │   │
│   └───────────┬─────────────┘    │    │   └───────────┬─────────────┘   │
│               │                  │    │               │                 │
│             eth0                 │    │             eth0                │
│          192.168.1.10            │    │          192.168.1.11           │
└───────────────┬──────────────────┘    └───────────────┬─────────────────┘
                │                                       │
                └───────────────┬───────────────────────┘
                                │
                         Physical Network
```

**Pod A (Node 1) → Pod C (Node 2):**

1. Pod A sends packet to 10.244.2.5
2. Bridge doesn't know 10.244.2.x - forwards to flannel.1
3. Flannel encapsulates in VXLAN (outer IP: Node1 → Node2)
4. Physical network delivers to Node 2
5. Node 2 flannel.1 decapsulates
6. Bridge forwards to Pod C via veth pair

### Multiple Pods on Different Nodes (BGP Routing)

```text
┌──────────────────────────────────┐    ┌──────────────────────────────────┐
│            Node 1                │    │            Node 2                │
│         192.168.1.10             │    │         192.168.1.11             │
│                                  │    │                                  │
│   ┌───────────┐  ┌───────────┐   │    │   ┌───────────┐  ┌───────────┐  │
│   │   Pod A   │  │   Pod B   │   │    │   │   Pod C   │  │   Pod D   │  │
│   │10.244.1.5 │  │10.244.1.6 │   │    │   │10.244.2.5 │  │10.244.2.6 │  │
│   └─────┬─────┘  └─────┬─────┘   │    │   └─────┬─────┘  └─────┬─────┘  │
│         │              │         │    │         │              │        │
│      veth pair      veth pair    │    │      veth pair      veth pair   │
│         │              │         │    │         │              │        │
│         ▼              ▼         │    │         ▼              ▼        │
│   ┌─────────────────────────┐    │    │   ┌─────────────────────────┐   │
│   │  Routing table:         │    │    │   │  Routing table:         │   │
│   │  10.244.1.0/24 → local  │    │    │   │  10.244.2.0/24 → local  │   │
│   │  10.244.2.0/24 → Node2  │    │    │   │  10.244.1.0/24 → Node1  │   │
│   └───────────┬─────────────┘    │    │   └───────────┬─────────────┘   │
│             eth0                 │    │             eth0                │
└───────────────┬──────────────────┘    └───────────────┬─────────────────┘
                │                                       │
                │      ┌─────────────────────┐         │
                └──────│  Router/Switch      │─────────┘
                       │  10.244.1.0/24→Node1│
                       │  10.244.2.0/24→Node2│
                       └─────────────────────┘
```

**No encapsulation** - just IP routing. Faster, but requires network support.

### Traffic Flow Summary

| Scenario                  | veth pairs  | Path                                       |
|---------------------------|-------------|--------------------------------------------|
| Same pod containers       | 1 shared    | localhost                                  |
| Same node pods            | 1 per pod   | Through bridge                             |
| Different node pods       | 1 per pod   | Bridge → CNI (overlay/routing) → Bridge    |

---

## Key Takeaways

1. **Bridges connect VMs to networks** - unavoidable in virtualization
2. **VLANs tag traffic** - can be done by host, switch, or NIC hardware
3. **In virtualized setups**, host does VLAN tagging, switch just forwards
4. **Physical routers/firewalls** need VLANs when receiving a trunk
5. **Virtual routers** don't need VLANs if hypervisor provides separate vNICs
6. **SR-IOV** bypasses host bridge, moves tagging to hardware
7. **Each host has separate bridges** - physical switch connects them
8. **K8s networking** uses same concepts (bridge + veth pairs)
9. **VMs use tap devices** - containers use veth pairs (different mechanisms)
10. **Containers in same pod** share network namespace - communicate via localhost
