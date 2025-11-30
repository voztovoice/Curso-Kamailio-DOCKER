# VIDEOCONFERENCIA 2 - CONTENIDO DOCKER
## RTPEngine en Docker - Multi-instancia y Configuración Avanzada

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
   - xt_RTPENGINE debe cargarse en el HOST
   - No puede cargarse desde dentro del contenedor
   - Cada instancia necesita un table ID único

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
│                    DOCKER HOST                       │
│                                                      │
│  ┌────────────────────────────────────────────┐    │
│  │  Kernel Module: xt_RTPENGINE              │    │
│  │  (cargado en host, compartido)            │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ RTPEngine #1 │  │ RTPEngine #2 │  │ RTPEng #3│ │
│  │ Table ID: 0  │  │ Table ID: 1  │  │ Table: 2 │ │
│  │ Port: 22222  │  │ Port: 22223  │  │ Port:2224│ │
│  │ RTP:10k-15k  │  │ RTP:15k-20k  │  │ RTP:20-25│ │
│  └──────────────┘  └──────────────┘  └──────────┘ │
│         ▲                 ▲                ▲        │
│         │                 │                │        │
│         └─────────────────┴────────────────┘        │
│                           │                          │
│                    ┌──────────────┐                 │
│                    │  Kamailio    │                 │
│                    │  Dispatcher  │                 │
│                    └──────────────┘                 │
└─────────────────────────────────────────────────────┘
```

---

## PARTE 2: DOCKERFILE PARA RTPENGINE (15 minutos)

### 2.1 Dockerfile RTPEngine (Compilación desde código)

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

LABEL maintainer="curso@mesaproyectos.com"
LABEL description="RTPEngine Media Proxy"

# Instalar solo dependencias de runtime
RUN dnf install -y epel-release && \
    dnf install -y \
    glib2 zlib openssl pcre libcurl \
    xmlrpc-c hiredis json-glib libevent \
    iptables libpcap mariadb-libs \
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
    mkdir -p /var/run/rtpengine /var/log/rtpengine && \
    chown -R rtpengine:rtpengine /var/run/rtpengine /var/log/rtpengine

# Crear archivo de configuración por defecto
RUN mkdir -p /etc/rtpengine

# Script de inicio
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# Volúmenes
VOLUME ["/etc/rtpengine", "/var/log/rtpengine"]

# Puertos
# Control port (UDP)
EXPOSE 22222/udp
# RTP range (ejemplo: 10000-20000)
EXPOSE 10000-20000/udp

# Variables de entorno (defaults)
ENV RTPE_INTERFACE=enp1s0
ENV RTPE_LISTEN_NG=22222
ENV RTPE_TABLE=0
ENV RTPE_PORT_MIN=10000
ENV RTPE_PORT_MAX=20000
ENV RTPE_LOG_LEVEL=6
ENV RTPE_RECORDING_DIR=/var/spool/rtpengine

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD /usr/local/bin/rtpengine-ctl list 2>/dev/null || exit 1

USER rtpengine

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["rtpengine"]
```

### 2.2 Script de inicio (docker-entrypoint.sh)

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
    log "WARNING: Kernel module xt_RTPENGINE not loaded on host!"
    log "Please run on host: modprobe xt_RTPENGINE"
    log "Continuing without kernel forwarding (user-space only)..."
fi

# Configurar parámetros por defecto si no están definidos
RTPE_INTERFACE=${RTPE_INTERFACE:-enp1s0}
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

# Codec transcoding (si está habilitado)
if [ "${RTPE_TRANSCODING}" = "yes" ]; then
    ARGS="$ARGS --codec-transcoding=always"
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

### 2.3 Configuración alternativa (archivo rtpengine.conf)

**Archivo: `config/rtpengine.conf`**

