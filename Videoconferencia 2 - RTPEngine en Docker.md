# VIDEOCONFERENCIA 2
## RTPEngine en Docker - Multi-instancia y Configuración Avanzada

---

## 📋 CÓMO USAR ESTE MATERIAL

### Estructura del documento

Este material está organizado siguiendo el **orden lógico de ejecución**. Cada paso depende del anterior:

```
1. Introducción y arquitectura (teoría)
2. Preparar el HOST (cargar módulo kernel) ← CRÍTICO
3. Crear estructura de directorios
4. Crear archivos de configuración
5. Crear Dockerfile y scripts
6. Construir imágenes Docker
7. Crear Docker Compose
8. Levantar servicios
9. Verificar y testear
```

### ⚠️ IMPORTANTE: Orden de ejecución

**NO puedes saltar pasos.** RTPEngine en Docker tiene requisitos específicos:

1. El **módulo kernel debe cargarse ANTES** de crear contenedores
2. Los **archivos de configuración** deben existir ANTES del build
3. Las **imágenes** deben construirse ANTES del compose

### 🎯 Objetivo de esta sesión

Al finalizar podrás:
- Comprender por qué RTPEngine necesita módulo kernel en el host
- Desplegar múltiples instancias RTPEngine containerizadas
- Integrar RTPEngine con Kamailio en Docker
- Balancear carga entre instancias
- Troubleshoot problemas comunes

---

## PARTE 1: RTPENGINE EN DOCKER - INTRODUCCIÓN (10 minutos)

### 1.1 ¿Por qué RTPEngine en contenedores?

**Ventajas específicas para RTPEngine:**

```
✅ AISLAMIENTO: Múltiples instancias sin conflictos de puertos/tables
✅ ESCALABILIDAD: Agregar/quitar instancias según carga
✅ TESTING: Probar diferentes versiones sin afectar producción
✅ PORTABILIDAD: Mismo setup en dev y producción
✅ UPDATES: Actualizar sin downtime (rolling updates)
✅ RECURSOS: Limitar CPU/RAM por instancia
```

**Consideraciones importantes:**

```
⚠️ KERNEL MODULES:
   - xt_RTPENGINE debe cargarse en el HOST (no en el contenedor)
   - No puede cargarse desde dentro del contenedor
   - Cada instancia necesita un table ID único
   - El módulo es compartido por todas las instancias

⚠️ NETWORKING:
   - Host mode recomendado para mejor rendimiento
   - Bridge mode requiere mapping de rangos UDP amplios
   - Cada instancia necesita su propio puerto de control

⚠️ RENDIMIENTO:
   - Docker añade overhead ~1-2% en media processing
   - Host mode minimiza overhead
   - Para 1000+ sesiones simultáneas, considerar bare metal
```

### 1.2 Arquitectura RTPEngine Docker

```
┌─────────────────────────────────────────────────────┐
│                    DOCKER HOST                      │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │  Kernel Module: xt_RTPENGINE               │     │
│  │  (cargado en host, compartido)             │     │
│  │  /proc/rtpengine/control                   │     │
│  └────────────────────────────────────────────┘     │
│                     ▲                               │
│                     │ (acceso compartido)           │
│  ┌──────────────┐  │  ┌──────────────┐  ┌────────┐  │
│  │ RTPEngine #1 ├──┘  │ RTPEngine #2 ├──┤RTPEng#3│  │
│  │ Table ID: 0  │     │ Table ID: 1  │  │Table: 2│  │
│  │ Port: 22222  │     │ Port: 22223  │  │Port:224│  │
│  │ RTP:10k-15k  │     │ RTP:15k-20k  │  │RTP:20-2│  │
│  └──────────────┘     └──────────────┘  └────────┘  │
│         ▲                   ▲                ▲      │
│         │                   │                │      │
│         └───────────────────┴────────────────┘      │
│                             │                       │
│                      ┌──────────────┐               │
│                      │  Kamailio    │               │
│                      │  Dispatcher  │               │
│                      └──────────────┘               │
└─────────────────────────────────────────────────────┘
```

