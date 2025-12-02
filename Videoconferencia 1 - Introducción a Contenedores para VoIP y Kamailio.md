# VIDEOCONFERENCIA 1
## Introducción a Contenedores para VoIP y Kamailio

---

## PARTE 1: INTRODUCCIÓN A DOCKER PARA VOIP (20 minutos)

### 1.1 ¿Qué es Docker y por qué usarlo con Kamailio?

**Conceptos básicos:**
- **Contenedor**: Entorno aislado que incluye la aplicación y todas sus dependencias
- **Imagen**: Plantilla inmutable para crear contenedores
- **Dockerfile**: Receta para construir una imagen
- **Volume**: Almacenamiento persistente fuera del contenedor
- **Network**: Red virtual para comunicación entre contenedores

**Ventajas para aplicaciones VoIP:**
```
✅ Portabilidad: "Funciona en mi máquina" = Funciona en todas
✅ Aislamiento: Múltiples versiones de Kamailio en el mismo servidor
✅ Escalabilidad: Crear/destruir instancias rápidamente
✅ Reproducibilidad: Mismo entorno en dev, testing y producción
✅ Versionado: Imágenes etiquetadas por versión
✅ CI/CD: Integración con pipelines de despliegue automatizado
```

**Casos de uso reales:**
1. Testing de configuraciones sin afectar producción
2. Despliegue rápido de nuevos nodos Kamailio
3. Desarrollo local sin instalar dependencias en host
4. Clusters de alta disponibilidad
5. Ambientes multi-tenant (varios clientes aislados)

**Consideraciones importantes para VoIP:**
```
⚠️ RENDIMIENTO RTP:
   - Docker añade overhead mínimo (~2-3%)
   - Network mode "host" elimina casi todo overhead
   - Para producción con alto tráfico: host mode recomendado

⚠️ PUERTOS:
   - SIP: 5060 UDP/TCP, 5061 TLS
   - RTP: Rango amplio (10000-20000 típicamente)
   - Gestión: 8080 (HTTP), 9060 (Binrpc)

⚠️ PERSISTENCIA:
   - Configuración debe estar en volumes
   - Logs deben persistir fuera del contenedor
   - Base de datos en contenedor separado con volume
```

### 1.2 Instalación de Docker

**En AlmaLinux 9 / Rocky Linux 9:**

```bash
# Actualizar sistema
sudo dnf update -y

# Instalar dependencias
sudo dnf install -y yum-utils device-mapper-persistent-data lvm2

# Agregar repositorio Docker
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Instalar Docker
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Iniciar y habilitar Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verificar instalación
sudo docker --version
sudo docker compose version

# Agregar usuario al grupo docker (evitar sudo)
sudo usermod -aG docker $USER
newgrp docker

# Test
docker run hello-world
```

**Comandos Docker esenciales:**

```bash
# IMÁGENES
docker images                    # Listar imágenes
docker pull kamailio/kamailio-ci # Descargar imagen
docker build -t mi-kamailio .    # Construir imagen
docker rmi imagen_id             # Eliminar imagen
docker image prune               # Limpiar imágenes sin usar

# CONTENEDORES
docker ps                        # Listar contenedores activos
docker ps -a                     # Listar todos (incluso detenidos)
docker run nombre_imagen         # Crear y ejecutar contenedor
docker start contenedor_id       # Iniciar contenedor existente
docker stop contenedor_id        # Detener contenedor
docker restart contenedor_id     # Reiniciar contenedor
docker rm contenedor_id          # Eliminar contenedor
docker logs contenedor_id        # Ver logs
docker logs -f contenedor_id     # Seguir logs en tiempo real
docker exec -it contenedor bash  # Ejecutar comando en contenedor

# VOLUMES
docker volume ls                 # Listar volumes
docker volume create nombre      # Crear volume
docker volume inspect nombre     # Inspeccionar volume
docker volume rm nombre          # Eliminar volume

# NETWORKS
docker network ls                # Listar redes
docker network create nombre     # Crear red
docker network inspect nombre    # Inspeccionar red
```

---

## PARTE 2: KAMAILIO EN DOCKER - INSTALACIÓN NATIVA VS DOCKER (15 minutos)

