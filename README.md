# 🏢 Proyecto Final - NetSecure Solutions (CECOMPE)

[Topología de Red]<img width="1903" height="1266" alt="Topologia" src="https://github.com/user-attachments/assets/578db4ec-9cbf-42e9-a70c-6a5e539112c4" />


## 📖 Introducción
NetSecure Solutions se dedicará a la venta o prestación de servicios en general, haciendo énfasis en el uso de Internet, Centros de atención al cliente (call centers) y las ventas directas. Esta organización de capital dominicano tendrá presencia a nivel nacional con sucursales en Santiago, La Romana, Puerto Plata, Barahona y su sede principal en Santo Domingo.

**CECOMPE (Centro de Cómputos Pelegrino)** es una compañía que se dedica a la elaboración de propuestas para soluciones de TI, la cual fue apoderada para que realizara el levantamiento correspondiente y presentara una propuesta tecnológica de comunicaciones unificadas acorde a la finalidad de la empresa y la disponibilidad de recursos.

Su grupo de la asignatura de Conmutación y Enrutamiento del ITLA ha sido elegido para implementar la propuesta y el diseño de CECOMPE en el emulador PNETLAB; emitir un informe de los resultados obtenidos.

El proyecto presenta un escenario completo para la implementación, sin embargo la misma puede ser modificado con justificación documentada. La empresa está dispuesta a implementar lo que sea necesario para el buen funcionamiento de la misma.

> **Nota de Despliegue de Servidores:** El **Servidor Web** está conectado y alojado directamente en la red del **ISP**, mientras que los demás servicios (DNS, DHCP, RADIUS, Correo) se encuentran alojados en el **Datacenter de la sede Santiago**.

---

## ⚙️ Aspectos Generales
- **Direccionamiento Privado** para toda la topología **`10.0.0.0/9`**, tomar en cuenta el crecimiento de la red 40% en 5 años.
- **Direccionamiento Publico `1.0.0.0/24`**
- Enrutamiento OSPF **Multiárea**
- Acceso remoto seguro para **todos** los dispositivos
- Implementa Seguridad en **todos** los dispositivos (Control de acceso, Seguridad de Puertos, Ataques de VLAN, Ataques DHCP, Ataques de ARP, Ataques de STP)
- Implementar DHCP por Sede **(En la Sede de Santiago usar un servidor del datacenter para que sea el server DHCP de esa Sede)**
- Implementar VPN dinámica con IPSec (DMVPN) para la comunicación entre las sucursales y la sede Principal
- Implementar ACL según el criterio del equipo

---

## 📍 Información Específica de las Sedes

### Sede Principal Central Santo Domingo
Controla el Gran Santo Domingo y la región Sur Central. Concentra la mayor demanda corporativa y gubernamental.
Crear Mínimo 4 Departamentos:
- **CALLCENTER-SD** (110 host + 40% a 5 años)
- **VENTAS-SD** (80 host + 40% a 5 años)
- **NOC-SOC-SD** (64 host + 40% a 5 años)
- **ADMIN-GERENCIA-SD** (51 host + 40% a 5 años)

*Implementar redundancia de capa 2 (EtherChannel y SpanningTree) y capa 3 (HSRP) donde aplique. En Santo Domingo la doble conexion con los SMW y el R son subredes de 2 host y ponerle una subred al EtherChannel.*

### Sede Santiago
Domina el Cibao Central. Es la segunda capital económica del país.
Crear Mínimo 3 Departamentos:
- **VENTAS-ST** (15 host + 40% a 5años)
- **DATACENTER-ST** (10 host + 40% a 5años)
- **ADMIN-ST** (5 host + 40% a 5años)

*Implementar en un Servidor cada uno de los siguientes servicios:*
- Servicio DNS (NombreEmpresa.com.do)
- Servicio DHCP (Debe asiganar ip a las Vlans de esa Sede)
- Servicio Web (URL:NombreEmpresa.com.do, se debe montar la pagina web de la empresa)
- Servicio Radius (Implementar un Servidor Radius, y crear un usuario para cada integarnte del grupo)
- Servicio De Correo (Dominio: NombreEmpresa.com.do, cree una cuenta para cada integrante)

### Sede Romana
Domina la Región Este. Permite atender el polo turístico e industrial más grande del país.
Crear Mínimo 3 Departamentos:
- **VENTAS-ATENCION-ROM** (52 host + 40% a 5 años)
- **LOGISTICA-ROM** (25 host + 40% a 5 años)
- **RRHH-ROM** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