**Flujo de datos:**
1. Kamailio recibe solicitud SIP con SDP
2. Kamailio contacta RTPEngine vía puerto control (22222/22223/22224)
3. RTPEngine procesa media (RTP) usando el módulo kernel
4. El módulo kernel reescribe paquetes en espacio de kernel (más rápido)

---

## PARTE 2: PREPARACIÓN - ESTRUCTURA DE DIRECTORIOS (5 minutos)

### 2.1 Crear estructura del proyecto

**IMPORTANTE: Hacer esto ANTES de crear Dockerfiles**

```bash
# Crear estructura completa
mkdir -p ~/rtpengine-docker/{config,logs,scripts,recordings}
cd ~/rtpengine-docker

# Crear subdirectorios para logs de cada instancia
mkdir -p logs/{rtpengine-1,rtpengine-2,rtpengine-3}

# Verificar estructura
tree -L 2
```

**Estructura esperada:**
```
rtpengine-docker/
├── config/              # Archivos de configuración
│   └── rtpengine.conf
├── logs/                # Logs persistentes
│   ├── rtpengine-1/
│   ├── rtpengine-2/
│   └── rtpengine-3/
├── scripts/             # Scripts de utilidad
│   ├── load-rtpengine-module.sh
│   └── check-kernel-module.sh
├── recordings/          # Grabaciones de llamadas
├── Dockerfile.rtpengine
├── docker-entrypoint.sh
├── docker-compose.yml
└── .env
```

---

## PARTE 3: PREPARAR EL HOST - MÓDULO KERNEL (15 minutos)

### ⚠️ CRÍTICO: Este paso es OBLIGATORIO antes de crear contenedores

El módulo `xt_RTPENGINE` debe cargarse en el **HOST**, no en los contenedores. Sin este módulo, RTPEngine funcionará en modo user-space (más lento).

### 3.1 Instalar dependencias en el host

```bash
# Instalar kernel headers (necesarios para compilar/cargar módulos)
sudo dnf install -y kernel-devel kernel-headers kernel-devel-$(uname -r)

# Instalar herramientas de compilación
sudo dnf install -y gcc make elfutils-libelf-devel

# Verificar versión del kernel
uname -r

# Entrar en la carpeta de trabajo
cd rtpengine-docker
```

### 3.2 Script para cargar el módulo

**Archivo: `scripts/load-rtpengine-module.sh`**

```bash
#!/bin/bash

# Colores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}=== RTPEngine Kernel Module Loader ===${NC}"

# Verificar si ya está cargado
if lsmod | grep -q xt_RTPENGINE; then
    echo -e "${YELLOW}Module xt_RTPENGINE already loaded${NC}"
    lsmod | grep xt_RTPENGINE
    echo -e "\n${GREEN}/proc/rtpengine status:${NC}"
    ls -la /proc/rtpengine/ 2>/dev/null || echo "Directory not found"
    exit 0
fi

echo "Loading kernel module..."

# Opción 1: Si el módulo está disponible en el sistema
if modprobe xt_RTPENGINE 2>/dev/null; then
    echo -e "${GREEN}Module loaded from system${NC}"
else
    echo -e "${YELLOW}Module not found in system, will compile from source${NC}"
    
    # Descargar y compilar el módulo
    cd /tmp
    git clone https://github.com/sipwise/rtpengine.git
    cd rtpengine/kernel-module
    
    echo "Compiling kernel module..."
    make
    
    if [ $? -eq 0 ]; then
        echo "Installing module..."
        insmod xt_RTPENGINE.ko
        
        if [ $? -eq 0 ]; then
            echo -e "${GREEN}Module compiled and loaded successfully${NC}"
        else
            echo -e "${RED}Failed to load module${NC}"
            exit 1
        fi
    else
        echo -e "${RED}Failed to compile module${NC}"
        exit 1
    fi
fi

# Verificar que /proc/rtpengine existe
echo -e "\n${GREEN}Verifying /proc/rtpengine:${NC}"
if [ -d /proc/rtpengine ]; then
    echo -e "${GREEN}✓ /proc/rtpengine directory exists${NC}"
    ls -la /proc/rtpengine/
else
    echo -e "${RED}✗ /proc/rtpengine NOT found${NC}"
    echo "This is required for RTPEngine to work properly!"
    exit 1
fi

# Mostrar reglas iptables de RTPEngine
echo -e "\n${GREEN}Current iptables rules for RTPEngine:${NC}"
iptables -t mangle -L RTPENGINE 2>/dev/null || echo "No RTPENGINE chain yet (will be created by rtpengine daemon)"

echo -e "\n${GREEN}Module setup complete!${NC}"
echo "You can now start RTPEngine containers."
```

