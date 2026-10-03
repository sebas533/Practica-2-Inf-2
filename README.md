# Práctica #2 — Topología #2: VPN Site-to-Site FortiGate ↔ Cisco

> **Video de demostración:** [Ver video](PENDIENTE-URL-DEL-VIDEO)

**Asignatura:** [SR]
**Estudiante:** [Luis Sebastian Roble Perez]
**Matrícula:** 2168

---

## 1. Objetivo

Armar en GNS3 una red donde los usuarios de la VLAN 10 lleguen a un servidor web HTTPS ubicado en otra red, y que esa comunicación solo sea posible a través de una **VPN IPsec Site-to-Site entre un FortiGate y un router Cisco**.

Lo que se busca demostrar:

- Los usuarios trabajan en la **VLAN 10** y reciben su IP por DHCP desde el FortiGate.
- El FortiGate es el gateway de los usuarios y les da salida a Internet con NAT.
- El servidor HTTPS está en una subred `/28` detrás del router Cisco.
- El router ISP conecta las dos WAN y da salida a Internet con PAT.
- El tráfico entre `10.21.68.0/25` y `10.21.68.128/28` viaja protegido por IPsec.
- El router Cisco excluye ese tráfico del NAT (NAT exemption).
- Si se deshabilita el túnel, el usuario pierde el acceso al servidor; al habilitarlo otra vez, vuelve.

---

## 2. Topología

![Topología de la infraestructura 2](Infra%202/Topologia%20.png)

Los equipos son:

| Equipo en GNS3 | Rol |
|---|---|
| `pcusuarios` | PC Ubuntu del usuario (VLAN 10) |
| `Switch-usuarios` | Switch IOSvL2 con la VLAN 10 y el trunk |
| `Fortigate` | Gateway de usuarios, DHCP, firewall y extremo VPN |
| `ISP-2168` | Router de tránsito entre las dos WAN y salida a Internet |
| `R-CISCO-2168` | Segundo extremo de la VPN y gateway del servidor |
| `websv` | Servidor Ubuntu con Apache2 y HTTPS |
| `NAT1` | Salida a Internet del laboratorio (GNS3 NAT) |
| `Cloud1` | Enlace de administración hacia la GUI del FortiGate |

---

## 3. Direccionamiento IP

El direccionamiento está basado en mi matrícula, **2168**. De ahí salen los dos bloques que uso en toda la práctica:

- **Redes LAN:** `10.21.68.x`
- **Enlaces WAN con el ISP:** `21.68.x.x`

| Red | Máscara | Hosts útiles | Uso |
|---|---|---:|---|
| `10.21.68.0/25` | 255.255.255.128 | 126 | Usuarios (VLAN 10) |
| `10.21.68.128/28` | 255.255.255.240 | 14 | Servidor web |
| `21.68.3.0/30` | 255.255.255.252 | 2 | Enlace ISP ↔ FortiGate |
| `21.68.4.0/30` | 255.255.255.252 | 2 | Enlace ISP ↔ R-CISCO |

| Equipo | Interfaz | Dirección |
|---|---|---|
| ISP-2168 | Fa0/0 (hacia NAT1) | DHCP (observado: `192.168.42.44/24`) |
| ISP-2168 | Fa1/0 (hacia FortiGate) | `21.68.3.1/30` |
| ISP-2168 | Fa1/1 (hacia R-CISCO) | `21.68.4.1/30` |
| FortiGate | port1 `WAN-ISP` | `21.68.3.2/30` |
| FortiGate | `VLAN10-USERS` (sobre port2) | `10.21.68.1/25` |
| FortiGate | port3 (administración) | `192.168.11.2/24` |
| PC usuario | ens3 (DHCP) | `10.21.68.10/25` |
| R-CISCO-2168 | Fa1/0 (WAN) | `21.68.4.2/30` |
| R-CISCO-2168 | Fa1/1 (LAN servidor) | `10.21.68.129/28` |
| Servidor web | ens3 | `10.21.68.130/28` |

