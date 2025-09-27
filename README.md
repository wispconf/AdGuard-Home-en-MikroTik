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