### 3.3 Cargar el módulo

```bash
# Hacer ejecutable
chmod +x scripts/load-rtpengine-module.sh

# Ejecutar (requiere sudo)
sudo ./scripts/load-rtpengine-module.sh

# Verificar
lsmod | grep xt_RTPENGINE
ls -la /proc/rtpengine/
```

**Salida esperada:**
```
xt_RTPENGINE           20480  0
✓ /proc/rtpengine directory exists
total 0
dr-xr-xr-x.  2 root root 0 Nov 30 10:00 .
dr-xr-xr-x. XX root root 0 Nov 30 09:00 ..
-rw-r--r--.  1 root root 0 Nov 30 10:00 control
```

### 3.4 Persistir el módulo en reinicio

Para que el módulo se cargue automáticamente al reiniciar:

**Archivo: `/etc/modules-load.d/rtpengine.conf`**

```bash
# Crear archivo
sudo bash -c 'echo "xt_RTPENGINE" > /etc/modules-load.d/rtpengine.conf'

# Verificar
cat /etc/modules-load.d/rtpengine.conf
```

### 3.5 Script de verificación

**Archivo: `scripts/check-kernel-module.sh`**

```bash
#!/bin/bash

echo "=== RTPEngine Kernel Module Status ==="

# 1. Verificar módulo kernel
echo -e "\n1. Kernel Module:"
if lsmod | grep -q xt_RTPENGINE; then
    echo "✓ xt_RTPENGINE is loaded"
    lsmod | grep xt_RTPENGINE
else
    echo "✗ xt_RTPENGINE is NOT loaded"
    echo "Run: sudo ./scripts/load-rtpengine-module.sh"
    exit 1
fi

# 2. Verificar /proc/rtpengine
echo -e "\n2. /proc/rtpengine:"
if [ -d /proc/rtpengine ]; then
    echo "✓ Directory exists"
    ls -la /proc/rtpengine/
else
    echo "✗ Directory NOT found"
    exit 1
fi

# 3. Verificar permisos
echo -e "\n3. Permissions:"
if [ -w /proc/rtpengine/control ]; then
    echo "✓ /proc/rtpengine/control is writable"
else
    echo "⚠ /proc/rtpengine/control may not be writable by containers"
fi

echo -e "\n✓ Kernel module is ready for use!"
```

---

## PARTE 4: ARCHIVOS DE CONFIGURACIÓN (10 minutos)

### 4.1 Script de inicio (docker-entrypoint.sh)

**IMPORTANTE: Crear este archivo ANTES del Dockerfile**

**Archivo: `docker-entrypoint.sh`**

