# VIDEOCONFERENCIA 3
## Arquitectura de Microservicios VoIP y Balanceo con Contenedores

---

## PARTE 1: ARQUITECTURA DE MICROSERVICIOS VOIP (15 minutos)

### 1.1 De monolítico a microservicios

**Arquitectura Tradicional (Monolítica):**

```
┌─────────────────────────────────────┐
│     Servidor VoIP Monolítico        │
│                                     │
│  ┌─────────┐  ┌──────────┐        │
│  │Kamailio │  │ Asterisk │         │
│  │  + DB   │  │  + Apps  │         │
│  │ + RTP   │  │  + Media │         │
│  └─────────┘  └──────────┘         │
│                                     │
│  Todo en un único servidor          │
└─────────────────────────────────────┘

Problemas:
❌ Difícil escalar componentes independientemente
❌ Un fallo afecta todo el sistema
❌ Actualizaciones requieren downtime completo
❌ Recursos desperdiciados
```

**Arquitectura Microservicios (Docker):**

```
┌──────────────────────────────────────────────────────┐
│              Docker Host / Cluster                    │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │Kamailio 1│  │Kamailio 2│  │Kamailio 3│          │
│  │  SIP     │  │  SIP     │  │  SIP     │          │
│  │  Proxy   │  │  Proxy   │  │  Proxy   │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │             │             │                  │
│       └─────────────┴─────────────┘                  │
│                     │                                │
│         ┌───────────┴───────────┐                   │
│         │                       │                    │
│  ┌──────▼──────┐        ┌──────▼──────┐            │
│  │ RTPEngine 1 │        │ RTPEngine 2 │            │
│  │   Media     │        │   Media     │            │
│  └─────────────┘        └─────────────┘            │
│                                                      │
│  ┌─────────────┐        ┌─────────────┐            │
│  │ Asterisk 1  │        │ Asterisk 2  │            │
│  │   Apps      │        │   Apps      │            │
│  └─────────────┘        └─────────────┘            │
│                                                      │
│  ┌─────────────┐        ┌─────────────┐            │
│  │  MariaDB    │◄──────►│   Redis     │            │
│  │  Primary    │        │   Cache     │            │
│  └─────────────┘        └─────────────┘            │
└──────────────────────────────────────────────────────┘

Ventajas:
✅ Escalar cada componente independientemente
✅ Aislamiento de fallos
✅ Actualizaciones sin downtime (rolling updates)
✅ Optimización de recursos por servicio
✅ Desarrollo y testing independiente
```

### 1.2 Componentes de la arquitectura VoIP containerizada

**1. Capa de Señalización (SIP):**
```
- Kamailio (Proxy SIP, registrar, dispatcher)
- Load Balancer externo (HAProxy/Nginx) opcional
- Múltiples instancias para HA
```

**2. Capa de Media (RTP/SRTP):**
```
- RTPEngine (transcoding, relay, SRTP)
- Múltiples instancias balanceadas
- Redis para compartir estado
```

**3. Capa de Aplicaciones:**
```
- Asterisk/FreeSWITCH (IVR, voicemail, conferencias)
- Separado del proxy SIP
- Escalable según carga
```

**4. Capa de Datos:**
```
- MariaDB/PostgreSQL (usuarios, CDRs)
- Redis (cache, sesiones)
- Replicación para HA
```

**5. Capa de Monitoreo:**
```
- Prometheus (métricas)
- Grafana (visualización)
- Homer (captura SIP)
- ELK Stack (logs centralizados)
```

### 1.3 Patrones de comunicación entre contenedores

**1. Service Discovery:**
```yaml
# Docker compose automáticamente crea DNS interno
services:
  kamailio:
    # Puede acceder a "mariadb" por nombre
    environment:
      - DB_HOST=mariadb  # Resuelve a IP del contenedor
```

**2. Network Isolation:**
```yaml
networks:
  frontend:  # Acceso público (SIP)
  backend:   # Solo interno (DB, Redis)
  media:     # RTP traffic
```

**3. Shared Volumes:**
```yaml
volumes:
  recordings:  # Compartido entre RTPEngine y apps
  configs:     # Configuraciones compartidas
```

---