### 2.1 Comparativa

| Aspecto | Instalación Nativa | Docker |
|---------|-------------------|--------|
| **Instalación** | 30-45 min (compilación) | 5 min (pull image) |
| **Dependencias** | Manual, conflictos posibles | Incluidas en imagen |
| **Múltiples versiones** | Difícil (conflictos) | Fácil (diferentes images) |
| **Respaldo** | Archivos dispersos | Imagen completa |
| **Portabilidad** | Dependiente del OS | Funciona en cualquier host Docker |
| **Rendimiento** | 100% | 97-99% (host mode ~99.5%) |
| **Complejidad inicial** | Media | Media-Alta |
| **Escalabilidad** | Manual | Automatizable |
| **Debugging** | Directo | A través de docker exec |

### 2.2 Cuándo usar cada opción

**Usar instalación nativa cuando:**
- Sistema legacy sin posibilidad de contenedores
- Máximo rendimiento crítico (aunque diferencia es mínima)
- Equipo sin experiencia Docker
- Integración profunda con sistema operativo host

**Usar Docker cuando:**
- Necesitas múltiples entornos (dev, test, prod)
- Despliegues frecuentes
- Clusters y alta disponibilidad
- CI/CD automatizado
- Ambientes multi-tenant
- Testing de configuraciones

---

## PARTE 3: DOCKERFILE PARA KAMAILIO (25 minutos)

### 3.1 Preparación: Crear estructura de directorios

**IMPORTANTE: Hacer esto ANTES de crear los Dockerfiles**

```bash
# Crear estructura de proyecto
mkdir -p ~/kamailio-docker/{config,logs,scripts}
cd ~/kamailio-docker

# Verificar estructura
tree -L 1
# Salida esperada:
# .
# ├── config
# ├── logs
# └── scripts
```

### 3.2 Dockerfile Básico

**Archivo: `Dockerfile.kamailio`**

```dockerfile
# Imagen base - AlmaLinux 9
FROM almalinux:9

# Metadata
LABEL maintainer="campus@mesaproyectos.com"
LABEL description="Kamailio 6.0.x SIP Server"
LABEL version="1.0"

# Variables de construccion
ARG KAMAILIO_VERSION=6.0
ARG KAMAILIO_BUILD=kamailio60

# Instalar dependencias
RUN dnf install -y epel-release && \
    dnf config-manager --set-enabled crb && \
    dnf install -y \
    # Compilación
    gcc gcc-c++ make bison flex \
    # Librerías
    openssl-devel libcurl-devel \
    mysql-devel postgresql-devel \
    pcre-devel expat-devel \
    libxml2-devel libunistring-devel \
    json-c-devel libevent-devel \
    # Utilidades
    git wget vim net-tools \
    # Limpiar cache
    && dnf clean all

# Crear usuario kamailio
RUN useradd -r -s /bin/false kamailio

# Descargar y compilar Kamailio
WORKDIR /usr/local/src

RUN git clone --depth 1 --branch ${KAMAILIO_VERSION} https://github.com/kamailio/kamailio.git kamailio && \
    cd kamailio && \
    make cfg && \
    # Incluir modulos importantes
    make include_modules="db_mysql db_postgres tls websocket dmq presence presence_xml debugger htable pike \
                          dispatcher dialog nathelper rtpengine usrloc registrar auth auth_db \
                          sanity textops siputils tm sl rr maxfwd jsonrpcs xlog corex secsipid" \
    cfg && \
    make all && \
    make install && \
    # Limpiar archivos de compilacion
    cd .. && rm -rf kamailio

# Crear directorios necesarios
RUN mkdir -p /etc/kamailio \
    /var/run/kamailio \
    /var/log/kamailio && \
    chown -R kamailio:kamailio /var/run/kamailio /var/log/kamailio

# Copiar configuracion básica (sera reemplazada por volume)
COPY config/kamailio.cfg /etc/kamailio/kamailio.cfg
RUN chown kamailio:kamailio /etc/kamailio/kamailio.cfg

# Exponer puertos
EXPOSE 5060/udp 5060/tcp 5061/tcp 8080/tcp

# Variables de entorno
ENV KAMAILIO_LISTEN_IP=0.0.0.0
ENV KAMAILIO_LISTEN_PORT=5060
ENV KAMAILIO_LOG_LEVEL=3

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD /usr/local/sbin/kamctl fifo get_statistics all || exit 1

# Usuario no privilegiado
USER kamailio

# Comando por defecto
CMD ["/usr/local/sbin/kamailio", "-DD", "-E", "-f", "/etc/kamailio/kamailio.cfg"]
```