```bash
#!/bin/bash
set -e

# Función para logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

# Verificar que el módulo del kernel esté cargado en el host
if [ ! -f /proc/rtpengine/control ]; then
    log "ERROR: Kernel module xt_RTPENGINE not loaded on host!"
    log "Please run on host: sudo modprobe xt_RTPENGINE"
    log "Or execute: sudo ./scripts/load-rtpengine-module.sh"
    exit 1
fi

# Configurar parámetros por defecto si no están definidos
RTPE_INTERFACE=${RTPE_INTERFACE:-eth0}
RTPE_LISTEN_NG=${RTPE_LISTEN_NG:-22222}
RTPE_TABLE=${RTPE_TABLE:-0}
RTPE_PORT_MIN=${RTPE_PORT_MIN:-10000}
RTPE_PORT_MAX=${RTPE_PORT_MAX:-20000}
RTPE_LOG_LEVEL=${RTPE_LOG_LEVEL:-6}
RTPE_RECORDING_DIR=${RTPE_RECORDING_DIR:-/var/spool/rtpengine}
RTPE_REDIS=${RTPE_REDIS:-}
RTPE_REDIS_DB=${RTPE_REDIS_DB:-0}

# Construir argumentos
ARGS=""
ARGS="$ARGS --interface=${RTPE_INTERFACE}"
ARGS="$ARGS --listen-ng=${RTPE_LISTEN_NG}"
ARGS="$ARGS --table=${RTPE_TABLE}"
ARGS="$ARGS --port-min=${RTPE_PORT_MIN}"
ARGS="$ARGS --port-max=${RTPE_PORT_MAX}"
ARGS="$ARGS --log-level=${RTPE_LOG_LEVEL}"
ARGS="$ARGS --pidfile=/var/run/rtpengine/rtpengine.pid"
ARGS="$ARGS --recording-dir=${RTPE_RECORDING_DIR}"
ARGS="$ARGS --recording-method=proc"
ARGS="$ARGS --log-stderr"
ARGS="$ARGS --no-fallback"

# Codec transcoding (si está habilitado)
if [ "${RTPE_TRANSCODING}" = "yes" ]; then
    ARGS="$ARGS --codec-accept=all"
fi

# Redis (si está configurado)
if [ -n "${RTPE_REDIS}" ]; then
    ARGS="$ARGS --redis=${RTPE_REDIS}"
    ARGS="$ARGS --redis-db=${RTPE_REDIS_DB}"
fi

# TOS/DSCP marking
if [ -n "${RTPE_TOS}" ]; then
    ARGS="$ARGS --tos=${RTPE_TOS}"
fi

log "Starting RTPEngine with parameters:"
log "  Interface: ${RTPE_INTERFACE}"
log "  Control Port: ${RTPE_LISTEN_NG}"
log "  Table ID: ${RTPE_TABLE}"
log "  RTP Ports: ${RTPE_PORT_MIN}-${RTPE_PORT_MAX}"
log "  Log Level: ${RTPE_LOG_LEVEL}"

# Ejecutar RTPEngine
exec /usr/local/bin/rtpengine --foreground $ARGS
```

```bash
# Hacer ejecutable
chmod +x docker-entrypoint.sh
```

### 4.2 Archivo de variables de entorno

**Archivo: `.env`**

```bash
# Interfaz de red (ajustar según tu sistema)
# Usa: ip a  para ver tus interfaces
RTPE_INTERFACE=eth0

# Timezone
TZ=America/Bogota

# Redis (opcional, para HA)
# RTPE_REDIS=redis:6379
# RTPE_REDIS_DB=0

# Transcoding (opcional)
# RTPE_TRANSCODING=yes
```

---

## PARTE 5: DOCKERFILE PARA RTPENGINE (15 minutos)

### 5.1 Dockerfile RTPEngine (Multi-stage build)

**IMPORTANTE: Los archivos docker-entrypoint.sh deben existir ANTES de este build**

**Archivo: `Dockerfile.rtpengine`**

