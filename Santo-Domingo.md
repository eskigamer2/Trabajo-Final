# NetSecure Solutions — Sede Principal Santo Domingo

## Rol en la topología

Santo Domingo es la **sede central**: hub DMVPN (`10.7.0.1`) hacia las 5
sedes regionales y punto de origen de la ruta por defecto para toda la red
(`default-information originate`). Tiene redundancia real de Capa 2
(EtherChannel de Capa 3 entre los dos switches de distribución) y de Capa 3
(HSRP activo/standby en cada VLAN, con `SWMD-SD-1` como primario, prioridad
110). Los enlaces internos de infraestructura (`R-CORE-SD` ↔ cada switch de
distribución, y el EtherChannel entre ambos) usan subredes punto a punto
`/30`.

### Departamentos / VLANs

| VLAN | Nombre | Subred |
|---:|---|---|
| 100 | DEFAULT-SD | — |
| 110 | CALLCENTER-SD | `10.0.0.0/24` |
| 120 | VENTAS-SD | `10.0.1.0/25` |
| 130 | NOC-SOC-SD | `10.0.1.128/25` |
| 140 | ADMIN-GERENCIA-SD | `10.0.2.0/25` |
| 199 | MGMT-SD | `10.0.3.0/28` |

### Enlaces de infraestructura interna

| Enlace | Subred |
|---|---|
| R-CORE-SD ↔ SWMD-SD-1 | `10.6.0.0/30` |
| R-CORE-SD ↔ SWMD-SD-2 | `10.6.0.4/30` |
| EtherChannel L3 SWMD-SD-1 ↔ SWMD-SD-2 | `10.6.0.8/30` |

### Dispositivos

- `R-CORE-SD` — Router núcleo: WAN al ISP, hub DMVPN, ruta por defecto para toda la red, DHCP relay para todas las VLANs
- `SWMD-SD-1` — Switch de distribución 1 (L3), HSRP **activo** (prioridad 110), trunk hacia los 3 switches de acceso
- `SWMD-SD-2` — Switch de distribución 2 (L3), HSRP **standby**, trunk hacia los 3 switches de acceso, EtherChannel L3 con SWMD-SD-1
- `SWA-SD-1` — Switch de acceso: VLAN 110 CALLCENTER-SD
- `SWA-SD-2` — Switch de acceso: VLAN 120 VENTAS-SD + VLAN 140 ADMIN-GERENCIA-SD
- `SWA-SD-3` — Switch de acceso: VLAN 130 NOC-SOC-SD

---

## Configuraciones

<details>
<summary><strong>R-CORE-SD (Router Núcleo)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname R-CORE-SD
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
NETSECURE SOLUTIONS (SANTO DOMINGO)
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
! INTERFACES
! ============================================================
!

interface Ethernet0/0
 description ENLACE-WAN-ISP
 ip address 1.0.0.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 no shutdown
exit

interface Ethernet0/1
 description ENLACE-SWMD-SD-1
 ip address 10.6.0.1 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf network point-to-point
 ip ospf 21 area 0
 duplex full
 no shutdown
exit

interface Ethernet0/2
 description ENLACE-SWMD-SD-2
 ip address 10.6.0.5 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf network point-to-point
 ip ospf 21 area 0
 duplex full
 no shutdown
exit

!
! ============================================================
! DHCP
! ============================================================
!

ip dhcp relay information trust-all

ip dhcp excluded-address 10.0.0.1 10.0.0.3
ip dhcp excluded-address 10.0.1.1 10.0.1.3
ip dhcp excluded-address 10.0.1.129 10.0.1.131
ip dhcp excluded-address 10.0.2.1 10.0.2.3
ip dhcp excluded-address 10.0.3.1 10.0.3.6

ip dhcp pool VLAN110
 network 10.0.0.0 255.255.255.0
 default-router 10.0.0.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN120
 network 10.0.1.0 255.255.255.128
 default-router 10.0.1.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN130
 network 10.0.1.128 255.255.255.128
 default-router 10.0.1.129
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN140
 network 10.0.2.0 255.255.255.128
 default-router 10.0.2.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN199
 network 10.0.3.0 255.255.255.240
 default-router 10.0.3.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

!
! ============================================================
! OSPF
! ============================================================
!

router ospf 21
 router-id 0.0.0.1
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2
 no passive-interface Tunnel1
 default-information originate
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.1

!
! ============================================================
! NAT
! ============================================================
!

