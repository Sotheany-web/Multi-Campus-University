# Multi-Campus-University
The network connects a Main Campus (Admin and Library) with a Remote Dormitory.  Different user groups (Admin, Student, Faculty, Guest) must be separated.  The design aims to provide stable connections, high availability, and secure communication.

# Packet Tracer Build Guide — Campus ↔ Dorm (VLSM, VLANs, LACP EtherChannel, Multi‑Area OSPF, WPA2)

This is a complete, **copy/paste** configuration guide with **all design choices made** (addressing, VLAN IDs, ports, routing areas, security hardening, verification).

---

## 0) Devices (as in your diagram)

- **Campus-Router**: Cisco **2911**
- **Dorm-Router**: Cisco **2911**
- **Core-Switch**: Cisco **3560-24PS**
- **Distribution-Switch**: Cisco **3560-24PS**
- **Access-Switch-Admin**: Cisco **2960-24TT**
- **Access-Switch-Library**: Cisco **2960-24TT**
- **Dorm-Switch**: Cisco **2960-24TT**
- **Wi‑Fi Router/AP**: **WRT300N**
- Endpoints: Admin-PC, Faculty-PC, Student-PC, Guest-Laptop

---

## 1) Mandatory Change to Your Diagram (WRT300N placement)

**Remove** the cable from **Dorm-Router ↔ WRT300N**.

**Add** a straight-through cable from:

- **Dorm-Switch Fa0/10** → **WRT300N LAN Port (Ethernet 1)**

**Important:** Do **NOT** use WRT300N “Internet/WAN” port. We are using it as an **Access Point** for Guest VLAN.

---

## 2) VLSM Addressing Plan (custom, minimal waste)

Base block: **10.10.0.0/24**

| Purpose | VLAN | Subnet | Mask | Gateway |
|---|---:|---|---|---|
| Student (largest) | 20 | 10.10.0.0/26 | 255.255.255.192 | 10.10.0.1 |
| Guest Wi‑Fi | 40 | 10.10.0.64/27 | 255.255.255.224 | 10.10.0.65 |
| Admin | 10 | 10.10.0.96/28 | 255.255.255.240 | 10.10.0.97 |
| Faculty | 30 | 10.10.0.112/28 | 255.255.255.240 | 10.10.0.113 |
| Switch Mgmt | 99 | 10.10.0.128/28 | 255.255.255.240 | 10.10.0.129 |
| WAN link | — | 10.10.0.144/30 | 255.255.255.252 | Campus 10.10.0.145 / Dorm 10.10.0.146 |

**OSPF Areas**
- **Area 0 (Backbone):** WAN + Dorm networks
- **Area 1 (Campus):** Campus VLANs

---

## 3) Port / Link Map (chosen so you can follow exactly)

### Campus side
- **CORE Gi0/1 + Gi0/2** ↔ **DIST Gi0/1 + Gi0/2** (LACP EtherChannel Po1, trunk)
- **CORE Fa0/1** ↔ **Campus-Router Gi0/0** (802.1Q trunk)
- **DIST Fa0/23** ↔ **ACC_ADMIN Fa0/24** (trunk)
- **DIST Fa0/24** ↔ **ACC_LIBRARY Fa0/24** (trunk)

### Dorm side
- **Dorm-Router Gi0/0** ↔ **DORM_SW Fa0/1** (trunk)
- **DORM_SW Fa0/2** ↔ Student-PC (access VLAN 20)
- **DORM_SW Fa0/10** ↔ WRT300N LAN (access VLAN 40)

### WAN
- **Campus-Router Gi0/1** ↔ **Dorm-Router Gi0/1** (routed /30)

---

## 4) Standard Device Hardening (apply to EVERY router and switch)

Run this on each device (adjust hostname line only if you want): REMINDER: when use `crypto key generate rsa` write `1024`

```cisco
conf t
enable secret C1sco@123
service password-encryption
no ip domain-lookup
banner motd ^CUnauthorized access prohibited.^C

username admin privilege 15 secret Adm1n@123
ip domain-name campus.local
crypto key generate rsa
ip ssh version 2

line con 0
 logging synchronous
 password Con@123
 login

line vty 0 4
 login local
 transport input ssh
 exec-timeout 10 0
end
wr
```