```dockerfile
# ETAPA 1: Builder
FROM almalinux:9 AS builder

# Instalar dependencias de compilación
RUN dnf install -y epel-release && \
    dnf install -y \
    # Compilación
    gcc gcc-c++ make pkgconfig redhat-rpm-config \
    # Dependencias RTPEngine
    glib2-devel zlib-devel openssl-devel \
    pcre-devel libcurl-devel xmlrpc-c-devel \
    hiredis-devel json-glib-devel libevent-devel \
    iptables-devel libpcap-devel mariadb-devel \
    gperf perl-IPC-Cmd \
    # Herramientas
    git wget

# Clonar y compilar RTPEngine
WORKDIR /usr/local/src
RUN git clone https://github.com/sipwise/rtpengine.git && \
    cd rtpengine && \
    # Compilar daemon
    cd daemon && \
    make && \
    # Compilar kernel module (solo source, se carga en host)
    cd ../kernel-module && \
    make && \
    # Compilar utils
    cd ../utils && \
    make

# ETAPA 2: Runtime
FROM almalinux:9

LABEL maintainer="campus@mesaproyectos.com"
LABEL description="RTPEngine Media Proxy"
LABEL version="1.0"

# Instalar solo dependencias de runtime
RUN dnf install -y epel-release && \
    dnf install -y \
    glib2 zlib openssl pcre libcurl \
    xmlrpc-c hiredis json-glib libevent \
    iptables libpcap mariadb-connector-c \
    kmod iproute net-tools procps-ng && \
    dnf clean all

# Copiar binarios compilados
COPY --from=builder /usr/local/src/rtpengine/daemon/rtpengine /usr/local/bin/
COPY --from=builder /usr/local/src/rtpengine/utils/rtpengine-ctl /usr/local/bin/
COPY --from=builder /usr/local/src/rtpengine/utils/rtpengine-ng-client /usr/local/bin/

# Kernel module (para referencia, debe cargarse en host)
COPY --from=builder /usr/local/src/rtpengine/kernel-module/xt_RTPENGINE.ko /opt/

# Crear usuario rtpengine
RUN groupadd -r rtpengine && \
    useradd -r -g rtpengine -s /sbin/nologin rtpengine && \
    mkdir -p /var/run/rtpengine /var/log/rtpengine /var/spool/rtpengine && \
    chown -R rtpengine:rtpengine /var/run/rtpengine /var/log/rtpengine /var/spool/rtpengine

# Crear directorio de configuración
RUN mkdir -p /etc/rtpengine

# Copiar script de inicio
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# Volúmenes
VOLUME ["/etc/rtpengine", "/var/log/rtpengine", "/var/spool/rtpengine"]

# Puertos
# Control port (UDP)
EXPOSE 22222/udp
# RTP range (ejemplo: 10000-20000)
EXPOSE 10000-20000/udp

# Variables de entorno (defaults)
ENV RTPE_INTERFACE=eth0
ENV RTPE_LISTEN_NG=22222
ENV RTPE_TABLE=0
ENV RTPE_PORT_MIN=10000
ENV RTPE_PORT_MAX=20000
ENV RTPE_LOG_LEVEL=6

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD /usr/local/bin/rtpengine-ctl list 2>/dev/null || exit 1

USER rtpengine

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["rtpengine"]
```

### 5.2 Construir la imagen

```bash
# Construir imagen
docker build -f Dockerfile.rtpengine -t rtpengine:latest .

# Verificar imagen creada
docker images | grep rtpengine

# Ver tamaño
docker images rtpengine:latest --format "{{.Size}}"
```

**Tamaño esperado:** ~300-400MB (multi-stage build optimizado)

---

## PARTE 6: DOCKER COMPOSE MULTI-INSTANCIA (20 minutos)

### 6.1 Docker Compose - 3 instancias RTPEngine (Host Mode)

**Archivo: `docker-compose-rtpengine.yml`**

```yaml
services:
  # RTPEngine Instancia 1
  rtpengine-1:
    image: rtpengine:latest
    container_name: rtpengine-1
    hostname: rtpengine-1
    restart: unless-stopped
    
    # Network mode host para mejor rendimiento
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-1:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
      # Acceso al módulo kernel (crítico)
      - /proc/rtpengine:/proc/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE:-eth0}
      - RTPE_LISTEN_NG=22222
      - RTPE_TABLE=0
      - RTPE_PORT_MIN=10000
      - RTPE_PORT_MAX=15000
      - RTPE_LOG_LEVEL=6
      - TZ=${TZ:-America/Bogota}
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # RTPEngine Instancia 2
  rtpengine-2:
    image: rtpengine:latest
    container_name: rtpengine-2
    hostname: rtpengine-2
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-2:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
      - /proc/rtpengine:/proc/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE:-eth0}
      - RTPE_LISTEN_NG=22223
      - RTPE_TABLE=1
      - RTPE_PORT_MIN=15001
      - RTPE_PORT_MAX=20000
      - RTPE_LOG_LEVEL=6
      - TZ=${TZ:-America/Bogota}
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # RTPEngine Instancia 3
  rtpengine-3:
    image: rtpengine:latest
    container_name: rtpengine-3
    hostname: rtpengine-3
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-3:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
      - /proc/rtpengine:/proc/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE:-eth0}
      - RTPE_LISTEN_NG=22224
      - RTPE_TABLE=2
      - RTPE_PORT_MIN=20001
      - RTPE_PORT_MAX=25000
      - RTPE_LOG_LEVEL=6
      - TZ=${TZ:-America/Bogota}
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

**Puntos clave:**
- Cada instancia tiene **Table ID único** (0, 1, 2)
- Cada instancia tiene **puerto control único** (22222, 22223, 22224)
- Cada instancia tiene **rango RTP no solapado**
- Todas comparten `/proc/rtpengine` del host

### 6.2 Levantar servicios

```bash
# Verificar que el módulo kernel está cargado
sudo ./scripts/check-kernel-module.sh