ip access-list extended NAT-ACL-SD
 deny ip 10.0.0.0 0.0.255.255 10.0.0.0 0.127.255.255
 permit ip 10.0.0.0 0.0.255.255 any
exit

ip nat inside source list NAT-ACL-SD interface Ethernet0/0 overload

!
! ============================================================
! ISAKMP / VPN
! ============================================================
!

crypto isakmp policy 21
 encr aes
 hash sha512
 authentication pre-share
 group 14
exit

crypto isakmp key Cisco123 address 1.0.0.6
crypto isakmp key Cisco123 address 1.0.0.10
crypto isakmp key Cisco123 address 1.0.0.14
crypto isakmp key Cisco123 address 1.0.0.18
crypto isakmp key Cisco123 address 1.0.0.22

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
 description DMVPN-HUB-SANTO-DOMINGO
 ip address 10.7.0.1 255.255.255.240
 no ip redirects
 ip mtu 1400
 ip nhrp authentication NETSECURE
 ip nhrp map multicast dynamic
 ip nhrp network-id 2026
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
<summary><strong>SWMD-SD-1 (Switch de Distribución 1)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWMD-SD-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

ip routing
ip cef
no ipv6 cef

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTO DOMINGO)
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

vlan 100
 name DEFAULT-SD
exit

vlan 110
 name CALLCENTER-SD
exit

vlan 120
 name VENTAS-SD
exit

vlan 130
 name NOC-SOC-SD
exit

vlan 140
 name ADMIN-GERENCIA-SD
exit

vlan 199
 name MGMT-SD
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 100,110,120,130,140,199 priority 24576

!
! ============================================================
! ENLACE CAPA 3 HACIA R-CORE-SD
! ============================================================
!

interface Ethernet0/0
 description CONEXION-R-CORE-SD
 no switchport
 ip address 10.6.0.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 21 area 0
 duplex full
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL CAPA 3
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-CAPA3-SWMD-SD-2
 no switchport
 no ip address
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description ENLACE-CAPA3-SWMD-SD-2
 no switchport
 ip address 10.6.0.9 255.255.255.252
 ip ospf network point-to-point
 ip ospf 21 area 0
 no shutdown
exit

!
! ============================================================
! TRUNKS HACIA SWITCHES DE ACCESO
! ============================================================
!

interface Ethernet1/0
 description ENLACE-TRUNK-SWA-SD-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit

interface Ethernet1/1
 description ENLACE-TRUNK-SWA-SD-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit

interface Ethernet1/2
 description ENLACE-TRUNK-SWA-SD-3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit

!
! ============================================================
! SVI + HSRP
! ============================================================
!

interface Vlan110
 description GATEWAY-VLAN-CALLCENTER
 ip address 10.0.0.2 255.255.255.0
 ip helper-address 10.6.0.1
 standby 110 ip 10.0.0.1
 standby 110 priority 110
 standby 110 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan120
 description GATEWAY-VLAN-VENTAS
 ip address 10.0.1.2 255.255.255.128
 ip helper-address 10.6.0.1
 standby 120 ip 10.0.1.1
 standby 120 priority 110
 standby 120 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan130
 description GATEWAY-VLAN-NOC-SOC
 ip address 10.0.1.130 255.255.255.128
 ip helper-address 10.6.0.1
 standby 130 ip 10.0.1.129
 standby 130 priority 110
 standby 130 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan140
 description GATEWAY-VLAN-ADMIN
 ip address 10.0.2.2 255.255.255.128
 ip helper-address 10.6.0.1
 standby 140 ip 10.0.2.1
 standby 140 priority 110
 standby 140 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan199
 description MANAGEMENT
 ip address 10.0.3.2 255.255.255.240
 ip helper-address 10.6.0.1
 standby 199 ip 10.0.3.1
 standby 199 priority 110
 standby 199 preempt
 ip ospf 21 area 0
 no shutdown
exit

!
! ============================================================
! OSPF
! ============================================================
!

router ospf 21
 router-id 0.0.0.11
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Ethernet0/0
 no passive-interface Port-channel1
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
<summary><strong>SWMD-SD-2 (Switch de Distribución 2)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWMD-SD-2
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

ip routing
ip cef
no ipv6 cef

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTO DOMINGO)
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

vlan 100
 name DEFAULT-SD
exit

vlan 110
 name CALLCENTER-SD
exit

vlan 120
 name VENTAS-SD
exit