### Sede Barahona
Controla el Suroeste. Posición estratégica ante el desarrollo de Pedernales-Cabo Rojo.
Crear Mínimo 3 Departamentos:
- **VENTAS-NEGOCIOS-BAR** (52 host + 40% a 5 años)
- **SOPORTE-TECNICO-BAR** (25 host + 40% a 5 años)
- **ADMINISTRACION-BAR** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

### Sede Puerto Plata
Asegura el Cibao Norte y la Costa Atlántica.
Crear Mínimo 3 Departamentos:
- **VENTAS-COMERCIAL-PPL** (52 host + 40% a 5 años)
- **MARKETING-PPL** (25 host + 40% a 5 años)
- **CONTABILIDAD-PPL** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

### Sede San Francisco de Macorís
Cubre el Nordeste. Centro de operaciones productoras con alta demanda de digitalización.
Crear Mínimo 3 Departamentos:
- **VENTAS-REGIONAL-SFM** (52 host + 40% a 5 años)
- **OPERACIONES-SFM** (25 host + 40% a 5 años)
- **LEGAL-CUMPLIMIENTO-SFM** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

---

## 📊 Tablas de Direccionamiento

### Direccionamiento Público
| Descripción | Red | Mascara | Primera utilizable | Broadcast |
| :--- | :--- | :--- | :--- | :--- |
| Santo Domingo ↔ ISP | `1.0.0.0/30` | `255.255.255.252` | `1.0.0.1` | `1.0.0.3` |
| Santiago ↔ ISP | `1.0.0.4/30` | `255.255.255.252` | `1.0.0.5` | `1.0.0.7` |
| La Romana ↔ ISP | `1.0.0.8/30` | `255.255.255.252` | `1.0.0.9` | `1.0.0.11` |
| Puerto Plata ↔ ISP | `1.0.0.12/30` | `255.255.255.252` | `1.0.0.13` | `1.0.0.15` |
| Barahona ↔ ISP | `1.0.0.16/30` | `255.255.255.252` | `1.0.0.17` | `1.0.0.19` |
| San Francisco de Macorís ↔ ISP | `1.0.0.20/30` | `255.255.255.252` | `1.0.0.21` | `1.0.0.23` |
| NAT estático — Web/Correo | `1.0.0.24/29` | `255.255.255.248` | `1.0.0.25` | `1.0.0.31` |

### DMVPN/IPSec - 10.7.0.0/28
| Dispositivo | Rol | IP del túnel |
| :--- | :--- | :--- |
| R-CORE — Santo Domingo | Hub | `10.7.0.1` |
| Router Santiago | Spoke | `10.7.0.2` |
| Router La Romana | Spoke | `10.7.0.3` |
| Router Puerto Plata | Spoke | `10.7.0.4` |
| Router Barahona | Spoke | `10.7.0.5` |
| Router San Francisco de Macorís | Spoke | `10.7.0.6` |

* **Red:** `10.7.0.0/28`
* **Máscara:** `255.255.255.240`
* **Primera IP utilizable:** `10.7.0.1` | **Última:** `10.7.0.14` | **Broadcast:** `10.7.0.15`

### Direccionamiento Privado por Sede

<details>
<summary><b>Santo Domingo</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 100 | DEFAULT-SD | VLAN nativa | — | — | — | — | — |
| 110 | CALLCENTER-SD | Departamento 1 | 154 | `10.0.0.0/24` | `10.0.0.1` | `10.0.0.254` | `10.0.0.255` |
| 120 | VENTAS-SD | Departamento 2 | 112 | `10.0.1.0/25` | `10.0.1.1` | `10.0.1.126` | `10.0.1.127` |
| 130 | NOC-SOC-SD | Departamento 3 | 90 | `10.0.1.128/25` | `10.0.1.129` | `10.0.1.254` | `10.0.1.255` |
| 140 | ADMIN-GERENCIA-SD | Departamento 4 | 72 | `10.0.2.0/25` | `10.0.2.1` | `10.0.2.126` | `10.0.2.127` |
| 199 | MGMT-SD | Gestión/OOB | — | `10.0.3.0/28` | `10.0.3.1` | `10.0.3.14` | `10.0.3.15` |

**Enlaces de Infraestructura Interna:**
* R-CORE ↔ Switch núcleo 1: `10.6.0.0/30` (`.1` al `.2`)
* R-CORE ↔ Switch núcleo 2: `10.6.0.4/30` (`.5` al `.6`)
* EtherChannel L3 R8 ↔ R9: `10.6.0.8/30` (`.9` al `.10`)
</details>