**DHCP de usuarios:** rango `10.21.68.10` – `10.21.68.100`, gateway `10.21.68.1`, DNS `8.8.8.8` y `1.1.1.1`.

**Redes protegidas por la VPN:** `10.21.68.0/25` (usuarios) ↔ `10.21.68.128/28` (servidor). El peer del FortiGate es `21.68.3.2` y el del Cisco es `21.68.4.2`.

---

## 4. Entorno

- GNS3 con GNS3 VM
- 1 × FortiGate VM64-KVM 7.0.9
- 2 × Cisco C7200 (`ISP-2168` y `R-CISCO-2168`)
- Cisco IOSvL2 como switch
- Ubuntu Server para el usuario y para el servidor web
- Apache2 con HTTPS (certificado autofirmado)

---

## 5. Configuración

Las configuraciones completas (sin secretos) están en [`running-configs/`](running-configs/). Aquí se explica lo importante de cada equipo.

### 5.1 Switch: VLAN 10 y trunk

El switch lleva la VLAN 10 hasta el FortiGate:

- `Gi0/0` es un trunk 802.1Q hacia `port2` del FortiGate y solo permite la VLAN 10.
- `Gi0/1` es un puerto de acceso en la VLAN 10 para el PC del usuario.

```text
interface GigabitEthernet0/0
 description HACIA-FG-PORT2
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface GigabitEthernet0/1
 description HACIA-PC-USER-2168
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
```

![VLAN 10 en el switch](Infra%202/Show%20vlan%20brief.png)

`show vlan brief` muestra la VLAN 10 (`USERS-2168`) con el puerto `Gi0/1` asignado.

![Trunk 802.1Q hacia el FortiGate](Infra%202/Show%20Interface%20trunk.png)

`show interfaces trunk` confirma que `Gi0/0` está en modo trunk con encapsulación 802.1Q y que la VLAN 10 está permitida y activa.

### 5.2 FortiGate

El FortiGate hace de gateway, servidor DHCP y firewall de los usuarios, y es uno de los extremos de la VPN.

**Interfaces**

- `port1` (`WAN-ISP`): `21.68.3.2/30`, gateway `21.68.3.1`. Solo permite ping.
- `VLAN10-USERS`: interfaz VLAN con ID 10 sobre `port2`, IP `10.21.68.1/25`.
- `port3`: administración por GUI en `192.168.11.2`.

![Interfaces del FortiGate](Infra%202/Fortigate%20Interfaces.png)

**DHCP de la VLAN 10**

El servidor DHCP está activo en `VLAN10-USERS`, con el rango `10.21.68.10 – 10.21.68.100`, gateway `10.21.68.1` y DNS `8.8.8.8` y `1.1.1.1`.

```text
config system dhcp server
    edit 2
        set default-gateway 10.21.68.1
        set netmask 255.255.255.128
        set interface "VLAN10-USERS"
        config ip-range
            edit 1
                set start-ip 10.21.68.10
                set end-ip 10.21.68.100
            next
        end
        set dns-server1 8.8.8.8
        set dns-server2 1.1.1.1
    next
end
```

El PC del usuario recibe su configuración sin tocar nada:

![IP del usuario por DHCP](Infra%202/direccionamiento%20ip%20via%20dhcp%20usuarios.png)

Con `ip -br a` y `ip route` se ve que `ens3` tomó `10.21.68.10/25` y que la ruta por defecto es `10.21.68.1`.

**Política de salida a Internet**

La política `USER-TO-INTERNET` deja salir a la VLAN 10 por `port1` con NAT, usando la dirección de la interfaz de salida.

![Configuración de la política USER-TO-INTERNET](Infra%202/User%20to%20Internet%20conffig.png)

**Políticas del firewall**