### 3.3 Dockerfile Optimizado (Multi-stage build)

**Archivo: `Dockerfile.kamailio-optimized`**

```dockerfile
# ETAPA 1: Compilación
FROM almalinux:9 AS builder

ARG KAMAILIO_VERSION=6.0

# Instalar dependencias de compilación
RUN dnf install -y epel-release && \
    dnf config-manager --set-enabled crb && \
    dnf install -y \
    # Compilación
    gcc gcc-c++ make bison flex \
    # Librerías
    openssl-devel libcurl-devel \
    mysql-devel postgresql-devel \
    pcre-devel expat-devel \
    libxml2-devel libunistring-devel \
    json-c-devel libevent-devel \
    # Utilidades
    git wget vim net-tools

# Compilar Kamailio
WORKDIR /usr/local/src
RUN git clone --depth 1 --branch ${KAMAILIO_VERSION} https://github.com/kamailio/kamailio.git && \
    cd kamailio && \
    make cfg && \
    make include_modules="db_mysql db_postgres tls websocket dmq presence presence_xml debugger htable pike \
                          dispatcher dialog nathelper rtpengine usrloc registrar auth auth_db \
                          sanity textops siputils tm sl rr maxfwd jsonrpcs xlog corex secsipid" \
    cfg && \
    make all && \
    make install

# ETAPA 2: Imagen final (solo runtime)
FROM almalinux:9

# Metadata
LABEL maintainer="campus@mesaproyectos.com"
LABEL description="Kamailio 6.0.x SIP Server - Optimized"

# Instalar solo dependencias de runtime (mas liviano)
RUN dnf install -y epel-release && \
    dnf install -y \
    openssl mariadb-connector-c postgresql-libs \
    pcre expat libxml2 libunistring json-c libevent \
    net-tools procps-ng && \
    dnf clean all

# Copiar binarios compilados desde builder
COPY --from=builder /usr/local/sbin/kamailio /usr/local/sbin/
COPY --from=builder /usr/local/sbin/kamctl /usr/local/sbin/
COPY --from=builder /usr/local/sbin/kamcmd /usr/local/sbin/
COPY --from=builder /usr/local/lib64/kamailio /usr/local/lib64/kamailio

# Crear usuario y directorios
RUN useradd -r -s /bin/false kamailio && \
    mkdir -p /etc/kamailio /var/run/kamailio /var/log/kamailio && \
    chown -R kamailio:kamailio /var/run/kamailio /var/log/kamailio

# Volumenes
VOLUME ["/etc/kamailio", "/var/log/kamailio"]

# Puertos
EXPOSE 5060/udp 5060/tcp 5061/tcp 8080/tcp

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD /usr/local/sbin/kamctl fifo get_statistics all || exit 1

USER kamailio

CMD ["/usr/local/sbin/kamailio", "-DD", "-E", "-f", "/etc/kamailio/kamailio.cfg"]
```

### 3.4 Configuración básica de Kamailio

**IMPORTANTE: Crear este archivo ANTES de construir las imágenes**

**Archivo: `config/kamailio.cfg` (simplificado para Docker)**