```ini
[rtpengine]
# Interface configuration
interface = enp1s0

# Listening ports
listen-ng = 22222

# Port range for RTP/RTCP
port-min = 10000
port-max = 20000

# Kernel table (for xt_RTPENGINE forwarding)
table = 0

# Timeout values (in seconds)
timeout = 60
silent-timeout = 3600
final-timeout = 10800

# Recording
recording-dir = /var/spool/rtpengine
recording-method = proc
recording-format = pcap

# Logging
log-level = 6
log-facility = local1
log-stderr = yes

# Performance
num-threads = 4

# Redis (optional, for HA)
# redis = 127.0.0.1:6379
# redis-db = 0
# no-redis-required = false

# Homer SIP capture (optional)
# homer = 192.168.1.100:9060
# homer-protocol = udp
# homer-id = 2001

# DTLS
# dtls-passive = yes

# Codec preferences (transcoding)
# codec-transcoding = always
# codec-prefer = PCMA, PCMU, opus
```

---

## PARTE 3: MULTI-INSTANCIA CON DOCKER COMPOSE (20 minutos)

### 3.1 Docker Compose - 3 instancias RTPEngine

**Archivo: `docker-compose-rtpengine.yml`**

```yaml
version: '3.8'

services:
  # RTPEngine Instancia 1
  rtpengine-1:
    build:
      context: .
      dockerfile: Dockerfile.rtpengine
    container_name: rtpengine-1
    hostname: rtpengine-1
    restart: unless-stopped
    
    # Network mode host para mejor rendimiento
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-1:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=enp1s0
      - RTPE_LISTEN_NG=22222
      - RTPE_TABLE=0
      - RTPE_PORT_MIN=10000
      - RTPE_PORT_MAX=15000
      - RTPE_LOG_LEVEL=6
      - RTPE_TRANSCODING=yes
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    healthcheck:
      test: ["CMD", "/usr/local/bin/rtpengine-ctl", "list"]
      interval: 30s
      timeout: 5s
      retries: 3

  # RTPEngine Instancia 2
  rtpengine-2:
    build:
      context: .
      dockerfile: Dockerfile.rtpengine
    container_name: rtpengine-2
    hostname: rtpengine-2
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-2:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=enp1s0
      - RTPE_LISTEN_NG=22223
      - RTPE_TABLE=1
      - RTPE_PORT_MIN=15000
      - RTPE_PORT_MAX=20000
      - RTPE_LOG_LEVEL=6
      - RTPE_TRANSCODING=yes
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    healthcheck:
      test: ["CMD", "/usr/local/bin/rtpengine-ctl", "-u", "22223", "list"]
      interval: 30s
      timeout: 5s
      retries: 3

  # RTPEngine Instancia 3
  rtpengine-3:
    build:
      context: .
      dockerfile: Dockerfile.rtpengine
    container_name: rtpengine-3
    hostname: rtpengine-3
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-3:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=enp1s0
      - RTPE_LISTEN_NG=22224
      - RTPE_TABLE=2
      - RTPE_PORT_MIN=20000
      - RTPE_PORT_MAX=25000
      - RTPE_LOG_LEVEL=6
      - RTPE_TRANSCODING=yes
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    healthcheck:
      test: ["CMD", "/usr/local/bin/rtpengine-ctl", "-u", "22224", "list"]
      interval: 30s
      timeout: 5s
      retries: 3

  # Redis (opcional, para HA)
  redis:
    image: redis:7-alpine
    container_name: rtpengine-redis
    restart: unless-stopped
    
    ports:
      - "6379:6379"
    
    volumes:
      - redis_data:/data
    
    command: redis-server --appendonly yes
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  redis_data:
```

### 3.2 Versión con network bridge (alternativa)

**Archivo: `docker-compose-rtpengine-bridge.yml`**

