# Scripts

## `web-server-https-setup.sh`

Instala Apache2 y habilita HTTPS con un certificado autofirmado en `WEB-SV-2168`.
Se ejecuta en el servidor web, con permisos de administrador:

```bash
chmod +x web-server-https-setup.sh
sudo ./web-server-https-setup.sh
```

## `test-vpn-connectivity.sh`

Ejecuta las tres pruebas principales hacia el servidor:

1. ICMP (`ping`)
2. HTTPS (`curl`)
3. Recorrido (`tracepath`)

Se ejecuta desde el PC del usuario:

```bash
chmod +x test-vpn-connectivity.sh
./test-vpn-connectivity.sh
```

o especificando otra IP:

```bash
./test-vpn-connectivity.sh 10.21.68.130
```

Con la VPN activa, las tres pruebas responden. Con el túnel deshabilitado,
el ping y el HTTPS fallan y el script lo indica al final.
