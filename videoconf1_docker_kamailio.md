# VIDEOCONFERENCIA 1 - CONTENIDO DOCKER
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

### 3.1 Dockerfile Básico

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
    gcc gcc-c++ make bison flex \
    openssl-devel curl-devel \
    mysql-devel postgresql-devel \
    pcre-devel expat-devel \
    libxml2-devel libunistring-devel \
    json-c-devel libevent-devel \
    git wget vim net-tools \
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
                          sanity textops siputils tm sl rr maxfwd jsonrpcs xlog corex" \
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
COPY kamailio.cfg /etc/kamailio/kamailio.cfg
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

### 3.2 Dockerfile Optimizado (Multi-stage build)

**Archivo: `Dockerfile.kamailio-optimized`**

```dockerfile
# ETAPA 1: Compilación
FROM almalinux:9 AS builder

ARG KAMAILIO_VERSION=6.0

# Instalar dependencias
RUN dnf install -y epel-release && \
    dnf config-manager --set-enabled crb && \
    dnf install -y \
    gcc gcc-c++ make bison flex \
    openssl-devel curl-devel \
    mysql-devel postgresql-devel \
    pcre-devel expat-devel \
    libxml2-devel libunistring-devel \
    json-c-devel libevent-devel \
    git wget vim net-tools \
    && dnf clean all

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
    openssl mysql-libs postgresql-libs \
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

### 3.3 Construir la imagen

```bash
# Construccion basica
docker build -f Dockerfile.kamailio -t mi-kamailio:6.0 .

# Construccion optimizada
docker build -f Dockerfile.kamailio-optimized -t mi-kamailio:6.0-optimized .

# Con argumentos personalizados
docker build \
    --build-arg KAMAILIO_VERSION=6.0 \
    -t mi-kamailio:6.0-custom \
    -f Dockerfile.kamailio .

# Ver imagenes creadas
docker images | grep kamailio

# Ver tamaño de imagen
docker images mi-kamailio:6.0-optimized --format "{{.Size}}"
```

### 3.4 Configuración básica de Kamailio

**Archivo: `kamailio.cfg` (simplificado para Docker)**

```
#!KAMAILIO

####### Global Parameters #########
debug=3
log_stderror=yes
memdbg=5
memlog=5
log_facility=LOG_LOCAL0
log_prefix="{$mt $hdr(CSeq) $ci} "

children=4
tcp_children=4

# Escuchar en todas las interfaces
listen=udp:0.0.0.0:5060

# Alias (será configurado por variable de entorno)
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
loadmodule "nathelper.so"
loadmodule "rtpengine.so"

modparam("nathelper|registrar", "received_avp", "$avp(RECEIVED)")

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
            route(NATMANAGE);
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
    
    route(NATMANAGE);
    route(RELAY);
}

route[RELAY] {
    if (!t_relay()) {
        sl_reply_error();
    }
    exit;
}

route[NATMANAGE] {
    if (nat_uac_test("19")) {
        if (is_method("REGISTER")) {
            fix_nated_register();
        } else {
            if(is_first_hop()) {
                set_contact_alias();
            }
        }
    }
    return;
}
```

---

## PARTE 4: DOCKER COMPOSE PARA KAMAILIO (15 minutos)

### 4.1 Docker Compose Básico

**Archivo: `docker-compose.yml`**

```yaml
services:
  kamailio:
    build:
      context: .
      dockerfile: Dockerfile.kamailio-optimized
    container_name: kamailio-server
    hostname: kamailio
    restart: unless-stopped
    
    # Network mode host para mejor rendimiento SIP/RTP
    # Alternativa: usar bridge y mapear puertos específicos
    network_mode: host
    
    volumes:
      # Configuracion persistente
      - ./config/kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
      - ./config/kamctlrc:/etc/kamailio/kamctlrc:ro
      # Logs persistentes
      - ./logs:/var/log/kamailio
      # Scripts personalizados
      - ./scripts:/opt/scripts:ro
    
    environment:
      - KAMAILIO_LOG_LEVEL=3
      - TZ=America/Bogota
    
    # Capabilities para binding a puertos < 1024
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    healthcheck:
      test: ["CMD", "/usr/local/sbin/kamctl", "fifo", "get_statistics", "all"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  # Base de datos MariaDB
  mariadb:
    image: mariadb:10.11
    container_name: kamailio-db
    hostname: mariadb
    restart: unless-stopped
    
    environment:
      MYSQL_ROOT_PASSWORD: kamailiorootpass
      MYSQL_DATABASE: kamailio
      MYSQL_USER: kamailio
      MYSQL_PASSWORD: kamailiopass
      TZ: America/Bogota
    
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
    
    ports:
      - "3306:3306"
    
    command: 
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --max_connections=500
    
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "kamailio", "-pkamailiopass"]
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

### 4.2 Docker Compose con network bridge (producción recomendada)

**Archivo: `docker-compose-bridge.yml`**