```yaml
version: '3.8'

services:
  rtpengine-1:
    build:
      context: .
      dockerfile: Dockerfile.rtpengine
    container_name: rtpengine-1
    hostname: rtpengine-1
    restart: unless-stopped
    
    networks:
      rtpe-net:
        ipv4_address: 172.25.0.10
    
    ports:
      # Control port
      - "22222:22222/udp"
      # RTP range
      - "10000-15000:10000-15000/udp"
    
    volumes:
      - ./config/rtpengine-1.conf:/etc/rtpengine/rtpengine.conf:ro
      - ./logs/rtpengine-1:/var/log/rtpengine
    
    environment:
      - RTPE_INTERFACE=eth0
      - RTPE_LISTEN_NG=22222
      - RTPE_TABLE=0
      - RTPE_PORT_MIN=10000
      - RTPE_PORT_MAX=15000
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE

  rtpengine-2:
    build:
      context: .
      dockerfile: Dockerfile.rtpengine
    container_name: rtpengine-2
    hostname: rtpengine-2
    restart: unless-stopped
    
    networks:
      rtpe-net:
        ipv4_address: 172.25.0.11
    
    ports:
      - "22223:22223/udp"
      - "15000-20000:15000-20000/udp"
    
    volumes:
      - ./config/rtpengine-2.conf:/etc/rtpengine/rtpengine.conf:ro
      - ./logs/rtpengine-2:/var/log/rtpengine
    
    environment:
      - RTPE_INTERFACE=eth0
      - RTPE_LISTEN_NG=22223
      - RTPE_TABLE=1
      - RTPE_PORT_MIN=15000
      - RTPE_PORT_MAX=20000
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE

networks:
  rtpe-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.0.0/24
```

---

## PARTE 4: INTEGRACIÓN KAMAILIO + RTPENGINE DOCKER (15 minutos)

### 4.1 Stack completo: Kamailio + RTPEngine + MariaDB

**Archivo: `docker-compose-full.yml`**

```yaml
version: '3.8'

services:
  # Base de datos
  mariadb:
    image: mariadb:10.11
    container_name: voip-db
    hostname: mariadb
    restart: unless-stopped
    
    networks:
      voip-net:
        ipv4_address: 172.30.0.10
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: kamailio
      MYSQL_USER: kamailio
      MYSQL_PASSWORD: ${DB_PASSWORD}
    
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./scripts/init-kamailio-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis para RTPEngine HA
  redis:
    image: redis:7-alpine
    container_name: voip-redis
    hostname: redis
    restart: unless-stopped
    
    networks:
      voip-net:
        ipv4_address: 172.30.0.11
    
    volumes:
      - redis_data:/data
    
    command: redis-server --appendonly yes
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  # RTPEngine 1
  rtpengine-1:
    build:
      context: ./rtpengine
      dockerfile: Dockerfile
    container_name: rtpengine-1
    hostname: rtpengine-1
    restart: unless-stopped
    
    # Host mode para mejor rendimiento
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-1:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE:-enp1s0}
      - RTPE_LISTEN_NG=22222
      - RTPE_TABLE=0
      - RTPE_PORT_MIN=10000
      - RTPE_PORT_MAX=15000
      - RTPE_LOG_LEVEL=6
      - RTPE_REDIS=172.30.0.11:6379
      - RTPE_REDIS_DB=0
      - RTPE_TRANSCODING=yes
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    depends_on:
      redis:
        condition: service_healthy

  # RTPEngine 2
  rtpengine-2:
    build:
      context: ./rtpengine
      dockerfile: Dockerfile
    container_name: rtpengine-2
    hostname: rtpengine-2
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-2:/var/log/rtpengine
      - ./recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE:-enp1s0}
      - RTPE_LISTEN_NG=22223
      - RTPE_TABLE=1
      - RTPE_PORT_MIN=15000
      - RTPE_PORT_MAX=20000
      - RTPE_LOG_LEVEL=6
      - RTPE_REDIS=172.30.0.11:6379
      - RTPE_REDIS_DB=0
      - RTPE_TRANSCODING=yes
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    depends_on:
      redis:
        condition: service_healthy

  # Kamailio
  kamailio:
    build:
      context: ./kamailio
      dockerfile: Dockerfile
    container_name: kamailio-server
    hostname: kamailio
    restart: unless-stopped
    
    networks:
      voip-net:
        ipv4_address: 172.30.0.20
    
    ports:
      - "5060:5060/udp"
      - "5060:5060/tcp"
      - "5061:5061/tcp"
      - "8080:8080/tcp"
    
    volumes:
      - ./config/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./logs/kamailio:/var/log/kamailio
    
    environment:
      - KAMAILIO_LOG_LEVEL=3
      - DB_HOST=mariadb
      - DB_USER=kamailio
      - DB_PASS=${DB_PASSWORD}
      - RTPE_SET_1=udp:127.0.0.1:22222
      - RTPE_SET_2=udp:127.0.0.1:22223
    
    depends_on:
      mariadb:
        condition: service_healthy
      rtpengine-1:
        condition: service_healthy
      rtpengine-2:
        condition: service_healthy

volumes:
  mariadb_data:
  redis_data:

networks:
  voip-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/24
```

