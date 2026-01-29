# Building a Network Lab: Understanding TCP/UDP Through Controlled Chaos

<div align="center">

![Status](https://img.shields.io/badge/Status-Completed-success)
![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20VirtualBox-blue)
![Focus](https://img.shields.io/badge/Focus-Networking%20Fundamentals-orange)

**"Networking clicked for me when I started breaking it on purpose."**

</div>

---

## 📖 Introduction
This guide documents a practical learning journey into computer networking fundamentals. The learner, already comfortable with Linux installations and basic Python, sought to deepen their understanding of operating systems, databases, and computer networks before starting their first job. 

Following advice from a Reddit community, they embarked on an enlightening project: **building a multi-VM network topology and intentionally breaking it** to observe how different protocols respond to network impairments.

### The Learning Objective
Rather than memorizing textbook diagrams, this hands-on approach aims to answer fundamental questions:
* How do packets actually travel between machines that can't talk directly?
* What happens when networks become unreliable (latency, packet loss)?
* Why do TCP and UDP behave so differently under poor network conditions?
* How does a router actually forward traffic?

---

## 🗺️ Project Overview

```mermaid
graph LR
    A[VM A<br/>Client<br/>192.168.10.2] -->|LAN A| B[VM B<br/>Router<br/>192.168.10.1 & 192.168.20.1]
    B -->|LAN C| C[VM C<br/>Server<br/>192.168.20.2]
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px
    style B fill:#fff3cd,stroke:#333,stroke-width:2px
    style C fill:#d4edda,stroke:#333,stroke-width:2px
```

### Key Components
* **Three Virtual Machines:** Client, Router, and Server.
* **Two Isolated Networks:** Simulating separate network segments.
* **Manual Routing:** Learning how packets find their destination.
* **Network Impairments:** Artificial latency and packet loss.
* **Protocol Comparison:** TCP vs UDP behavior under stress.

---

## 🏗️ Phase 1: The Setup (Creating the Sandbox)

### Why Virtual Machines on Windows?
The learner initially wondered whether to boot Linux from disk or use their existing Windows installation. The answer: **stay on Windows and use virtualization.**
* **Need all three machines running simultaneously.**
* Type 2 hypervisors (VirtualBox, VMware) run as normal Windows applications.

### Network Architecture Design

```mermaid
graph TB
    subgraph "Physical Host (Windows)"
        subgraph "LAN A (192.168.10.0/24)"
            VMA[VM A - Client<br/>192.168.10.2]
            VMB1[VM B - eth0<br/>192.168.10.1]
        end
        
        subgraph "LAN C (192.168.20.0/24)"
            VMB2[VM B - eth1<br/>192.168.20.1]
            VMC[VM C - Server<br/>192.168.20.2]
        end
        
        VMB1 -.Router VM B.- VMB2
    end
    
    style VMA fill:#e1f5ff
    style VMB1 fill:#fff3cd
    style VMB2 fill:#fff3cd
    style VMC fill:#d4edda
```

### VirtualBox Network Configuration

| VM | Adapter | Network Type | Network Name |
| :--- | :--- | :--- | :--- |
| **VM A** | Adapter 1 | Internal Network | "LanA" |
| **VM B** | Adapter 1 | Internal Network | "LanA" |
| **VM B** | Adapter 2 | Internal Network | "LanC" |
| **VM C** | Adapter 1 | Internal Network | "LanC" |

> **Note:** An optional NAT adapter on VM B can be temporarily enabled for package installation, then disabled afterward.

---

## 🛣️ Phase 2: Manual Routing (Understanding Packet Flow)

### 1. IP Address Assignment
Configure network interfaces with static IP addresses:

```bash
# On VM A (Client)
sudo ip addr add 192.168.10.2/24 dev eth0

# On VM C (Server)
sudo ip addr add 192.168.20.2/24 dev eth0

# On VM B (Router)
sudo ip addr add 192.168.10.1/24 dev eth0  # Facing LAN A
sudo ip addr add 192.168.20.1/24 dev eth1  # Facing LAN C
```

### 2. The Routing Problem
VM A only knows about the `192.168.10.0/24` network. It has no knowledge of how to reach `192.168.20.0/24`.

```mermaid
sequenceDiagram
    participant A as VM A<br/>(192.168.10.2)
    participant B as VM B<br/>(Router)
    participant C as VM C<br/>(192.168.20.2)
    
    Note over A: Wants to reach 192.168.20.2
    Note over A: Only knows about 192.168.10.0/24
    A-xC: ❌ Can't reach directly
    Note over A: No route to destination
```

### 3. The Solution: Routing Tables
Tell machines where to send packets destined for networks they're not directly connected to:

```bash
# On VM A: Tell it to use VM B as gateway for the 192.168.20.0/24 network
sudo ip route add 192.168.20.0/24 via 192.168.10.1

# On VM C: Tell it to use VM B as gateway for the 192.168.10.0/24 network
sudo ip route add 192.168.10.0/24 via 192.168.20.1
```

### 4. Enabling IP Forwarding (The "Secret Sauce")
Even with routing tables, Linux drops packets not addressed to itself by default (Security Policy).

**The Critical Command:**
```bash
# On VM B (Router)
sudo sysctl -w net.ipv4.ip_forward=1
```

**Analogy: The Hotel Receptionist**
* **Forwarding OFF:** "Sorry, policy says I can't deliver items to guests." (Packet dropped)
* **Forwarding ON:** "Sure, I'll run this up to Room 202 for you." (Packet forwarded)

```mermaid
sequenceDiagram
    participant A as VM A<br/>(192.168.10.2)
    participant B as VM B<br/>(ip_forward=1)
    participant C as VM C<br/>(192.168.20.2)
    
    A->>B: Packet for 192.168.20.2
    Note over B: Check routing table<br/>Destination is on eth1
    B->>C: Forward packet
    C->>B: Reply packet
    B->>A: Forward reply
    Note over A: ✅ Communication established!
```

---

## ⚡ Phase 3: Controlled Chaos (Breaking the Network)

### Establishing a Baseline
Before breaking anything, we confirm Gbps speeds and 0 retransmissions using `iperf3`.

### Experiment A: Adding Latency
Simulating satellite or intercontinental links.
```bash
# On VM B (Router)
sudo tc qdisc add dev eth0 root netem delay 200ms
```

```mermaid
graph LR
    A[Packet Arrives] --> B[Wait 200ms]
    B --> C[Forward Packet]
    style B fill:#ffcccc
```

### Experiment B: Packet Loss
Simulating faulty hardware or congestion.
```bash
# On VM B (Router)
sudo tc qdisc change dev eth0 root netem delay 200ms loss 10%
```

```mermaid
graph TD
    A[Incoming Packets] --> B{Random<br/>Decision}
    B -->|90%| C[Forward Packet]
    B -->|10%| D[Drop Packet]
    style D fill:#ff6b6b
```

---

## ⚔️ Phase 4: TCP vs UDP Battle Royale

With **200ms latency** and **10% packet loss**, how do protocols cope?

```mermaid
graph LR
    A[VM A] -->|200ms delay<br/>10% loss| B[VM B Router]
    B --> C[VM C]
    style B fill:#fff3cd
```

### TCP: The Perfectionist
TCP guarantees delivery. When packets are lost, it assumes congestion and slows down.

```mermaid
sequenceDiagram
    participant A as Client
    participant B as Router
    participant C as Server
    
    A->>B: Packet 1
    B->>C: Packet 1 ✓
    A->>B: Packet 2
    B-xC: Packet 2 ✗ (Lost!)
    A->>B: Packet 3
    B->>C: Packet 3 ✓
    Note over A: Timeout waiting for ACK 2
    A->>B: Retransmit Packet 2
    B->>C: Packet 2 ✓
    Note over A: Slow down transmission<br/>(Congestion Control)
```

### UDP: The Honey Badger
UDP fires and forgets. It doesn't care about reliability, only speed.

```mermaid
sequenceDiagram
    participant A as Client
    participant B as Router
    participant C as Server
    
    A->>B: Packet 1
    B->>C: Packet 1 ✓
    A->>B: Packet 2
    B-xC: Packet 2 ✗ (Lost!)
    A->>B: Packet 3
    B->>C: Packet 3 ✓
    A->>B: Packet 4
    B->>C: Packet 4 ✓
    Note over A: Keep sending at full speed<br/>No retransmissions
```

### Performance Comparison

| Metric | TCP | UDP |
| :--- | :--- | :--- |
| **Target Rate** | Dynamic | 100 Mbps |
| **Achieved Rate** | **~10 Mbps** (Throttled) | **~75 Mbps** (Consistent) |
| **Reliability** | 100% (Eventually) | ~75% |
| **Retransmissions** | High | None |
| **Behavior** | Adapts to conditions | Maintains speed |
| **Best For** | File transfers, Web pages | Streaming, VoIP, Gaming |

---

## 🔍 Phase 5: Visualization with Wireshark

Using `tcpdump` on the headless VMs and analyzing the `.pcap` files on the host machine.

```mermaid
graph TD
    A[Open capture_tcp.pcap] --> B{Filter Traffic}
    B -->|TCP| C[Look for Red/Black Lines]
    B -->|UDP| D[No retransmissions]
    
    C --> E[Retransmissions]
    C --> F[Duplicate ACKs]
    C --> G[Out-of-Order Packets]
    
    style E fill:#ff6b6b
    style F fill:#ffd43b
    style G fill:#ffa94d
```

**Key Insights:**
* **Black lines:** TCP retransmissions.
* **Red lines:** Duplicate ACKs and errors.
* **Pattern density:** Shows how hard TCP is working to fix the broken network.

---

## 🧠 Conceptual Summary

```mermaid
---
config:
  theme: 'base'
  themeVariables:
    primaryColor: '#b302fd'
    primaryTextColor: '#1565c0'
    primaryBorderColor: '#7C00'
    lineColor: '#F8B229'
    secondaryColor: '#0ad2fb'
    tertiaryColor: '#9a0af9'
---
mindmap
  root((Network Lab<br/>Learning))
    Setup
      VirtualBox VMs
      Internal Networks
      Static IPs
    Routing
      Routing Tables
      IP Forwarding
      Gateway Configuration
    Breaking
      Latency Injection
      Packet Loss
      Traffic Control
    Analysis
      TCP Behavior
      UDP Behavior
      Performance Metrics
    Tools
      iperf3
      tcpdump
      Wireshark
      tc netem
```

## What We Would Be Doing Next
**Implement a Firewall:** Use `iptables` to block specific traffic.
---

<div align="center">
  <i>"The best way to understand networking is to build something, break it, and watch what happens."</i>
</div>
