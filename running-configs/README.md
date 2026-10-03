# Running-configs — Infraestructura 2

Configuraciones de los equipos de la práctica #2, topología #2
(VPN IPsec Site-to-Site FortiGate ↔ Cisco).

| Archivo | Equipo | Función |
|---|---|---|
| `SW-USERS-2168.txt` | Switch IOSvL2 | VLAN 10 y trunk hacia el FortiGate |
| `ISP-2168-T2.txt` | Router ISP | Tránsito entre WAN y salida a Internet con PAT |
| `R-CISCO-2168-sanitized.txt` | Router Cisco | Extremo VPN, gateway del servidor y NAT exemption |
| `FG-T2-2168-sanitized.conf` | FortiGate | Gateway de usuarios, DHCP, políticas y extremo VPN |

> **Seguridad:** los archivos `sanitized` fueron limpiados. La PSK fue
> reemplazada por `<PSK-OMITIDA>` y, en el FortiGate, se publica solo un
> extracto sin certificados, claves privadas ni hashes de contraseñas.