### 4.2 Configuración Kamailio para múltiples RTPEngine

**Fragmento de `kamailio.cfg` con dispatcher para RTPEngine:**

```
# Cargar módulo rtpengine
loadmodule "rtpengine.so"

# Configuración RTPEngine con múltiples sets
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:22222")
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:22223")

# Balanceo round-robin
modparam("rtpengine", "rtpengine_disable_tout", 20)
modparam("rtpengine", "rtpengine_retr", 5)
modparam("rtpengine", "rtpengine_tout_ms", 1000)

# En route de manejo de media
route[RTPENGINE] {
    if (is_method("INVITE") && has_body("application/sdp")) {
        # INVITE inicial - offer
        if (rtpengine_offer("codec-mask-all codec-transcode-PCMA codec-transcode-PCMU")) {
            t_on_reply("MANAGE_REPLY");
        } else {
            xlog("L_ERR", "RTPEngine offer failed\n");
        }
    }
}

onreply_route[MANAGE_REPLY] {
    if (status =~ "18[0-9]|2[0-9][0-9]" && has_body("application/sdp")) {
        # Respuesta con SDP - answer
        rtpengine_answer("codec-mask-all codec-transcode-PCMA codec-transcode-PCMU");
    }
}

# Al terminar la llamada
route[RTPENGINE_DELETE] {
    if (is_method("BYE|CANCEL")) {
        rtpengine_delete();
    }
}
```

### 4.3 Archivo .env para variables

**Archivo: `.env`**

```bash
# Database
DB_ROOT_PASSWORD=SuperSecureRootPass123!
DB_PASSWORD=KamailioSecurePass456!

# RTPEngine
RTPE_INTERFACE=enp1s0

# Network
EXTERNAL_IP=192.168.1.100
SIP_DOMAIN=sip.miempresa.com

# Timezone
TZ=America/Bogota
```

---

## PARTE 5: CARGA DEL MÓDULO KERNEL Y TROUBLESHOOTING (10 minutos)

### 5.1 Cargar módulo xt_RTPENGINE en el host

**Script de carga del módulo:**

**Archivo: `load-rtpengine-module.sh`**

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
else
    echo "Loading kernel module..."
    
    # Opción 1: Si el módulo está en el sistema
    if modprobe xt_RTPENGINE 2>/dev/null; then
        echo -e "${GREEN}Module loaded from system${NC}"
    else
        # Opción 2: Compilar e insertar desde contenedor
        echo "Extracting and loading module from container..."
        
        # Obtener el módulo desde el contenedor
        docker run --rm rtpengine:latest cat /opt/xt_RTPENGINE.ko > /tmp/xt_RTPENGINE.ko
        
        # Insertar el módulo
        insmod /tmp/xt_RTPENGINE.ko
        
        if [ $? -eq 0 ]; then
            echo -e "${GREEN}Module loaded successfully${NC}"
        else
            echo -e "${RED}Failed to load module${NC}"
            exit 1
        fi
    fi
fi

