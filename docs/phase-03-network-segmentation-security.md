# Phase 3 - Network Segmentation & Firewalling

## Overview

Phase 3 replaces the original flat ForgeLine network with a segmented architecture.

Until this point, domain controllers, servers, management systems and workstations shared the same network. This worked for the initial deployment, but provided no meaningful network boundary between user devices and infrastructure.

In this phase, OPNsense was introduced as the central Layer 3 router and firewall. Management, server and client systems were moved into separate networks, and explicit firewall policy was implemented between them.

The goal was not simply to create different subnets, but to control which systems are allowed to communicate and on which services.

---

## Architecture

| Zone | Subnet | Gateway | Purpose |
|---|---|---|---|
| WAN | `192.168.250.0/24` | VMware NAT | Upstream connectivity |
| MGMT | `10.10.10.0/24` | `10.10.10.1` | Administrative systems |
| SERVERS | `10.10.20.0/24` | `10.10.20.1` | Domain and infrastructure servers |
| CLIENTS | `10.10.30.0/24` | `10.10.30.1` | Domain workstations |

```text
                    Internet
                       |
                  VMware NAT
                192.168.250.0/24
                       |
                 +-------------+
                 |  OPNsense   |
                 |-------------|
                 | WAN         |
                 | MGMT        | 10.10.10.1
                 | SERVERS     | 10.10.20.1
                 | CLIENTS     | 10.10.30.1
                 +-------------+
                    /    |    \
                   /     |     \
                MGMT  SERVERS  CLIENTS
                  |      |        |
               MGMT01   DC01   CLIENT01
                        DC02   CLIENT02
                       FILE01
```

Separate VMware virtual networks provide the Layer 2 boundaries:

| VMware Network | Purpose |
|---|---|
| VMnet8 | WAN / NAT |
| VMnet2 | MGMT |
| VMnet3 | SERVERS |
| VMnet4 | CLIENTS |

Internal VMware DHCP was disabled. Routing and policy between internal networks are handled by OPNsense.

---

## Infrastructure Migration

The existing servers were migrated from the original `10.10.10.0/24` network to the dedicated SERVERS network.

| System | Address | Role |
|---|---|---|
| DC01 | `10.10.20.10` | AD DS / DNS |
| DC02 | `10.10.20.11` | AD DS / DNS |
| FILE01 | `10.10.20.20` | File Server |
| MGMT01 | `10.10.10.21` | Management Server |

Both domain controllers use `10.10.20.1` as their default gateway and continue to use the internal AD DNS infrastructure.

After migration, DNS registration and Netlogon registration were refreshed and stale records from the previous addressing were removed.

AD replication was then validated:

```powershell
repadmin /replsummary
repadmin /showrepl DC01
repadmin /showrepl DC02
```

Final replication status:

```text
Source DSA     fails/total
DC01           0 / 5
DC02           0 / 5

Destination DSA
DC01           0 / 5
DC02           0 / 5
```

Domain, Configuration, Schema, DomainDnsZones and ForestDnsZones replication all completed successfully after the network migration.

---

## DNS Architecture

Domain clients continue to use the ForgeLine domain controllers as their DNS servers:

```text
DC01    10.10.20.10
DC02    10.10.20.11
```

Clients do not use public DNS resolvers directly.

External queries follow the internal DNS infrastructure:

```text
CLIENT
   |
   | DNS
   v
DC01 / DC02
   |
   | Forward unresolved external query
   v
External DNS Forwarders
   |
   v
Internet
```

This preserves Active Directory DNS resolution while still providing external name resolution.

Forward and reverse DNS are configured for the infrastructure.

Examples:

```text
DC01.corp.forgeline.test   -> 10.10.20.10
DC02.corp.forgeline.test   -> 10.10.20.11
FILE01.corp.forgeline.test -> 10.10.20.20
```

Reverse:

```text
10.10.20.10 -> DC01.corp.forgeline.test
10.10.20.11 -> DC02.corp.forgeline.test
10.10.20.20 -> FILE01.corp.forgeline.test
```

The reverse lookup zone was added during Phase 3 validation after identifying it as a missing part of the original Phase 1 DNS configuration.

---

## Client DHCP

OPNsense Kea DHCP provides addressing for the CLIENTS network.

```text
Subnet:      10.10.30.0/24
Pool:        10.10.30.100 - 10.10.30.199
Gateway:     10.10.30.1

DNS:
10.10.20.10
10.10.20.11

Domain:
corp.forgeline.test
```

A client therefore receives everything required to operate as a domain workstation without relying on VMware DHCP or external DNS.

---

## Firewall Design

Firewall policy is based around explicit access between security zones rather than unrestricted routing.

Aliases were created for infrastructure groups and service sets:

```text
FORGELINE_DCS
    DC01
    DC02

FORGELINE_FILE_SERVERS
    FILE01

EXTERNAL_DNS
    Approved DNS forwarders

AD_TCP_PORTS
AD_UDP_PORTS
WEB_PORTS
```

Using aliases keeps policy readable and allows infrastructure members or service definitions to be changed without rebuilding individual rules.

### CLIENTS Policy

Domain workstations are allowed to reach the domain controllers only on services required for Active Directory operation.

This includes DNS, Kerberos, LDAP, SMB, RPC, Global Catalog and the required dynamic RPC range.

Clients are also allowed SMB access to FILE01.

The resulting policy is logically:

