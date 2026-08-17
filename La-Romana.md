# NetSecure Solutions — Sede La Romana

## Rol en la topología

Sede regional conectada al [ISP](./ISP.md) mediante un enlace WAN dedicado y
un túnel DMVPN/IPSec (spoke) hacia el hub en Santo Domingo (área OSPF 4).
DHCP para las VLANs de esta sede se resuelve **localmente en el router**
(no depende de un servidor externo). Autenticación AAA vía RADIUS contra el
servidor del datacenter en [Santiago](./Santiago.md) (`10.1.0.34`).

### Departamentos / VLANs

| VLAN | Nombre | Subred |
|---:|---|---|
| 300 | DEFAULT-ROM | — |
| 310 | VENTAS-ATENCION-ROM | `10.2.0.0/25` |
| 320 | LOGISTICA-ROM | `10.2.0.128/26` |
| 330 | RRHH-ROM | `10.2.0.192/28` |
| 399 | MGMT-ROM | `10.2.1.0/28` |

### Dispositivos

- `R-ROM` — Router de sede (WAN al ISP, subinterfaces 802.1Q, DHCP, NAT, DMVPN spoke)
- `SWD-ROM-1` — Switch de distribución (trunk + EtherChannel L2 hacia el switch de acceso)
- `SWA-ROM-1` — Switch de acceso (puertos de usuario, port-security, DHCP snooping, DAI)

# Configuraciones

<details>
<summary><strong>R-ROM (Router)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname R-ROM
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
NETSECURE SOLUTIONS (LA ROMANA)
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
 ip address 1.0.0.10 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex full
 no shutdown
exit

!
! ============================================================
! TRUNK HACIA SWD-ROM-1
! ============================================================
!

interface Ethernet0/1
 description TRUNK-SWD-ROM-1
 no ip address
 duplex full
 no shutdown
exit

interface Ethernet0/1.310
 description GATEWAY-VENTAS-ATENCION-ROM
 encapsulation dot1Q 310
 ip address 10.2.0.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 4
exit

interface Ethernet0/1.320
 description GATEWAY-LOGISTICA-ROM
 encapsulation dot1Q 320
 ip address 10.2.0.129 255.255.255.192
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 4
exit

interface Ethernet0/1.330
 description GATEWAY-RRHH-ROM
 encapsulation dot1Q 330
 ip address 10.2.0.193 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 4
exit

interface Ethernet0/1.399
 description GATEWAY-MGMT-ROM
 encapsulation dot1Q 399
 ip address 10.2.1.1 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 4
exit

!
! ============================================================
! DHCP
! ============================================================
!

ip dhcp excluded-address 10.2.0.1 10.2.0.3
ip dhcp excluded-address 10.2.0.129 10.2.0.131
ip dhcp excluded-address 10.2.0.193 10.2.0.195
ip dhcp excluded-address 10.2.1.1 10.2.1.3

ip dhcp pool VLAN310-VENTAS-ATENCION
 network 10.2.0.0 255.255.255.128
 default-router 10.2.0.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN320-LOGISTICA
 network 10.2.0.128 255.255.255.192
 default-router 10.2.0.129
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN330-RRHH
 network 10.2.0.192 255.255.255.240
 default-router 10.2.0.193
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN399-MGMT
 network 10.2.1.0 255.255.255.240
 default-router 10.2.1.1
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
 router-id 3.3.3.3
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Tunnel1
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.9

!
! ============================================================
! NAT
! ============================================================
!

ip access-list extended NAT-ACL-ROM
 deny ip 10.2.0.0 0.0.1.255 10.0.0.0 0.255.255.255
 permit ip 10.2.0.0 0.0.1.255 any
exit

ip nat inside source list NAT-ACL-ROM interface Ethernet0/0 overload

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
 description DMVPN-SPOKE-LA-ROMANA
 ip address 10.7.0.3 255.255.255.240
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

</details>

<details>
<summary><strong>SWD-ROM-1 (Switch de Distribución)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWD-ROM-1
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
NETSECURE SOLUTIONS (LA ROMANA)
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

vlan 300
 name DEFAULT-ROM
exit

vlan 310
 name VENTAS-ATENCION-ROM
exit

vlan 320
 name LOGISTICA-ROM
exit

vlan 330
 name RRHH-ROM
exit

vlan 399
 name MGMT-ROM
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 300,310,320,330,399 priority 24576

!
! ============================================================
! TRUNK HACIA R-ROM
! ============================================================
!

interface Ethernet0/0
 description TRUNK-R-ROM
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 300,310,320,330,399
 switchport trunk native vlan 300
 switchport nonegotiate
 duplex full
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL / TRUNK HACIA SWA-ROM-1
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWA-ROM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 300,310,320,330,399
 switchport trunk native vlan 300
 switchport nonegotiate
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWA-ROM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 300,310,320,330,399
 switchport trunk native vlan 300
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! MANAGEMENT
! ============================================================
!

interface Vlan399
 description MANAGEMENT-SWD-ROM-1
 ip address 10.2.1.2 255.255.255.240
 no shutdown
exit

ip default-gateway 10.2.1.1
ip radius source-interface Vlan399

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory```

</details>

<details>
<summary><strong>SWA-ROM-1 (Switch de Acceso)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-ROM-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (LA ROMANA)
ADVERTENCIA: ACCESO RESTRINGIDO. SOLO PERSONAL AUTORIZADO.
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

vlan 300
 name DEFAULT-ROM
exit

vlan 310
 name VENTAS-ATENCION-ROM
exit

vlan 320
 name LOGISTICA-ROM
exit

vlan 330
 name RRHH-ROM
exit

vlan 399
 name MGMT-ROM
exit

spanning-tree mode pvst

!
! ============================================================
! ETHERCHANNEL / TRUNK
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWD-ROM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 300,310,320,330,399
 switchport trunk native vlan 300
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWD-ROM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 300,310,320,330,399
 switchport trunk native vlan 300
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! PUERTO MANAGEMENT
! ============================================================
!

interface Ethernet0/0
 description MGMT-ROM
 switchport mode access
 switchport access vlan 399
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
! VENTAS / ATENCION - VLAN 310
! ============================================================
!

interface range Ethernet1/0-3
 description VENTAS-ATENCION-ROM
 switchport mode access
 switchport access vlan 310
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
! LOGISTICA - VLAN 320
! ============================================================
!

interface range Ethernet2/0-3
 description LOGISTICA-ROM
 switchport mode access
 switchport access vlan 320
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
! RRHH - VLAN 330
! ============================================================
!

interface range Ethernet3/0-3
 description RRHH-ROM
 switchport mode access
 switchport access vlan 330
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

interface Vlan399
 description MANAGEMENT-SWA-ROM-1
 ip address 10.2.1.3 255.255.255.240
 no shutdown
exit

ip default-gateway 10.2.1.1
ip radius source-interface Vlan399

no ip http server
no ip http secure-server

!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 310,320,330
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

ip arp inspection vlan 310,320,330

interface Port-channel1
 ip arp inspection trust
exit

interface range Ethernet0/1-3
 ip arp inspection trust
exit

end
write memory```

</details>