# Levantar servicios
docker compose -f docker-compose-rtpengine.yml up -d

# Ver logs
docker compose -f docker-compose-rtpengine.yml logs -f

# Ver estado
docker compose -f docker-compose-rtpengine.yml ps
```

### 6.3 Verificar funcionamiento

```bash
# Ver estadísticas de cada instancia
docker exec rtpengine-1 rtpengine-ctl list
docker exec rtpengine-2 rtpengine-ctl -u 22223 list
docker exec rtpengine-3 rtpengine-ctl -u 22224 list

# Ver puertos en escucha
ss -ulnp | grep rtpengine

# Verificar tablas iptables
sudo iptables -t mangle -L RTPENGINE -v
```

---

## PARTE 7: INTEGRACIÓN KAMAILIO + RTPENGINE (20 minutos)

### 7.1 Stack completo con Kamailio

**Archivo: `docker-compose-full.yml`**

```yaml
services:
  kamailio:
    image: mi-kamailio:6.0-optimized
    command: ["/usr/local/sbin/kamailio", "-DD", "-E", "-f", "/etc/kamailio/kamailio.cfg"]
    container_name: kamailio
    hostname: kamailio
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./config/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./logs/kamailio:/var/log/kamailio
    
    environment:
      - KAMAILIO_LOG_LEVEL=3
      - TZ=${TZ}
    
    depends_on:
      - rtpengine-1
      - rtpengine-2
      - mariadb
    
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE

  rtpengine-1:
    image: rtpengine:latest
    container_name: rtpengine-1
    hostname: rtpengine-1
    restart: unless-stopped
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-1:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
      - /proc/rtpengine:/proc/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE}
      - RTPE_LISTEN_NG=22222
      - RTPE_TABLE=0
      - RTPE_PORT_MIN=10000
      - RTPE_PORT_MAX=15000
      - RTPE_LOG_LEVEL=6
      - TZ=${TZ}
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE

  rtpengine-2:
    image: rtpengine:latest
    container_name: rtpengine-2
    hostname: rtpengine-2
    restart: unless-stopped
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-2:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
      - /proc/rtpengine:/proc/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE}
      - RTPE_LISTEN_NG=22223
      - RTPE_TABLE=1
      - RTPE_PORT_MIN=15001
      - RTPE_PORT_MAX=20000
      - RTPE_LOG_LEVEL=6
      - TZ=${TZ}
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE

  mariadb:
    image: mariadb:10.11
    container_name: kamailio-db
    hostname: mariadb
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: kamailio
      MYSQL_USER: kamailio
      MYSQL_PASSWORD: ${DB_PASSWORD}
      TZ: ${TZ}
    
    volumes:
      - mariadb_data:/var/lib/mysql
    
    ports:
      - "3306:3306"
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mariadb_data:
```

### 7.2 Configuración Kamailio para múltiples RTPEngine

**Archivo: `config/kamailio.cfg` (fragmento relevante)**

```
#!KAMAILIO

# ... (configuración básica)

# Módulo RTPEngine
loadmodule "rtpengine.so"

# Configurar múltiples instancias RTPEngine
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:22222 udp:127.0.0.1:22223")
modparam("rtpengine", "rtpengine_allow_op", 1)
modparam("rtpengine", "rtpengine_retr", 3)
modparam("rtpengine", "rtpengine_tout_ms", 500)

# Algoritmo de balanceo: 0=round-robin, 1=random
modparam("rtpengine", "setid_default", 0)

# ... (routing logic)