<details>
<summary><b>Santiago</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 200 | DEFAULT-ST | VLAN nativa | — | — | — | — | — |
| 210 | VENTAS-ST | Ventas | 21 | `10.1.0.0/27` | `10.1.0.1` | `10.1.0.30` | `10.1.0.31` |
| 220 | DATACENTER-ST | DNS/DHCP/Web/RADIUS | 14 | `10.1.0.32/27` | `10.1.0.33` | `10.1.0.62` | `10.1.0.63` |
| 230 | ADMIN-ST | Administración | 7 | `10.1.0.64/28` | `10.1.0.65` | `10.1.0.78` | `10.1.0.79` |
| 299 | MGMT-ST | Gestión/OOB | — | `10.1.1.0/28` | `10.1.1.1` | `10.1.1.14` | `10.1.1.15` |

**Servidor Data Center:** `10.1.0.34` (Gateway: `10.1.0.33`)
</details>

<details>
<summary><b>La Romana</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 300 | DEFAULT-ROM | VLAN nativa | — | — | — | — | — |
| 310 | VENTAS-ATENCION-ROM | Departamento 1 | 73 | `10.2.0.0/25` | `10.2.0.1` | `10.2.0.126` | `10.2.0.127` |
| 320 | LOGISTICA-ROM | Departamento 2 | 35 | `10.2.0.128/26` | `10.2.0.129` | `10.2.0.190` | `10.2.0.191` |
| 330 | RRHH-ROM | Departamento 3 | 10 | `10.2.0.192/28` | `10.2.0.193` | `10.2.0.206` | `10.2.0.207` |
| 399 | MGMT-ROM | Gestión/OOB | — | `10.2.1.0/28` | `10.2.1.1` | `10.2.1.14` | `10.2.1.15` |
</details>

<details>
<summary><b>Puerto Plata</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 400 | DEFAULT-PPL | VLAN nativa | — | — | — | — | — |
| 410 | VENTAS-COMERCIAL-PPL | Departamento 1 | 73 | `10.3.0.0/25` | `10.3.0.1` | `10.3.0.126` | `10.3.0.127` |
| 420 | MARKETING-PPL | Departamento 2 | 35 | `10.3.0.128/26` | `10.3.0.129` | `10.3.0.190` | `10.3.0.191` |
| 430 | CONTABILIDAD-PPL | Departamento 3 | 10 | `10.3.0.192/28` | `10.3.0.193` | `10.3.0.206` | `10.3.0.207` |
| 499 | MGMT-PPL | Gestión/OOB | — | `10.3.1.0/28` | `10.3.1.1` | `10.3.1.14` | `10.3.1.15` |
</details>

<details>
<summary><b>Barahona</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 500 | DEFAULT-BAR | VLAN nativa | — | — | — | — | — |
| 510 | VENTAS-NEGOCIOS-BAR | Departamento 1 | 73 | `10.4.0.0/25` | `10.4.0.1` | `10.4.0.126` | `10.4.0.127` |
| 520 | SOPORTE-TECNICO-BAR | Departamento 2 | 35 | `10.4.0.128/26` | `10.4.0.129` | `10.4.0.190` | `10.4.0.191` |
| 530 | ADMINISTRACION-BAR | Departamento 3 | 10 | `10.4.0.192/28` | `10.4.0.193` | `10.4.0.206` | `10.4.0.207` |
| 599 | MGMT-BAR | Gestión/OOB | — | `10.4.1.0/28` | `10.4.1.1` | `10.4.1.14` | `10.4.1.15` |
</details>

<details>
<summary><b>San Francisco de Macorís</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 600 | DEFAULT-SFM | VLAN nativa | — | — | — | — | — |
| 610 | VENTAS-REGIONAL-SFM | Departamento 1 | 73 | `10.5.0.0/25` | `10.5.0.1` | `10.5.0.126` | `10.5.0.127` |
| 620 | OPERACIONES-SFM | Departamento 2 | 35 | `10.5.0.128/26` | `10.5.0.129` | `10.5.0.190` | `10.5.0.191` |
| 630 | LEGAL-CUMPLIMIENTO-SFM | Departamento 3 | 10 | `10.5.0.192/28` | `10.5.0.193` | `10.5.0.206` | `10.5.0.207` |
| 699 | MGMT-SFM | Gestión/OOB | — | `10.5.1.0/28` | `10.5.1.1` | `10.5.1.14` | `10.5.1.15` |

---

```
</details>