```
#!KAMAILIO

####### Global Parameters #########
debug=3
log_stderror=yes
memdbg=5
memlog=5
log_facility=LOG_LOCAL0

children=4
tcp_children=4

# Escuchar en todas las interfaces
listen=udp:0.0.0.0:5060
listen=tcp:0.0.0.0:5060

# Alias
alias="kamailio.local"

####### Modules Section ########
mpath="/usr/local/lib64/kamailio/modules/"

loadmodule "jsonrpcs.so"
loadmodule "kex.so"
loadmodule "corex.so"
loadmodule "tm.so"
loadmodule "tmx.so"
loadmodule "sl.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "maxfwd.so"
loadmodule "usrloc.so"
loadmodule "registrar.so"
loadmodule "textops.so"
loadmodule "siputils.so"
loadmodule "xlog.so"
loadmodule "sanity.so"
loadmodule "ctl.so"
loadmodule "cfg_rpc.so"

####### Routing Logic ########
request_route {
    
    # Log inicial
    xlog("L_INFO", "New $rm from $fu to $ru (IP:$si:$sp)\n");
    
    # Sanity checks
    if (!sanity_check()) {
        xlog("L_WARN", "Malformed SIP message from $si:$sp\n");
        exit;
    }
    
    # Max-Forwards
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }
    
    # Record routing para dialogos
    if (is_method("INVITE|SUBSCRIBE")) {
        record_route();
    }
    
    # Handle requests dentro de dialogos
    if (has_totag()) {
        if (loose_route()) {
            route(RELAY);
            exit;
        }
    }
    
    # REGISTER requests
    if (is_method("REGISTER")) {
        save("location");
        exit;
    }
    
    # Lookup location
    if (!lookup("location")) {
        sl_send_reply("404", "Not Found");
        exit;
    }
    
    route(RELAY);
}

route[RELAY] {
    if (!t_relay()) {
        sl_reply_error();
    }
    exit;
}
```

**Nota:** Si necesitas módulos NAT (nathelper, rtpengine), agrégalos después de validar el funcionamiento básico.

### 3.5 Construir las imágenes

**Ahora sí, con el kamailio.cfg ya creado, construimos:**

```bash
# Construccion básica
docker build -f Dockerfile.kamailio -t mi-kamailio:6.0 .

# Construccion optimizada (RECOMENDADA)
docker build -f Dockerfile.kamailio-optimized -t mi-kamailio:6.0-optimized .

# Con argumentos personalizados
docker build \
    --build-arg KAMAILIO_VERSION=6.0 \
    -t mi-kamailio:6.0-custom \
    -f Dockerfile.kamailio-optimized .

# Ver imagenes creadas
docker images | grep kamailio

# Ver tamaño de imagen
docker images mi-kamailio:6.0-optimized --format "{{.Size}}"
```

**Comparación de tamaños esperados:**
- Dockerfile básico: ~800MB
- Dockerfile optimizado: ~400MB

---

## PARTE 4: DOCKER COMPOSE PARA KAMAILIO (15 minutos)

### 4.1 Archivo .env para variables

**IMPORTANTE: Crear este archivo ANTES de los docker-compose**

**Archivo: `.env`**

# IMPORTANTE: Este archivo contiene credenciales sensibles
# - NO subir a repositorios Git (agregar a .gitignore)
# - Usar permisos restrictivos: chmod 600 .env
# - En produccion, usar secrets management (Docker Secrets, Vault)

```bash
cat > .env <<EOF
# Base de datos
DB_ROOT_PASSWORD=SuperSecureRootPass123!
DB_NAME=kamailio
DB_USER=kamailio
DB_PASS=KamailioSecurePass456!

# Kamailio
KAMAILIO_LOG_LEVEL=3
KAMAILIO_DOMAIN=$HOSTNAME
KAMAILIO_EXTERNAL_IP=$(curl -s ifconfig.me)

# Timezone
TZ=America/Bogota

# Versione
KAMAILIO_VERSION=6.0
MARIADB_VERSION=10.11
EOF

Luego:

chmod 600 .env

```



### 4.2 Docker Compose Básico (Network Host Mode)

**Archivo: `docker-compose.yml`**