route[RTPENGINE] {
    if (is_method("INVITE")) {
        # Flags para rtpengine_offer
        # trust-address: confiar en la dirección del SDP
        # replace-origin: reemplazar origin en SDP
        # replace-session-connection: reemplazar connection en SDP
        
        if (has_body("application/sdp")) {
            if (rtpengine_offer("trust-address replace-origin replace-session-connection")) {
                xlog("L_INFO", "RTPEngine offer successful\n");
                t_on_reply("RTPENGINE_REPLY");
            } else {
                xlog("L_ERR", "RTPEngine offer failed\n");
                sl_send_reply("503", "Service Unavailable - Media Proxy Error");
                exit;
            }
        }
    }
    
    if (is_method("ACK") && has_body("application/sdp")) {
        rtpengine_answer("trust-address replace-origin replace-session-connection");
    }
    
    if (is_method("BYE|CANCEL")) {
        rtpengine_delete();
    }
}

onreply_route[RTPENGINE_REPLY] {
    if (has_body("application/sdp")) {
        rtpengine_answer("trust-address replace-origin replace-session-connection");
    }
}
```

### 7.3 Actualizar archivo .env

```bash
# Base de datos
DB_ROOT_PASSWORD=RootPass123!
DB_PASSWORD=KamPass456!

# RTPEngine
RTPE_INTERFACE=eth0

# Timezone
TZ=America/Bogota
```

---

## PARTE 8: VERIFICACIÓN Y TROUBLESHOOTING (10 minutos)

### 8.1 Script de verificación completo

**Archivo: `scripts/check-rtpengine-stack.sh`**

```bash
#!/bin/bash

echo "=== RTPEngine Docker Stack Status Check ==="

# 1. Verificar módulo kernel
echo -e "\n1. Kernel Module:"
if lsmod | grep -q xt_RTPENGINE; then
    echo "✓ xt_RTPENGINE loaded"
    lsmod | grep xt_RTPENGINE
else
    echo "✗ xt_RTPENGINE NOT loaded - CRITICAL ERROR"
    exit 1
fi

# 2. Verificar /proc/rtpengine
echo -e "\n2. /proc/rtpengine:"
if [ -d /proc/rtpengine ]; then
    echo "✓ Directory exists"
    ls -la /proc/rtpengine/
else
    echo "✗ Directory NOT found - CRITICAL ERROR"
    exit 1
fi

# 3. Verificar contenedores
echo -e "\n3. Docker Containers:"
docker ps --filter "name=rtpengine" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 4. Verificar logs recientes
echo -e "\n4. Recent logs:"
for container in rtpengine-1 rtpengine-2 rtpengine-3; do
    echo -e "\n--- $container ---"
    docker logs --tail 5 $container 2>/dev/null || echo "Container not running"
done

# 5. Verificar estadísticas
echo -e "\n5. RTPEngine Statistics:"
for port in 22222 22223 22224; do
    echo -e "\n--- Instance on port $port ---"
    docker exec rtpengine-1 rtpengine-ctl -u $port list 2>/dev/null | head -10 || echo "Not accessible"
done

# 6. Verificar puertos en escucha
echo -e "\n6. Listening Ports:"
ss -ulnp | grep rtpengine || echo "No ports found"

# 7. Test de conectividad
echo -e "\n7. Connectivity Test:"
for port in 22222 22223 22224; do
    nc -zu 127.0.0.1 $port && echo "✓ Port $port reachable" || echo "✗ Port $port NOT reachable"
done

# 8. Verificar iptables
echo -e "\n8. IPTables Rules:"
sudo iptables -t mangle -L RTPENGINE -v 2>/dev/null || echo "No RTPENGINE chain found"

echo -e "\n=== Check Complete ==="
```

### 8.2 Problemas comunes y soluciones

**Problema 1: "Kernel module not loaded"**
```bash
# Solución:
sudo ./scripts/load-rtpengine-module.sh
lsmod | grep xt_RTPENGINE
```

**Problema 2: "Table ID conflict"**
```bash
# Verificar que cada instancia tiene Table ID único
docker compose -f docker-compose-rtpengine.yml config | grep RTPE_TABLE

# Deben ser: 0, 1, 2 (no duplicados)
```

**Problema 3: "Port already in use"**
```bash
# Verificar puertos en uso
ss -ulnp | grep 2222

# Asegurar puertos únicos: 22222, 22223, 22224
```

**Problema 4: "No audio en llamadas"**
```bash
# Verificar sesiones RTP
docker exec rtpengine-1 rtpengine-ctl list

