# NetSecure Solutions — Sede Puerto Plata

> Proyecto: Conmutación y Enrutamiento — ITLA — PNETLAB
> Empresa: NetSecure Solutions

## Rol en la topología

Sede regional conectada al [ISP](./ISP.md) mediante un enlace WAN dedicado y
un túnel DMVPN/IPSec (spoke) hacia el hub en Santo Domingo (área OSPF 3).
DHCP para las VLANs de esta sede se resuelve **localmente en el router**
(no depende de un servidor externo). Autenticación AAA vía RADIUS contra el
servidor del datacenter en [Santiago](./Santiago.md) (`10.1.0.34`).

### Departamentos / VLANs

| VLAN | Nombre | Subred |
|---:|---|---|
| 400 | DEFAULT-PPL | — |
| 410 | VENTAS-COMERCIAL-PPL | `10.3.0.0/25` |
| 420 | MARKETING-PPL | `10.3.0.128/26` |
| 430 | CONTABILIDAD-PPL | `10.3.0.192/28` |
| 499 | MGMT-PPL | `10.3.1.0/28` |

### Dispositivos

- `R-PPL` — Router de sede (WAN al ISP, subinterfaces 802.1Q, DHCP, NAT, DMVPN spoke)
- `SWD-PPL-1` — Switch de distribución (trunk + EtherChannel L2 hacia el switch de acceso)
- `SWA-PPL-1` — Switch de acceso (puertos de usuario, port-security, DHCP snooping, DAI)

---

## R-PPL (Router)

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname R-PPL
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
NETSECURE SOLUTIONS (PUERTO PLATA)
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
 ip address 1.0.0.14 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 no shutdown
exit

!
! ============================================================
! TRUNK HACIA SWD-PPL-1
! ============================================================
!

interface Ethernet0/1
 description TRUNK-SWD-PPL-1
 no ip address
 duplex full
 no shutdown
exit

interface Ethernet0/1.410
 description GATEWAY-VENTAS-COMERCIAL-PPL
 encapsulation dot1Q 410
 ip address 10.3.0.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 3
exit

interface Ethernet0/1.420
 description GATEWAY-MARKETING-PPL
 encapsulation dot1Q 420
 ip address 10.3.0.129 255.255.255.192
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 3
exit

interface Ethernet0/1.430
 description GATEWAY-CONTABILIDAD-PPL
 encapsulation dot1Q 430
 ip address 10.3.0.193 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 3
exit

interface Ethernet0/1.499
 description GATEWAY-MGMT-PPL
 encapsulation dot1Q 499
 ip address 10.3.1.1 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 3
exit

!
! ============================================================
! DHCP
! ============================================================
!

ip dhcp excluded-address 10.3.0.1 10.3.0.3
ip dhcp excluded-address 10.3.0.129 10.3.0.131
ip dhcp excluded-address 10.3.0.193 10.3.0.195
ip dhcp excluded-address 10.3.1.1 10.3.1.3

ip dhcp pool VLAN410-VENTAS
 network 10.3.0.0 255.255.255.128
 default-router 10.3.0.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN420-MARKETING
 network 10.3.0.128 255.255.255.192
 default-router 10.3.0.129
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN430-CONTABILIDAD
 network 10.3.0.192 255.255.255.240
 default-router 10.3.0.193
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN499-MGMT
 network 10.3.1.0 255.255.255.240
 default-router 10.3.1.1
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
 router-id 4.4.4.4
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Tunnel1
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.13

!
! ============================================================
! NAT
! ============================================================
!

ip access-list extended NAT-ACL-PPL
 deny ip 10.3.0.0 0.0.1.255 10.0.0.0 0.255.255.255
 permit ip 10.3.0.0 0.0.1.255 any
exit

ip nat inside source list NAT-ACL-PPL interface Ethernet0/0 overload

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
 description DMVPN-SPOKE-PUERTO-PLATA
 ip address 10.7.0.4 255.255.255.240
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
write memory```

## SWD-PPL-1 (Switch de Distribución)

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWD-PPL-1
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
NETSECURE SOLUTIONS (PUERTO PLATA)
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

vlan 400
 name DEFAULT-PPL
exit

vlan 410
 name VENTAS-COMERCIAL-PPL
exit

vlan 420
 name MARKETING-PPL
exit

vlan 430
 name CONTABILIDAD-PPL
exit

vlan 499
 name MGMT-PPL
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 400,410,420,430,499 priority 24576

!
! ============================================================
! TRUNK HACIA R-PPL
! ============================================================
!

interface Ethernet0/0
 description TRUNK-R-PPL
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 400,410,420,430,499
 switchport trunk native vlan 400
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL / TRUNK HACIA SWA-PPL-1
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWA-PPL-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 400,410,420,430,499
 switchport trunk native vlan 400
 switchport nonegotiate
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWA-PPL-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 400,410,420,430,499
 switchport trunk native vlan 400
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! MANAGEMENT
! ============================================================
!

interface Vlan499
 description MANAGEMENT-SWD-PPL-1
 ip address 10.3.1.2 255.255.255.240
 no shutdown
exit

ip default-gateway 10.3.1.1
ip radius source-interface Vlan499

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory```

## SWA-PPL-1 (Switch de Acceso)

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-PPL-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (PUERTO PLATA)
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

vlan 400
 name DEFAULT-PPL
exit

vlan 410
 name VENTAS-COMERCIAL-PPL
exit

vlan 420
 name MARKETING-PPL
exit

vlan 430
 name CONTABILIDAD-PPL
exit

vlan 499
 name MGMT-PPL
exit

spanning-tree mode pvst

!
! ============================================================
! ETHERCHANNEL / TRUNK
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWD-PPL-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 400,410,420,430,499
 switchport trunk native vlan 400
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWD-PPL-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 400,410,420,430,499
 switchport trunk native vlan 400
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! PUERTO MANAGEMENT
! ============================================================
!

interface Ethernet0/0
 description MGMT-PPL
 switchport mode access
 switchport access vlan 499
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
! VENTAS / COMERCIAL - VLAN 410
! ============================================================
!

interface range Ethernet1/0-3
 description VENTAS-COMERCIAL-PPL
 switchport mode access
 switchport access vlan 410
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
! MARKETING - VLAN 420
! ============================================================
!

interface range Ethernet2/0-3
 description MARKETING-PPL
 switchport mode access
 switchport access vlan 420
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
! CONTABILIDAD - VLAN 430
! ============================================================
!

interface range Ethernet3/0-3
 description CONTABILIDAD-PPL
 switchport mode access
 switchport access vlan 430
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

interface Vlan499
 description MANAGEMENT-SWA-PPL-1
 ip address 10.3.1.3 255.255.255.240
 no shutdown
exit

ip default-gateway 10.3.1.1
ip radius source-interface Vlan499

no ip http server
no ip http secure-server

!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 410,420,430
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

ip arp inspection vlan 410,420,430

interface Port-channel1
 ip arp inspection trust
exit

interface range Ethernet0/1-3
 ip arp inspection trust
exit

end
write memory
```