---

## 5) Switch Configurations

### 5.1 CORE (3560)

```cisco
enable
conf t
hostname CORE

vlan 10
 name ADMIN
vlan 20
 name STUDENT
vlan 30
 name FACULTY
vlan 40
 name GUEST
vlan 99
 name MGMT

! LACP EtherChannel to DIST
interface range gi0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 channel-group 1 mode active
 no shut

interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 no shut

! Trunk to Campus Router (router-on-a-stick)
interface fa0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,30,99
 no shut

end
wr
```

> Note: Campus router only needs VLAN 10 + 30 + 99 in this design. Student/Guest live in Dorm.

---

### 5.2 DIST (3560)

```cisco
enable
conf t
hostname DIST

vlan 10
 name ADMIN
vlan 20
 name STUDENT
vlan 30
 name FACULTY
vlan 40
 name GUEST
vlan 99
 name MGMT

! LACP EtherChannel to CORE
interface range gi0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 channel-group 1 mode active
 no shut

interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 no shut

! Trunks to access switches
interface fa0/23
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 no shut

interface fa0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 no shut

end
wr
```

---

### 5.3 ACC_ADMIN (2960)

Admin-PC is on **Fa0/1** in **VLAN 10**.

```cisco
enable
conf t
hostname ACC_ADMIN

vlan 10
 name ADMIN
vlan 30
 name FACULTY
vlan 99
 name MGMT

interface fa0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,30,99
 no shut

interface fa0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shut

! Switch management
interface vlan 99
 ip address 10.10.0.130 255.255.255.240
 no shut
ip default-gateway 10.10.0.129

end
wr
```

---

### 5.4 ACC_LIBRARY (2960)

Faculty-PC is on **Fa0/1** in **VLAN 30**.

```cisco
enable
conf t
hostname ACC_LIBRARY

vlan 10
 name ADMIN
vlan 30
 name FACULTY
vlan 99
 name MGMT

interface fa0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,30,99
 no shut

interface fa0/1
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shut

! Switch management
interface vlan 99
 ip address 10.10.0.131 255.255.255.240
 no shut
ip default-gateway 10.10.0.129

end
wr
```

---

### 5.5 DORM_SW (2960) — includes the WRT change

```cisco
enable
conf t
hostname DORM_SW

vlan 20
 name STUDENT
vlan 40
 name GUEST
vlan 99
 name MGMT

! Trunk to Dorm Router
interface fa0/1
 switchport mode trunk
 switchport trunk allowed vlan 20,40,99
 no shut

! Student-PC
interface fa0/2
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shut

! WRT300N LAN (Guest Wi-Fi)
interface fa0/10
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shut

! Switch management
interface vlan 99
 ip address 10.10.0.132 255.255.255.240
 no shut
ip default-gateway 10.10.0.129

end
wr
```

---

## 6) Router Configurations

### 6.1 Campus-Router (2911)

- Gi0/0 to CORE trunk
- Gi0/1 to Dorm-Router WAN

```cisco
enable
conf t
hostname CAMPUS_RTR

! WAN to Dorm
interface gi0/1
 ip address 10.10.0.145 255.255.255.252
 no shut

! Trunk to CORE
interface gi0/0
 no shut

interface gi0/0.10
 encapsulation dot1Q 10
 ip address 10.10.0.97 255.255.255.240

interface gi0/0.30
 encapsulation dot1Q 30
 ip address 10.10.0.113 255.255.255.240

interface gi0/0.99
 encapsulation dot1Q 99
 ip address 10.10.0.129 255.255.255.240

! DHCP (Campus only)
ip dhcp excluded-address 10.10.0.97 10.10.0.100
ip dhcp excluded-address 10.10.0.113 10.10.0.116

ip dhcp pool ADMIN
 network 10.10.0.96 255.255.255.240
 default-router 10.10.0.97
 dns-server 8.8.8.8

ip dhcp pool FACULTY
 network 10.10.0.112 255.255.255.240
 default-router 10.10.0.113
 dns-server 8.8.8.8

! OSPF multi-area
router ospf 1
 router-id 1.1.1.1
 network 10.10.0.144 0.0.0.3 area 0
 network 10.10.0.96 0.0.0.15 area 1
 network 10.10.0.112 0.0.0.15 area 1
 network 10.10.0.128 0.0.0.15 area 1

end
wr
```