| # | Nombre | Origen → Destino | Servicio | NAT |
|---|---|---|---|---|
| 1 | `USER-TO-INTERNET` | `VLAN10-USERS` → `WAN-ISP (port1)` | ALL | Sí |
| 2 | `vpn_VPN-FG-CISCO_local_0` | `VLAN10-USERS` → `VPN-FG-CISCO` | ALL | No |
| 3 | `vpn_VPN-FG-CISCO_remote_0` | `VPN-FG-CISCO` → `VLAN10-USERS` | ALL | No |

![Políticas del FortiGate](Infra%202/Politicas.png)

Las políticas 2 y 3 son las que permiten el tráfico en los dos sentidos por el túnel. El NAT va desactivado en ellas para que las direcciones reales viajen dentro del túnel. Todo lo demás cae en el *Implicit Deny*.

**Rutas estáticas**

| # | Destino | Salida | Función |
|---|---|---|---|
| 1 | `0.0.0.0/0` | `port1` vía `21.68.3.1` | Salida hacia el ISP |
| 2 | `10.21.68.128/28` | `VPN-FG-CISCO` | Tráfico al servidor por el túnel |
| 3 | `10.21.68.128/28` | blackhole (distancia 254) | Descarta el tráfico si el túnel cae |

La ruta *blackhole* es la que explica la prueba de dependencia del túnel (sección 7.5): sin túnel, el tráfico hacia el servidor se descarta en lugar de salir sin cifrar por Internet.

### 5.3 ISP

El `ISP-2168` conecta las dos WAN y da salida a Internet al laboratorio:

- `Fa1/0` → `21.68.3.1/30` (hacia el FortiGate), `ip nat inside`
- `Fa1/1` → `21.68.4.1/30` (hacia R-CISCO), `ip nat inside`
- `Fa0/0` → IP por DHCP desde GNS3 NAT, `ip nat outside`
- PAT de los dos enlaces WAN a través de `Fa0/0`

```text
ip nat inside source list 1 interface FastEthernet0/0 overload
access-list 1 permit 21.68.3.0 0.0.0.3
access-list 1 permit 21.68.4.0 0.0.0.3
```

![Interfaces del ISP](Infra%202/Showipinterfacesbrief%20ISP.png)

Las tres interfaces están `up/up`: `Fa0/0` con IP por DHCP, `Fa1/0` con `21.68.3.1` y `Fa1/1` con `21.68.4.1`.

![Tabla de rutas del ISP](Infra%202/Show%20Ip%20route%20ISP.png)

La tabla tiene las redes `21.68.3.0/30` y `21.68.4.0/30` conectadas y una ruta por defecto hacia el gateway de GNS3 NAT. El ISP **no conoce** las redes `10.21.68.x`: por eso el tráfico entre usuario y servidor solo funciona dentro del túnel.

### 5.4 Router Cisco (R-CISCO-2168)

Es el otro extremo de la VPN y el gateway del servidor.

- `Fa1/0` → `21.68.4.2/30`, `ip nat outside`, con el crypto map `VPN-MAP` aplicado.
- `Fa1/1` → `10.21.68.129/28`, `ip nat inside`.
- Ruta por defecto hacia `21.68.4.1` (el ISP).

![Interfaces del R-CISCO](Infra%202/Rcisco%20inerfaces%20brief.png)

`Fa1/0` y `Fa1/1` están `up/up`. `Fa0/0` está sin IP y apagada porque no se usa.

![Tabla de rutas del R-CISCO](Infra%202/Show%20ip%20route%20rcisco.png)

Se ven las redes conectadas `10.21.68.128/28` y `21.68.4.0/30`, y la ruta por defecto `S*` hacia `21.68.4.1`.

**NAT exemption**

El router hace PAT por `Fa1/0` para el tráfico normal de la LAN del servidor, pero el tráfico que va por la VPN no se debe traducir. Para eso la ACL 101 lo niega primero:

```text
ip nat inside source list 101 interface FastEthernet1/0 overload
access-list 101 deny   ip 10.21.68.128 0.0.0.15 10.21.68.0 0.0.0.127
access-list 101 permit ip 10.21.68.128 0.0.0.15 any
```