vlan 130
 name NOC-SOC-SD
exit

vlan 140
 name ADMIN-GERENCIA-SD
exit

vlan 199
 name MGMT-SD
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 100,110,120,130,140,199 priority 28672

!
! ============================================================
! ENLACE CAPA 3 HACIA R-CORE-SD
! ============================================================
!

interface Ethernet0/0
 description CONEXION-R-CORE-SD
 no switchport
 ip address 10.6.0.6 255.255.255.252
 ip ospf network point-to-point
 ip ospf 21 area 0
 duplex full
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL CAPA 3 HACIA SWMD-SD-1
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-CAPA3-SWMD-SD-1
 no switchport
 no ip address
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description ENLACE-CAPA3-SWMD-SD-1
 no switchport
 ip address 10.6.0.10 255.255.255.252
 ip ospf network point-to-point
 ip ospf 21 area 0
 no shutdown
exit

!
! ============================================================
! TRUNKS HACIA SWITCHES DE ACCESO
! ============================================================
!

interface Ethernet1/0
 description ENLACE-TRUNK-SWA-SD-3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit

interface Ethernet1/1
 description ENLACE-TRUNK-SWA-SD-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit

interface Ethernet1/2
 description ENLACE-TRUNK-SWA-SD-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit

!
! ============================================================
! SVI + HSRP
! ============================================================
!

interface Vlan110
 description GATEWAY-VLAN-CALLCENTER
 ip address 10.0.0.3 255.255.255.0
 ip helper-address 10.6.0.5
 standby 110 ip 10.0.0.1
 standby 110 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan120
 description GATEWAY-VLAN-VENTAS
 ip address 10.0.1.3 255.255.255.128
 ip helper-address 10.6.0.5
 standby 120 ip 10.0.1.1
 standby 120 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan130
 description GATEWAY-VLAN-NOC-SOC
 ip address 10.0.1.131 255.255.255.128
 ip helper-address 10.6.0.5
 standby 130 ip 10.0.1.129
 standby 130 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan140
 description GATEWAY-VLAN-ADMIN
 ip address 10.0.2.3 255.255.255.128
 ip helper-address 10.6.0.5
 standby 140 ip 10.0.2.1
 standby 140 preempt
 ip ospf 21 area 0
 no shutdown
exit

interface Vlan199
 description MANAGEMENT
 ip address 10.0.3.3 255.255.255.240
 ip helper-address 10.6.0.5
 standby 199 ip 10.0.3.1
 standby 199 preempt
 ip ospf 21 area 0
 no shutdown
exit

!
! ============================================================
! OSPF
! ============================================================
!

router ospf 21
 router-id 0.0.0.12
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Ethernet0/0
 no passive-interface Port-channel1
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
<summary><strong>SWA-SD-1 (Switch de Acceso — CALLCENTER-SD)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-SD-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTO DOMINGO)
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

ip radius source-interface Vlan199


vlan 100
 name DEFAULT-SD
exit

vlan 110
 name CALLCENTER-SD
exit

vlan 120
 name VENTAS-SD
exit

vlan 130
 name NOC-SOC-SD
exit

vlan 140
 name ADMIN-GERENCIA-SD
exit

vlan 199
 name MGMT-SD
exit


spanning-tree mode pvst


interface Ethernet0/0
 description ENLACE TRUNK A SWMD-SD-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 no shutdown
exit

interface Ethernet0/1
 description ENLACE TRUNK A SWMD-SD-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 no shutdown
exit


interface range Ethernet1/0-3
 description CALLCENTER-SD
 switchport mode access
 switchport access vlan 110
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit


interface Ethernet3/3
 description MANAGEMENT
 switchport mode access
 switchport access vlan 199
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit


interface range Ethernet0/2-3, Ethernet2/0-3, Ethernet3/0-2
 description PUERTOS-SIN-ASIGNAR
 switchport mode access
 switchport access vlan 100
 shutdown
exit


interface Vlan199
 description MANAGEMENT-SWA-SD-1
 ip address 10.0.3.4 255.255.255.240
 no shutdown
exit

ip default-gateway 10.0.3.1


no ip http server
no ip http secure-server


!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 110
no ip dhcp snooping information option
ip dhcp snooping

interface Ethernet0/0
 ip dhcp snooping trust
exit

interface Ethernet0/1
 ip dhcp snooping trust
exit

interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15
exit


!
! ============================================================
! DAI / ARP INSPECTION
! ============================================================
!

