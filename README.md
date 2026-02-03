# Docker-Status-Buttons (GNOME + Ubuntu 24)

> **Repo público, transparente y audit-friendly** para crear:
>
> 1) Un **toggle tipo “pill”** en **Quick Settings** (panel superior derecho) para **encender/apagar TODO Docker**:
>    - Docker Engine del host (`docker.service`, `docker.socket`, `containerd.service`)
>    - Docker Desktop (servicio de usuario `docker-desktop.service`)
> 2) Un **indicador siempre visible** en la barra superior (top bar) con **Executor** que muestre:
>    - `🐳 DOCKER: ON` / `🐳 DOCKER: OFF`
>
> ✅ Filosofía: **control total**, **cero magia**, **código corto**, **sin descargas automáticas**, **sin `curl | bash`**.

---

## Filosofía: control total y transparencia

Este repo está diseñado para que **cualquiera pueda replicarlo manualmente** y entender exactamente qué pasa:

- ✅ Vos creás cada archivo y pegás su contenido
- ✅ Scripts cortos, fáciles de auditar
- ✅ No hay instalaciones automáticas desde Internet
- ✅ No hay “comandos mágicos”
- ✅ Solo se usan comandos estándar: `systemctl`, `systemctl --user` y `pkexec`

Esto reduce al mínimo la superficie de riesgo y aumenta la confianza para uso personal y para terceros.

---

## ¿Por qué “apagar Docker por completo”?

En Ubuntu podés tener dos “Dockers” distintos corriendo:

1) **Docker Engine (host)** como servicios systemd:
   - `docker.service`
   - `docker.socket`  ⚠️ si queda activo, systemd puede reactivar Docker automáticamente
   - `containerd.service`

2) **Docker Desktop en Linux** (normalmente VM/stack separado) como servicio de usuario:
   - `docker-desktop.service`

Si tu PC también es para juegos, lo ideal es:
- Docker **OFF** (0 recursos consumidos por Docker) cuando jugás
- Docker **ON** solo cuando lo necesitás (dev / tests / entornos locales)

---

## Requisitos

- Ubuntu 24.x con GNOME
- Docker Engine instalado
- Docker Desktop instalado (opcional, pero soportado por esta guía)
- Permisos de administrador (se usa `pkexec` para start/stop del Engine)
- Extensiones GNOME:
  - **Custom Command Toggle** (toggle pill en Quick Settings):
    - https://extensions.gnome.org/extension/7012/custom-command-toggle/
  - **Executor** (texto visible siempre en top bar):
    - https://extensions.gnome.org/extension/2932/executor/

---

# Guía completa (Manual, paso a paso)

> En esta guía vas a:
> 1) Deshabilitar autostart (para que Docker no se prenda solo)
> 2) Crear scripts locales y ejecutables
> 3) Instalar extensiones GNOME manualmente
> 4) Configurar el toggle (Quick Settings)
> 5) Configurar el status siempre visible (Executor)
> 6) Verificar con comandos “de auditoría” que Docker está realmente OFF

---

## Paso 0 — Preparación (recomendado)

Para que Docker **no se encienda solo** al iniciar el sistema / sesión, deshabilitá autostart.

### 0.1 Deshabilitar autostart de Docker Engine (host)
~~~bash
sudo systemctl disable docker docker.socket containerd
~~~

### 0.2 Deshabilitar autostart de Docker Desktop (user service)
~~~bash
systemctl --user disable docker-desktop
~~~

### 0.3 (Opcional) Apagar todo ahora mismo
~~~bash
systemctl --user stop docker-desktop
sudo systemctl stop docker docker.socket containerd
~~~

> A partir de acá, lo vas a encender/apagar cuando quieras con el toggle.

---

## Paso 1 — Crear scripts localmente 

Estos scripts viven en tu HOME y son 100% auditables.

### 1.1 Crear carpetas
~~~bash
mkdir -p ~/.local/bin
mkdir -p ~/.config/executor
~~~

---

## Paso 2 — Crear los archivos (uno por uno) y pegar contenido

Podés usar `nano`, `gedit`, `code`, `notepadqq`, etc.  
Ejemplo con `nano` :

