# Kali Linux Network Troubleshooting and Recovery Report

**Status: Resolved and verified**

## 1. Overview

This report documents the network configuration issue encountered while setting up Kali Linux in Oracle VirtualBox, the changes made to the VirtualBox NAT Network and Kali's network profile, and the connectivity checks completed after recovery.

The lab required Kali to use the static IPv4 address `10.0.0.2/24`. Initially, Kali received `10.0.0.3/24` through DHCP. When the static address was configured, NetworkManager reported that `10.0.0.2` was already in use and did not activate the connection. The VirtualBox DHCP configuration was then corrected, Kali was restarted, and the required static address became active.

The final checks confirmed the expected IP address and route, successful gateway connectivity, successful HTTPS access to an external website, working DNS resolution, and availability of Nmap. A VirtualBox snapshot named `Clean Kali - Network Verified` was created, and network checks were completed after using the snapshot.

## 2. Environment and target configuration

| Component | Configuration |
|---|---|
| Virtualization platform | Oracle VirtualBox |
| Guest operating system | Kali Linux |
| VM image shown in VirtualBox | `kali-linux-2026.3-virtualbox-amd64` |
| VirtualBox adapter attachment | NAT Network |
| NAT Network name | `NatNetwork` |
| IPv4 network | `10.0.0.0/24` |
| Kali network interface | `eth0` |
| Required Kali address | `10.0.0.2/24` |
| Default gateway | `10.0.0.1` |
| Final DHCP server address | `10.0.0.254` |
| Final DHCP allocation range | `10.0.0.3`–`10.0.0.253` |
| DHCP server | Enabled |
| NAT Network IPv6 | Disabled |

## 3. Issue 1 — DHCP assigned an address outside the lab requirement

### Initial observation

The interface and route were inspected using:

```bash
ip -br addr
ip route
```

Kali initially received `10.0.0.3/24` on `eth0`. The interface was active and had a route through `10.0.0.1`, but its address did not match the lab requirement of `10.0.0.2/24`.

### Investigation

The VirtualBox DHCP server configuration was checked from Windows:

```cmd
G:\VBox\VBoxManage list dhcpservers
```

The initial settings showed the DHCP server using `10.0.0.2`, with a lease range beginning at `10.0.0.3` and ending at `10.0.0.254`. This conflicted with the address reserved for the Kali VM.

The NAT Network settings were also checked:

```cmd
G:\VBox\VBoxManage list natnets
```

The network was `10.0.0.0/24`, with gateway `10.0.0.1`, DHCP enabled, and IPv6 disabled.

### Resolution

The VirtualBox DHCP server was moved to `10.0.0.254`, and its allocation range was changed to `10.0.0.3`–`10.0.0.253`:

```cmd
G:\VBox\VBoxManage dhcpserver modify --network=NatNetwork --server-ip=10.0.0.254 --lower-ip=10.0.0.3 --upper-ip=10.0.0.253
```

The updated configuration was verified with:

```cmd
G:\VBox\VBoxManage list dhcpservers
```

The resulting configuration showed the DHCP server at `10.0.0.254`, the allocation range `10.0.0.3`–`10.0.0.253`, and DHCP enabled. This left `10.0.0.2` outside the dynamic allocation range for the Kali VM's static address.

## 4. Issue 2 — NetworkManager rejected the static address

### Observed failure

Kali's NetworkManager connection profile was configured for manual IPv4 addressing. During an activation attempt, NetworkManager returned an error indicating that IP configuration could not be reserved. Its logs reported that `10.0.0.2` was already in use on the network, identifying a MAC address of `08:00:27:DB:80:4B` in the duplicate-address message.

The earlier DHCP configuration explained why the address allocation needed correction. However, the exact owner of the address reported by duplicate-address detection was not independently identified, so the report does not attribute the warning to a specific device without proof.

### Resolution and final interface state

After the VirtualBox DHCP configuration was corrected, the Kali VM was restarted. The final interface inspection showed:

```text
eth0    UP    10.0.0.2/24
```

The routing table showed:

```text
default via 10.0.0.1 dev eth0 proto static metric 100
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.2 metric 100
```

The NetworkManager profile inspection confirmed `ipv4.method: manual`, `ipv4.addresses: 10.0.0.2/24`, and `ipv4.gateway: 10.0.0.1`. The per-connection `ipv4.dns` value was blank in the captured final profile; DNS resolution was therefore verified separately rather than reporting an unverified DNS-server setting.

## 5. Final network configuration

| Parameter | Final value / result |
|---|---|
| VirtualBox attachment | NAT Network |
| NAT Network | `NatNetwork` |
| Network CIDR | `10.0.0.0/24` |
| Kali interface | `eth0` — UP |
| Kali IPv4 address | `10.0.0.2/24` |
| IPv4 method | Manual/static |
| Default gateway | `10.0.0.1` |
| DHCP server | `10.0.0.254` |
| DHCP allocation range | `10.0.0.3`–`10.0.0.253` |
| DHCP enabled | Yes |
| NAT Network IPv6 | Disabled |
| DNS resolution | Successful for `example.com` |

## 6. Connectivity and tool verification

The tests below were run in Kali after the static IP configuration was active. They were also repeated as part of the post-snapshot recovery check.

### 6.1 Gateway connectivity — PASS

Command:

```bash
ping -c 4 10.0.0.1
```

Result: **4 packets transmitted, 4 received, 0% packet loss.** Kali could reach the NAT Network gateway.

### 6.2 External HTTPS connectivity — PASS

Command:

```bash
curl -I --max-time 10 https://example.com
```

Result: **`HTTP/2 200`**. The request completed successfully, confirming HTTPS connectivity to the external website.

### 6.3 DNS resolution — PASS

Command:

```bash
getent ahostsv4 example.com
```

Result: The command returned IPv4 address records for `example.com`, including `104.20.23.154` and `172.66.147.243` in the captured output. Name resolution was working at the time of the test.

### 6.4 Nmap availability — PASS

Command:

```bash
nmap --version
```

Result: **Nmap version 7.99**. This verified that Nmap was installed and executable; this step was a version check, not a port scan.

## 7. VirtualBox snapshot and recovery verification

A VirtualBox snapshot named **`Clean Kali - Network Verified`** was created on **27 October 2026**, as shown in VirtualBox Manager. The snapshot tree subsequently displayed `Current State (changed)`, indicating that the VM had continued to change after the snapshot was taken.

After using the snapshot, the recovery check re-ran the interface, routing, gateway, and HTTPS tests. The captured output showed `eth0` active at `10.0.0.2/24`, the default route via `10.0.0.1`, four successful gateway ping replies with `0% packet loss`, and an `HTTP/2 200` response from `https://example.com`.

**Recovery result: PASS.** The required static network configuration and external connectivity were present during the post-snapshot check.

## 8. Lessons learned

The incident demonstrated that successful DHCP connectivity does not necessarily satisfy a lab's required addressing scheme. The DHCP server address and lease range should be checked before assigning a static address within the same subnet. NetworkManager logs are useful for diagnosing duplicate-address detection, while interface and route output confirm the configuration actually in use. Gateway reachability, external HTTPS access, DNS resolution, and tool availability should be checked independently because each validates a different part of the environment.

The final configuration and recovery checks were successful. The duplicate-address warning was observed and documented, while its exact originating device was left unspecified because it was not independently identified.


## Evidence file references

The main repository README maps the captured checks to evidence files in `../evidence/`. The key captures are `08-kali-ip-address.png`, `09-kali-routing-table.png`, `10-networkmanager-profile.png`, `11-gateway-ping-test.png`, `12-external-https-test.png`, `13-dns-resolution-test.png`, `14-nmap-version.png`, `15-virtualbox-snapshot-created.png`, and `17-post-snapshot-recovery-test.png`.