ip arp inspection vlan 110

interface Ethernet0/0
 ip arp inspection trust
exit

interface Ethernet0/1
 ip arp inspection trust
exit


end
write memory
```

</details>

<details>
<summary><strong>SWA-SD-2 (Switch de Acceso — VENTAS-SD / ADMIN-GERENCIA-SD)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-SD-2
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTO DOMINGO)
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

ip radius source-interface Vlan199

vlan 100
 name DEFAULT-SD
exit

vlan 110
 name CALLCENTER-SD
exit

vlan 120
 name VENTAS-SD
exit

vlan 130
 name NOC-SOC-SD
exit

vlan 140
 name ADMIN-GERENCIA-SD
exit

vlan 199
 name MGMT-SD
exit

spanning-tree mode rapid-pvst

interface Ethernet0/0
 description ENLACE TRUNK A SWMD-SD-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 no shutdown
exit

interface Ethernet0/1
 description ENLACE TRUNK A SWMD-SD-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 no shutdown
exit

interface range Ethernet1/0-3
 description VENTAS-SD
 switchport mode access
 switchport access vlan 120
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface range Ethernet2/0-3
 description ADMIN-GERENCIA-SD
 switchport mode access
 switchport access vlan 140
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface Ethernet3/3
 description MANAGEMENT
 switchport mode access
 switchport access vlan 199
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

interface range Ethernet0/2-3, Ethernet3/0-2
 description PUERTOS-SIN-ASIGNAR
 switchport mode access
 switchport access vlan 100
 shutdown
exit

interface Vlan199
 description MANAGEMENT-SWA-SD-2
 ip address 10.0.3.5 255.255.255.240
 no shutdown
exit

ip default-gateway 10.0.3.1

no ip http server
no ip http secure-server


!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 120,140
no ip dhcp snooping information option
ip dhcp snooping

interface Ethernet0/0
 ip dhcp snooping trust
exit

interface Ethernet0/1
 ip dhcp snooping trust
exit

interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15
exit

interface range Ethernet2/0-3
 ip dhcp snooping limit rate 15
exit


!
! ============================================================
! DAI / ARP INSPECTION
! ============================================================
!

ip arp inspection vlan 120,140

interface Ethernet0/0
 ip arp inspection trust
exit

interface Ethernet0/1
 ip arp inspection trust
exit


end
write memory
```

</details>

<details>
<summary><strong>SWA-SD-3 (Switch de Acceso — NOC-SOC-SD)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-SD-3
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1


banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTO DOMINGO)
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

ip radius source-interface Vlan199


vlan 100
 name DEFAULT-SD
exit

vlan 110
 name CALLCENTER-SD
exit

vlan 120
 name VENTAS-SD
exit

vlan 130
 name NOC-SOC-SD
exit

vlan 140
 name ADMIN-GERENCIA-SD
exit

vlan 199
 name MGMT-SD
exit


spanning-tree mode rapid-pvst


interface Ethernet0/0
 description ENLACE TRUNK A SWMD-SD-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 no shutdown
exit

interface Ethernet0/1
 description ENLACE TRUNK A SWMD-SD-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,110,120,130,140,199
 switchport trunk native vlan 100
 switchport nonegotiate
 duplex full
 no shutdown
exit


interface range Ethernet1/0-3
 description NOC-SOC-SD
 switchport mode access
 switchport access vlan 130
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit


interface Ethernet3/3
 description MANAGEMENT
 switchport mode access
 switchport access vlan 199
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit


interface range Ethernet0/2-3, Ethernet2/0-3, Ethernet3/0-2
 description PUERTOS-SIN-ASIGNAR
 switchport mode access
 switchport access vlan 100
 shutdown
exit


interface Vlan199
 description MANAGEMENT-SWA-SD-3
 ip address 10.0.3.6 255.255.255.240
 no shutdown
exit

ip default-gateway 10.0.3.1


no ip http server
no ip http secure-server


!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 130
no ip dhcp snooping information option
ip dhcp snooping

interface Ethernet0/0
 ip dhcp snooping trust
exit

interface Ethernet0/1
 ip dhcp snooping trust
exit

interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15
exit


!
! ============================================================
! DAI / ARP INSPECTION
! ============================================================
!

ip arp inspection vlan 130

interface Ethernet0/0
 ip arp inspection trust
exit

interface Ethernet0/1
 ip arp inspection trust
exit


end
write memory
```

</details>