---

### 2.1 Crear `~/.local/bin/docker-all-status`

**Qué hace:** detecta si hay algo de Docker activo (Engine host o Desktop) y muestra:
- `🐳 DOCKER: ON` si algo está activo
- `🐳 DOCKER: OFF` si está todo apagado

1) Abrí el archivo:
~~~bash
nano ~/.local/bin/docker-all-status
~~~

2) Pegá este contenido:
~~~bash
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
~~~

3) Guardá y salí (en nano: `Ctrl+O`, Enter, `Ctrl+X`)

---

### 2.2 Crear `~/.local/bin/docker-all-on`

**Qué hace:** enciende TODO:
- Docker Engine del host (requiere autorización, usa `pkexec`)
- Docker Desktop (si existe el servicio de usuario)

1) Abrí el archivo:
~~~bash
nano ~/.local/bin/docker-all-on
~~~

2) Pegá este contenido:
~~~bash
#!/usr/bin/env bash
set -euo pipefail

# Enciende Docker Engine (host). Requiere auth vía Polkit (pkexec).
/usr/bin/pkexec /usr/bin/systemctl start containerd.service docker.socket docker.service

# Enciende Docker Desktop (servicio de usuario), si existe.
# Si no está instalado o no existe el unit, no rompe.
 /usr/bin/systemctl --user start docker-desktop.service 2>/dev/null || true
~~~

3) Guardá y salí

---

### 2.3 Crear `~/.local/bin/docker-all-off`

**Qué hace:** apaga TODO:
- Docker Desktop primero
- Docker Engine + docker.socket + containerd después

⚠️ Importante: apagamos también `docker.socket` porque si queda activo, systemd puede reactivar Docker ante el primer acceso al socket.

1) Abrí el archivo:
~~~bash
nano ~/.local/bin/docker-all-off
~~~

2) Pegá este contenido:
~~~bash
#!/usr/bin/env bash
set -euo pipefail

# Apaga Docker Desktop primero (VM / stack propio).
/usr/bin/systemctl --user stop docker-desktop.service 2>/dev/null || true

# Apaga Docker Engine (host) + socket + containerd.
# Importante: si dejás docker.socket activo, puede reactivar dockerd.
 /usr/bin/pkexec /usr/bin/systemctl stop docker.service docker.socket containerd.service
~~~

3) Guardá y salí

---

### 2.4 Crear wrapper para Executor: `~/.config/executor/docker-status.sh`

**Qué hace:** Executor ejecuta este wrapper y muestra en la top bar lo que imprima el script de status.

1) Abrí el archivo:
~~~bash
nano ~/.config/executor/docker-status.sh
~~~

2) Pegá este contenido:
~~~bash
#!/usr/bin/env bash
set -euo pipefail
"$HOME/.local/bin/docker-all-status"
~~~

3) Guardá y salí

---

### 2.5 Dar permisos de ejecución (obligatorio)
~~~bash
chmod +x ~/.local/bin/docker-all-status ~/.local/bin/docker-all-on ~/.local/bin/docker-all-off
chmod +x ~/.config/executor/docker-status.sh
~~~

---

## Paso 3 — Pruebas rápidas en terminal (sanity check)

### 3.1 Ver estado actual
~~~bash
~/.local/bin/docker-all-status
~~~

### 3.2 Encender todo (te va a pedir autorización)
~~~bash
~/.local/bin/docker-all-on
~~~

### 3.3 Verificar ON
~~~bash
~/.local/bin/docker-all-status
~~~

### 3.4 Apagar todo (te va a pedir autorización)
~~~bash
~/.local/bin/docker-all-off
~~~

### 3.5 Verificar OFF
~~~bash
~/.local/bin/docker-all-status
~~~

---

## Paso 4 — Instalar extensiones GNOME (manual)

Tenés dos opciones:

### Opción A (recomendada): Extension Manager (GUI)
1) Instalá Extension Manager:
~~~bash
sudo apt update
sudo apt install -y gnome-shell-extension-manager
~~~

2) Abrí **Extension Manager** (Actividades → “Extension Manager”)

3) En la pestaña “Browse”, buscá e instalá:
- `Custom Command Toggle`
- `Executor`