## PARTE 2: DOCKER COMPOSE AVANZADO - STACK COMPLETO (20 minutos)

### 2.1 Stack VoIP completo con microservicios

**Archivo: `docker-compose-microservices.yml`**

```yaml
version: '3.8'

#############################################
# VOIP MICROSERVICES STACK
# - 3x Kamailio (SIP Proxy)
# - 2x RTPEngine (Media Proxy)
# - 2x Asterisk (App Server)
# - 1x MariaDB (Database)
# - 1x Redis (Cache/State)
# - 1x Homer (SIP Capture)
#############################################

services:
  #==========================================
  # CAPA DE BASE DE DATOS
  #==========================================
  
  mariadb:
    image: mariadb:10.11
    container_name: voip-db
    hostname: mariadb
    restart: unless-stopped
    
    networks:
      backend:
        ipv4_address: 172.31.0.10
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: kamailio
      MYSQL_USER: kamailio
      MYSQL_PASSWORD: ${DB_PASSWORD}
      TZ: ${TZ}
    
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./scripts/init-kamailio-db.sql:/docker-entrypoint-initdb.d/01-kamailio.sql:ro
      - ./scripts/init-homer-db.sql:/docker-entrypoint-initdb.d/02-homer.sql:ro
      - ./config/mysql/my.cnf:/etc/mysql/conf.d/custom.cnf:ro
    
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --max_connections=1000
      - --innodb_buffer_pool_size=1G
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    container_name: voip-redis
    hostname: redis
    restart: unless-stopped
    
    networks:
      backend:
        ipv4_address: 172.31.0.11
    
    volumes:
      - redis_data:/data
      - ./config/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    
    command: redis-server /usr/local/etc/redis/redis.conf
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "2"

  #==========================================
  # CAPA DE MEDIA (RTP)
  #==========================================
  
  rtpengine-1:
    build:
      context: ./rtpengine
      dockerfile: Dockerfile
    container_name: rtpengine-1
    hostname: rtpengine-1
    restart: unless-stopped
    
    network_mode: host
    
    volumes:
      - ./logs/rtpengine-1:/var/log/rtpengine
      - recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE}
      - RTPE_LISTEN_NG=22222
      - RTPE_TABLE=0
      - RTPE_PORT_MIN=10000
      - RTPE_PORT_MAX=20000
      - RTPE_LOG_LEVEL=6
      - RTPE_REDIS=172.31.0.11:6379
      - RTPE_REDIS_DB=0
      - RTPE_TRANSCODING=yes
      - RTPE_TOS=184
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    depends_on:
      redis:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "/usr/local/bin/rtpengine-ctl", "list"]
      interval: 30s
      timeout: 5s
      retries: 3

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
      - recordings:/var/spool/rtpengine
    
    environment:
      - RTPE_INTERFACE=${RTPE_INTERFACE}
      - RTPE_LISTEN_NG=22223
      - RTPE_TABLE=1
      - RTPE_PORT_MIN=20000
      - RTPE_PORT_MAX=30000
      - RTPE_LOG_LEVEL=6
      - RTPE_REDIS=172.31.0.11:6379
      - RTPE_REDIS_DB=0
      - RTPE_TRANSCODING=yes
      - RTPE_TOS=184
    
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    
    depends_on:
      redis:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "/usr/local/bin/rtpengine-ctl", "-u", "22223", "list"]
      interval: 30s
      timeout: 5s
      retries: 3

  #==========================================
  # CAPA DE SEÑALIZACIÓN (SIP)
  #==========================================
  
  kamailio-1:
    build:
      context: ./kamailio
      dockerfile: Dockerfile
    container_name: kamailio-1
    hostname: kamailio-1
    restart: unless-stopped
    
    networks:
      frontend:
        ipv4_address: 172.32.0.10
      backend:
        ipv4_address: 172.31.0.20
    
    ports:
      - "5060:5060/udp"
      - "5060:5060/tcp"
      - "5061:5061/tcp"
      - "8080:8080/tcp"
    
    volumes:
      - ./config/kamailio/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./config/kamailio/dispatcher.list:/etc/kamailio/dispatcher.list:ro
      - ./logs/kamailio-1:/var/log/kamailio
    
    environment:
      - KAMAILIO_LOG_LEVEL=3
      - KAMAILIO_ALIAS=${SIP_DOMAIN}
      - DB_HOST=mariadb
      - DB_USER=kamailio
      - DB_PASS=${DB_PASSWORD}
      - REDIS_HOST=redis
      - RTPE_SET_1=udp:127.0.0.1:22222
      - RTPE_SET_2=udp:127.0.0.1:22223
      - HOMER_HOST=homer
      - HOMER_PORT=9060
    
    depends_on:
      mariadb:
        condition: service_healthy
      redis:
        condition: service_healthy
      rtpengine-1:
        condition: service_healthy
      rtpengine-2:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "/usr/local/sbin/kamctl", "fifo", "get_statistics", "all"]
      interval: 30s
      timeout: 5s
      retries: 3

  kamailio-2:
    build:
      context: ./kamailio
      dockerfile: Dockerfile
    container_name: kamailio-2
    hostname: kamailio-2
    restart: unless-stopped
    
    networks:
      frontend:
        ipv4_address: 172.32.0.11
      backend:
        ipv4_address: 172.31.0.21
    
    ports:
      - "5070:5060/udp"
      - "5070:5060/tcp"
      - "5071:5061/tcp"
      - "8081:8080/tcp"
    
    volumes:
      - ./config/kamailio/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./config/kamailio/dispatcher.list:/etc/kamailio/dispatcher.list:ro
      - ./logs/kamailio-2:/var/log/kamailio
    
    environment:
      - KAMAILIO_LOG_LEVEL=3
      - KAMAILIO_ALIAS=${SIP_DOMAIN}
      - DB_HOST=mariadb
      - DB_USER=kamailio
      - DB_PASS=${DB_PASSWORD}
      - REDIS_HOST=redis
      - RTPE_SET_1=udp:127.0.0.1:22222
      - RTPE_SET_2=udp:127.0.0.1:22223
      - HOMER_HOST=homer
      - HOMER_PORT=9060
    
    depends_on:
      mariadb:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "/usr/local/sbin/kamctl", "fifo", "get_statistics", "all"]
      interval: 30s
      timeout: 5s
      retries: 3

  kamailio-3:
    build:
      context: ./kamailio
      dockerfile: Dockerfile
    container_name: kamailio-3
    hostname: kamailio-3
    restart: unless-stopped
    
    networks:
      frontend:
        ipv4_address: 172.32.0.12
      backend:
        ipv4_address: 172.31.0.22
    
    ports:
      - "5080:5060/udp"
      - "5080:5060/tcp"
      - "5081:5061/tcp"
      - "8082:8080/tcp"
    
    volumes:
      - ./config/kamailio/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./config/kamailio/dispatcher.list:/etc/kamailio/dispatcher.list:ro
      - ./logs/kamailio-3:/var/log/kamailio
    
    environment:
      - KAMAILIO_LOG_LEVEL=3
      - KAMAILIO_ALIAS=${SIP_DOMAIN}
      - DB_HOST=mariadb
      - DB_USER=kamailio
      - DB_PASS=${DB_PASSWORD}
      - REDIS_HOST=redis
      - RTPE_SET_1=udp:127.0.0.1:22222
      - RTPE_SET_2=udp:127.0.0.1:22223
      - HOMER_HOST=homer
      - HOMER_PORT=9060
    
    depends_on:
      mariadb:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "/usr/local/sbin/kamctl", "fifo", "get_statistics", "all"]
      interval: 30s
      timeout: 5s
      retries: 3

  #==========================================
  # CAPA DE APLICACIONES
  #==========================================
  
  asterisk-1:
    build:
      context: ./asterisk
      dockerfile: Dockerfile
    container_name: asterisk-1
    hostname: asterisk-1
    restart: unless-stopped
    
    networks:
      backend:
        ipv4_address: 172.31.0.30
      media:
        ipv4_address: 172.33.0.10
    
    ports:
      - "5160:5060/udp"
      - "5160:5060/tcp"
      - "10100-10200:10100-10200/udp"
    
    volumes:
      - ./config/asterisk/asterisk-1:/etc/asterisk:ro
      - ./logs/asterisk-1:/var/log/asterisk
      - voicemail:/var/spool/asterisk/voicemail
      - recordings:/var/spool/asterisk/monitor
    
    environment:
      - AST_LOG_LEVEL=verbose
    
    healthcheck:
      test: ["CMD", "asterisk", "-rx", "core show version"]
      interval: 30s
      timeout: 5s
      retries: 3

  asterisk-2:
    build:
      context: ./asterisk
      dockerfile: Dockerfile
    container_name: asterisk-2
    hostname: asterisk-2
    restart: unless-stopped
    
    networks:
      backend:
        ipv4_address: 172.31.0.31
      media:
        ipv4_address: 172.33.0.11
    
    ports:
      - "5170:5060/udp"
      - "5170:5060/tcp"
      - "10200-10300:10200-10300/udp"
    
    volumes:
      - ./config/asterisk/asterisk-2:/etc/asterisk:ro
      - ./logs/asterisk-2:/var/log/asterisk
      - voicemail:/var/spool/asterisk/voicemail
      - recordings:/var/spool/asterisk/monitor
    
    environment:
      - AST_LOG_LEVEL=verbose
    
    healthcheck:
      test: ["CMD", "asterisk", "-rx", "core show version"]
      interval: 30s
      timeout: 5s
      retries: 3

  #==========================================
  # MONITOREO Y CAPTURA
  #==========================================
  
  homer:
    image: sipcapture/homer-app:latest
    container_name: voip-homer
    hostname: homer
    restart: unless-stopped
    
    networks:
      backend:
        ipv4_address: 172.31.0.40
      frontend:
        ipv4_address: 172.32.0.40
    
    ports:
      - "9060:9060/udp"  # HEP capture
      - "9080:9080/tcp"  # Web UI
    
    environment:
      - DB_HOST=mariadb
      - DB_USER=homer
      - DB_PASS=${DB_PASSWORD}
    
    depends_on:
      mariadb:
        condition: service_healthy
    
    volumes:
      - homer_data:/usr/local/homer

  #==========================================
  # LOAD BALANCER (OPCIONAL)
  #==========================================
  
  haproxy:
    image: haproxy:2.8-alpine
    container_name: voip-lb
    hostname: haproxy
    restart: unless-stopped
    
    networks:
      frontend:
        ipv4_address: 172.32.0.50
    
    ports:
      - "5000:5000/tcp"  # SIP TCP load balanced
      - "5000:5000/udp"  # SIP UDP load balanced
      - "9999:9999/tcp"  # HAProxy stats
    
    volumes:
      - ./config/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    
    depends_on:
      - kamailio-1
      - kamailio-2
      - kamailio-3

#==========================================
# VOLUMES
#==========================================

volumes:
  mariadb_data:
    driver: local
  redis_data:
    driver: local
  homer_data:
    driver: local
  recordings:
    driver: local
  voicemail:
    driver: local

#==========================================
# NETWORKS
#==========================================

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.32.0.0/24
    driver_opts:
      com.docker.network.bridge.name: br-voip-front
  
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.31.0.0/24
    driver_opts:
      com.docker.network.bridge.name: br-voip-back
  
  media:
    driver: bridge
    ipam:
      config:
        - subnet: 172.33.0.0/24
    driver_opts:
      com.docker.network.bridge.name: br-voip-media
```

