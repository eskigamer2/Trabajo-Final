# NetSecure Solutions — Sede San Francisco de Macorís

## Rol en la topología

Sede regional conectada al [ISP](./ISP.md) mediante un enlace WAN dedicado y
un túnel DMVPN/IPSec (spoke) hacia el hub en Santo Domingo (área OSPF 6).
DHCP para las VLANs de esta sede se resuelve **localmente en el router**
(no depende de un servidor externo). Autenticación AAA vía RADIUS contra el
servidor del datacenter en [Santiago](./Santiago.md) (`10.1.0.34`).

### Departamentos / VLANs

| VLAN | Nombre | Subred |
|---:|---|---|
| 600 | DEFAULT-SFM | — |
| 610 | VENTAS-REGIONAL-SFM | `10.5.0.0/25` |
| 620 | OPERACIONES-SFM | `10.5.0.128/26` |
| 630 | LEGAL-CUMPLIMIENTO-SFM | `10.5.0.192/28` |
| 699 | MGMT-SFM | `10.5.1.0/28` |

### Dispositivos

- `R-SFM` — Router de sede (WAN al ISP, subinterfaces 802.1Q, DHCP, NAT, DMVPN spoke)
- `SWD-SFM-1` — Switch de distribución (trunk + EtherChannel L2 hacia el switch de acceso)
- `SWA-SFM-1` — Switch de acceso (puertos de usuario, port-security, DHCP snooping, DAI)

---

## Configuraciones

<details>
<summary><strong>R-SFM (Router)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname R-SFM
no ip domain lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

ip cef
no ipv6 cef

!
! ============================================================
! BANNER
! ============================================================
!

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SAN FRANCISCO DE MACORIS)
ADVERTENCIA: ACCESO RESTRINGIDO. AUTENTICACION RADIUS ACTIVA.
*********************************************************************
#

!
! ============================================================
! SSH
! ============================================================
!

crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2

line console 0
 logging synchronous
 exec-timeout 10 0
exit

line vty 0 4
 transport input ssh
 exec-timeout 10 0
exit

!
! ============================================================
! RADIUS / AAA
! ============================================================
!

aaa new-model
aaa authentication login default group radius local

radius server SANTIAGO-RADIUS
 address ipv4 10.1.0.34 auth-port 1812 acct-port 1813
 key Netsecure-EMP1
exit

!
! ============================================================
! INTERFAZ WAN
! ============================================================
!

interface Ethernet0/0
 description ENLACE-WAN-ISP
 ip address 1.0.0.22 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex full
 no shutdown
exit

!
! ============================================================
! TRUNK HACIA SWD-SFM-1
! ============================================================
!

interface Ethernet0/1
 description TRUNK-A-SWD-SFM-1
 no ip address
 duplex full
 no shutdown
exit

interface Ethernet0/1.610
 description GATEWAY-VENTAS
 encapsulation dot1Q 610
 ip address 10.5.0.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 6
exit

interface Ethernet0/1.620
 description GATEWAY-OPERACIONES
 encapsulation dot1Q 620
 ip address 10.5.0.129 255.255.255.192
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 6
exit

interface Ethernet0/1.630
 description GATEWAY-LEGAL
 encapsulation dot1Q 630
 ip address 10.5.0.193 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 6
exit

interface Ethernet0/1.699
 description GATEWAY-MGMT
 encapsulation dot1Q 699
 ip address 10.5.1.1 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 6
exit

!
! ============================================================
! DHCP
! ============================================================
!

ip dhcp excluded-address 10.5.0.1 10.5.0.3
ip dhcp excluded-address 10.5.0.129 10.5.0.131
ip dhcp excluded-address 10.5.0.193 10.5.0.195
ip dhcp excluded-address 10.5.1.1 10.5.1.3

ip dhcp pool VLAN610-VENTAS
 network 10.5.0.0 255.255.255.128
 default-router 10.5.0.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN620-OPERACIONES
 network 10.5.0.128 255.255.255.192
 default-router 10.5.0.129
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN630-LEGAL
 network 10.5.0.192 255.255.255.240
 default-router 10.5.0.193
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN699-MGMT
 network 10.5.1.0 255.255.255.240
 default-router 10.5.1.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

!
! ============================================================
! OSPF / ENRUTAMIENTO
! ============================================================
!

router ospf 21
 router-id 6.6.6.6
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Tunnel1
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.21

!
! ============================================================
! NAT
! ============================================================
!

ip access-list extended NAT-ACL-SFM
 deny ip 10.5.0.0 0.0.255.255 10.0.0.0 0.255.255.255
 permit ip 10.5.0.0 0.0.255.255 any
exit

ip nat inside source list NAT-ACL-SFM interface Ethernet0/0 overload

!
! ============================================================
! VPN / ISAKMP
! ============================================================
!

crypto isakmp policy 21
 encr aes
 hash sha512
 authentication pre-share
 group 14
exit

crypto isakmp key Cisco123 address 1.0.0.2

!
! ============================================================
! IPSEC
! ============================================================
!

crypto ipsec transform-set TS-VPN esp-aes esp-sha512-hmac
 mode transport
exit

crypto ipsec profile PF-VPN
 set transform-set TS-VPN
exit

!
! ============================================================
! DMVPN / NHRP
! ============================================================
!