![ACL 101 y NAT en el R-CISCO](Infra%202/Acces%20list%20101%20y%20nat.png)

La primera línea de la ACL excluye del NAT el tráfico entre `10.21.68.128/28` y `10.21.68.0/25`. La segunda permite el resto para que salga por PAT.

### 5.5 Servidor web

- IP: `10.21.68.130/28`
- Gateway: `10.21.68.129`
- Servicio: Apache2 con HTTPS (443) y certificado autofirmado

---

## 6. VPN IPsec Site-to-Site

El túnel se levanta entre las IP WAN de los dos extremos. Los dos lados tienen que coincidir en todos los parámetros.

| Parámetro | FortiGate | R-CISCO |
|---|---|---|
| Nombre | `VPN-FG-CISCO` | `VPN-MAP` (crypto map) |
| Peer | `21.68.4.2` | `21.68.3.2` |
| IKE | IKEv1 | IKEv1 |
| Fase 1 | `des-md5`, `des-sha1` | DES / SHA1, DH grupo 5 |
| Autenticación | Pre-Shared Key | Pre-Shared Key |
| Fase 2 | `des-md5`, `des-sha1` | `esp-des esp-sha-hmac`, PFS grupo 5, lifetime 43200 s |
| Tráfico protegido (local) | `10.21.68.0/25` | `10.21.68.128/28` |
| Tráfico protegido (remoto) | `10.21.68.128/28` | `10.21.68.0/25` |

Configuración del lado Cisco:

```text
crypto isakmp policy 10
 authentication pre-share
 group 5
crypto isakmp key <PSK-OMITIDA> address 21.68.3.2
!
crypto ipsec transform-set TS-FG-CISCO esp-des esp-sha-hmac
 mode tunnel
!
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 21.68.3.2
 set security-association lifetime seconds 43200
 set transform-set TS-FG-CISCO
 set pfs group5
 match address VPN-TRAFFIC
!
ip access-list extended VPN-TRAFFIC
 permit ip 10.21.68.128 0.0.0.15 10.21.68.0 0.0.0.127
```

> **Seguridad:** la PSK real no se publica en este repositorio.

> **Nota:** DES y SHA1 se usaron para que el FortiGate y el router Cisco negociaran sin problemas en el laboratorio. No son parámetros recomendables para un entorno real, donde se usaría AES con SHA-256 o superior.

---

## 7. Validación

### 7.1 Túnel activo en el FortiGate

En *VPN → IPsec Tunnels*, el túnel `VPN-FG-CISCO` aparece en estado **Up** sobre `WAN-ISP (port1)`.

![Túnel VPN activo en el FortiGate](Infra%202/vpn%20activa.png)

### 7.2 IKE activo en el Cisco

```text
show crypto isakmp sa
```

![IKE activo en el R-CISCO](Infra%202/IKE%20ACTIVO%20en%20cisco.png)

La asociación entre `21.68.3.2` y `21.68.4.2` aparece en estado `QM_IDLE` / `ACTIVE`, lo que confirma que la fase 1 quedó establecida.

### 7.3 Asociación IPsec

```text
show crypto ipsec sa
```

![show crypto ipsec sa en el R-CISCO](Infra%202/IPsec%20cifrado.png)

Muestra el crypto map `VPN-MAP` aplicado en `Fa1/0`, el peer `21.68.3.2` y los selectores del tráfico protegido: `10.21.68.128/28` como identidad local y `10.21.68.0/25` como remota. Los contadores `#pkts encaps` y `#pkts decaps` indican el tráfico que realmente pasa cifrado por el túnel.

### 7.4 Pruebas con la VPN activa

Desde el PC del usuario:

```bash
ping -c 3 10.21.68.130
```

![Ping con la VPN activa](Infra%202/ping%20icmp%20vpn%20se%20actica.png)

