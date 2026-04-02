# VPN + Proxy con Docker (WireGuard + Gluetun)

Este proyecto permite que únicamente el tráfico de un navegador (o aplicaciones específicas) sea enroutado a través de una VPN WireGuard usando un proxy HTTP local, mientras que el resto del sistema mantiene su conectividad normal utilizando la IP local.

La configuración está pensada para funcionar tanto en **Windows** como en **Linux**, siempre que Docker tenga soporte para contenedores Linux.

---

# 🧩 Contexto y alcance del repositorio

Este repositorio contiene la **implementación práctica** de la solución descrita en el artículo.

Aquí encontrarás todo lo necesario para:

* Configurar la conexión VPN mediante Docker
* Exponer un proxy HTTP local
* Aislar el tráfico del navegador
* Ejecutar la solución en **Windows o Linux**
* Validar y depurar el funcionamiento

👉 La explicación completa del problema, motivación y decisiones técnicas está documentada en el artículo de [Medium](https://medium.com/@manuel.bouza07/3a0642b7bdc5) enlazado en el repositorio.

---

# 🎯 Objetivo

* Ejecutar una VPN WireGuard dentro de Docker
* Exponer un proxy HTTP en `127.0.0.1:8888`
* Configurar un navegador (por ejemplo, **Edge / Chrome / Brave / Firefox**) para usar ese proxy
* Aislar el tráfico del navegador del resto del sistema

---

# 🧱 Arquitectura

```text
Navegador (Edge / Chrome / Brave / Firefox)
        ↓
Proxy HTTP (127.0.0.1:8888)
        ↓
Docker (port mapping / NAT)
        ↓
Contenedor Gluetun
   ├── HTTP Proxy
   ├── Firewall (kill switch)
   ├── DNS (DoT)
   └── WireGuard VPN (tun0)
        ↓
Internet (IP pública de la VPN)
```

---

# ⚙️ Requisitos

## 🪟 Windows

* Windows 10/11
* Docker Desktop con backend Linux habilitado (normalmente WSL2)
* Configuración válida de WireGuard (`.conf`)
* Puerto UDP del servidor accesible

---

## 🐧 Linux

* Docker Engine + Docker Compose Plugin
* Permisos para ejecutar Docker (`docker` o `sudo docker`)
* Configuración válida de WireGuard (`.conf`)
* Puerto UDP del servidor accesible

---

# 📁 Estructura del proyecto

```text
vpn-proxy/
│
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
└── README.md
```

---

# 📥 1. Descargar el repositorio

Ubícate en el directorio donde deseas clonar el proyecto.

## 🪟 Windows

```powershell
cd C:\ruta\donde\guardar\el\proyecto
git clone https://github.com/ManuelBouza/vpn-proxy
cd vpn-proxy
```

---

## 🐧 Linux

```bash
cd /ruta/donde/guardar/el/proyecto
git clone https://github.com/ManuelBouza/vpn-proxy
cd vpn-proxy
```

---

# 🧾 2. Crear el fichero .env

El fichero `.env` debe completarse manualmente en tu entorno antes de levantar los contenedores.

Por seguridad:

* `.env` **NO se versiona**
* Contiene credenciales sensibles

---

## 🪟 Windows

### Crear archivo desde plantilla

```powershell
Copy-Item .env.example .env
```

> Nota: en PowerShell moderno `cp` suele ser alias de `Copy-Item`, pero se usa el comando explícito por compatibilidad.

---

## 🐧 Linux

### Crear archivo desde plantilla

```bash
cp .env.example .env
```

---

## ✏️ Configuración del archivo

Editar el archivo `.env` con tus valores:

```env
GLUETUN_CONTAINER_NAME=gluetun
HTTP_PROXY_BIND_IP=127.0.0.1
HTTP_PROXY_BIND_PORT=8888

WIREGUARD_ENDPOINT_IP=TU_IP_VPN
WIREGUARD_ENDPOINT_PORT=TU_PUERTO
WIREGUARD_PUBLIC_KEY=CLAVE_PUBLICA_SERVIDOR
WIREGUARD_PRIVATE_KEY=CLAVE_PRIVADA_CLIENTE
WIREGUARD_PRESHARED_KEY=CLAVE_PRESHARED
WIREGUARD_ADDRESSES=IP_DEL_CLIENTE/32
TZ=Europe/Madrid
```

---

## ⚠️ Importante

* Nunca subas `.env` a GitHub
* Añádelo a `.gitignore`
* Trata este archivo como **secreto** (contiene claves privadas)
* Valida que los datos de WireGuard sean correctos

---

# 🧾 3. Revisar docker-compose.yml

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: ${GLUETUN_CONTAINER_NAME:-gluetun}
    cap_add:
      - NET_ADMIN
      - NET_RAW
    ports:
      - "${HTTP_PROXY_BIND_IP:-127.0.0.1}:${HTTP_PROXY_BIND_PORT:-8888}:8888/tcp"
    environment:
      - VPN_SERVICE_PROVIDER=custom
      - VPN_TYPE=wireguard

      # WireGuard config desde .env
      - WIREGUARD_ENDPOINT_IP=${WIREGUARD_ENDPOINT_IP}
      - WIREGUARD_ENDPOINT_PORT=${WIREGUARD_ENDPOINT_PORT}
      - WIREGUARD_PUBLIC_KEY=${WIREGUARD_PUBLIC_KEY}
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_PRESHARED_KEY=${WIREGUARD_PRESHARED_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}

      # Proxy HTTP
      - HTTPPROXY=on
      - HTTPPROXY_LOG=on
      - HTTPPROXY_STEALTH=on

      # Opcional
      - TZ=${TZ:-Europe/Madrid}

    restart: unless-stopped
```

👉 Tras clonar el repositorio, crea `.env`, rellénalo con tus datos y luego ejecuta Docker.

---

# 🔑 4. Obtener datos desde WireGuard (.conf)

Ejemplo:

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = IP_DEL_CLIENTE/32

[Peer]
PublicKey = SERVER_PUBLIC_KEY
PresharedKey = PSK
Endpoint = IP:PORT
AllowedIPs = 0.0.0.0/0
```

### Traducción a Docker:

| WireGuard     | Docker                  |
| ------------- | ----------------------- |
| PrivateKey    | WIREGUARD_PRIVATE_KEY   |
| PublicKey     | WIREGUARD_PUBLIC_KEY    |
| PresharedKey  | WIREGUARD_PRESHARED_KEY |
| Address       | WIREGUARD_ADDRESSES     |
| Endpoint IP   | WIREGUARD_ENDPOINT_IP   |
| Endpoint Port | WIREGUARD_ENDPOINT_PORT |

---

# 🚀 5. Levantar el servicio

## 🪟 Windows

```powershell
docker compose up -d
```

---

## 🐧 Linux

```bash
docker compose up -d
```

---

# 📜 6. Ver logs

## 🪟 Windows

```powershell
docker logs -f gluetun
```

---

## 🐧 Linux

```bash
docker logs -f gluetun
```

---

# 🧪 7. Verificar conexión

## 🪟 Windows

Sin proxy:

```powershell
curl.exe https://ifconfig.me
```

Con proxy:

```powershell
curl.exe -x http://127.0.0.1:8888 https://ifconfig.me
```

---

## 🐧 Linux

Sin proxy:

```bash
curl https://ifconfig.me
```

Con proxy:

```bash
curl -x http://127.0.0.1:8888 https://ifconfig.me
```

👉 Debe devolver la IP de la VPN

---

# 🌐 8. Navegador

## 🥇 Edge / Chrome / Brave

Para navegadores basados en Chromium se recomienda usar **Proxy SwitchyOmega 3 (ZeroOmega)**:

[Proxy SwitchyOmega 3 (ZeroOmega)](https://microsoftedge.microsoft.com/addons/detail/proxy-switchyomega-3-zer/dmaldhchmoafliphkijbfhaomcgglmgd)

### Configuración inicial recomendada

1. Configurar el perfil `proxy` con estos valores:

* **Scheme**: `(default)`
* **Protocol**: `HTTP`
* **Server**: `127.0.0.1`
* **Port**: `8888`

Y deja el resto así:

* `http://` → `(use default)`

* `https://` → `(use default)`

* `ftp://` → `(use default)`

* **Bypass List**:

```text
127.0.0.1
::1
localhost
```

---

### Ajustes obligatorios para que cargue siempre por el proxy

En la sección **Interface**:

1. Cambia el **Startup Profile** a `proxy`
2. Pulsa **Apply changes**
3. Reinicia el navegador

> ⚠️ Importante:
> No configures manualmente `https://` como proxy.
> Gluetun utiliza un proxy HTTP con soporte CONNECT para HTTPS.

---

### Verificación

Abre:

```text
https://ifconfig.me
```

👉 Debe mostrar la IP de la VPN (no la IP local)

---

## 🥈 Firefox (segunda opción recomendada)

Firefox permite configurar proxy de forma independiente sin afectar el sistema.

Configuración:

* Proxy HTTP: `127.0.0.1`
* Puerto: `8888`
* Activar: usar para todos los protocolos

---

## 💡 Nota importante

Los navegadores basados en Chromium (Chrome, Edge, Brave):

* No aíslan el tráfico por sí solos como Firefox
* Requieren una extensión o configuración explícita de proxy
* Para este proyecto, **ZeroOmega** es la opción recomendada

---

# ⚠️ Problemas comunes

## ❌ Proxy devuelve 503

Causa:

* VPN no conectada o en proceso de reconexión

👉 El proxy depende completamente del túnel VPN.

Solución:

```powershell
docker logs gluetun
```

---

## ❌ DNS timeout

Causa:

* La VPN no está enroutando tráfico correctamente
* El túnel WireGuard no está funcionando aunque no haya error explícito

---

## ❌ No hay conexión VPN

Verificar:

* Endpoint accesible (IP/puerto)
* Claves correctas
* PresharedKey (si aplica)
* Que el servidor permita tu peer

---

## ❌ WireGuard funciona en Windows o Linux pero no en Docker

Causa común:

👉 WireGuard **no permite múltiples conexiones simultáneas con el mismo peer**

Solución:

* Desactivar el cliente WireGuard en el host
  **o**
* Crear un peer adicional en el servidor

---

## ❌ Navegador no usa el proxy

Verificar:

* ZeroOmega → perfil `proxy` activo
* `Startup Profile = proxy`
* `Apply changes` ejecutado
* El navegador fue reiniciado

👉 Si no está activo, el tráfico saldrá por la IP local.

---

# 🧠 Reglas importantes

* 1 peer = 1 conexión activa
* WireGuard es “silencioso” (puede fallar sin errores claros)
* “Connected” ≠ tráfico funcionando
* Si la VPN cae → el proxy deja de funcionar (comportamiento esperado)

---

# 🔐 Opcional: proteger proxy con usuario

```yaml
- HTTPPROXY_USER=usuario
- HTTPPROXY_PASSWORD=password
```

Uso:

```text
Windows: curl.exe -x http://usuario:password@127.0.0.1:8888 https://ifconfig.me
Linux:   curl -x http://usuario:password@127.0.0.1:8888 https://ifconfig.me
```

---

# 🛑 Detener servicio

## 🪟 Windows

```powershell
docker compose down
```

---

## 🐧 Linux

```bash
docker compose down
```

---

# 🔁 Reiniciar

## 🪟 Windows

```powershell
docker restart gluetun
```

---

## 🐧 Linux

```bash
docker restart gluetun
```

---

# 🧪 Debug útil

## 🪟 Windows

```powershell
docker logs gluetun
docker ps
docker exec -it gluetun sh
```

---

## 🐧 Linux

```bash
docker logs gluetun
docker ps
docker exec -it gluetun sh
```
