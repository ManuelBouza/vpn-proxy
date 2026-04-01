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

## ⚡ Uso rápido (TL;DR)

```bash
git clone https://github.com/ManuelBouza/vpn-proxy
cd vpn-proxy

# Windows (PowerShell)
Copy-Item .env.example .env

# Linux
cp .env.example .env

# editar .env

docker compose up -d
```

Proxy disponible en:

```
http://127.0.0.1:8888
```

---

# 🎯 Objetivo

* Ejecutar una VPN WireGuard dentro de Docker
* Exponer un proxy HTTP en `127.0.0.1:8888`
* Configurar un navegador (ej. Firefox o Edge/Chrome con extensión) para usar ese proxy
* Aislar el tráfico del navegador del resto del sistema

---

# 🧱 Arquitectura

```id="rcjgdu"
Navegador (Firefox / Edge / Chrome)
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

```id="2nsqxh"
vpn-proxy/
│
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
└── README.md
```

---

# 🧾 1. Crear el fichero .env

El fichero `.env` debe completarse manualmente en tu entorno antes de levantar los contenedores.

Por seguridad:

* `.env` **NO se versiona**
* Contiene credenciales sensibles

---

## 🪟 Windows

### Crear archivo desde plantilla

```powershell id="env_win_copy"
Copy-Item .env.example .env
```

> Nota: en PowerShell moderno `cp` suele ser alias de `Copy-Item`, pero se usa el comando explícito por compatibilidad.

---

## 🐧 Linux

### Crear archivo desde plantilla

```bash id="env_linux_copy"
cp .env.example .env
```

---

## ✏️ Configuración del archivo

Editar el archivo `.env` con tus valores:

```env id="env_vars"
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
* Validar que los datos de WireGuard sean correctos

---

# 🧾 2. Revisar docker-compose.yml

```yaml id="xkkte5"
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

# 🔑 3. Obtener datos desde WireGuard (.conf)

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

# 🚀 4. Levantar el servicio

## 🪟 Windows

```powershell id="run_win"
cd C:\ruta\al\proyecto
docker compose up -d
```

---

## 🐧 Linux

```bash id="run_linux"
cd /ruta/al/proyecto
docker compose up -d
```

---

# 📜 5. Ver logs

## 🪟 Windows

```powershell id="logs_win"
docker logs -f gluetun
```

---

## 🐧 Linux

```bash id="logs_linux"
docker logs -f gluetun
```

---

# 🧪 6. Verificar conexión

## 🪟 Windows

Sin proxy:

```powershell id="test_win_1"
curl.exe https://ifconfig.me
```

Con proxy:

```powershell id="test_win_2"
curl.exe -x http://127.0.0.1:8888 https://ifconfig.me
```

---

## 🐧 Linux

Sin proxy:

```bash id="test_linux_1"
curl https://ifconfig.me
```

Con proxy:

```bash id="test_linux_2"
curl -x http://127.0.0.1:8888 https://ifconfig.me
```

👉 Debe devolver la IP de la VPN

---

# 🌐 7. Navegador

## 🥇 Firefox (recomendado)

Permite configurar proxy de forma independiente sin afectar el sistema.

Configuración:

* Proxy HTTP: `127.0.0.1`
* Puerto: `8888`
* Activar: usar para todos los protocolos

---

## 🥈 Chrome / Edge / Brave

Usar extensión: **FoxyProxy Standard**

Configuración recomendada:

* Tipo: HTTP
* Host: `127.0.0.1`
* Puerto: `8888`

👉 Activar manualmente el perfil desde la extensión.

---

## 🧪 Verificación

Abrir:

```
https://ifconfig.me
```

👉 Debe mostrar la IP de la VPN

---

## 💡 Nota importante

Los navegadores basados en Chromium (Chrome, Edge, Brave):

* Usan el proxy del sistema por defecto
* No permiten aislamiento sin extensiones

👉 Por eso se recomienda:

* Firefox (nativo)
  **o**
* FoxyProxy (alternativa en Chromium)

---

# ⚠️ Problemas comunes

## ❌ Proxy devuelve 503

Causa:

* VPN no conectada o en proceso de reconexión

👉 El proxy depende completamente del túnel VPN.

Solución:

```powershell id="24hvx4"
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

* Firefox → configuración manual activa
* FoxyProxy → perfil seleccionado

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

```text id="wfztug"
Windows: curl.exe -x http://usuario:password@127.0.0.1:8888 https://ifconfig.me
Linux:   curl -x http://usuario:password@127.0.0.1:8888 https://ifconfig.me
```

---

# 🛑 Detener servicio

## 🪟 Windows

```powershell id="stop_win"
docker compose down
```

---

## 🐧 Linux

```bash id="stop_linux"
docker compose down
```

---

# 🔁 Reiniciar

## 🪟 Windows

```powershell id="restart_win"
docker restart gluetun
```

---

## 🐧 Linux

```bash id="restart_linux"
docker restart gluetun
```

---

# 🧪 Debug útil

## 🪟 Windows

```powershell id="debug_win"
docker logs gluetun
docker ps
docker exec -it gluetun sh
```

---

## 🐧 Linux

```bash id="debug_linux"
docker logs gluetun
docker ps
docker exec -it gluetun sh
```
---