# Verificar rangos RTP no solapados
docker compose -f docker-compose-rtpengine.yml config | grep RTPE_PORT
```

---

## PARTE 9: EJERCICIO PRÁCTICO (20 minutos)

### Ejercicio: Desplegar stack completo y probar llamadas

**Objetivo:** Stack funcional Kamailio + 2 RTPEngine + MariaDB

**Resumen del flujo completo:**

```bash
# 1. Preparar host (CRÍTICO - PRIMER PASO)
sudo ./scripts/load-rtpengine-module.sh
sudo ./scripts/check-kernel-module.sh

# 2. Verificar estructura
tree -L 2

# 3. Verificar archivos de configuración
ls -la docker-entrypoint.sh .env

# 4. Construir imagen RTPEngine
docker build -f Dockerfile.rtpengine -t rtpengine:latest .

# 5. Levantar stack completo
docker compose -f docker-compose-full.yml up -d

# 6. Verificar todo
./scripts/check-rtpengine-stack.sh

# 7. Ver logs
docker compose -f docker-compose-full.yml logs -f
```

**Test de llamada:**

1. Registrar 2 softphones:
   - Usuario 1: 1001@<IP>:5060
   - Usuario 2: 1002@<IP>:5060

2. Hacer llamada de 1001 a 1002

3. Verificar sesiones RTP:
```bash
# Ver sesiones activas
docker exec rtpengine-1 rtpengine-ctl list

# Ver estadísticas en tiempo real
watch -n 1 'docker exec rtpengine-1 rtpengine-ctl list'
```

---

## RESUMEN Y MEJORES PRÁCTICAS

### ✅ Checklist RTPEngine Docker:

```
□ Módulo kernel xt_RTPENGINE cargado en HOST (CRÍTICO)
□ /proc/rtpengine accesible y writable
□ Estructura de directorios creada ANTES del build
□ docker-entrypoint.sh existe ANTES del build
□ Cada instancia con table ID único (0, 1, 2...)
□ Cada instancia con puerto control único (22222, 22223...)
□ Rangos RTP no solapados entre instancias
□ Network mode "host" para producción
□ Capabilities NET_ADMIN y SYS_NICE
□ Logs en volumes persistentes
□ Healthchecks configurados
□ Variables de entorno en .env
```

### 🔥 Errores comunes a evitar:

```
❌ No cargar módulo kernel en host (ERROR MÁS COMÚN)
❌ Intentar cargar módulo desde dentro del contenedor
❌ No montar /proc/rtpengine en los contenedores
❌ Table IDs duplicados entre instancias
❌ Puertos control duplicados
❌ Rangos RTP solapados
❌ No usar CAP_ADD NET_ADMIN
❌ Crear Dockerfile antes que docker-entrypoint.sh
❌ No verificar módulo antes de levantar contenedores
❌ Olvidar hacer chmod +x docker-entrypoint.sh
```

### 📋 Orden correcto de ejecución:

```
1. Cargar módulo kernel en HOST (scripts/load-rtpengine-module.sh)
2. Verificar módulo (scripts/check-kernel-module.sh)
3. Crear estructura de directorios
4. Crear docker-entrypoint.sh y .env
5. Crear Dockerfile
6. Construir imagen (docker build)
7. Crear docker-compose.yml
8. Levantar servicios (docker compose up)
9. Verificar (scripts/check-rtpengine-stack.sh)
```

### 📊 Métricas a monitorear:

```
- Sesiones activas (rtpengine-ctl list)
- CPU y memoria por instancia (docker stats)
- Packets procesados
- Latencia de red
- Disk I/O (si hay recording)
- Errores en logs
```

### 📚 Recursos adicionales:

- RTPEngine GitHub: https://github.com/sipwise/rtpengine
- RTPEngine Wiki: https://github.com/sipwise/rtpengine/wiki
- Docker Networking: https://docs.docker.com/network/
- Kamailio RTPEngine Module: https://www.kamailio.org/docs/modules/stable/modules/rtpengine.html

---

**FIN VIDEOCONFERENCIA 2 - PARTE DOCKER**

**Próxima sesión:** Arquitectura de microservicios y balanceo con contenedores
