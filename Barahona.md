# NetSecure Solutions — Sede Barahona

## Rol en la topología

Sede regional conectada al [ISP](./ISP.md) mediante un enlace WAN dedicado y
un túnel DMVPN/IPSec (spoke) hacia el hub en Santo Domingo (área OSPF 5).
DHCP para las VLANs de esta sede se resuelve **localmente en el router**
(no depende de un servidor externo). Autenticación AAA vía RADIUS contra el
servidor del datacenter en [Santiago](./Santiago.md) (`10.1.0.34`).

### Departamentos / VLANs

| VLAN | Nombre | Subred |
|---:|---|---|
| 500 | DEFAULT-BAR | — |
| 510 | VENTAS-NEGOCIOS-BAR | `10.4.0.0/25` |
| 520 | SOPORTE-TECNICO-BAR | `10.4.0.128/26` |
| 530 | ADMINISTRACION-BAR | `10.4.0.192/28` |
| 599 | MGMT-BAR | `10.4.1.0/28` |

### Dispositivos

- `R-BAR` — Router de sede (WAN al ISP, subinterfaces 802.1Q, DHCP, NAT, DMVPN spoke)
- `SWD-BAR-1` — Switch de distribución (trunk + EtherChannel L2 hacia el switch de acceso)
- `SWA-BAR-1` — Switch de acceso (puertos de usuario, port-security, DHCP snooping, DAI)

---

## Configuraciones

<details>
<summary><strong>R-BAR (Router)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname R-BAR
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
NETSECURE SOLUTIONS (BARAHONA)
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
 ip address 1.0.0.18 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 no shutdown
exit

!
! ============================================================
! TRUNK HACIA SWD-BAR-1
! ============================================================
!

interface Ethernet0/1
 description TRUNK-A-SWD-BAR-1
 no ip address
 duplex full
 no shutdown
exit

interface Ethernet0/1.510
 description GATEWAY-VENTAS
 encapsulation dot1Q 510
 ip address 10.4.0.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

interface Ethernet0/1.520
 description GATEWAY-SOPORTE
 encapsulation dot1Q 520
 ip address 10.4.0.129 255.255.255.192
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

interface Ethernet0/1.530
 description GATEWAY-ADMIN
 encapsulation dot1Q 530
 ip address 10.4.0.193 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

interface Ethernet0/1.599
 description GATEWAY-MGMT
 encapsulation dot1Q 599
 ip address 10.4.1.1 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

!
! ============================================================
! DHCP
! ============================================================
!

ip dhcp excluded-address 10.4.0.1 10.4.0.3
ip dhcp excluded-address 10.4.0.129 10.4.0.131
ip dhcp excluded-address 10.4.0.193 10.4.0.195
ip dhcp excluded-address 10.4.1.1 10.4.1.3

ip dhcp pool VLAN510-VENTAS
 network 10.4.0.0 255.255.255.128
 default-router 10.4.0.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN520-SOPORTE
 network 10.4.0.128 255.255.255.192
 default-router 10.4.0.129
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN530-ADMIN
 network 10.4.0.192 255.255.255.240
 default-router 10.4.0.193
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN599-MGMT
 network 10.4.1.0 255.255.255.240
 default-router 10.4.1.1
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
 router-id 5.5.5.5
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Tunnel1
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.17

!
! ============================================================
! NAT
! ============================================================
!

ip access-list extended NAT-ACL-BAR
 deny ip 10.4.0.0 0.0.255.255 10.0.0.0 0.255.255.255
 permit ip 10.4.0.0 0.0.255.255 any
exit

ip nat inside source list NAT-ACL-BAR interface Ethernet0/0 overload

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
 description DMVPN-SPOKE-BARAHONA
 ip address 10.7.0.5 255.255.255.240
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
<summary><strong>SWD-BAR-1 (Switch de Distribución)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWD-BAR-1
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
NETSECURE SOLUTIONS (BARAHONA)
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

vlan 500
 name DEFAULT-BAR
exit

vlan 510
 name VENTAS-NEGOCIOS-BAR
exit

vlan 520
 name SOPORTE-TECNICO-BAR
exit

vlan 530
 name ADMINISTRACION-BAR
exit

vlan 599
 name MGMT-BAR
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 500,510,520,530,599 priority 24576

!
! ============================================================
! TRUNK HACIA R-BAR
! ============================================================
!

interface Ethernet0/0
 description TRUNK-A-R-BAR
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 duplex full
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL / TRUNK HACIA SWA-BAR-1
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWA-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWA-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! MANAGEMENT
! ============================================================
!

interface Vlan599
 description MANAGEMENT-SWD-BAR-1
 ip address 10.4.1.2 255.255.255.240
 no shutdown
exit

ip default-gateway 10.4.1.1
ip radius source-interface Vlan599

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
<summary><strong>SWA-BAR-1 (Switch de Acceso)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-BAR-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (BARAHONA)
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

vlan 500
 name DEFAULT-BAR
exit

vlan 510
 name VENTAS-NEGOCIOS-BAR
exit

vlan 520
 name SOPORTE-TECNICO-BAR
exit

vlan 530
 name ADMINISTRACION-BAR
exit

vlan 599
 name MGMT-BAR
exit

spanning-tree mode pvst

!
! ============================================================
! ETHERCHANNEL / TRUNK
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWD-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWD-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! PUERTO MANAGEMENT
! ============================================================
!

interface Ethernet0/0
 description MGMT-BAR
 switchport mode access
 switchport access vlan 599
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
! VENTAS / NEGOCIOS - VLAN 510
! ============================================================
!

interface range Ethernet1/0-3
 description VENTAS-NEGOCIOS-BAR
 switchport mode access
 switchport access vlan 510
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
! SOPORTE TECNICO - VLAN 520
! ============================================================
!

interface range Ethernet2/0-3
 description SOPORTE-TECNICO-BAR
 switchport mode access
 switchport access vlan 520
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
! ADMINISTRACION - VLAN 530
! ============================================================
!

interface range Ethernet3/0-3
 description ADMINISTRACION-BAR
 switchport mode access
 switchport access vlan 530
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

interface Vlan599
 description MANAGEMENT-SWA-BAR-1
 ip address 10.4.1.3 255.255.255.240
 no shutdown
exit

ip default-gateway 10.4.1.1
ip radius source-interface Vlan599

no ip http server
no ip http secure-server

!
! ============================================================
! DHCP SNOOPING
! PRIMERO SE CONFIGURA DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 510,520,530
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
! SE CONFIGURA DESPUES DE DHCP SNOOPING
! ============================================================
!

ip arp inspection vlan 510,520,530

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