### 2.2 Configuración HAProxy para balanceo SIP

**Archivo: `config/haproxy/haproxy.cfg`**

```
global
    log stdout format raw local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

# Frontend para SIP TCP
frontend sip_tcp_frontend
    bind *:5000
    mode tcp
    default_backend sip_tcp_servers

# Backend Kamailio servers (SIP TCP)
backend sip_tcp_servers
    mode tcp
    balance roundrobin
    option tcp-check
    
    server kamailio-1 172.32.0.10:5060 check inter 2000 rise 2 fall 3
    server kamailio-2 172.32.0.11:5060 check inter 2000 rise 2 fall 3
    server kamailio-3 172.32.0.12:5060 check inter 2000 rise 2 fall 3

# Frontend para SIP UDP (usando socat o similar)
# Nota: HAProxy no balancea UDP nativamente en versiones antiguas

# Stats interface
listen stats
    bind *:9999
    mode http
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
```

### 2.3 Configuración Redis para HA

**Archivo: `config/redis/redis.conf`**

```
# Bind to all interfaces dentro del contenedor
bind 0.0.0.0

# Protección con password
requirepass YourRedisPassword123!

# Persistencia
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec

# Timeouts
timeout 300

# Max clients
maxclients 10000

# Max memory
maxmemory 512mb
maxmemory-policy allkeys-lru

# Logging
loglevel notice
```

