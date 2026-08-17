# NetSecure Solutions — ISP / Nube

> Empresa: NetSecure Solutions

## Rol en la topología

El router **ISP-NUBE** Concentra los 6 enlaces
públicos hacia cada sede (`1.0.0.0/24`), realiza NAT overload hacia "internet"
 y expone el **servidor Web/Correo** mediante una subred pública
dedicada (`1.0.0.24/29`).

| Enlace | Red pública |
|---|---|
| Santo Domingo (Hub) | `1.0.0.0/30` |
| Santiago | `1.0.0.4/30` |
| La Romana | `1.0.0.8/30` |
| Puerto Plata | `1.0.0.12/30` |
| Barahona | `1.0.0.16/30` |
| San Francisco de Macorís | `1.0.0.20/30` |
| **NAT estático — Web/Correo** | `1.0.0.24/29` (`1.0.0.25` usable) |

> El servidor Web/Correo público vive físicamente en el datacenter de
> **Santiago** (ver [Santiago.md](./Santiago.md)); aquí en el ISP solo se
> documenta su salida/NAT hacia Internet.

---

## ISP-NUBE (Router)

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname ISP-NUBE
no ip domain lookup

service timestamps debug datetime msec
service timestamps log datetime msec

ip cef
no ipv6 cef

!
! ============================================================
! ENLACE SANTO DOMINGO
! ============================================================
!

interface Ethernet0/0
 description ENLACE-A-SANTO-DOMINGO-HUB
 ip address 1.0.0.1 255.255.255.252
 ip nat inside
 no shutdown
exit

!
! ============================================================
! ENLACE PUERTO PLATA
! ============================================================
!

interface Ethernet0/1
 description ENLACE-A-PUERTO-PLATA
 ip address 1.0.0.13 255.255.255.252
 ip nat inside
 no shutdown
exit

!
! ============================================================
! ENLACE SANTIAGO
! ============================================================
!

interface Ethernet0/2
 description ENLACE-A-SANTIAGO
 ip address 1.0.0.5 255.255.255.252
 ip nat inside
 duplex full
 no shutdown
exit

!
! ============================================================
! ENLACE LA ROMANA
! ============================================================
!

interface Ethernet0/3
 description ENLACE-A-LA-ROMANA
 ip address 1.0.0.9 255.255.255.252
 ip nat inside
 no shutdown
exit

!
! ============================================================
! ENLACE SAN FRANCISCO DE MACORIS
! ============================================================
!

interface Ethernet1/0
 description ENLACE-A-SAN-FRANCISCO-MACORIS
 ip address 1.0.0.21 255.255.255.252
 ip nat inside
 no shutdown
exit

!
! ============================================================
! ENLACE BARAHONA
! ============================================================
!

interface Ethernet1/1
 description ENLACE-A-BARAHONA
 ip address 1.0.0.17 255.255.255.252
 ip nat inside
 no shutdown
exit

!
! ============================================================
! ENLACE WEB SERVER
! ============================================================
!

interface Ethernet2/0
 description ENLACE-A-WEB-SERVER
 ip address 1.0.0.25 255.255.255.248
 ip nat inside
 no shutdown
exit

!
! ============================================================
! ENLACE ALTICE CLOUD / INTERNET
! ============================================================
!

interface Ethernet3/0
 description ENLACE-A-ALTICE-CLOUD
 ip address dhcp
 ip nat outside
 no shutdown
exit

!
! ============================================================
! NAT
! ============================================================
!

access-list 100 permit ip any any

ip nat inside source list 100 interface Ethernet3/0 overload

!
! ============================================================
! RUTAS
! ============================================================
!

ip route 0.0.0.0 0.0.0.0 10.0.137.1

ip route 10.1.0.0 255.255.254.0 1.0.0.6
ip route 10.1.0.32 255.255.255.224 1.0.0.6

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory```