```yaml
services:
  kamailio:
    image: mi-kamailio:6.0-optimized
    container_name: kamailio-server
    hostname: kamailio
    restart: unless-stopped
    
    # Network mode host para mejor rendimiento SIP/RTP
    network_mode: host
    
    volumes:
      # Configuracion persistente
      - ./config/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      # Logs persistentes
      - ./logs:/var/log/kamailio
      # Scripts personalizados
      - ./scripts:/opt/scripts:ro
    
    environment:
      - KAMAILIO_LOG_LEVEL=${KAMAILIO_LOG_LEVEL}
      - TZ=${TZ}
    
    # Capabilities para binding a puertos < 1024
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Base de datos MariaDB
  mariadb:
    image: mariadb:${MARIADB_VERSION}
    container_name: kamailio-db
    hostname: mariadb
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASS}
      TZ: ${TZ}
    
    volumes:
      - mariadb_data:/var/lib/mysql
    
    ports:
      - "3306:3306"
    
    command: 
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --max_connections=500
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "${DB_USER}", "-p${DB_PASS}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mariadb_data:
    driver: local

networks:
  default:
    name: kamailio-network
```

### 4.3 Docker Compose con network bridge (producción recomendada)

**Archivo: `docker-compose-bridge.yml`**

```yaml
services:
  kamailio:
    image: mi-kamailio:6.0-optimized
    command: ["/usr/local/sbin/kamailio", "-DD", "-E", "-f", "/etc/kamailio/kamailio.cfg"]
    container_name: kamailio-server
    hostname: kamailio.local
    restart: unless-stopped
    
    networks:
      kamailio-net:
        ipv4_address: 172.20.0.10
    
    ports:
      # SIP
      - "5060:5060/udp"
      - "5060:5060/tcp"
      - "5061:5061/tcp"
      # HTTP/JSONRPC
      - "8080:8080/tcp"
      # RTP range (ejemplo reducido, en produccion ampliar)
      - "10000-10100:10000-10100/udp"
    
    volumes:
      - ./config/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./logs/kamailio:/var/log/kamailio
    
    environment:
      - KAMAILIO_LOG_LEVEL=${KAMAILIO_LOG_LEVEL}
      - DB_HOST=mariadb
      - DB_USER=${DB_USER}
      - DB_PASS=${DB_PASS}
      - DB_NAME=${DB_NAME}
      - TZ=${TZ}
    
    depends_on:
      mariadb:
        condition: service_healthy
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  mariadb:
    image: mariadb:${MARIADB_VERSION}
    container_name: kamailio-db
    hostname: mariadb
    restart: unless-stopped
    
    networks:
      kamailio-net:
        ipv4_address: 172.20.0.20
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASS}
      TZ: ${TZ}
    
    volumes:
      - mariadb_data:/var/lib/mysql
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mariadb_data:

networks:
  kamailio-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

### 4.4 Comandos Docker Compose

```bash
# Levantar servicios (si el archivo se llama docker-compose.yml)
docker compose up -d

# Levantar servicios (con nombre personalizado)
docker compose -f docker-compose-bridge.yml up -d

# Ver logs
docker compose logs -f kamailio
docker compose -f docker-compose-bridge.yml logs -f kamailio

# Ver logs de MariaDB
docker compose logs -f mariadb
docker compose -f docker-compose-bridge.yml logs -f mariadb

# Ver estado de servicios
docker compose ps
docker compose -f docker-compose-bridge.yml ps

# Detener servicios
docker compose stop
docker compose -f docker-compose-bridge.yml stop

# Detener y eliminar contenedores
docker compose down
docker compose -f docker-compose-bridge.yml down

# Detener, eliminar contenedores Y volumes (CUIDADO: borra la BD)
docker compose down -v
docker compose -f docker-compose-bridge.yml down -v

# Reconstruir imágenes
docker compose build --no-cache

# Ejecutar comando en contenedor
docker compose exec kamailio kamctl fifo get_statistics all
docker compose exec kamailio kamcmd stats.get_statistics all

# Entrar al contenedor
docker compose exec kamailio bash
docker compose exec mariadb mysql -u kamailio -p${DB_PASS}

# Ver usuarios registrados
docker compose exec kamailio kamctl ul show
```

---

## PARTE 5: EJERCICIO PRÁCTICO (25 minutos)

### Ejercicio 1: Desplegar Kamailio básico con Docker

**Objetivo:** Levantar un servidor Kamailio containerizado con registro de usuarios

**Resumen del flujo completo:**

```bash
# 1. Crear estructura
mkdir -p ~/kamailio-docker/{config,logs,scripts}
cd ~/kamailio-docker