### 2.4 Dispatcher list para Kamailio

**Archivo: `config/kamailio/dispatcher.list`**

```
# Dispatcher list for Asterisk servers
# Format: setid destination flags priority attrs

# Set 1: Asterisk servers para media services
1 sip:172.31.0.30:5060 0 50 weight=50;rweight=50
1 sip:172.31.0.31:5060 0 50 weight=50;rweight=50

# Set 2: Backup servers (si existen)
# 2 sip:172.31.0.32:5060 0 0

# Set 10: RTPEngine servers (si dispatcher se usa para media)
# 10 udp:127.0.0.1:22222 0 0
# 10 udp:127.0.0.1:22223 0 0
```

---

## PARTE 3: BALANCEO AVANZADO CON CONTENEDORES (15 minutos)

### 3.1 Estrategias de balanceo

**1. Round Robin (básico):**
```
Kamailio dispatcher con algorithm=0
- Distribuye secuencialmente
- Simple pero efectivo
- No considera carga actual
```

**2. Weight-based (ponderado):**
```
Kamailio dispatcher con pesos
- Servers más potentes reciben más tráfico
- Configuración en dispatcher.list
- weight=50 vs weight=100
```

**3. Least-loaded (menor carga):**
```
Dispatcher con algorithm=10
- Envía a servidor con menos sesiones activas
- Requiere dialog tracking
- Mejor distribución de carga
```

