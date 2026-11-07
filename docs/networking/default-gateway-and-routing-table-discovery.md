When a Kubernetes node pings external destinations (such as Google or public internet APIs), it forwards packets to its configured **Default Gateway**

---

## 1. Identification Commands

To inspect the routing table on a node:

```bash
# Returns only the default gateway entry:
ip route show default

# Lists the entire kernel routing table (inspect the top line):
ip route

# Legacy / alternative command:
route -n
```

---

## 2. Full `ip route` Output Analysis

```text
default via 192.168.1.1 dev eth0 proto dhcp metric 100 
172.17.0.0/24 dev cni0 proto kernel scope link src 172.17.0.1 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

- **Line 1 (`default via 192.168.1.1`):** The default route used for any traffic not matching local subnets (e.g., pinging Google or public APIs).
- **`default` (or `0.0.0.0/0`):** Matches all non-local destination IPs.
- **`via 192.168.1.1`:** The IPv4 address of the **Default Gateway** (the authoritative target).
- **`dev eth0`:** The network device used to reach this gateway.
- **Remaining Lines (`scope link`):** Directly connected subnets (`172.17.0.0/24` Pod subnet, `192.168.1.0/24` local node LAN). These require no gateway and are reached via local switching.

---

## 3. CKA Exam Elimination Strategy

When presented with multiple choice options for Default Gateway:

- Values like `255.255.255.0` are **Subnet Masks**, not gateways. Eliminate immediately.
- Values like `8.8.8.8` are public **DNS Servers**, not gateways. Eliminate immediately.
- The Default Gateway is always the router IP indicated after the `via` keyword on the `default` route line.