El servidor responde. El primer paquete se pierde porque el túnel todavía se está negociando; a partir del segundo, las respuestas llegan sin problema (TTL 62, que corresponde a pasar por el FortiGate y por el R-CISCO).

```bash
curl -k -I https://10.21.68.130
```

![HTTPS a través de la VPN](Infra%202/https%20atraves%20de%20vpn%20ok.png)

La respuesta es `HTTP/1.1 200 OK` con `Server: Apache/2.4.66 (Ubuntu)`: el servidor web se alcanza por HTTPS a través del túnel.

### 7.5 Dependencia del túnel (VPN deshabilitada)

Se deshabilita el túnel `VPN-FG-CISCO` y su estado pasa a **Inactive**.

![Túnel VPN inactivo en el FortiGate](Infra%202/tunel%20vpn%20en%20el%20fortigate.png)

Con el túnel caído, el mismo `curl` desde el usuario falla:

![HTTPS sin la VPN](Infra%202/dependencia%20del%20tunel.png)

`curl` responde `Failed to connect to 10.21.68.130 port 443` en apenas 2 ms. Esto pasa porque, sin el túnel, desaparece la ruta por `VPN-FG-CISCO` y la ruta *blackhole* descarta el tráfico hacia `10.21.68.128/28`. Así queda demostrado que el acceso al servidor depende de la VPN.

### 7.6 Restauración

Se habilita otra vez el túnel y se repiten el ping y el `curl`. El túnel vuelve a **Up** y el servidor vuelve a responder.

<!-- Agregar aquí la captura de la VPN restaurada, por ejemplo:
![VPN restaurada](Infra%202/NOMBRE-DE-LA-CAPTURA.png)
-->

---

## 8. Running-configs

Las configuraciones están en [`running-configs/`](running-configs/):

- `SW-USERS-2168.txt`
- `ISP-2168-T2.txt`
- `R-CISCO-2168-sanitized.txt`
- `FG-T2-2168-sanitized.conf`

> **Seguridad:** los archivos marcados como `sanitized` se limpiaron antes de publicarlos. La PSK se reemplazó por `<PSK-OMITIDA>` y, del FortiGate, solo se publica un extracto (interfaces, DHCP, objetos, VPN, políticas y rutas), sin certificados, claves privadas ni hashes de contraseñas.

---

## 9. Scripts

La carpeta [`scripts/`](https://github.com/sebas533/Practica-2-Inf-2/blob/main/Scripts/Scripts.md) tiene dos scripts de apoyo:

- `web-server-https-setup.sh`: instala Apache2 y activa HTTPS con un certificado autofirmado en `WEB-SV-2168`.
- `test-vpn-connectivity.sh`: ejecuta las tres pruebas principales hacia el servidor (ping, HTTPS con curl y recorrido con tracepath).

Uso rápido de las pruebas:

```bash
./test-vpn-connectivity.sh
./test-vpn-connectivity.sh 10.21.68.130
```

---

## 10. Resultado

El usuario de la VLAN 10 llega al servidor HTTPS remoto mediante una VPN IPsec Site-to-Site entre un FortiGate y un router Cisco. Las pruebas mostraron que:

- el túnel se establece, tanto en el FortiGate (estado Up) como en el Cisco (`QM_IDLE` / `ACTIVE`);
- el servidor responde por ICMP y por HTTPS (`200 OK`) con el túnel activo;
- al deshabilitar el túnel, el servidor deja de ser alcanzable;
- el NAT del Cisco excluye el tráfico de la VPN y el FortiGate solo aplica NAT a la salida a Internet.

---

## Estructura del repositorio

```text
.
├── README.md
├── Infra 2/                         # capturas de la práctica
├── running-configs/
│   ├── FG-T2-2168-sanitized.conf
│   ├── ISP-2168-T2.txt
│   ├── R-CISCO-2168-sanitized.txt
│   ├── SW-USERS-2168.txt
│   └── README.md
└── scripts/
    ├── README.md
    ├── test-vpn-connectivity.sh
    └── web-server-https-setup.sh
```