**4. Hash-based (por Call-ID):**
```
Dispatcher con algorithm=4
- Mismo usuario siempre al mismo servidor
- Útil para stateful applications
- Basado en Call-ID o From URI
```

### 3.2 Configuración Kamailio para multi-instancia

**Fragmento de `kamailio.cfg` para dispatcher:**

```
#!KAMAILIO

# Cargar módulos necesarios
loadmodule "dispatcher.so"
loadmodule "dialog.so"
loadmodule "htable.so"

# Configuración dispatcher
modparam("dispatcher", "db_url", "mysql://kamailio:pass@mariadb/kamailio")
modparam("dispatcher", "table_name", "dispatcher")
modparam("dispatcher", "flags", 2)
modparam("dispatcher", "dst_avp", "$avp(dst)")
modparam("dispatcher", "grp_avp", "$avp(grp)")
modparam("dispatcher", "cnt_avp", "$avp(cnt)")
modparam("dispatcher", "attrs_avp", "$avp(attrs)")
modparam("dispatcher", "ds_ping_interval", 30)
modparam("dispatcher", "ds_ping_from", "sip:pinger@kamailio.local")
modparam("dispatcher", "ds_probing_mode", 1)
modparam("dispatcher", "ds_probing_threshold", 2)

# Algorithm: 4 = hash over callid
modparam("dispatcher", "ds_hash_size", 10)

# Dialog tracking para estadísticas
modparam("dialog", "enable_stats", 1)
modparam("dialog", "dlg_flag", 4)

# Hash table para tracking
modparam("htable", "htable", "servers=>size=8;autoexpire=300")

# Routing logic
request_route {
    
    # ... verificaciones previas ...
    
    # INVITE hacia Asterisk
    if (is_method("INVITE") && !has_totag()) {
        
        # Set dialog flag
        setflag(4);
        
        # Dispatch a Set 1 (Asterisk servers)
        # Algorithm 4: hash over callid
        if (!ds_select_dst("1", "4")) {
            xlog("L_ERR", "No Asterisk servers available\n");
            sl_send_reply("503", "Service Unavailable");
            exit;
        }
        
        # Store destination info
        $avp(dst_addr) = $du;
        xlog("L_INFO", "Dispatching to: $du\n");
        
        # Set reply route para failover
        t_on_failure("DISPATCH_FAILURE");
        
        # Relay
        route(RELAY);
        exit;
    }
    
    # ... resto del routing ...
}

# Failover route
failure_route[DISPATCH_FAILURE] {
    
    if (t_is_canceled()) {
        exit;
    }
    
    # Si el destino falló, probar siguiente
    if (t_check_status("408|503")) {
        xlog("L_WARN", "Server $du failed, trying next\n");
        
        # Marcar destino como inactivo
        ds_mark_dst("i");
        
        # Seleccionar siguiente destino
        if (ds_next_dst()) {
            xlog("L_INFO", "Trying failover to: $du\n");
            t_relay();
            exit;
        } else {
            xlog("L_ERR", "All servers failed\n");
            send_reply("503", "All Servers Unavailable");
            exit;
        }
    }
}

# Route para RTPEngine balancing
route[RTPENGINE_SELECT] {
    
    # Balancear entre RTPEngine instances
    # Usando hash sobre Call-ID
    $var(rtpe_idx) = crc32("$ci") mod 2;
    
    if ($var(rtpe_idx) == 0) {
        # RTPEngine 1
        rtpengine_offer("codec-mask-all codec-transcode-PCMA");
    } else {
        # RTPEngine 2
        set_rtpengine_set("1");
        rtpengine_offer("codec-mask-all codec-transcode-PCMA");
    }
}

# Event route para dispatcher
event_route[dispatcher:dst-down] {
    xlog("L_WARN", "Destination down: $rm $ru ($du)\n");
    
    # Notificar a sistema de monitoreo
    # ...
}

event_route[dispatcher:dst-up] {
    xlog("L_INFO", "Destination up: $rm $ru ($du)\n");
    
    # Notificar a sistema de monitoreo
    # ...
}
```