```yaml

services:
  kamailio:
    build:
      context: .
      dockerfile: Dockerfile.kamailio-optimized
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
      - KAMAILIO_LOG_LEVEL=3
      - DB_HOST=mariadb
      - DB_USER=kamailio
      - DB_PASS=kamailiopass
      - DB_NAME=kamailio
      - TZ=America/Bogota
    
    depends_on:
      mariadb:
        condition: service_healthy
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  mariadb:
    image: mariadb:10.11
    container_name: kamailio-db
    hostname: mariadb
    restart: unless-stopped
    
    networks:
      kamailio-net:
        ipv4_address: 172.20.0.20
    
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-kamailiorootpass}
      MYSQL_DATABASE: ${DB_NAME:-kamailio}
      MYSQL_USER: ${DB_USER:-kamailio}
      MYSQL_PASSWORD: ${DB_PASS:-kamailiopass}
    
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./config/my.cnf:/etc/mysql/conf.d/custom.cnf:ro
    
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

### 4.3 Archivo .env para variables

**Archivo: `.env`**

```bash
# Base de datos
DB_ROOT_PASSWORD=SuperSecureRootPass123!
DB_NAME=kamailio
DB_USER=kamailio
DB_PASS=KamailioSecurePass456!

# Kamailio
KAMAILIO_LOG_LEVEL=3
KAMAILIO_DOMAIN=sip1.kamailio.xyz
KAMAILIO_EXTERNAL_IP=144.202.68.137

# Timezone
TZ=America/Bogota

# Versiones
KAMAILIO_VERSION=6.0
MARIADB_VERSION=10.11
```

### 4.4 Comandos Docker Compose

```bash
# Levantar servicios si el archivo de docker tiene el nombre estandar docker-compose.yml
docker compose up -d
# sino
docker compose -f nombrearchivodocker up -d

# Ver logs
docker compose logs -f kamailio
# o
docker compose -f nombrearchivodocker logs kamailio
docker compose logs -f mariadb
# o
docker compose -f nombrearchivodocker logs mariadb

# Ver estado
docker compose ps

# Detener servicios
docker compose stop

# Detener y eliminar contenedores
docker compose down
# o
docker compose -f nombrearchivodocker down

# Detener, eliminar contenedores y volumes
docker compose down -v
# o
docker compose -f nombrearchivodocker down -v

# Reconstruir imágenes
docker compose build --no-cache

# Escalar servicio (múltiples instancias)
docker compose up -d --scale kamailio=3

# Ejecutar comando en contenedor
docker compose exec kamailio kamctl fifo get_statistics all
docker compose exec kamailio kamcmd stats.get_statistics all

# Entrar al contenedor
docker compose exec kamailio bash
docker compose exec mariadb mysql -u kamailio -p
```

---

## PARTE 5: EJERCICIO PRÁCTICO (25 minutos)

### Ejercicio 1: Desplegar Kamailio básico con Docker

**Objetivo:** Levantar un servidor Kamailio containerizado con registro de usuarios

**Pasos:**

1. **Crear estructura de directorios:**
```bash
mkdir -p ~/kamailio-docker/{config,logs,scripts}
cd ~/kamailio-docker
```

2. **Crear Dockerfile:** (usar Dockerfile.kamailio-optimized de arriba)

3. **Crear kamailio.cfg básico:** (usar configuración de arriba)

4. **Crear docker-compose.yml:** (usar versión básica)

5. **Construir y levantar:**
```bash
docker compose build
docker compose up -d
docker compose logs -f
```

6. **Verificar funcionamiento:**
```bash
# Ver estadísticas
docker compose exec kamailio kamctl fifo get_statistics all

# Ver usuarios registrados
docker compose exec kamailio kamctl ul show

# Ver logs en tiempo real
docker compose logs -f kamailio
```

7. **Registrar un softphone:**
- Configurar un softphone (Zoiper, Linphone, etc.)
- Server: IP del host
- Usuario: test@kamailio.local
- Password: (si tienes auth configurado)

### Ejercicio 2: Agregar base de datos

**Objetivo:** Integrar MariaDB para persistencia de usuarios

**Pasos:**

1. **Usar docker-compose-bridge.yml**

2. **Crear script de inicialización de BD:**

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

3. **Modificar kamailio.cfg para usar DB:**

```
# Agregar módulos de DB
loadmodule "db_mysql.so"
loadmodule "auth.so"
loadmodule "auth_db.so"

# Configurar conexión DB
modparam("auth_db", "db_url", "mysql://kamailio:kamailiopass@mariadb/kamailio")
modparam("usrloc", "db_url", "mysql://kamailio:kamailiopass@mariadb/kamailio")
modparam("usrloc", "db_mode", 2)

# En route[AUTH]:
route[AUTH] {
    if (is_method("REGISTER")) {
        if (!auth_check("$fd", "subscriber", "1")) {
            auth_challenge("$fd", "0");
            exit;
        }
    }
}
```

4. **Levantar stack completo:**
```bash
docker compose -f docker-compose-bridge.yml up -d
docker compose logs -f
```

5. **Verificar conectividad a DB:**
```bash
docker compose exec kamailio kamctl db show
docker compose exec mariadb mysql -u kamailio -pkamailiopass -e "USE kamailio; SELECT * FROM subscriber;"
```

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
❌ Exponer puertos innecesarios
❌ Credenciales hardcodeadas en Dockerfile
❌ No usar .dockerignore (imágenes grandes)
❌ No limitar rango RTP (mapear todo 10000-60000)
❌ Configuración dentro del contenedor (no en volume)
❌ No configurar healthchecks
❌ Logs dentro del contenedor (se pierden)
❌ Usar tag "latest" en producción
```

### 📚 Recursos adicionales:

- Docker docs: https://docs.docker.com
- Kamailio Docker Hub: https://hub.docker.com/u/kamailio
- Docker Compose reference: https://docs.docker.com/compose/compose-file/

---

**FIN VIDEOCONFERENCIA 1 - PARTE DOCKER**

**Próxima sesión:** RTPEngine en Docker (multi-instancia)