---

### Opción B: GNOME Extensions web
1) Entrá a https://extensions.gnome.org/
2) Instalá el soporte del navegador si te lo pide
3) Instalá:
- Custom Command Toggle: https://extensions.gnome.org/extension/7012/custom-command-toggle/
- Executor: https://extensions.gnome.org/extension/2932/executor/

---

## Paso 5 — Configurar el toggle pill (Custom Command Toggle)

### 5.1 Abrir configuración
Desde Extension Manager o por terminal:
~~~bash
gnome-extensions prefs custom-command-toggle@storageb.github.com
~~~

### 5.2 Crear el toggle “Docker-Status”

Completar así (MUY IMPORTANTE: usar rutas absolutas):

- **Button name:** `Docker-Status`

- **Toggle ON command:**
  - `/home/TU_USUARIO/.local/bin/docker-all-on`

- **Toggle OFF command:**
  - `/home/TU_USUARIO/.local/bin/docker-all-off`

- **Check Status command:**
  - `/home/TU_USUARIO/.local/bin/docker-all-status`

- **Search term:** `ON`

Si tu UI ofrece estas opciones, activarlas:
- “Keep toggle state synced”
- “Check command exit code”
- “Initial state from command output” (o equivalente)

⚠️ Nota: muchas extensiones no expanden `~`. Por eso **NO** uses `~/.local/bin/...` aquí.

### 5.3 Dónde aparece el toggle
Abrí **Quick Settings** (click arriba a la derecha: Wi-Fi/sonido/batería)  
Ahí debe aparecer `Docker-Status`.

---

## Paso 6 — Configurar el estado visible siempre (Executor)

### 6.1 Abrir configuración
Desde Extension Manager o por terminal:
~~~bash
gnome-extensions prefs executor@raujonas.github.io
~~~

### 6.2 Agregar comando
En Executor, agregá un comando que ejecute:
- `~/.config/executor/docker-status.sh`

Configuración recomendada:
- **Intervalo:** 5–10 segundos
- **Posición:** Center (o donde prefieras)

### 6.3 Resultado esperado
En la barra superior deberías ver:
- `🐳 DOCKER: OFF` cuando esté apagado
- `🐳 DOCKER: ON` cuando esté encendido

---

## Verificación (comandos “de auditoría”)

Estos comandos sirven para confirmar que el toggle realmente apagó TODO.

### 1) Servicios del host (Engine + socket + containerd)
~~~bash
systemctl is-active docker.service docker.socket containerd.service
~~~
Apagado: deberían ser `inactive`.

### 2) Docker Desktop (servicio de usuario)
~~~bash
systemctl --user is-active docker-desktop.service
~~~
Apagado: `inactive`.

### 3) Procesos que NO deberían existir cuando está OFF
~~~bash
ps -eo pid,comm,args | grep -E "dockerd|containerd|docker-desktop|qemu|vpnkit" | grep -v grep
~~~
Apagado: no debería imprimir nada.

### 4) Prueba rápida: con Docker OFF, la CLI debería fallar
~~~bash
docker info
~~~

---

## Troubleshooting

### No veo el toggle
- Está dentro de **Quick Settings** (no es un botón permanente).
- Probá cerrar sesión y volver a entrar.

### El toggle no ejecuta nada / no cambia el estado
- Verificá permisos:
  ~~~bash
  ls -la ~/.local/bin/docker-all-*
  ~~~
  Deben ser `-rwxr-xr-x`.
- Usá rutas absolutas en la configuración del toggle.

### Executor muestra “ON/OFF” pero no muestra el emoji 🐳
Instalá fuente de emojis:
~~~bash
sudo apt update
sudo apt install -y fonts-noto-color-emoji
~~~
Luego logout/login.

---

## Seguridad (qué hace y qué NO hace)

✅ Hace:
- `systemctl start/stop ...`
- `systemctl --user start/stop ...`
- `pkexec systemctl ...` (pide autorización)

❌ No hace:
- descargas automáticas
- ejecución remota
- modificaciones permanentes del sistema
- cambios fuera de tu HOME (salvo iniciar/detener servicios)