### 3.3 Monitoreo de carga distribuida

**Script de monitoreo: `monitor-cluster.sh`**

```bash
#!/bin/bash

# Colores
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

# Función para obtener estadísticas de Kamailio
get_kamailio_stats() {
    local container=$1
    local stats=$(docker exec $container kamctl fifo get_statistics dialog: 2>/dev/null)
    local active=$(echo "$stats" | grep "active_dialogs" | awk '{print $3}')
    echo ${active:-0}
}

# Función para obtener estadísticas de RTPEngine
get_rtpengine_stats() {
    local port=$1
    local stats=$(docker exec rtpengine-1 rtpengine-ctl -u $port list 2>/dev/null)
    local sessions=$(echo "$stats" | grep -c "call-id")
    echo ${sessions:-0}
}

# Función para obtener CPU/RAM
get_container_resources() {
    local container=$1
    docker stats $container --no-stream --format "{{.CPUPerc}}\t{{.MemUsage}}" 2>/dev/null || echo "0%\t0B/0B"
}

# Loop de monitoreo
while true; do
    clear
    echo -e "${GREEN}╔════════════════════════════════════════════════════════════╗${NC}"
    echo -e "${GREEN}║         VOIP CLUSTER STATUS - $(date +%H:%M:%S)                  ║${NC}"
    echo -e "${GREEN}╚════════════════════════════════════════════════════════════╝${NC}"
    
    # Kamailio Instances
    echo -e "\n${YELLOW}KAMAILIO INSTANCES:${NC}"
    echo "┌──────────────┬──────────────┬────────────────────────────┐"
    echo "│ Instance     │ Active Calls │ CPU / Memory               │"
    echo "├──────────────┼──────────────┼────────────────────────────┤"
    
    for i in 1 2 3; do
        container="kamailio-$i"
        calls=$(get_kamailio_stats $container)
        resources=$(get_container_resources $container)
        
        if [ "$calls" -gt 50 ]; then
            color=$RED
        elif [ "$calls" -gt 20 ]; then
            color=$YELLOW
        else
            color=$GREEN
        fi
        
        printf "│ %-12s │ ${color}%-12s${NC} │ %-26s │\n" "$container" "$calls" "$resources"
    done
    echo "└──────────────┴──────────────┴────────────────────────────┘"
    
    # RTPEngine Instances
    echo -e "\n${YELLOW}RTPENGINE INSTANCES:${NC}"
    echo "┌──────────────┬──────────────┬────────────────────────────┐"
    echo "│ Instance     │ Active Sess  │ CPU / Memory               │"
    echo "├──────────────┼──────────────┼────────────────────────────┤"
    
    for i in 1 2; do
        container="rtpengine-$i"
        port=$((22221 + i))
        sessions=$(get_rtpengine_stats $port)
        resources=$(get_container_resources $container)
        
        if [ "$sessions" -gt 100 ]; then
            color=$RED
        elif [ "$sessions" -gt 50 ]; then
            color=$YELLOW
        else
            color=$GREEN
        fi
        
        printf "│ %-12s │ ${color}%-12s${NC} │ %-26s │\n" "$container" "$sessions" "$resources"
    done
    echo "└──────────────┴──────────────┴────────────────────────────┘"
    
    # Asterisk Instances
    echo -e "\n${YELLOW}ASTERISK INSTANCES:${NC}"
    echo "┌──────────────┬────────────────────────────────────────────┐"
    echo "│ Instance     │ CPU / Memory                               │"
    echo "├──────────────┼────────────────────────────────────────────┤"
    
    for i in 1 2; do
        container="asterisk-$i"
        resources=$(get_container_resources $container)
        printf "│ %-12s │ %-42s │\n" "$container" "$resources"
    done
    echo "└──────────────┴────────────────────────────────────────────┘"
    
    # Database & Cache
    echo -e "\n${YELLOW}BACKEND SERVICES:${NC}"
    echo "┌──────────────┬────────────────────────────────────────────┐"
    echo "│ Service      │ CPU / Memory                               │"
    echo "├──────────────┼────────────────────────────────────────────┤"
    
    for service in mariadb redis; do
        container="voip-$service"
        resources=$(get_container_resources $container)
        printf "│ %-12s │ %-42s │\n" "$service" "$resources"
    done
    echo "└──────────────┴────────────────────────────────────────────┘"
    
    # Totals
    total_calls=0
    for i in 1 2 3; do
        calls=$(get_kamailio_stats "kamailio-$i")
        total_calls=$((total_calls + calls))
    done
    
    echo -e "\n${GREEN}TOTAL ACTIVE CALLS: $total_calls${NC}"
    
    sleep 3
done
```

