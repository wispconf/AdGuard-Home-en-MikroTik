# 📘 AdGuard Home en OPNsense – Versión 3.0

En esta versión, AdGuard Home se instala **directamente en OPNsense** mediante un plugin comunitario.  
Esto permite que el firewall actúe como servidor DNS con filtrado avanzado, integrando AdGuard Home con **Unbound** (resolver nativo de OPNsense).

---

## 1. Requisitos previos

- OPNsense actualizado (23.x o superior).
- Acceso SSH o consola.
- Conexión a Internet para instalar el repositorio adicional.

---

## 2. Instalar repositorio comunitario

1. Accede por SSH a OPNsense.
2. Ejecuta:

```bash
pkg add -f https://www.routerperformance.net/opnsense-repo/opnsense-routerperformance-repo.txz
pkg install -y os-adguardhome-maxit
```

3. Verifica que el plugin aparece en **System → Firmware → Plugins** como instalado.

---

## 3. Ajustar Unbound para coexistir con AdGuard Home

1. En la GUI de OPNsense, ve a:  
   **Services → Unbound DNS → General**

2. Cambia el puerto de escucha de **53** a **5353**.  
   (Esto libera el puerto 53 para AdGuard Home).

3. Guarda y aplica cambios.

---

## 4. Configurar AdGuard Home

1. En la GUI de OPNsense, ve a:  
   **Services → AdGuard Home**

2. Activa el servicio y define:
   - **Listen interface**: LAN (ej. `10.5.50.1`)
   - **Listen port**: `53`
   - **Web interface**: `3000`

3. Guarda y aplica.

4. Accede a la interfaz web de AdGuard Home:  
   `http://<IP_OPNsense>:3000`

5. Completa el asistente:
   - Idioma
   - Usuario y contraseña
   - DNS Upstream: apunta a Unbound en el puerto 5353  
     Ejemplo:
     ```text
     127.0.0.1:5353
     ```

---

## 5. Configurar DHCP para entregar AdGuard Home como DNS

En OPNsense:

1. Ve a **Services → DHCPv4 → [LAN]**.
2. En **DNS servers**, coloca la IP LAN de OPNsense (ej. `10.5.50.1`).
3. Guarda y aplica.

---

## 6. Verificación

1. En un cliente de la LAN, renueva DHCP.
2. Ejecuta:

```bash
nslookup google.com
```

- El servidor DNS debe ser la IP de OPNsense (`10.5.50.1`).
- En el dashboard de AdGuard Home, deben aparecer las IPs reales de los clientes (`10.5.50.x`).

---

## 7. Reglas de firewall (opcional)

Para forzar todo el tráfico DNS a pasar por AdGuard Home:

```bash
Firewall → NAT → Port Forward
```

- Redirige todo tráfico UDP/TCP puerto 53 hacia `10.5.50.1:53`.

---

## 8. Comparación con otras versiones

| Versión | Ubicación de AdGuard | Visibilidad de clientes | Complejidad |
|---------|----------------------|--------------------------|-------------|
| 1.0     | Contenedor en MikroTik | ❌ Solo gateway contenedor | Media |
| 2.0     | Dispositivo externo (10.5.50.2) | ✅ IPs reales | Baja |
| 3.0     | Integrado en OPNsense | ✅ IPs reales | Media |

---

## 9. Finalización

Con esta versión, AdGuard Home queda **integrado en OPNsense**, trabajando junto a Unbound.  
- AdGuard Home filtra y bloquea.  
- Unbound resuelve recursivamente y mantiene overrides locales.  
- Los clientes aparecen con sus IPs reales en el dashboard.  

👉 Esta es la opción más limpia si ya usas OPNsense como firewall principal.
