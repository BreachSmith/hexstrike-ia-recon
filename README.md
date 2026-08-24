# HexStrike IA — Guía de Instalación

Interfaz de IA para reconocimiento ofensivo. Un **operador en Kali Linux** (`demo_ia.py`) manda órdenes en lenguaje natural a un **backend HexStrike AI en un LXC de Proxmox** (`hexstrike_server.py`), que ejecuta `nmap` y —en órdenes libres— consulta un modelo local de Ollama. Todo el tráfico viaja por una malla cifrada de **Tailscale**; nada expuesto a Internet.

```
┌─────────────────┐        Tailscale (WireGuard)        ┌──────────────────────┐
│  Kali Linux     │  ────────────────────────────────►  │  ia-upy (LXC/Proxmox)│
│  demo_ia.py     │        100.90.158.75:8888           │  hexstrike_server.py │
│  (HexStrike TUI)│  ◄────────────────────────────────  │  Flask + Ollama +Nmap│
└─────────────────┘                                     └──────────────────────┘
     CLIENTE                                                    BACKEND
```

El backend es [HexStrike AI](https://github.com/0x4m4/hexstrike-ai) (0x4m4, v6.0). Aunque orquesta 150+ herramientas, este cliente solo genera comandos de `nmap`, así que es lo único imprescindible para operar este cliente.

> **Cómo leer esta guía.** Cada paso trae al final un bloque **`# ✓ Verificar`** con el comando para confirmar que quedó bien antes de pasar al siguiente. Si la verificación no da lo esperado, no avances.

---

## Requisitos previos

| Componente | Backend (`ia-upy`) | Cliente (Kali) |
|---|---|---|
| SO | Ubuntu 24.04 LTS / Debian (LXC) | Kali Linux |
| Rol | Servidor HexStrike + motor de IA | Operador / TUI |
| Software | git, nmap, python3, venv, Ollama, Tailscale | python3, rich, requests, Tailscale |

Necesitas `root`/`sudo` en ambas máquinas y una cuenta de Tailscale para unir los dos nodos a la misma malla.

---

## Parte A — Backend (`ia-upy`, LXC en Proxmox)

### 1. Actualizar el sistema e instalar dependencias base

`nmap` es el binario que HexStrike invoca; `git` clona el proyecto; `python3-venv` aísla las dependencias.

```bash
apt update && apt upgrade -y
apt install -y git nmap python3 python3-pip python3-venv curl
```

```bash
# ✓ Verificar: deben imprimir una versión, no "command not found"
nmap --version | head -n1
python3 --version
git --version
```

### 2. Instalar y conectar Tailscale

Une el backend a la malla privada. `tailscale up` te da un enlace para autenticar el nodo.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

```bash
# ✓ Verificar: el estado debe decir "Logged in" y darte la IP de la malla
tailscale status
tailscale ip -4          # debe coincidir con la que el cliente tiene fija: 100.90.158.75
```

> **LXC en Proxmox:** si `tailscale up` falla por falta de TUN, en el **host** de Proxmox edita `/etc/pve/lxc/<ID>.conf`, agrega estas líneas y reinicia el contenedor:
> ```
> lxc.cgroup2.devices.allow: c 10:200 rwm
> lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
> ```

### 3. Instalar Ollama y descargar el modelo

Ollama corre la inferencia 100% local. En el LXC sin GPU trabaja en CPU (más lento, pero funcional). **Solo se usa en órdenes de lenguaje libre** — ver la sección final sobre cuándo entra la IA.

```bash
curl -fsSL https://ollama.com/install.sh | sh
systemctl enable --now ollama
ollama pull qwen2.5:7b
```

```bash
# ✓ Verificar: el servicio activo y el modelo en la lista
systemctl is-active ollama          # debe decir: active
ollama list                         # debe aparecer qwen2.5:7b
```

> **Sobre el modelo.** Este proyecto usa `qwen2.5:7b` (Qwen 2.5, 7B parámetros) por su buen desempeño siguiendo instrucciones y generando comandos, con un tamaño que corre en CPU. Si prefieres otro modelo, ajusta el tag en el `pull` y confirma con `ollama list` que coincide **exacto** con el que espera HexStrike.

### 4. Clonar HexStrike AI e instalar sus dependencias

HexStrike trae su propio `requirements.txt` (Flask y demás). Se instala dentro de un entorno virtual para no romper el Python del sistema.

```bash
git clone https://github.com/0x4m4/hexstrike-ai.git
cd hexstrike-ai
python3 -m venv hexstrike-env
source hexstrike-env/bin/activate
pip3 install -r requirements.txt
```

```bash
# ✓ Verificar: el prompt debe traer el prefijo (hexstrike-env) y Flask estar instalado
which python3                       # debe apuntar a .../hexstrike-env/bin/python3
pip3 show flask | grep -i version   # confirma que las dependencias se instalaron
```

### 5. Levantar el servidor HexStrike

Usa el Python del venv directamente para que `nohup` lo mantenga vivo sin dejar la sesión activada. La salida queda en `hexstrike.log`.

```bash
nohup hexstrike-env/bin/python3 hexstrike_server.py --port 8888 > hexstrike.log 2>&1 &
```

Alternativa: para **ver la salida en tiempo real** (en lugar de segundo plano), arráncalo en primer plano en su propia terminal. Con el venv activado:

```bash
source hexstrike-env/bin/activate
python3 hexstrike_server.py --port 8888
```

Déjalo corriendo ahí; abre otra terminal para lo demás. Para detenerlo, `Ctrl+C`.

```bash
# ✓ Verificar: proceso vivo, puerto escuchando y la API responde
pgrep -af hexstrike_server.py       # debe listar el proceso con su PID
ss -tuln | grep 8888                # debe mostrar el puerto en LISTEN
curl http://localhost:8888/health   # Flask debe devolver un JSON de estado
```

> **No actives la API key.** El cliente `demo_ia.py` no envía header de autenticación. Si arrancas con `HEXSTRIKE_REQUIRE_API_KEY=true`, el cliente recibe 401 y falla. Déjalo en el modo por defecto (sin key).

---

## Parte B — Cliente (Kali Linux)

### 1. Instalar y conectar Tailscale

Mismo procedimiento: une Kali a la misma malla.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

```bash
# ✓ Verificar: Kali debe VER al backend por la malla
tailscale status                    # el nodo ia-upy debe aparecer activo
ping -c 3 100.90.158.75             # debe responder con tiempos, no "unreachable"
```

### 2. Instalar dependencias de la TUI

`rich` renderiza las tablas y colores; `requests` habla con la API del backend. Son las únicas librerías externas que importa `demo_ia.py`.

```bash
pip3 install rich requests --break-system-packages
```

```bash
# ✓ Verificar: ambas importan sin error
python3 -c "import rich, requests; print('rich y requests OK')"
```

### 3. Confirmar el endpoint y ejecutar

`demo_ia.py` trae la IP del backend fija en la constante `HEXSTRIKE_URL = "http://100.90.158.75:8888"`. Si tu backend usa otra IP de Tailscale, edítala ahí antes de correr.

```bash
# ✓ Verificar ANTES de abrir la TUI: el backend responde desde Kali
curl http://100.90.158.75:8888/health

# Ahora sí, lanzar la consola
python3 demo_ia.py
```

Ejemplos de órdenes (estas traen palabras clave, así que el cliente arma el `nmap` directo, **sin** Ollama):

```
escanea 100.90.158.75
   -> nmap -sT -F 100.90.158.75

audita versiones 100.90.158.75
   -> nmap -sT -sV -Pn --top-ports 20 100.90.158.75

Audita el objetivo 100.90.158.75 en busca de vulnerabilidades comunes en los servicios web
   -> nmap -sT -p 80,443,8888 --script=http-title 100.90.158.75
```

Esa última frase larga cae en la ruta web porque contiene `audita`, `vulnerabilidades` y `web`; el parser la reconoce y ejecuta el escaneo NSE de servicios web sin invocar al modelo.

Para **invocar a Ollama (Qwen 2.5)** se usa una orden libre, sin esas palabras clave de escaneo. Por ejemplo:

```
Analiza qué riesgos tendría esta máquina si estuviera expuesta a Internet
   -> ruta de IA: t_00--ai_reconnaissance_workflow (procesa el modelo local)
```

Escribe `salir` para cerrar la sesión.

---

## Puesta en marcha

Secuencia para levantar el stack completo tras un reinicio de las máquinas. No reinstala nada — solo inicia los servicios en orden.

**En el BACKEND (`ia-upy`):**

```bash
# 1) Malla y motor de IA arriba
tailscale up
systemctl start ollama

# 2) Levantar el servidor HexStrike (desde la carpeta del proyecto)
cd ~/hexstrike-ai
nohup hexstrike-env/bin/python3 hexstrike_server.py --port 8888 > hexstrike.log 2>&1 &

# 3) Confirmar en tres golpes
tailscale ip -4
ss -tuln | grep 8888
curl http://localhost:8888/health
```

**En el CLIENTE (Kali):**

```bash
# 1) Conectar a la malla y confirmar que ves al backend
sudo tailscale up
curl http://100.90.158.75:8888/health

# 2) Lanzar la TUI
python3 demo_ia.py
```

Si los tres comandos del backend responden y el `/health` contesta desde Kali, el stack está operativo.

---

## Gestión del servicio (reiniciar componentes)

Comandos para reiniciar cada componente de forma independiente **sin reinstalar nada**. Útiles para aplicar cambios de configuración o recuperar un servicio detenido.

**Reiniciar el servidor HexStrike:**

```bash
# Matar la instancia colgada
pkill -f hexstrike_server.py

# Confirmar que el puerto quedó libre (no debe imprimir nada)
ss -tuln | grep 8888

# Volver a levantarlo
cd ~/hexstrike-ai
nohup hexstrike-env/bin/python3 hexstrike_server.py --port 8888 > hexstrike.log 2>&1 &
curl http://localhost:8888/health
```

**Reiniciar Ollama:**

```bash
systemctl restart ollama
ollama ps                # muestra el modelo cargado en memoria
```

**Reconectar Tailscale:**

```bash
sudo tailscale down && sudo tailscale up
tailscale status
```

**Reinicio completo del stack (en orden):**

```bash
# BACKEND
pkill -f hexstrike_server.py
systemctl restart ollama tailscaled
cd ~/hexstrike-ai
nohup hexstrike-env/bin/python3 hexstrike_server.py --port 8888 > hexstrike.log 2>&1 &

# CLIENTE
sudo systemctl restart tailscaled && sudo tailscale up
```

---

## Verificación final: ¿los datos son REALES o el fallback?

El cliente tiene un respaldo: **si el backend devuelve salida vacía, la TUI rellena la tabla con puertos hardcodeados** (`22 ssh`, `80 http`, `8888 http`). Para confirmar que lo que ves es real, consulta la API **por fuera de la TUI**, donde el relleno no aparece.

```bash
# 1) Lanzar un escaneo real directo a la API (desde el backend o desde Kali)
TASK=$(curl -s -X POST http://100.90.158.75:8888/api/process/execute-async \
  -H "Content-Type: application/json" \
  -d '{"command": "nmap -sT -F 100.90.158.75", "args": []}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['task_id'])")

echo "task_id = $TASK"

# 2) Pedir el resultado crudo (espera unos segundos y repite si sale vacío)
curl -s http://100.90.158.75:8888/api/process/get-task-result/$TASK
```

Si el segundo comando devuelve la salida del `nmap` con puertos, **son datos reales**. Si devuelve vacío, lo que la TUI muestre es el relleno.

Como apoyo, en el backend deja corriendo el log en otra terminal para ver el `nmap` ejecutándose en tiempo real:

```bash
tail -f hexstrike.log
```

---

## Solución de problemas

**La TUI muestra siempre los mismos tres puertos.**
Es el respaldo defensivo: el backend no devolvió salida. Revisa `hexstrike.log`, confirma que el server está arriba (`curl .../health`) y que no activaste la API key.

**El escaneo falla dentro del LXC.**
El cliente usa **TCP Connect (`-sT`)** a propósito, porque los *raw sockets* del SYN scan (`-sS`) suelen estar bloqueados en contenedores sin privilegios. Mantén `-sT` si tocas el tipo de escaneo.

**El cliente no llega al backend.**
Casi siempre es Tailscale. Revisa `tailscale status` en ambos lados; los dos nodos deben verse activos. Si uno dice `offline`, vuelve a correr `tailscale up`.

**Error 401 / conexión rechazada.**
Tienes la API key activada en el server. Reinícialo sin `HEXSTRIKE_REQUIRE_API_KEY`, o agrega el header en el cliente.

**Ollama tarda mucho.**
En CPU la primera inferencia es lenta mientras carga el modelo en RAM. Confirma el servicio con `systemctl status ollama`. Recuerda que solo se usa en órdenes de lenguaje libre.

---

## Cómo enruta las órdenes el cliente

`demo_ia.py` decide la ruta **antes** de llamar a la IA:

- Si la orden trae palabras clave de escaneo (`escanea`, `nmap`, `puertos`, `versiones`, `vuln`, `web`…), el propio cliente arma el comando `nmap -sT ...` y lo manda al endpoint `/api/process/execute-async`. **No pasa por Ollama.**
- Solo si la orden es lenguaje libre sin esas palabras, el cliente delega en el workflow de IA (`t_00--ai_reconnaissance_workflow`), y ahí sí entra Qwen 2.5 vía Ollama.

Para dirigir la orden al modelo, usa una descripción en lenguaje libre sin las palabras clave de escaneo.

---

## Nota de OpSec

Todo el tráfico entre cliente y backend va cifrado punto a punto por Tailscale (WireGuard). El backend **nunca** expone el puerto 8888 a Internet: solo es accesible dentro de la malla. No abras ese puerto en el firewall público ni le hagas port-forward — rompe todo el modelo de aislamiento.

---

*UPY · Coordinación de Ingeniería en Ciberseguridad*