# 2. Crear archivos de configuración
# - Crear config/kamailio.cfg (sección 3.4)
# - Crear .env (sección 4.1)
# - Crear Dockerfile.kamailio-optimized (sección 3.3)

# 3. Construir imagen
docker build -f Dockerfile.kamailio-optimized -t mi-kamailio:6.0-optimized .

# 4. Crear docker-compose-bridge.yml (sección 4.3)

# 5. Levantar servicios
docker compose -f docker-compose-bridge.yml up -d

# 6. Verificar logs
docker compose -f docker-compose-bridge.yml logs -f
```

**Verificar funcionamiento:**

```bash
# Ver estadísticas de Kamailio
docker compose -f docker-compose-bridge.yml exec kamailio kamctl fifo get_statistics all

# Ver usuarios registrados
docker compose -f docker-compose-bridge.yml exec kamailio kamctl ul show

# Ver logs en tiempo real
docker compose -f docker-compose-bridge.yml logs -f kamailio

# Verificar conectividad a MariaDB
docker compose -f docker-compose-bridge.yml exec mariadb mysql -u kamailio -p${DB_PASS} -e "SHOW DATABASES;"
```

**Registrar un softphone:**
- Configurar un softphone (Zoiper, Linphone, etc.)
- Server: IP del host (144.202.68.137)
- Puerto: 5060
- Usuario: 1001@kamailio.local
- Password: (sin autenticación por ahora)

### Ejercicio 2: Agregar base de datos y autenticación

**Objetivo:** Integrar MariaDB para persistencia de usuarios con autenticación

**Pasos:**

1. **Crear script de inicialización de BD:**

**Archivo: `scripts/init-db.sql`**

```sql
-- Crear tablas básicas Kamailio
CREATE TABLE IF NOT EXISTS subscriber (
    id INT(10) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(64) NOT NULL DEFAULT '',
    domain VARCHAR(64) NOT NULL DEFAULT '',
    password VARCHAR(64) NOT NULL DEFAULT '',
    email_address VARCHAR(64) NOT NULL DEFAULT '',
    ha1 VARCHAR(64) NOT NULL DEFAULT '',
    ha1b VARCHAR(64) NOT NULL DEFAULT '',
    rpid VARCHAR(64) DEFAULT NULL,
    UNIQUE KEY account_idx (username, domain)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS location (
    id INT(10) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    ruid VARCHAR(64) NOT NULL DEFAULT '',
    username VARCHAR(64) NOT NULL DEFAULT '',
    domain VARCHAR(64) DEFAULT NULL,
    contact VARCHAR(512) NOT NULL DEFAULT '',
    received VARCHAR(128) DEFAULT NULL,
    path VARCHAR(512) DEFAULT NULL,
    expires DATETIME NOT NULL DEFAULT '2030-05-28 21:32:15',
    q FLOAT(10,2) NOT NULL DEFAULT 1.00,
    callid VARCHAR(255) NOT NULL DEFAULT 'Default-Call-ID',
    cseq INT(11) NOT NULL DEFAULT 1,
    last_modified DATETIME NOT NULL DEFAULT '2000-01-01 00:00:01',
    flags INT(11) NOT NULL DEFAULT 0,
    cflags INT(11) NOT NULL DEFAULT 0,
    user_agent VARCHAR(255) NOT NULL DEFAULT '',
    socket VARCHAR(64) DEFAULT NULL,
    methods INT(11) DEFAULT NULL,
    instance VARCHAR(255) DEFAULT NULL,
    reg_id INT(11) NOT NULL DEFAULT 0,
    server_id INT(11) NOT NULL DEFAULT 0,
    connection_id INT(11) NOT NULL DEFAULT 0,
    keepalive INT(11) NOT NULL DEFAULT 0,
    partition INT(11) NOT NULL DEFAULT 0,
    KEY account_contact_idx (username, domain, contact)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Insertar usuarios de prueba
INSERT INTO subscriber (username, domain, password, ha1, ha1b) VALUES
('1001', 'kamailio.local', 'test1001', MD5('1001:kamailio.local:test1001'), MD5('1001@kamailio.local:kamailio.local:test1001')),
('1002', 'kamailio.local', 'test1002', MD5('1002:kamailio.local:test1002'), MD5('1002@kamailio.local:kamailio.local:test1002')),
('1003', 'kamailio.local', 'test1003', MD5('1003:kamailio.local:test1003'), MD5('1003@kamailio.local:kamailio.local:test1003'));
```

2. **Añadir el script al docker-compose-bridge.yml:**

En la sección de `mariadb`, añadir:

```yaml
volumes:
  - mariadb_data:/var/lib/mysql
  - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
```

3. **Modificar kamailio.cfg para usar DB:**

```
# Agregar módulos de DB (después de los módulos existentes)
loadmodule "db_mysql.so"
loadmodule "auth.so"
loadmodule "auth_db.so"

# Configurar conexión DB
modparam("auth_db", "db_url", "mysql://kamailio:kamailiopass@mariadb/kamailio")
modparam("usrloc", "db_url", "mysql://kamailio:kamailiopass@mariadb/kamailio")
modparam("usrloc", "db_mode", 2)

# Modificar la sección REGISTER en request_route:
# REGISTER requests
if (is_method("REGISTER")) {
    # Autenticación
    if (!auth_check("$fd", "subscriber", "1")) {
        auth_challenge("$fd", "0");
        exit;
    }
    save("location");
    exit;
}
```

4. **Reconstruir imagen con nuevos módulos:**

```bash
docker build -f Dockerfile.kamailio-optimized -t mi-kamailio:6.0-optimized .
```

5. **Levantar stack completo:**

```bash
docker compose -f docker-compose-bridge.yml down
docker compose -f docker-compose-bridge.yml up -d
docker compose -f docker-compose-bridge.yml logs -f
```

6. **Verificar usuarios en BD:**

```bash
docker compose -f docker-compose-bridge.yml exec mariadb mysql -u kamailio -p${DB_PASS} -e "USE kamailio; SELECT username, domain FROM subscriber;"
```

7. **Registrar softphone con autenticación:**
- Usuario: 1001
- Dominio: kamailio.local
- Password: test1001

---

## RESUMEN Y MEJORES PRÁCTICAS

### ✅ Checklist de producción:

```
□ Usar multi-stage build para imágenes más ligeras
□ Network mode "host" para alta carga RTP
□ Volumes para configuración y logs
□ Variables de entorno para parametrización
□ Healthchecks configurados
□ Límites de recursos (CPU, memoria)
□ Logging estructurado (JSON)
□ Rotación de logs configurada
□ Secrets en archivos .env (no en código)
□ Usuario no-root en contenedor
□ Imágenes etiquetadas con versión
□ Backup de volumes configurado
```

### 🔥 Errores comunes a evitar:

```
❌ Crear Dockerfile antes que kamailio.cfg (build fallará)
❌ No crear estructura de carpetas (config/, logs/, scripts/)
❌ Exponer puertos innecesarios
❌ Credenciales hardcodeadas en Dockerfile
❌ No usar .dockerignore (imágenes grandes)
❌ No limitar rango RTP (mapear todo 10000-60000)
❌ Configuración dentro del contenedor (no en volume)
❌ No configurar healthchecks
❌ Logs dentro del contenedor (se pierden)
❌ Usar tag "latest" en producción
❌ Olvidar sobrescribir command: en docker-compose para bridge mode
```

### 📋 Orden correcto de ejecución:

```
1. Crear estructura de carpetas (mkdir -p config logs scripts)
2. Crear config/kamailio.cfg
3. Crear .env
4. Crear Dockerfiles
5. Construir imágenes (docker build)
6. Crear docker-compose.yml
7. Levantar servicios (docker compose up -d)
```

### 📚 Recursos adicionales:

- Docker docs: https://docs.docker.com
- Kamailio Docker Hub: https://hub.docker.com/u/kamailio
- Docker Compose reference: https://docs.docker.com/compose/compose-file/
- Kamailio Wiki: https://www.kamailio.org/wiki/

---

**FIN VIDEOCONFERENCIA 1 - PARTE DOCKER**

**Próxima sesión:** RTPEngine en Docker (multi-instancia)
