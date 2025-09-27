# 📘 AdGuard Home en MikroTik RB5009

Este repositorio contiene dos versiones del tutorial para implementar AdGuard Home en un entorno con MikroTik RB5009:

- **Versión 1.0** → AdGuard Home ejecutándose como contenedor dentro de MikroTik (limitación: no muestra IPs reales de clientes).
- **Versión 2.0** → AdGuard Home migrado a un dispositivo externo con IP directa en la LAN (`10.5.50.2`), lo que permite trazabilidad completa de clientes.

---

# 🚀 Versión 2.0 – AdGuard Home fuera del contenedor (IP 10.5.50.2)

En esta versión, AdGuard Home se ejecuta en un **dispositivo externo** (ej. Raspberry Pi, VM o servidor físico) con IP directa en la LAN (`10.5.50.2`).  
Esto resuelve la limitación de la versión 1.0 y permite que AdGuard Home registre correctamente las IPs reales de los clientes (`10.5.50.x`).

---

## 1. Infraestructura de red base para clientes

```bash
/interface/bridge/add name=bridgeLan comment="Bridge principal para clientes"
/ip/address/add address=10.5.50.254/24 interface=bridgeLan comment="IP del router en LAN"
/ip/pool/add name=Hotspot ranges=10.5.50.10-10.5.50.200 comment="Pool DHCP para clientes"
/ip/dhcp-server/add name=Hotspot interface=bridgeLan address-pool=Hotspot disabled=no comment="Servidor DHCP activo"
/ip/dhcp-server/network/add address=10.5.50.0/24 gateway=10.5.50.254 dns-server=10.5.50.2 comment="Entregar DNS de AdGuard externo a clientes"
```

---

## 2. Instalación de AdGuard Home en dispositivo externo

En el dispositivo con IP `10.5.50.2` (ej. Raspberry Pi, VM):

```bash
curl -s -S -L https://static.adguard.com/adguardhome/release/AdGuardHome_linux_arm64.tar.gz -o adguard.tar.gz
tar -xvf adguard.tar.gz
cd AdGuardHome
./AdGuardHome -s install
```

La interfaz web quedará disponible en:  
`http://10.5.50.2:3000`

---

## 3. Configuración inicial de AdGuard Home

1. Accede a `http://10.5.50.2:3000`
2. Completa el asistente:
   - Idioma
   - Interfaz de escucha: `10.5.50.2`, puerto `53`
   - Web interface: `10.5.50.2:3000`
   - Usuario y contraseña

---

## 4. Configurar proveedores DNS en AdGuard Home

En la interfaz web → **Settings → DNS settings**:

```text
1.1.1.1
8.8.8.8
9.9.9.9
```

---

## 5. Desactivar resolución DNS en MikroTik

```bash
/ip/dns/set servers=10.5.50.2 allow-remote-requests=no cache-size=2048KiB query-server-timeout=2s
```

---

## 6. Verificación de conectividad

Desde un cliente de la LAN:

```bash
nslookup google.com
```

- El servidor DNS debe ser `10.5.50.2`
- En el dashboard de AdGuard Home, el cliente debe aparecer con su IP real (`10.5.50.x`)

---

## 7. Reglas de firewall (opcional)

Si deseas restringir el acceso DNS solo a clientes LAN:

```bash
/ip/firewall/filter/add chain=forward src-address=10.5.50.0/24 dst-address=10.5.50.2 protocol=udp dst-port=53 action=accept comment="Permitir DNS UDP hacia AdGuard"
/ip/firewall/filter/add chain=forward src-address=10.5.50.0/24 dst-address=10.5.50.2 protocol=tcp dst-port=53 action=accept comment="Permitir DNS TCP hacia AdGuard"
/ip/firewall/filter/add chain=forward dst-address=10.5.50.2 protocol=udp dst-port=53 action=drop comment="Bloquear DNS UDP externo"
/ip/firewall/filter/add chain=forward dst-address=10.5.50.2 protocol=tcp dst-port=53 action=drop comment="Bloquear DNS TCP externo"
```

---

## 8. Verificación en AdGuard Home

- Accede a **Dashboard → Clientes más frecuentes**
- Ahora sí deberías ver las IPs reales de cada cliente (`10.5.50.252`, `10.5.50.247`, etc.)

---

## 9. Comparación entre versiones

| Versión | Ubicación de AdGuard | IP entregada por DHCP | Visibilidad de clientes |
|---------|----------------------|-----------------------|--------------------------|
| 1.0     | Contenedor en MikroTik | 172.17.0.2           | ❌ Solo `172.17.0.1` o AP |
| 2.0     | Dispositivo externo   | 10.5.50.2            | ✅ IPs reales de clientes |

---

## 10. Finalización

Con esta versión, AdGuard Home funciona con IP directa en la LAN (`10.5.50.2`), lo que permite trazabilidad completa de clientes y estadísticas precisas.  
La versión 1.0 queda como referencia para entornos donde no se disponga de hardware adicional, pero la **versión 2.0 es la recomendada para producción**.