interface Tunnel1
 description DMVPN-SPOKE-SFM
 ip address 10.7.0.6 255.255.255.240
 no ip redirects
 ip mtu 1400
 ip nhrp authentication NETSECURE
 ip nhrp map 10.7.0.1 1.0.0.2
 ip nhrp map multicast 1.0.0.2
 ip nhrp network-id 2026
 ip nhrp nhs 10.7.0.1
 ip tcp adjust-mss 1360
 ip ospf network point-to-multipoint
 ip ospf 21 area 0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PF-VPN
 no shutdown
exit

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory
```

</details>

<details>
<summary><strong>SWD-SFM-1 (Switch de Distribución)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWD-SFM-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

no ip routing

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SAN FRANCISCO DE MACORIS)
ADVERTENCIA: ACCESO RESTRINGIDO. AUTENTICACION RADIUS ACTIVA.
*********************************************************************
#

crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2

line console 0
 logging synchronous
 exec-timeout 10 0
exit

line vty 0 4
 transport input ssh
 exec-timeout 10 0
exit

aaa new-model
aaa authentication login default group radius local

radius server SANTIAGO-RADIUS
 address ipv4 10.1.0.34 auth-port 1812 acct-port 1813
 key Netsecure-EMP1
exit

!
! ============================================================
! VLANS
! ============================================================
!

vlan 600
 name DEFAULT-SFM
exit

vlan 610
 name VENTAS-REGIONAL-SFM
exit

vlan 620
 name OPERACIONES-SFM
exit

vlan 630
 name LEGAL-CUMPLIMIENTO-SFM
exit

vlan 699
 name MGMT-SFM
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 600,610,620,630,699 priority 24576

!
! ============================================================
! TRUNK HACIA R-SFM
! ============================================================
!

interface Ethernet0/0
 description TRUNK-A-R-SFM
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 600,610,620,630,699
 switchport trunk native vlan 600
 switchport nonegotiate
 duplex full
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL / TRUNK HACIA SWA-SFM-1
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWA-SFM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 600,610,620,630,699
 switchport trunk native vlan 600
 switchport nonegotiate
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWA-SFM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 600,610,620,630,699
 switchport trunk native vlan 600
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! MANAGEMENT
! ============================================================
!

interface Vlan699
 description MANAGEMENT-SWD-SFM-1
 ip address 10.5.1.2 255.255.255.240
 no shutdown
exit

ip default-gateway 10.5.1.1
ip radius source-interface Vlan699

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory
```

</details>

<details>
<summary><strong>SWA-SFM-1 (Switch de Acceso)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-SFM-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SAN FRANCISCO DE MACORIS)
ADVERTENCIA: ACCESO RESTRINGIDO. AUTENTICACION RADIUS ACTIVA.
*********************************************************************
#

crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2

line console 0
 logging synchronous
 exec-timeout 10 0
exit

line vty 0 4
 transport input ssh
 exec-timeout 10 0
exit

aaa new-model
aaa authentication login default group radius local

radius server SANTIAGO-RADIUS
 address ipv4 10.1.0.34 auth-port 1812 acct-port 1813
 key Netsecure-EMP1
exit

no ip routing

vlan 600
 name DEFAULT-SFM
exit

vlan 610
 name VENTAS-REGIONAL-SFM
exit

vlan 620
 name OPERACIONES-SFM
exit

vlan 630
 name LEGAL-CUMPLIMIENTO-SFM
exit

vlan 699
 name MGMT-SFM
exit

spanning-tree mode pvst

!
! ============================================================
! ETHERCHANNEL / TRUNK
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWD-SFM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 600,610,620,630,699
 switchport trunk native vlan 600
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWD-SFM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 600,610,620,630,699
 switchport trunk native vlan 600
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! PUERTO MANAGEMENT
! ============================================================
!

interface Ethernet0/0
 description MGMT-SFM
 switchport mode access
 switchport access vlan 699
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! VENTAS REGIONAL - VLAN 610
! ============================================================
!

interface range Ethernet1/0-3
 description VENTAS-REGIONAL-SFM
 switchport mode access
 switchport access vlan 610
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! OPERACIONES - VLAN 620
! ============================================================
!

interface range Ethernet2/0-3
 description OPERACIONES-SFM
 switchport mode access
 switchport access vlan 620
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! LEGAL / CUMPLIMIENTO - VLAN 630
! ============================================================
!

interface range Ethernet3/0-3
 description LEGAL-CUMPLIMIENTO-SFM
 switchport mode access
 switchport access vlan 630
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! MANAGEMENT SVI
! ============================================================
!

interface Vlan699
 description MANAGEMENT-SWA-SFM-1
 ip address 10.5.1.3 255.255.255.240
 no shutdown
exit

ip default-gateway 10.5.1.1
ip radius source-interface Vlan699

no ip http server
no ip http secure-server

!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 610,620,630
no ip dhcp snooping information option
ip dhcp snooping

interface Port-channel1
 ip dhcp snooping trust
exit

interface range Ethernet0/1-3
 ip dhcp snooping trust
exit

interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15
exit

interface range Ethernet2/0-3
 ip dhcp snooping limit rate 15
exit

interface range Ethernet3/0-3
 ip dhcp snooping limit rate 15
exit

!
! ============================================================
! DAI / ARP INSPECTION
! ============================================================
!

ip arp inspection vlan 610,620,630

interface Port-channel1
 ip arp inspection trust
exit

interface range Ethernet0/1-3
 ip arp inspection trust
exit

end
write memory
```

</details>
