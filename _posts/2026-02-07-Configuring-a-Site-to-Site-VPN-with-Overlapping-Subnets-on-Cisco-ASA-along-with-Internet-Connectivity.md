---
title: "Lab: Configuring a Site-to-Site VPN with Overlapping Subnets on Cisco ASA along with Internet Connectivity"
date: 2026-02-07 18:00:00 +0530
categories: [VPN]
tags: [vpn, ikev1, NAT, ASA, pnetlab]
img_path: /assets/img/2026-02-07-asa-overlap/
---

## Lab Objective

In a typical Site-to-Site VPN, both sides must have unique LAN subnets. But what happens during a corporate merger when both companies use the subnet space?

The objective of this lab is to build an IKEv1 IPsec tunnel between two Cisco ASAs where both local LANs overlap perfectly. We will use **Policy NAT (Twice NAT)** to hide the real subnets behind unique "mapped" subnets as traffic traverses the tunnel.

## Network Topology

Below is the logical setup. Note that Site A and Site B are identical subnet ranges.

![Topology Diagram](topology-diagram.png)

### The IP Addressing Scheme

| Location | Real Subnet | NAT Mapped Subnet (Visible to VPN) |
| :--- | :--- | :--- |
| **Site A (ASA-1)** | 192.1.100.0/24 | **192.1.101.0/24** |
| **Site B (ASA-2)** | 192.1.100.0/24 | **192.1.102.0/24** |

> {: .prompt-info }
> **The Key Concept:** When Host A (192.1.100.2) wants to talk to Host B, it won't try to ping Host B's real IP. It must ping Host B's *mapped* IP (e.g., 192.1.102.2). The ASAs will handle the translation in both directions.

---

## Configuration Guide

### Step 1: Defining Network Objects (ASA-1)

On ASA firewalls, clarity is king. We define objects for both the real IPs and the mapped IPs.

```cisco
! ASA-1 Configuration

! 1. Define Local (Site A) Object
object network REAL
 subnet 192.1.100.0 255.255.255.0
 nat (INSIDE,OUTSIDE) dynamic pat-pool interface
```
Above here while creating object for Host A LAN users, A **Dynamic PAT** is configured as **Section 2 NAT (Auto NAT)** which means this NAT Rule only be checked if a traffic didn't match with **Section 1 NAT (Manual NAT)**.

Now Let's create some more object

```cisco
! 2. Define Mapped Object
object network NOTREAL
 subnet 192.1.101.0 255.255.255.0

! 3. Define Remote (Site B) Object
object network VPNDEST          
 subnet 192.1.102.0 255.255.255.0
```
In Policy Based VPN, It is must to match the interesting traffic which actually triggers the VPN. To overwrite the Dynamic PAT Rule only for the interesting VPN traffic except for the traffic towards the internet, Let's configure a Manual NAT in global configuration mode.

```cisco
nat (INSIDE,OUTSIDE) source static REAL NOTREAL destination static VPNDEST VPNDEST
```

When traffic on the INSIDE interface matches:
 - a source of object REAL
 - a destination of object VPNDEST

Send it to the OUTSIDE interface and translate:
 - the source using static translation to object NOTREAL
 - the destination using static translation to object VPNDEST

To match and keep the Destination IP Address as intact, It is required to map the receiving destination IP from INSIDE as same as transmitting destination IP to OUTSIDE.

Let's configure isakmp SA policy, ACL for Interesting Traffic and IPSec SA using transform-set
```cisco
!Phase - I
crypto ikev1 policy 1
 hash sha
 auth pre-share
 group 14
 enc aes-192

tunnel-group 10.1.22.2 type ipsec-l2l
tunnel-group 10.1.22.2 ipsec-attributes
ikev1 pre-shared-key cisco123

!Phase - II
crypto ipsec ikev1 transform-set TR esp-aes-256 esp-sha-hmac

!ACL for interesting traffic
access-list VPNTRF permit ip object NOTREAL object VPNDEST

!Enable ikev1 on OUTISDE interface
crypto ikev1 enable OUTSIDE

!Bind everything and apply the crypto map on the OUTSIDE interface
crypto map CMAP 1 match address VPNTRF
crypto map CMAP 1 set ikev1 transform-set TR
crypto map CMAP 1 set peer 10.1.22.2
crypto map CMAP interface OUTSIDE
```
Same configuration is needed on other Site with logical change in commands of ASA-2 

```cisco
! ASA-2 Configuration

object network REAL
 subnet 192.1.100.0 255.255.255.0
 nat (INSIDE,OUTSIDE) dynamic pat-pool interface

object network NOTREAL    
 subnet 192.1.102.0 255.255.255.0

object network VPNDEST          
 subnet 192.1.101.0 255.255.255.0

nat (INSIDE,OUTSIDE) source static REAL NOTREAL destination static VPNDEST VPNDEST

crypto ikev1 policy 1
 hash sha
 auth pre-share
 group 14
 enc aes-192

tunnel-group 10.1.12.1 type ipsec-l2l
tunnel-group 10.1.12.1 ipsec-attributes
ikev1 pre-shared-key cisco123

crypto ipsec ikev1 transform-set TR esp-aes-256 esp-sha-hmac

access-list VPNTRF permit ip object NOTREAL object VPNDEST

crypto ikev1 enable OUTSIDE

crypto map CMAP 1 match address VPNTRF
crypto map CMAP 1 set ikev1 transform-set TR
crypto map CMAP 1 set peer 10.1.12.1
crypto map CMAP interface OUTSIDE
```

It is required to initiate a traffic from either end in order to create a secure tunnel that gets triggered after matching Policy NAT configured earlier.


To check Phase - I status 
```
show crypto ikev1 sa
```

To check Phase - II status 
```
show crypto ipsec sa
```

To check NAT rule status 
```
show nat detail
```


Great! We've configured a little complex Lab today but It is quiet concerning that how ASA firewall allowing traffic other than TCP protocols? 
To find the answer for this question, Do some digging with 
```
show running-config all sysopt
```
You can download the topology file for this lab below.

<i class="fas fa-download"></i> Download Lab File (.unl) {: .btn .btn--primary }