---

### 6.2 Dorm-Router (2911)

- Gi0/0 to Dorm-Switch trunk
- Gi0/1 to Campus-Router WAN

```cisco
enable
conf t
hostname DORM_RTR

! WAN to Campus
interface gi0/1
 ip address 10.10.0.146 255.255.255.252
 no shut

! Trunk to Dorm switch
interface gi0/0
 no shut

interface gi0/0.20
 encapsulation dot1Q 20
 ip address 10.10.0.1 255.255.255.192

interface gi0/0.40
 encapsulation dot1Q 40
 ip address 10.10.0.65 255.255.255.224

interface gi0/0.99
 encapsulation dot1Q 99
 ip address 10.10.0.133 255.255.255.240

! DHCP (Dorm only)
ip dhcp excluded-address 10.10.0.1 10.10.0.10
ip dhcp excluded-address 10.10.0.65 10.10.0.70

ip dhcp pool STUDENT
 network 10.10.0.0 255.255.255.192
 default-router 10.10.0.1
 dns-server 8.8.8.8

ip dhcp pool GUEST
 network 10.10.0.64 255.255.255.224
 default-router 10.10.0.65
 dns-server 8.8.8.8

! (Optional) Guest isolation (recommended for “traffic separation”)
ip access-list extended GUEST_ISOLATION
 deny ip 10.10.0.64 0.0.0.31 10.10.0.96 0.0.0.15
 deny ip 10.10.0.64 0.0.0.31 10.10.0.112 0.0.0.15
 deny ip 10.10.0.64 0.0.0.31 10.10.0.0 0.0.0.63
 permit ip 10.10.0.64 0.0.0.31 any

interface gi0/0.40
 ip access-group GUEST_ISOLATION in

! OSPF backbone
router ospf 1
 router-id 2.2.2.2
 network 10.10.0.144 0.0.0.3 area 0
 network 10.10.0.0 0.0.0.63 area 0
 network 10.10.0.64 0.0.0.31 area 0
 network 10.10.0.128 0.0.0.15 area 0

end
wr
```

---

## 7) WRT300N Configuration (AP mode + WPA2)

**Cabling:** Dorm-Switch Fa0/10 → WRT300N **LAN** (Ethernet 1)

On **WRT300N → GUI**:

### Setup / Basic Setup
- Local IP: **10.10.0.66**
- Subnet Mask: **255.255.255.224**
- Default Gateway: **10.10.0.65**
- DHCP Server: **Disable**

### Wireless
- SSID: **DormGuest**
- Security: **WPA2-Personal**
- Passphrase: **DormGuest123!**

On Guest-Laptop:
- Join SSID **DormGuest**
- Enter passphrase
- IP config: **DHCP**

---

## 8) End Device Settings

Set each end device to **DHCP**:

- Admin-PC (VLAN 10)
- Faculty-PC (VLAN 30)
- Student-PC (VLAN 20)
- Guest-Laptop (Wi‑Fi VLAN 40)

---

## 9) Verification (required connectivity proof)

### 9.1 EtherChannel / Trunk checks
On CORE and DIST:

```cisco
show etherchannel summary
show interfaces port-channel 1 trunk
show interfaces trunk
```

### 9.2 OSPF checks
On both routers:

```cisco
show ip ospf neighbor
show ip route ospf
```

### 9.3 Ping + Traceroute (take screenshots for submission)
From **Admin-PC**:
- `ping 10.10.0.1`
- `tracert 10.10.0.1`

From **Student-PC**:
- `ping 10.10.0.97`
- `tracert 10.10.0.97`

From **Guest-Laptop** (after Wi‑Fi connects):
- `ping 10.10.0.65`
- `tracert 10.10.0.65`

If you enabled **Guest isolation ACL**, Guest-Laptop should **fail** ping/tracert to Admin/Faculty/Student subnets but still reach “any” routes allowed by your environment.

---

## 10) Save Often (Packet Tracer stability tip)

Save versions like:
- `v1.pkt`, `v2.pkt`, `v3.pkt`

---