---

## PARTE 4: ESCALADO HORIZONTAL (10 minutos)

### 4.1 Escalado automático con Docker Compose

```bash
# Escalar Kamailio a 5 instancias
docker compose up -d --scale kamailio=5

# Escalar RTPEngine a 4 instancias
docker compose up -d --scale rtpengine=4

# Ver instancias
docker compose ps
```

### 4.2 Limitación de recursos por contenedor

```yaml
services:
  kamailio-1:
    # ... configuración ...
    
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    
    # Restart policy
    restart: on-failure
    restart: max-restart-count: 3
```

### 4.3 Health checks avanzados

```yaml
healthcheck:
  test: |
    /bin/bash -c '
    # Check if Kamailio is responding
    kamctl fifo get_statistics all > /dev/null || exit 1
    
    # Check active connections
    CONNS=$(kamctl fifo get_statistics tcp: | grep current_opened_connections | awk "{print $3}")
    
    # Fail if no connections for too long (graceful degradation)
    # [ "$CONNS" -gt 0 ] || exit 1
    
    # All good
    exit 0
    '
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

---

## RESUMEN Y PRÓXIMA SESIÓN

### ✅ Lo que cubrimos:

```
□ Arquitectura de microservicios VoIP
□ Docker Compose multi-servicio
□ Balanceo de carga con dispatcher
□ HAProxy para load balancing
□ Monitoreo de cluster
□ Escalado horizontal
□ Limitación de recursos
```

### 📋 Checklist de producción:

```
□ Separar redes (frontend/backend/media)
□ Health checks en todos los servicios
□ Resource limits configurados
□ Logging centralizado
□ Monitoreo activo
□ Failover automático configurado
□ Backup de volumes
□ Secrets en archivos .env
```

---

**FIN VIDEOCONFERENCIA 3 - PARTE DOCKER**

**Próxima sesión:** Alta disponibilidad con Docker Swarm y Kubernetes
