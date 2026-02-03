# Docker-Status-Buttons (GNOME + Ubuntu 24)

> **Guía 100% manual, transparente y audit-friendly** para crear:
>
> 1) Un **toggle tipo “pill”** en **Quick Settings** para **encender/apagar TODO Docker**:
>    - Docker Engine del host (`docker.service`, `docker.socket`, `containerd.service`)
>    - Docker Desktop (servicio de usuario `docker-desktop.service`)
> 2) Un **indicador siempre visible** en la barra superior (top bar) con **Executor**:
>    - `🐳 DOCKER: ON` / `🐳 DOCKER: OFF`
>
> ✅ Filosofía: **control total**, **cero magia**, **código corto**, **sin descargas automáticas**, **sin `curl | bash`**.

---

## Tabla de contenidos

- [Filosofía](#filosofía)
- [Qué resuelve y por qué](#qué-resuelve-y-por-qué)
- [Requisitos](#requisitos)
- [Probado en](#probado-en)
- [Estructura del repo](#estructura-del-repo)
- [Guía paso a paso (manual)](#guía-paso-a-paso-manual)
  - [Paso 0 — Preparación](#paso-0--preparación)
  - [Paso 1 — Crear scripts](#paso-1--crear-scripts)
  - [Paso 2 — Probar desde terminal](#paso-2--probar-desde-terminal)
  - [Paso 3 — Instalar extensiones GNOME](#paso-3--instalar-extensiones-gnome)
  - [Paso 4 — Configurar el toggle](#paso-4--configurar-el-toggle)
  - [Paso 5 — Configurar el indicador siempre visible](#paso-5--configurar-el-indicador-siempre-visible)
  - [Paso 6 — Verificación](#paso-6--verificación)
- [Modelo de seguridad](#modelo-de-seguridad)
- [Desinstalación](#desinstalación)
- [Troubleshooting](#troubleshooting)

---

## Filosofía

Este repo está diseñado para que cualquiera pueda replicar el resultado con **control absoluto** y confianza:

- ✅ Vos creás cada archivo y pegás su contenido (sin automatizaciones opacas)
- ✅ Scripts cortos, legibles y fáciles de auditar
- ✅ Nada descarga código automáticamente
- ✅ No existe `curl | bash`
- ✅ Se usan solo herramientas estándar: `systemctl`, `systemctl --user` y `pkexec`

---

## Qué resuelve y por qué

En Ubuntu podés tener **dos “Dockers”** distintos corriendo:

1) **Docker Engine (host)** como servicios systemd:
   - `docker.service`
   - `docker.socket` (⚠️ si queda activo, systemd puede reactivar Docker automáticamente)
   - `containerd.service`

2) **Docker Desktop** (Linux) como servicio de usuario (stack/VM separado):
   - `docker-desktop.service`

Si usás tu PC para juegos, querés:
- Docker **OFF** cuando jugás (0 procesos/servicios consumiendo recursos)
- Docker **ON** solo cuando lo necesitás para dev

Este setup agrega:
- Un **toggle** para ON/OFF total desde GUI (Quick Settings)
- Un **status visible siempre** para saber si quedó algo prendido

---

## Requisitos

- Ubuntu 24.x con GNOME
- Docker Engine instalado
- Docker Desktop instalado (opcional, pero soportado)
- Permisos de administrador para start/stop del Engine (Polkit prompt via `pkexec`)
- Extensiones GNOME:
  - **Custom Command Toggle** (toggle pill en Quick Settings):  
    https://extensions.gnome.org/extension/7012/custom-command-toggle/
  - **Executor** (texto visible siempre en top bar):  
    https://extensions.gnome.org/extension/2932/executor/

---

## Probado en

- Ubuntu 24.x + GNOME (ej.: GNOME 46)
- Funciona tanto en X11 como en Wayland (control de systemd)
- Docker Engine + Docker Desktop instalados (aunque podés usar solo uno)

---

## Estructura del repo

Este repo contiene **plantillas de scripts** y documentación. Los archivos reales se crean en tu HOME:

**Archivos locales (tu PC):**
- `~/.local/bin/docker-all-status`
- `~/.local/bin/docker-all-on`
- `~/.local/bin/docker-all-off`
- `~/.config/executor/docker-status.sh`

**Archivos en el repo (referencia):**
- `scripts/` (plantillas)
- `executor/` (wrapper)

---

# Guía paso a paso (manual)

## Paso 0 — Preparación

Para que Docker **no se encienda solo**:

### 0.1 Deshabilitar autostart de Docker Engine (host)
```bash
sudo systemctl disable docker docker.socket containerd
```

### 0.2 Deshabilitar autostart de Docker Desktop (user service)
```bash
systemctl --user disable docker-desktop
```

### 0.3 (Opcional) Apagar todo ahora mismo
```bash
systemctl --user stop docker-desktop
sudo systemctl stop docker docker.socket containerd
```

---

## Paso 1 — Crear scripts

### 1.1 Crear carpetas
```bash
mkdir -p ~/.local/bin
mkdir -p ~/.config/executor
```

---

### 1.2 Crear `~/.local/bin/docker-all-status`

**Qué hace:** detecta si hay algo de Docker activo (Engine host o Desktop) y muestra:
- `🐳 DOCKER: ON`
- `🐳 DOCKER: OFF`

1) Abrí el archivo:
```bash
nano ~/.local/bin/docker-all-status
```

2) Pegá este contenido:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Detecta si está activo Docker Engine (host) o Docker Desktop (user).
is_on=0

for unit in docker.service docker.socket containerd.service; do
  if /usr/bin/systemctl is-active --quiet "$unit"; then
    is_on=1
  fi
done

if /usr/bin/systemctl --user is-active --quiet docker-desktop.service 2>/dev/null; then
  is_on=1
fi

if [ "$is_on" -eq 1 ]; then
  echo "🐳 DOCKER: ON"
else
  echo "🐳 DOCKER: OFF"
fi
```

3) Guardá y salí (nano: `Ctrl+O`, Enter, `Ctrl+X`)

---

### 1.3 Crear `~/.local/bin/docker-all-on`

**Qué hace:** enciende el Engine (host) y Docker Desktop (si existe).

1) Abrí el archivo:
```bash
nano ~/.local/bin/docker-all-on
```

2) Pegá este contenido:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Enciende Docker Engine (host). Requiere auth vía Polkit (pkexec).
/usr/bin/pkexec /usr/bin/systemctl start containerd.service docker.socket docker.service

# Enciende Docker Desktop (servicio de usuario), si existe.
# Si no está instalado o no existe el unit, no rompe.
 /usr/bin/systemctl --user start docker-desktop.service 2>/dev/null || true
```

3) Guardá y salí

---

### 1.4 Crear `~/.local/bin/docker-all-off`

**Qué hace:** apaga Desktop primero y luego Engine + socket + containerd.

⚠️ Importante: apagamos también `docker.socket` porque si queda activo, systemd puede reactivar Docker ante el primer acceso al socket.

1) Abrí el archivo:
```bash
nano ~/.local/bin/docker-all-off
```

2) Pegá este contenido:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Apaga Docker Desktop primero (VM / stack propio).
/usr/bin/systemctl --user stop docker-desktop.service 2>/dev/null || true

# Apaga Docker Engine (host) + socket + containerd.
# Importante: si dejás docker.socket activo, puede reactivar dockerd.
 /usr/bin/pkexec /usr/bin/systemctl stop docker.service docker.socket containerd.service
```

3) Guardá y salí

---

### 1.5 Crear wrapper para Executor: `~/.config/executor/docker-status.sh`

**Qué hace:** Executor ejecuta este wrapper y muestra el resultado en la top bar.

1) Abrí el archivo:
```bash
nano ~/.config/executor/docker-status.sh
```

2) Pegá este contenido:
```bash
#!/usr/bin/env bash
set -euo pipefail
"$HOME/.local/bin/docker-all-status"
```

3) Guardá y salí

---

### 1.6 Dar permisos de ejecución
```bash
chmod +x ~/.local/bin/docker-all-status ~/.local/bin/docker-all-on ~/.local/bin/docker-all-off
chmod +x ~/.config/executor/docker-status.sh
```

---

## Paso 2 — Probar desde terminal

Estado:
```bash
~/.local/bin/docker-all-status
```

Encender (te va a pedir autorización):
```bash
~/.local/bin/docker-all-on
```

Verificar:
```bash
~/.local/bin/docker-all-status
```

Apagar (te va a pedir autorización):
```bash
~/.local/bin/docker-all-off
```

Verificar:
```bash
~/.local/bin/docker-all-status
```

---

## Paso 3 — Instalar extensiones GNOME

### Opción A (recomendada): Extension Manager
Instalá la herramienta:
```bash
sudo apt update
sudo apt install -y gnome-shell-extension-manager
```

Abrí **Extension Manager** y buscá/instalá:
- **Custom Command Toggle**
- **Executor**

### Opción B: GNOME Extensions (web)
- Custom Command Toggle:
  https://extensions.gnome.org/extension/7012/custom-command-toggle/
- Executor:
  https://extensions.gnome.org/extension/2932/executor/

---

## Paso 4 — Configurar el toggle

Abrí preferencias:
```bash
gnome-extensions prefs custom-command-toggle@storageb.github.com
```

Creá un toggle con (usar rutas absolutas):

- **Button name:** `Docker-Status`

- **Toggle ON command:**
  - `/home/TU_USUARIO/.local/bin/docker-all-on`

- **Toggle OFF command:**
  - `/home/TU_USUARIO/.local/bin/docker-all-off`

- **Check Status command:**
  - `/home/TU_USUARIO/.local/bin/docker-all-status`

- **Search term:** `ON`

Activá si está disponible:
- “Keep toggle state synced”
- “Check command exit code”
- “Initial state from command output”

> Nota: muchas extensiones no expanden `~`. Por eso se usan rutas absolutas `/home/TU_USUARIO/...`.

El toggle aparece en **Quick Settings** (panel superior derecho: Wi-Fi/sonido/batería).

---

## Paso 5 — Configurar el indicador siempre visible

Abrí preferencias:
```bash
gnome-extensions prefs executor@raujonas.github.io
```

En Executor, agregá un comando que ejecute:
- `~/.config/executor/docker-status.sh`

Recomendado:
- Intervalo: 5–10 segundos
- Posición: Center (o donde prefieras)

Resultado esperado en top bar:
- `🐳 DOCKER: OFF` / `🐳 DOCKER: ON`

---

## Paso 6 — Verificación

### 6.1 Servicios del host
```bash
systemctl is-active docker.service docker.socket containerd.service
```
OFF: deberían ser `inactive`.

### 6.2 Servicio de usuario (Desktop)
```bash
systemctl --user is-active docker-desktop.service
```
OFF: `inactive`

### 6.3 Procesos (OFF = no debería mostrar nada)
```bash
ps -eo pid,comm,args | grep -E "dockerd|containerd|docker-desktop|qemu|vpnkit" | grep -v grep
```

### 6.4 Docker CLI (OFF = debería fallar)
```bash
docker info
```

---

## Modelo de seguridad

**Qué hace:**
- Inicia/detiene servicios con `systemctl`
- Inicia/detiene servicios de usuario con `systemctl --user`
- Usa `pkexec systemctl ...` para operaciones que requieren permisos de admin

**Qué NO hace:**
- No descarga nada
- No ejecuta código remoto
- No modifica archivos del sistema
- No agrega servicios persistentes
- No toca tu configuración de Docker (solo prende/apaga)

**Por qué se usa `pkexec`:**
- Es un mecanismo estándar de Polkit para pedir autorización de forma explícita y segura.

---

## Desinstalación

### 1) Apagar todo
```bash
systemctl --user stop docker-desktop
sudo systemctl stop docker docker.socket containerd
```

### 2) Eliminar scripts y wrapper
```bash
rm -f ~/.local/bin/docker-all-status ~/.local/bin/docker-all-on ~/.local/bin/docker-all-off
rm -f ~/.config/executor/docker-status.sh
```

### 3) (Opcional) Deshabilitar extensiones
Desde Extension Manager o:
```bash
gnome-extensions disable custom-command-toggle@storageb.github.com
gnome-extensions disable executor@raujonas.github.io
```

### 4) (Opcional) Volver a habilitar autostart (comportamiento original)
```bash
sudo systemctl enable docker docker.socket containerd
systemctl --user enable docker-desktop
```

---

## Troubleshooting

### No veo el toggle
- Está dentro de **Quick Settings**, no es un botón fijo en la barra superior.
- Probá logout/login para refrescar GNOME Shell.

### El toggle no ejecuta nada
- Verificá permisos:
```bash
ls -la ~/.local/bin/docker-all-*
```
Deben ser ejecutables: `-rwxr-xr-x`

- Usá rutas absolutas en la configuración del toggle (no `~`).

### Executor no muestra el emoji 🐳
Instalá fuente de emojis:
```bash
sudo apt update
sudo apt install -y fonts-noto-color-emoji
```
Luego logout/login.

### Docker se vuelve a encender “solo”
- Asegurate de haber deshabilitado `docker.socket`:
```bash
systemctl is-enabled docker docker.socket containerd
```
Debería decir `disabled` para los tres.
