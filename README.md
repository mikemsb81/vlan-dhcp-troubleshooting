# DHCP Configuration and Troubleshooting Across VLANs

Cisco Packet Tracer lab by Michael Bessard.

## Overview

Configured R1 as a DHCP server for two VLANs using an existing router-on-a-stick topology. Verified client settings, automatic leases, and connectivity, then deliberately configured an incorrect DHCP default gateway for VLAN20. Renewing the client's lease reproduced a fault where local communication worked but communication across VLANs failed. Correcting the pool and refreshing the client settings restored connectivity.

**Lab file:** [vlan-dhcp-troubleshooting.pkt](vlan-dhcp-troubleshooting.pkt)

## Topology

| Connection | Purpose |
|---|---|
| R1 G0/0 to SW1 G0/2 | 802.1Q trunk carrying VLANs 10 and 20 |
| SW1 G0/1 to SW2 G0/1 | Trunk carrying VLANs 10 and 20 |
| PC1 to SW1 Fa0/1; PC3 to SW2 Fa0/1 | Access ports in VLAN10, USERS |
| PC2 to SW1 Fa0/2; PC4 to SW2 Fa0/2 | Access ports in VLAN20, SERVERS |

R1 provides both the VLAN gateways and DHCP service. No DHCP relay is required because R1 has an interface in each client subnet. SERVERS is the VLAN name; PC2 and PC4 are PC endpoints in this lab.

## Addressing

| Device/interface | Address | Default gateway | Assignment |
|---|---|---|---|
| R1 G0/0.10 | 192.168.10.1/24 | N/A | Static |
| R1 G0/0.20 | 192.168.20.1/24 | N/A | Static |
| PC1 | 192.168.10.21/24 | 192.168.10.1 | DHCP, observed |
| PC3 | 192.168.10.13/24 | 192.168.10.1 | Existing static configuration |
| PC2 | 192.168.20.12/24 | 192.168.20.1 | Existing static configuration |
| PC4 | 192.168.20.21/24 | 192.168.20.1 | DHCP, observed after recovery |

Addresses .1 through .20 are excluded from DHCP in both subnets, protecting the gateways and the existing static PCs. The remaining allocation range is .21 through .254. A future lease is not guaranteed to use the same address shown here. DNS service was outside this exercise.

## Router configuration

Existing inter-VLAN routing configuration:

```text
interface GigabitEthernet0/0
 no ip address
 no shutdown
!
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```

DHCP configuration added on R1:

```text
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
!
ip dhcp pool VLAN10-USERS
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
!
ip dhcp pool VLAN20-SERVERS
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
```

Selected DHCP under Desktop > IP Configuration on PC1 and PC4. PC2 and PC3 remained static.

## Baseline verification

Router checks:

```text
show ip dhcp pool
show running-config | include ip dhcp excluded-address
show ip dhcp binding
```

The pool display showed both /24 subnet ranges. Its excluded-address count displayed 2 per pool; the running configuration was checked separately and confirmed the intended .1 through .20 ranges. Client assignments were .21 in each subnet.

R1 listed automatic bindings for 192.168.10.21 and 192.168.20.21. Both clients showed the correct /24 mask and their VLAN's .1 gateway.

From PC1:

```text
ping 192.168.10.1
ping 192.168.20.21
```

Both tests returned four replies with 0% loss.

## Controlled fault

Changed only the default gateway supplied by the VLAN20 pool:

```text
configure terminal
ip dhcp pool VLAN20-SERVERS
 no default-router 192.168.20.1
 default-router 192.168.20.254
end
```

On PC4, released and renewed the lease to obtain the changed settings:

```text
ipconfig /release
ipconfig /renew
ipconfig
```

PC4 showed 192.168.20.21/24 with the incorrect gateway 192.168.20.254.

```text
ping 192.168.20.12
ping 192.168.10.21
```

The same-subnet ping to PC2 succeeded with 0% loss. The cross-subnet ping to PC1 failed with 100% loss. PC4 could communicate locally without a gateway, but its supplied gateway did not provide a path to VLAN10.

## Recovery

Corrected the source of the client settings on R1:

```text
configure terminal
ip dhcp pool VLAN20-SERVERS
 no default-router 192.168.20.254
 default-router 192.168.20.1
end
```

Refreshed PC4's lease and verified recovery:

```text
ipconfig /release
ipconfig /renew
ipconfig
ping 192.168.10.21
```

PC4 again showed gateway 192.168.20.1 and received four replies from PC1 with 0% loss. Saved R1's corrected configuration using `copy running-config startup-config` and saved the Packet Tracer file.

## Observed results

| Stage | Check | Result |
|---|---|---|
| Baseline | PC1 and PC4 client settings | Correct address, mask, and gateway |
| Baseline | R1 DHCP bindings | Two automatic leases |
| Baseline | PC1 to 192.168.10.1 | 4/4 replies |
| Baseline | PC1 to PC4, 192.168.20.21 | 4/4 replies |
| Fault | PC4 default gateway | Incorrect: 192.168.20.254 |
| Fault | PC4 to PC2, 192.168.20.12 | 4/4 replies |
| Fault | PC4 to PC1, 192.168.10.21 | 0/4 replies |
| Recovery | PC4 default gateway | Correct: 192.168.20.1 |
| Recovery | PC4 to PC1, 192.168.10.21 | 4/4 replies |

## Screenshot evidence

| Screenshot | Evidence |
|---|---|
| [01-dhcp-pools.png](01-dhcp-pools.png) | DHCP pools and subnet ranges |
| [02-dhcp-exclusions.png](02-dhcp-exclusions.png) | Configured exclusion ranges |
| [03-pc1-dhcp-settings.png](03-pc1-dhcp-settings.png) | PC1 baseline client settings |
| [04-pc4-dhcp-settings.png](04-pc4-dhcp-settings.png) | PC4 baseline client settings |
| [05-dhcp-bindings.png](05-dhcp-bindings.png) | Two automatic leases on R1 |
| [06-dhcp-baseline-connectivity.png](06-dhcp-baseline-connectivity.png) | PC1 gateway and cross-VLAN pings |
| [07-dhcp-wrong-gateway.png](07-dhcp-wrong-gateway.png) | Incorrect gateway received by PC4 |
| [08-dhcp-connectivity-fault.png](08-dhcp-connectivity-fault.png) | Same-subnet success and cross-subnet failure |
| [09-dhcp-recovery-verified.png](09-dhcp-recovery-verified.png) | Corrected gateway and restored connectivity |

## Lessons learned

- DHCP supplies configuration; a successful lease does not prove that all supplied settings are correct.
- Exclusions keep DHCP from allocating addresses reserved for static devices.
- A pool change does not immediately update settings already held by clients. Refresh the client lease and inspect the result.
- Comparing same-subnet and cross-subnet connectivity helps isolate a gateway problem.
- Fixing the DHCP pool addresses the configuration source rather than manually overriding one client.
- Verify recovery with both client settings and an actual connectivity test.

This exercise tested DHCP assignment and one incorrect-gateway fault. It did not test DNS, Internet access, pool exhaustion, DHCP relay, rogue DHCP servers, or DHCP snooping. No packet capture of the DHCP exchange was performed.