# Verificar que /proc/rtpengine existe
if [ -d /proc/rtpengine ]; then
    echo -e "${GREEN}/proc/rtpengine directory exists${NC}"
    ls -la /proc/rtpengine/
else
    echo -e "${RED}/proc/rtpengine NOT found${NC}"
    exit 1
fi

# Mostrar reglas iptables de RTPEngine
echo -e "\n${GREEN}Current iptables rules for RTPEngine:${NC}"
iptables -t mangle -L RTPENGINE 2>/dev/null || echo "No RTPENGINE chain yet"

echo -e "\n${GREEN}Setup complete!${NC}"
```

### 5.2 Persistir el módulo en reinicio

**Archivo: `/etc/modules-load.d/rtpengine.conf`**

```
xt_RTPENGINE
```

**Archivo: `/etc/modprobe.d/rtpengine.conf`**

```
# RTPEngine kernel module configuration
options xt_RTPENGINE
```

### 5.3 Script de verificación

**Archivo: `check-rtpengine.sh`**

```bash
#!/bin/bash

echo "=== RTPEngine Docker Status Check ==="

# 1. Verificar módulo kernel
echo -e "\n1. Kernel Module:"
if lsmod | grep -q xt_RTPENGINE; then
    echo "✓ xt_RTPENGINE loaded"
    lsmod | grep xt_RTPENGINE
else
    echo "✗ xt_RTPENGINE NOT loaded"
fi

# 2. Verificar /proc/rtpengine
echo -e "\n2. /proc/rtpengine:"
if [ -d /proc/rtpengine ]; then
    echo "✓ Directory exists"
    ls -la /proc/rtpengine/
else
    echo "✗ Directory NOT found"
fi

# 3. Verificar contenedores
echo -e "\n3. Docker Containers:"
docker ps --filter "name=rtpengine" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 4. Verificar logs
echo -e "\n4. Recent logs from rtpengine-1:"
docker logs --tail 20 rtpengine-1 2>/dev/null || echo "Container not running"

# 5. Verificar estadísticas
echo -e "\n5. RTPEngine Statistics:"
for port in 22222 22223 22224; do
    echo -e "\nInstance on port $port:"
    docker exec rtpengine-1 rtpengine-ctl -u $port list 2>/dev/null || echo "Not accessible"
done

# 6. Verificar puertos en escucha
echo -e "\n6. Listening Ports:"
ss -ulnp | grep rtpengine || echo "No ports found"

# 7. Test de conectividad
echo -e "\n7. Connectivity Test:"
nc -zu 127.0.0.1 22222 && echo "✓ Port 22222 (rtpengine-1) reachable" || echo "✗ Port 22222 NOT reachable"
nc -zu 127.0.0.1 22223 && echo "✓ Port 22223 (rtpengine-2) reachable" || echo "✗ Port 22223 NOT reachable"
```

---

## PARTE 6: EJERCICIO PRÁCTICO (20 minutos)

### Ejercicio: Desplegar stack completo Kamailio + RTPEngine multi-instancia

**Objetivo:** Levantar Kamailio con 2 instancias RTPEngine balanceadas

**Pasos:**

#### 1. Preparar entorno

```bash
# Crear estructura de directorios
mkdir -p ~/voip-stack/{kamailio,rtpengine,config,logs,scripts,recordings}
cd ~/voip-stack

# Estructura:
# .
# ├── kamailio/
# │   └── Dockerfile
# ├── rtpengine/
# │   ├── Dockerfile
# │   └── docker-entrypoint.sh
# ├── config/
# │   └── kamailio.cfg
# ├── logs/
# │   ├── kamailio/
# │   ├── rtpengine-1/
# │   └── rtpengine-2/
# ├── scripts/
# │   ├── init-db.sql
# │   └── load-rtpengine-module.sh
# ├── recordings/
# ├── docker-compose.yml
# └── .env
```

#### 2. Cargar módulo kernel (EN EL HOST)

```bash
# Instalar headers del kernel si no están
sudo dnf install -y kernel-devel kernel-headers

