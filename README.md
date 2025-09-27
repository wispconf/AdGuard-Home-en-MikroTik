# 📘 AdGuard Home en MikroTik RB5009 (sin USB) con redirección DNS y dominio wifi1.info

Este tutorial documenta la instalación de AdGuard Home como contenedor en MikroTik RB5009, con redirección DNS para clientes LAN y visibilidad estructurada. Incluye aclaraciones sobre limitaciones de visibilidad por cliente y recomendaciones para trazabilidad completa.

---

## 1. Infraestructura de red base para clientes

```bash
/interface/bridge/add name=bridgeLan comment="Bridge principal para clientes"
/ip/address/add address=10.5.50.254/24 interface=bridgeLan comment="IP del router en LAN"
/ip/pool/add name=Hotspot ranges=10.5.50.10-10.5.50.200 comment="Pool DHCP para clientes"
/ip/dhcp-server/add name=Hotspot interface=bridgeLan address-pool=Hotspot disabled=no comment="Servidor DHCP activo"
/ip/dhcp-server/network/add address=10.5.50.0/24 gateway=10.5.50.254 dns-server=172.17.0.2 comment="Entregar DNS de AdGuard directamente a clientes"
```

---

## 2. Red interna para el contenedor

```bash
/interface/bridge/add name=containers comment="Bridge aislado para contenedores"
/ip/address/add address=172.17.0.1/24 interface=containers comment="IP del bridge de contenedores"
/interface/veth/add name=veth1 address=172.17.0.2/24 gateway=172.17.0.1 comment="Interfaz virtual para AdGuard"
/interface/bridge/port/add bridge=containers interface=veth1 comment="Vincular veth1 al bridge de contenedores"
```

---

## 3. Configurar parámetros globales de contenedores

```bash
/container/config/set registry-url=https://registry-1.docker.io tmp-dir=/adguard/tmp
```

---

## 4. Crear el contenedor AdGuard Home

```bash
/container/add \
  name=adguardhome \
  interface=veth1 \
  logging=yes \
  root-dir=adguard \
  remote-image=adguard/adguardhome:latest
```

---

## 5. Activar manualmente el contenedor

```bash
/container/start 0
```

---

## 6. Activar inicio automático

```bash
/container/set 0 start-on-boot=yes
```

---

## 7. Reglas de firewall y NAT

```bash
/ip/firewall/filter/add chain=forward src-address=172.17.0.0/24 action=accept comment="Permitir salida desde contenedor"

/ip/firewall/nat/add chain=dstnat protocol=udp dst-port=53 \
  src-address=10.5.50.0/24 dst-address-type=local \
  in-interface=bridgeLan action=dst-nat \
  to-addresses=172.17.0.2 to-ports=53 disabled=yes \
  comment="(Desactivada) Redirigir DNS UDP a AdGuard"

/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=53 \
  src-address=10.5.50.0/24 dst-address-type=local \
  in-interface=bridgeLan action=dst-nat \
  to-addresses=172.17.0.2 to-ports=53 disabled=yes \
  comment="(Desactivada) Redirigir DNS TCP a AdGuard"

/ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 \
  out-interface=ether1 action=masquerade \
  comment="Permitir salida a Internet desde contenedor"
```

---

## 8. Verificación de conectividad

```bash
/tool/ping 1.1.1.1
/tool/ping 8.8.8.8
/tool/ping 172.17.0.2
```

---

## 9. Acceder a la interfaz web de AdGuard Home

- Abre tu navegador y visita:  
  `http://172.17.0.2:3000`

- Completa el asistente de configuración:
  - Idioma
  - Interfaz de escucha: `172.17.0.2`, puerto `53`
  - Web interface: `172.17.0.2:3000`
  - Usuario y contraseña

---

## 10. Configurar proveedores DNS en AdGuard Home

- Accede a:  
  `http://172.17.0.2/#dns`

- Introduce estos DNS:

```bash
1.1.1.1
8.8.8.8
9.9.9.9
```

---

## 11. Desactivar resolución DNS en MikroTik

```bash
/ip/dns/set servers=172.17.0.2 allow-remote-requests=no cache-size=2048KiB query-server-timeout=2s
```

---

## 12. Verificación en AdGuard Home

- Accede al panel:  
  `Dashboard → Clientes más frecuentes`

- Si todo está correctamente configurado, deberías ver IPs como `10.5.50.x`

---

## 13. ⚠️ Limitación estructural de visibilidad por cliente

Cuando AdGuard Home se ejecuta como contenedor en MikroTik, las peticiones DNS llegan desde el gateway virtual (`172.17.0.1`), incluso si los clientes usan directamente la IP del contenedor (`172.17.0.2`).  
Esto se debe a cómo MikroTik encapsula el tráfico en su sistema de contenedores.  
En este escenario, AdGuard Home no puede ver la IP real del cliente (como `10.5.50.252` o `10.5.50.247`) porque el tráfico pasa por el router antes de llegar al contenedor.

---

## 📘 Nota importante

    Si AdGuard está dentro del contenedor de MikroTik, los clientes seguirán apareciendo como 172.17.0.1 en el dashboard (limitación estructural).
    Si AdGuard está en un dispositivo con IP directa en la LAN (ej. 10.5.50.5), entonces sí verás las IPs reales de cada cliente en AdGuard Home.