```text
CLIENTS -> Domain Controllers
    ALLOW required AD services

CLIENTS -> FILE01
    ALLOW SMB

CLIENTS -> MGMT
    BLOCK

CLIENTS -> SERVERS
    BLOCK everything not explicitly permitted

CLIENTS -> Internet
    ALLOW HTTP/HTTPS

Everything else
    DENY
```

This means a workstation can authenticate against the domain, receive Group Policy and access authorized file shares without receiving unrestricted access to the infrastructure network.

### SERVERS Policy

Servers are prevented from freely initiating connections toward user or management networks.

```text
SERVERS -> MGMT
    BLOCK

SERVERS -> CLIENTS
    BLOCK

Domain Controllers -> approved external DNS
    ALLOW TCP/UDP 53

SERVERS -> Internet
    ALLOW HTTP/HTTPS

Everything else
    DENY
```

OPNsense is stateful, so response traffic belonging to an established permitted connection does not require a mirrored rule in the opposite direction.

For example, allowing:

```text
CLIENT01 -> FILE01 : TCP 445
```

allows FILE01 to respond to that connection without granting FILE01 unrestricted permission to initiate connections toward the CLIENTS network.

---

## Firewall Rule Ordering

One important part of the implementation was ensuring that broad external access rules could not override internal segmentation.

The policy follows this order:

```text
Specific required internal ALLOW
              |
              v
Protected internal network BLOCK
              |
              v
Permitted external ALLOW
              |
              v
Implicit DENY
```

For example, an HTTP/HTTPS rule with destination `ANY` must not appear before a rule blocking CLIENTS from the management network.

Otherwise:

```text
CLIENT -> MGMT01:443
```

could match the broad web rule before reaching the internal block.

This reinforced an important firewall principle:

> A correct rule with incorrect ordering can still produce an incorrect security policy.

---

## Troubleshooting

### DNS Resolution After Network Migration

Following the migration, normal `nslookup` queries sometimes produced timeout messages despite eventually returning the correct record.

The DNS service was verified independently using:

```powershell
Resolve-DnsName
Test-NetConnection
```

and AD replication remained healthy.

Debugging `nslookup` exposed queries such as:

```text
dc02.corp.forgeline.test.corp.forgeline.test
```

showing that the configured DNS search suffix was being appended during resolution attempts.

Using the absolute FQDN:

```text
dc02.corp.forgeline.test.
```

returned immediately.

This was useful because the initial symptom looked like a DNS server or network failure, while inspection of the actual queries showed that the DNS infrastructure itself was responding correctly.

### Remote DNS Administration

After segmentation, remote DNS administration from MGMT01 was also validated across the firewall boundary.

Connectivity to DNS, RPC and SMB services was confirmed, and remote DNS management worked correctly when addressing the domain controller through its domain identity:

```text
DC01.corp.forgeline.test
```

rather than relying on the raw IP address.

This also provided an additional validation that management traffic was successfully crossing the new MGMT-to-SERVERS boundary.

### Reverse DNS

Reverse DNS had not been configured during the initial Phase 1 deployment.

This became visible during Phase 3 DNS validation when reverse lookups could not identify the DNS servers by hostname.

A reverse lookup zone and PTR records were added for the server network, completing the DNS configuration that should originally have been part of Phase 1.

---

## Validation

The final environment was tested from multiple zones rather than assuming that successful configuration meant successful enforcement.

### Network and DHCP

CLIENT01 successfully received:

```text
IPv4:       10.10.30.101
Gateway:    10.10.30.1
DNS:        10.10.20.10 / 10.10.20.11
DNS suffix: corp.forgeline.test
```

### DNS

Internal infrastructure records resolve through both domain controllers.

External DNS resolution works through the domain DNS servers and their configured forwarders.

### Active Directory

Domain authentication, domain controller discovery and required AD communication continue to function across the routed network.

Group Policy remains operational for domain workstations.

### File Services

Clients can reach FILE01 over SMB while the existing AGDLP permissions continue to determine authorization to individual shares.

This creates separate controls at two layers:

```text
Firewall
    -> Can this workstation reach the service?

NTFS / Share permissions
    -> Is this user authorized to access the data?
```

### Segmentation

CLIENTS cannot freely initiate connections toward MGMT.

CLIENTS cannot access arbitrary services in SERVERS.

Required AD and SMB traffic remains functional.

MGMT retains administrative connectivity toward SERVERS.

SERVERS cannot freely initiate new connections toward CLIENTS or MGMT.

### Internet

Client Internet access works through OPNsense and outbound NAT while DNS remains centralized through the domain controllers.

### Domain Controller Failover

DC failover was tested again after segmentation.

The environment remained functional when one domain controller became unavailable, confirming that the redundancy implemented earlier still works after introducing routed network boundaries and firewall policy.

---

## Result

Phase 3 changed ForgeLine from a flat lab network into a routed and policy-controlled environment.

The final architecture provides:

- Dedicated management, server and client networks
- Central routing and firewalling through OPNsense
- Kea DHCP for client systems
- Centralized AD DNS with controlled external forwarding
- Forward and reverse DNS
- Restricted workstation access to infrastructure
- Dedicated management access to servers
- Restricted server-initiated traffic
- Controlled Internet access
- Stateful firewall policy
- Preserved Active Directory replication and failover
- Preserved Group Policy and file services

The important change is not simply that ForgeLine now uses multiple subnets.

Traffic between security zones now crosses an explicit policy boundary, allowing access based on system role and required service rather than physical presence on the same network.