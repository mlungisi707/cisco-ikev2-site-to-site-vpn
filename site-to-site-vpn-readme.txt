# Secure Site-to-Site VPN Architecture (IKEv2 / IPsec) on Cisco IOS-XE

## Executive Project Overview
This repository documents the implementation of a permanent, high-security **Site-to-Site IKEv2/IPsec VPN tunnel** built within **Cisco Modeling Labs (CML)**. The architecture models a modern production environment where two physically separate private corporate subnets are securely interconnected over an unsecure public internet transit path.

The deployment utilizes modern, production-grade virtual appliances running **Cisco IOS-XE (CSR1000v / Catalyst 8000V)** and Linux end-hosts. 

### Why Enterprise Engineers Deploy This Architecture
* **Cost Efficiency:** Eliminates the dependency on high-cost, dedicated physical long-haul circuits (such as MPLS links) by leveraging existing, low-cost public internet infrastructure.
* **Data Confidentiality:** Employs Advanced Encryption Standard (**AES-256**) to ensure that intercepted packets are completely unreadable to unauthorized entities.
* **Data Integrity:** Implements Secure Hash Algorithm (**SHA-256**) hashing mechanics to guarantee data cannot be altered or injected while in transit.
* **Authentication Validation:** Restricts tunnel establishment exclusively to verified peers via pre-shared cryptographic keys.

---

## Network Topology & Interface Matrix

![Network Topology](./Site_to_Site_topology.png)
*Figure 1: Virtual Infrastructure Connection Scheme inside Cisco Modeling Labs.*

### Hardware Addressing Map

| Device Name | Interface | Layer 3 IP Address | Subnet Purpose | Next-Hop Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **HQ-PC** | `eth0` | 192.168.10.10/24 | Corporate LAN Segment | 192.168.10.1 |
| **HQ-Router** | `Gi2` (LAN)<br>`Gi1` (WAN) | 192.168.10.1/24<br>10.1.1.1/30 | Local Gateway<br>Public Public-Facing Link | N/A<br>10.1.1.2 (ISP) |
| **ISP-Router** | `Gi1` (HQ)<br>`Gi2` (BR) | 10.1.1.2/30<br>20.1.1.2/30 | Public Internet Backbone<br>Public Internet Backbone | N/A<br>N/A |
| **Branch-Router** | `Gi1` (WAN)<br>`Gi2` (LAN) | 20.1.1.1/30<br>192.168.20.1/24 | Public Public-Facing Link<br>Remote Gateway | 20.1.1.2 (ISP)<br>N/A |
| **Branch-PC** | `eth0` | 192.168.20.10/24 | Remote Office LAN Segment | 192.168.20.1 |

---

## Technical Concept & Command Decoding

### Phase 1: Key Exchange Management Plane (IKEv2)
* **`crypto ikev2 proposal`**: Initializes a modular template to specify security parameters. IKEv2 is heavily favored over IKEv1 due to reduced message exchanges (4 packets vs 9) and native MOBIKE stability.
* **`encryption aes-cbc-256`**: Mandates 256-bit block cipher protection.
* **`integrity sha256` & `prf sha256`**: Enforces Secure Hash Algorithm 2 to handle data fingerprints and drives the Pseudo-Random Function to mathematically derive secure cryptographic keys.
* **`group 14`**: Diffie-Hellman Group 14 utilizes a robust 2048-bit modulus space to securely trade key fragments over public wires.
* **`crypto ikev2 keyring`**: A secure software vault storing local/remote identities and matching pre-shared string passwords.
* **`crypto ikev2 profile`**: The crucial management layer linking the security policy to explicit source peer IP checking rules (`match identity remote`).

### Phase 2: Data Encryption Plane (IPsec)
* **`crypto ipsec transform-set`**: Defines the physical packaging engine for data traffic. It applies **ESP (Encapsulating Security Payload)** to wrap, seal, and protect the actual user data payloads passing through the router.
* **`ip access-list extended`**: Identifies "Interesting Traffic." This tells the router exactly which traffic profiles must be encrypted (Subnet A to Subnet B) vs standard public web lookups.
* **`crypto map`**: The master binder structure combining Phase 1 profiles, Phase 2 transform-sets, and Interesting Traffic ACLs into an executable policy.

---

## Engineering Troubleshooting & Case Study

### The Data Plane Black-Hole Fault
During initial validation tests, public WAN pings succeeded perfectly, but private host-to-host traffic suffered **100% packet loss**. Running diagnostic checks revealed a structural state flaw:

```text
HQ-Router# show crypto ipsec sa
#pkts encaps: 0, #pkts decaps: 0
```

#### Root-Cause Breakdown
1. **Interface Layer Mismatch:** Crypto maps were initially placed on internal LAN interfaces (`Gi2`), blinding the routers to traffic entering or leaving the external WAN interface (`Gi1`). This was resolved by re-binding the maps to the public-facing links.
2. **Static Route Hijacking:** A specific static route configuration (`ip route 192.168.20.0 ... GigabitEthernet1 10.1.1.2`) instructed the routing engine to forward private packets immediately as clear text to the ISP next-hop. This caused the packets to bypass the crypto map engine completely. The ISP router then dropped the unroutable RFC 1918 private packets.

#### The Resolution
By completely negating the specific static routes (`no ip route`), the edge routers were forced to process data using their broad default routes (`0.0.0.0 0.0.0.0`). As the private packet steps through to egress the public WAN interface, it cleanly collides with the bound **Crypto Map**, matching the extended ACL, triggering the IKEv2 handshake, and successfully establishing the encrypted tunnel.

---

## Empirical Verification Proofs

### 1. Verification of Phase 1 Management Tunnel (IKEv2)
Execution of the session lookup confirms the control plane negotiated security parameters cleanly and stands in a verified, active state.

```text
HQ-Router# show crypto ikev2 sa
```
![IKEv2 Phase 1 Status](./IKEv2_Phase1_Active.png)
*Figure 2: Production output demonstrating IKEv2 Security Association settled in a stable READY state.*

### 2. Verification of Phase 2 Data Encapsulation (IPsec)
Running traffic across the endpoints confirms data packets are actively hitting the security engine, getting encapsulated, and decrypted dynamically.

```text
HQ-Router# show crypto ipsec sa
```
![IPsec Phase 2 Counters](./Ipsec_Phase2_Counters.png)
*Figure 3: Data-plane metrics illustrating incremented encapsulation counters proving data is being actively encrypted over the secure link.*