# Cargar módulo
sudo chmod +x scripts/load-rtpengine-module.sh
sudo ./scripts/load-rtpengine-module.sh

# Verificar
lsmod | grep xt_RTPENGINE
ls -la /proc/rtpengine/
```

#### 3. Configurar variables de entorno

```bash
# Editar .env
cat > .env << 'EOF'
DB_ROOT_PASSWORD=RootPass123!
DB_PASSWORD=KamPass456!
RTPE_INTERFACE=enp1s0
EXTERNAL_IP=192.168.1.100
TZ=America/Bogota
EOF
```

#### 4. Construir imágenes

```bash
# Build RTPEngine
cd rtpengine
docker build -t rtpengine:latest .

# Build Kamailio
cd ../kamailio
docker build -t kamailio:6.0 .

cd ..
```

#### 5. Levantar servicios

```bash
# Levantar todo el stack
docker compose -f docker-compose-full.yml up -d

# Ver logs
docker compose logs -f
```

#### 6. Verificar funcionamiento

```bash
# Check containers
docker compose ps

# Check RTPEngine instances
docker exec rtpengine-1 rtpengine-ctl list
docker exec rtpengine-2 rtpengine-ctl -u 22223 list

# Check Kamailio
docker exec kamailio kamctl fifo get_statistics all
docker exec kamailio kamctl rtpengine show

# Ver logs en tiempo real
docker compose logs -f rtpengine-1
```

#### 7. Test de llamada

```bash
# Registrar 2 softphones
# SIP Server: <IP-HOST>:5060
# Usuario 1: 1001@kamailio.local / password: test1001
# Usuario 2: 1002@kamailio.local / password: test1002

# Hacer llamada de 1001 a 1002

# Verificar sesiones RTP activas
docker exec rtpengine-1 rtpengine-ctl list

# Ver estadísticas
watch -n 1 'docker exec rtpengine-1 rtpengine-ctl list | grep "session\|SSRC"'
```

#### 8. Monitoreo

```bash
# Script de monitoreo continuo
cat > monitor.sh << 'EOF'
#!/bin/bash
while true; do
    clear
    echo "=== RTPEngine Status ==="
    echo "Instance 1 (22222):"
    docker exec rtpengine-1 rtpengine-ctl list 2>/dev/null | head -20
    echo ""
    echo "Instance 2 (22223):"
    docker exec rtpengine-2 rtpengine-ctl -u 22223 list 2>/dev/null | head -20
    sleep 2
done
EOF

chmod +x monitor.sh
./monitor.sh
```

---

## RESUMEN Y MEJORES PRÁCTICAS

### ✅ Checklist RTPEngine Docker:

```
□ Módulo kernel xt_RTPENGINE cargado en HOST
□ Cada instancia con table ID único (0, 1, 2...)
□ Cada instancia con puerto control único (22222, 22223...)
□ Rangos RTP no solapados entre instancias
□ Network mode "host" para producción
□ Capabilities NET_ADMIN y SYS_NICE
□ Logs en volumes persistentes
□ Healthchecks configurados
□ Variables de entorno para configuración
□ Redis para HA (opcional)
□ Monitoreo con rtpengine-ctl
```

### 🔥 Errores comunes:

```
❌ No cargar módulo kernel en host
❌ Table IDs duplicados entre instancias
❌ Puertos control duplicados
❌ Rangos RTP solapados
❌ No usar CAP_ADD NET_ADMIN
❌ Olvidar mapear /proc en bridge mode
❌ No persistir logs
❌ No configurar healthchecks
```

### 📊 Métricas a monitorear:

```
- Sesiones activas (rtpengine-ctl list)
- CPU y memoria por instancia
- Packets procesados
- Errores de transcoding
- Latencia de red
- Disk I/O (si hay recording)
```

---

**FIN VIDEOCONFERENCIA 2 - PARTE DOCKER**

**Próxima sesión:** Arquitectura de microservicios y balanceo con contenedores
