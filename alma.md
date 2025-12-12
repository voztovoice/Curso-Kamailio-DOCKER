# RTPEngine en AlmaLinux 9: Migración de iptables a nftables


Este artículo documenta la evolución del módulo del kernel de RTPEngine, de `xt_RTPENGINE` a `nft_rtpengine`, y proporciona una guía completa de instalación en AlmaLinux 9 con configuración de nftables para entornos de producción VoIP.

Linux está en proceso de transición de iptables hacia nftables como sistema de filtrado de paquetes. Esta migración no es meramente cosmética; representa un rediseño arquitectónico del netfilter framework:

- **iptables**: Framework legacy basado en múltiples comandos (`iptables`, `ip6tables`, `arptables`, `ebtables`)
- **nftables**: Framework unificado que reemplaza todas las variantes anteriores con una sintaxis consistente

RTPEngine, como proxy de medios que depende del forwarding de paquetes en kernel space, debió adaptarse a esta realidad. El módulo del kernel evolucionó para mantener compatibilidad con el nuevo framework:

| Aspecto | Método Antiguo (≤11.5) | Método Moderno (12.x+) |
|---------|------------------------|------------------------|
| **Módulo del kernel** | `xt_RTPENGINE.ko` | `nft_rtpengine.ko` |
| **Sistema de filtrado** | iptables/ip6tables | nftables |
| **Gestión de reglas** | Manual vía CLI | Automática por daemon |
| **Extensión iptables** | Requerida (`libxt_RTPENGINE.so`) | No necesaria |
| **Comando de carga** | `modprobe xt_RTPENGINE` | `modprobe nft_rtpengine` |

### Aclaración importante: No es una sustitución completa

El módulo `nft_rtpengine` sigue siendo técnicamente un módulo x_tables (la infraestructura subyacente de iptables). Lo que cambió fue su integración con nftables, eliminando la necesidad de gestión manual de reglas y permitiendo al daemon de RTPEngine controlar el netfilter directamente.

Por esta razón, al ejecutar `nft list ruleset`, verás el mensaje `XT target RTPENGINE not found` en la salida. Este mensaje es **normal y esperado** - la herramienta `nft` no puede interpretar targets de x_tables, pero el kernel sí los procesa correctamente.

```bash
# Verificar versión del sistema
cat /etc/redhat-release
# AlmaLinux release 9.x (Turquoise Kodkod)

# Actualizar sistema base
dnf update -y
reboot

# Instalar Development Tools
dnf config-manager --set-enabled crb
dnf groupinstall "Development Tools" -y

# Instalar kernel headers (esencial para compilar módulos)
dnf install kernel-devel-$(uname -r) -y

```

### Dependencias de compilación

```bash
# Dependencias base de RTPEngine
dnf install -y \
    glib2-devel \
    zlib-devel \
    openssl-devel \
    pcre-devel \
    libcurl-devel \
    xmlrpc-c-devel \
    libevent-devel \
    json-glib-devel \
    hiredis-devel \
    gperf \
    libpcap-devel \
    perl-IPC-Cmd \
    libwebsockets-devel \
    nftables \
    pandoc \
    iptables-legacy-devel \
    ncurses-devel \
    libjwt-devel \
    libatomic \
    mysql-devel

# Dependencias para transcoding (opcional)
dnf install -y \
    libavcodec-free-devel libavdevice-free-devel libavfilter-free-devel \
    libavformat-free-devel libavutil-free-devel libpostproc-free-devel \
    libswresample-free-devel libswscale-free-devel \
    spandsp-devel \
    opus-devel \
    bcg729-devel

# Dependencias para DTLS/SRTP
dnf install -y \
    openssl-devel \
    libsrtp-devel
```

---

## Instalación de RTPEngine

### Descarga del código fuente

```bash
# Clonar repositorio
cd /usr/src
git clone https://github.com/sipwise/rtpengine.git
cd /usr/src/rtpengine/daemon
make
cp rtpengine /usr/local/bin
cd /usr/src/rtpengine/kernel-module
nano +6832 nft_rtpengine.c

se modifica este bloque:

static int rtpengine_expr_dump(struct sk_buff *skb, const struct nft_expr *expr
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6,2,0)
                , bool reset
#endif
)

para que quede:

static int rtpengine_expr_dump(struct sk_buff *skb, const struct nft_expr *expr, bool reset)

se guardan los cambios y 

make
make install
modinfo nft_rtpengine | head -n 10
```

**Salida esperada de modinfo:**
```bash
filename:       /lib/modules/5.14.0-611.11.1.el9_7.x86_64/updates/nft_rtpengine.ko
alias:          nft-expr-rtpengine
alias:          ip6t_RTPENGINE
alias:          ipt_RTPENGINE
alias:          xt_RTPENGINE
import_ns:      CRYPTO_INTERNAL
description:    rtpengine packet forwarding acceleration
author:         Sipwise GmbH <support@sipwise.com>
license:        GPL
rhelversion:    9.7

```

### Carga del módulo del kernel

```bash
# Cargar módulo manualmente
modprobe nft_rtpengine

# Verificar que está cargado
lsmod | grep nft_rtpengine
# nft_rtpengine    147456  0

# Verificar interfaz /proc
ls -la /proc/rtpengine/
# total 0
# dr-xr-xr-x 2 root root 0 Dec 12 10:10 .
# -w--w---- 1 root root 0 Dec 12 10:10 control

# Configurar carga automática al boot
echo "nft_rtpengine" > /etc/modules-load.d/rtpengine.conf

# Verificar configuración
cat /etc/modules-load.d/rtpengine.conf
# nft_rtpengine
```

---

## Configuración de nftables

```bash
nft flush ruleset
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy accept \; }
nft add rule inet filter input iif lo accept
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input ip protocol icmp accept
nft add rule inet filter input ip6 nexthdr ipv6-icmp accept
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input udp dport 5060 accept
nft add rule inet filter input udp dport 10000-20000 accept
nft add rule inet filter input counter drop
nft add chain inet filter forward { type filter hook forward priority 0 \; policy drop \; }
nft add chain inet filter output { type filter hook output priority 0 \; policy accept \; }
systemctl enable nftables
systemctl start nftables
nft list ruleset
nft list ruleset > /etc/nftables/rtpengine.nft

nano /etc/sysconfig/nftables.conf

al final del archvio se añade:

include "/etc/nftables/rtpengine.nft"

se guardan los cambios

```

## Configuración de RTPEngine

### Crear estructura de directorios

```bash
# Directorio de configuración
mkdir -p /etc/rtpengine

# Directorio de logs
mkdir -p /var/log/rtpengine
chown rtpengine:rtpengine /var/log/rtpengine

# Directorio de recordings (si se usa)
mkdir -p /var/spool/rtpengine
chown rtpengine:rtpengine /var/spool/rtpengine
```

### Archivo de configuración principal

```bash
cat > /etc/rtpengine/rtpengine.conf << 'EOF'
[rtpengine]

# Interfaces de red
# Formato: [nombre]/[IP_local]![IP_anunciada]
# Si no hay NAT, solo: [nombre]/[IP]
interface = pub/IP_PUBLICA

# Control de Kamailio/OpenSIPS
listen-ng = 127.0.0.1:2223

# Rango de puertos RTP
port-min = 10000
port-max = 20000

# Timeout configurations (en segundos)
timeout = 60
silent-timeout = 3600
final-timeout = 7200
offer-timeout = 60

# Kernel forwarding table ID
table = 0

# Logging
log-level = 6
log-facility = local1
log-facility-cdr = local2

# DTLS certificates (generados automáticamente si no existen)
# dtls-passive

# nftables configuration (gestión automática)
# El daemon creará y gestionará las chains automáticamente
nftables-chain = rtpengine
nftables-base-chain = input

# Redis para clustering (opcional)
# redis = 127.0.0.1:6379
# redis-db = 0

# Recording (opcional)
# recording-dir = /var/spool/rtpengine
# recording-method = pcap
# recording-format = eth

# Performance tuning
num-threads = 4

# B2BUA mode (si se usa)
# b2b-url = http://127.0.0.1:8080

# Delete delay para cleanup
delete-delay = 30

# TOS/DSCP marking
# tos = 184
EOF
```

### Configuración de rsyslog para logs de RTPEngine

```bash
cat > /etc/rsyslog.d/rtpengine.conf << 'EOF'
# RTPEngine call logs
local1.* /var/log/rtpengine/rtpengine.log

# RTPEngine CDR logs
local2.* /var/log/rtpengine/rtpengine-cdr.log

# Stop processing if matched
local1.* stop
local2.* stop
EOF

# Reiniciar rsyslog
systemctl restart rsyslog
```

### Configuración de logrotate

```bash
cat > /etc/logrotate.d/rtpengine << 'EOF'
/var/log/rtpengine/*.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /usr/bin/systemctl reload rsyslog > /dev/null 2>&1 || true
    endscript
}
EOF
```

### Crear systemd service

```bash
cat > /etc/systemd/system/rtpengine.service << 'EOF'
[Unit]
Description=RTPEngine Media Proxy
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/sipwise/rtpengine

[Service]
Type=forking
PIDFile=/run/rtpengine/rtpengine.pid
EnvironmentFile=-/etc/default/rtpengine
RuntimeDirectory=rtpengine

# Pre-start: verificar que el módulo está cargado
ExecStartPre=/bin/bash -c 'lsmod | grep -q nft_rtpengine || modprobe nft_rtpengine'

# Start daemon
ExecStart=/usr/local/bin/rtpengine \
    --pidfile=/run/rtpengine/rtpengine.pid \
    --config-file=/etc/rtpengine/rtpengine.conf

# Reload configuration
ExecReload=/bin/kill -HUP $MAINPID

# Stop daemon
ExecStop=/bin/kill -TERM $MAINPID

# Restart policy
Restart=on-failure
RestartSec=10

# Security hardening
NoNewPrivileges=true
PrivateTmp=true

# User/Group
User=root
Group=root

# Capabilities necesarias para kernel module
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_SYS_NICE
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW CAP_SYS_NICE

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
systemctl daemon-reload

# Habilitar servicio
systemctl enable rtpengine
```

---

## Iniciar y Verificar RTPEngine

### Iniciar el servicio

```bash
# Iniciar RTPEngine
systemctl start rtpengine

# Verificar estado
systemctl status rtpengine
```

**Salida esperada:**
```
â rtpengine.service - RTPEngine Media Proxy
     Loaded: loaded (/etc/systemd/system/rtpengine.service; enabled; preset: disabled)
     Active: active (running) since Fri 2025-12-12 17:00:05 UTC; 2s ago
       Docs: https://github.com/sipwise/rtpengine
    Process: 27278 ExecStartPre=/bin/bash -c lsmod | grep -q nft_rtpengine || modprobe nft_rtpengine (code=exited, status=0/SUCCESS)
    Process: 27281 ExecStart=/usr/local/bin/rtpengine --pidfile=/run/rtpengine/rtpengine.pid --config-file=/etc/rtpengine/rtpengine.conf (code=exited, status=0/SUCCESS)
   Main PID: 27282 (rtpengine)
      Tasks: 23 (limit: 4580)
     Memory: 27.2M (peak: 27.6M)
        CPU: 263ms
     CGroup: /system.slice/rtpengine.service
             ââ27282 /usr/local/bin/rtpengine --pidfile=/run/rtpengine/rtpengine.pid --config-file=/etc/rtpengine/rtpengine.conf

Dec 12 17:00:05 sip1.kamailio.xyz systemd[1]: Starting RTPEngine Media Proxy...
Dec 12 17:00:05 sip1.kamailio.xyz rtpengine[27281]: INFO: [core] compile-time OpenSSL library: OpenSSL 3.5.1 1 Jul 2025
Dec 12 17:00:05 sip1.kamailio.xyz rtpengine[27281]: INFO: [core] run-time OpenSSL library: OpenSSL 3.5.1 1 Jul 2025
Dec 12 17:00:05 sip1.kamailio.xyz rtpengine[27281]: INFO: [crypto] Generating new DTLS certificate
Dec 12 17:00:05 sip1.kamailio.xyz rtpengine[27282]: INFO: [core] Version git-master-addb93f0 initialising
Dec 12 17:00:05 sip1.kamailio.xyz systemd[1]: Started RTPEngine Media Proxy.
Dec 12 17:00:05 sip1.kamailio.xyz rtpengine[27282]: INFO: [core] Startup complete, version git-master-addb93f0
```

### Verificar módulo del kernel

```bash
# Verificar módulo cargado
lsmod | grep nft_rtpengine
# nft_rtpengine         102400  4

# Verificar /proc interface
ls -la /proc/rtpengine/
# dr-xr-xr-x 3 root root 0 Dec 12 10:30 .
# drwxr-xr-x 2 root root 0 Dec 12 10:30 0
# -w--w---- 1 root root 0 Dec 12 10:30 control

# Verificar tabla de forwarding (0 es el default)
ls -la /proc/rtpengine/0/
# dr-xr-xr-x. 6 root root 0 Dec 12 17:00 .
# dr-xr-xr-x. 5 root root 0 Dec 12 16:55 ..
# dr-xr-xr-x. 2 root root 0 Dec 12 17:02 calls
# -rw-rw----. 1 root root 0 Dec 12 17:00 control
# -r--r--r--. 1 root root 0 Dec 12 17:02 list
# -r--r--r--. 1 root root 0 Dec 12 17:02 status

```

### Verificar reglas de nftables

```bash
# Ver todas las reglas
nft list ruleset

# Ver específicamente la tabla filter
nft list table inet filter

# Ver específicamente la chain rtpengine
nft list chain inet filter rtpengine
```

**Salida esperada:**
```
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        iif "lo" accept comment "Allow loopback"
        ct state established,related accept comment "Allow established/related"
        ip protocol icmp accept comment "Allow ICMPv4"
        ip6 nexthdr ipv6-icmp accept comment "Allow ICMPv6"
        tcp dport 22 accept comment "Allow SSH"
        udp dport 5060 accept comment "Allow SIP UDP"
        tcp dport 5060 accept comment "Allow SIP TCP"
        tcp dport 5061 accept comment "Allow SIP TLS"
        udp dport 10000-20000 accept comment "Allow RTP media range"
        ip protocol udp counter packets 0 bytes 0 jump rtpengine
        counter packets 0 bytes 0 drop comment "Drop all other traffic"
    }
    
    chain rtpengine {
        XT target RTPENGINE not found
        counter packets 0 bytes 0
    }
    
    chain forward {
        type filter hook forward priority filter; policy drop;
    }
    
    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

### Verificar logs

```bash
# Ver logs en tiempo real
tail -f /var/log/rtpengine/rtpengine.log

# Buscar errores
grep -i error /var/log/rtpengine/rtpengine.log

# Buscar warnings
grep -i warning /var/log/rtpengine/rtpengine.log
```

### Monitoreo de contadores de nftables

```bash
# Ver contadores de la chain rtpengine
watch -n 1 'nft list chain inet filter rtpengine'
```

Verás los contadores incrementarse:
```
chain rtpengine {
    XT target RTPENGINE not found
    counter packets 123456 bytes 98765432
}
```

### Logs de RTPEngine

```bash
# Ver kernelization messages
tail -f /var/log/rtpengine/rtpengine.log | grep -i kernel
```

**Mensajes esperados:**
```
INFO: [call-id port 10000]: [core] Kernelizing media stream: 198.51.100.50:50000 -> 203.0.113.10:10000
INFO: [call-id port 10002]: [core] Kernelizing media stream: 198.51.100.50:50002 -> 203.0.113.10:10002
```

---

## Troubleshooting

### Problema: Módulo no carga

**Síntoma:**
```bash
modprobe nft_rtpengine
modprobe: ERROR: could not insert 'nft_rtpengine': Exec format error
```

**Causa:** Kernel headers no coinciden con kernel en ejecución

**Solución:**
```bash
# Verificar versión del kernel
uname -r
# 5.14.0-362.24.1.el9_3.x86_64

# Verificar headers instalados
rpm -qa | grep kernel-devel
# kernel-devel-5.14.0-362.8.1.el9_3.x86_64  <-- NO COINCIDE

# Instalar headers correctos
dnf install -y kernel-devel-$(uname -r)

# Recompilar módulo
cd /usr/src/rtpengine/kernel-module
make clean
make
make install
depmod -a
```

### Problema: "Failed to create nftables chains"

**Síntoma:**
```
rtpengine[12346]: Fatal error: Failed to create nftables chains or rules
```

**Causas posibles:**

1. **nftables no está instalado:**
```bash
dnf install -y nftables
systemctl enable nftables
systemctl start nftables
```

2. **Permisos insuficientes:**
```bash
# Verificar capabilities en systemd service
grep Capabilities /etc/systemd/system/rtpengine.service
# AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_SYS_NICE
```

3. **Chain base no existe:**
```bash
# Crear chain INPUT si no existe
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy accept \; }
```

### Problema: Paquetes no llegan al kernel module

**Síntoma:** `/proc/rtpengine/0/list` permanece vacío durante llamadas

**Diagnóstico:**
```bash
# 1. Verificar que el módulo está cargado
lsmod | grep nft_rtpengine

# 2. Verificar que la chain rtpengine existe
nft list chain inet filter rtpengine

# 3. Verificar que hay un jump desde INPUT a rtpengine
nft list chain inet filter input | grep rtpengine

# 4. Capturar tráfico RTP
tcpdump -i any -n 'udp portrange 10000-20000' -c 10

# 5. Verificar logs de rtpengine para mensajes de kernelization
grep -i kernel /var/log/rtpengine/rtpengine.log
```

**Solución común:** Verificar que el tráfico UDP realmente pasa por nftables:
```bash
# Agregar logging temporal para debug
nft insert rule inet filter input ip protocol udp log prefix "UDP packet: "

# Ver logs
dmesg -T | grep "UDP packet"

# Remover regla de logging después
nft list ruleset -a  # Encontrar handle de la regla
nft delete rule inet filter input handle <número>
```

### Problema: "XT target RTPENGINE not found" en nft output

**Síntoma:** Al ejecutar `nft list ruleset`, se ve:
```
chain rtpengine {
    XT target RTPENGINE not found
    counter packets 0 bytes 0
}
```

**Solución:** **Esto es completamente normal y esperado**. No es un error.

La herramienta `nft` no puede interpretar targets de x_tables (iptables legacy), pero el kernel sí los procesa. Si ves incrementarse los contadores de paquetes, el sistema está funcionando correctamente.

### Problema: Llamadas no procesan audio

**Diagnóstico completo:**

```bash
# 1. Verificar que RTPEngine recibe signaling de Kamailio
tail -f /var/log/rtpengine/rtpengine.log | grep offer

# 2. Verificar que puertos RTP están abiertos
ss -ulnp | grep rtpengine

# 3. Verificar firewall
nft list ruleset | grep 10000-20000

# 4. Verificar NAT (si aplica)
# Si RTPEngine está detrás de NAT, verificar interface configuration:
grep interface /etc/rtpengine/rtpengine.conf
# Debe ser: interface = external/[IP_local]![IP_pública]

# 5. Capturar paquetes RTP
tcpdump -i any -n -s 0 -vvv 'udp portrange 10000-20000' -w /tmp/rtp.pcap

# 6. Analizar con tshark
tshark -r /tmp/rtp.pcap -Y rtp
```

---

## Optimización y Tuning

### Ajustes del kernel para alta carga

```bash
cat >> /etc/sysctl.d/99-rtpengine.conf << 'EOF'
# Aumentar límites de sockets
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 67108864
net.core.wmem_default = 67108864
net.core.netdev_max_backlog = 30000

# UDP buffer sizes
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384

# Aumentar rango de puertos locales
net.ipv4.ip_local_port_range = 1024 65535

# Mejorar handling de conexiones
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192

# Reducir TIME_WAIT
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
EOF

# Aplicar cambios
sysctl -p /etc/sysctl.d/99-rtpengine.conf
```

### Ajustes de RTPEngine para alta concurrencia

```bash
# En /etc/rtpengine/rtpengine.conf, ajustar:

# Aumentar threads (1.5-2x número de cores)
num-threads = 8

# Aumentar rango de puertos
port-min = 10000
port-max = 30000

# Ajustar timeouts según carga
timeout = 60
silent-timeout = 1800

# Habilitar Homer para monitoring (opcional)
# homer = 127.0.0.1:9060
# homer-protocol = udp
# homer-id = 2001
```

### Monitoring con Prometheus (opcional)

RTPEngine puede exportar métricas para Prometheus:

```bash
# En /etc/rtpengine/rtpengine.conf, agregar:
# http-listen = 127.0.0.1:9101
# prometheus = /metrics
```

---

## Migración desde versión antigua (xt_RTPENGINE)

Si tienes un sistema en producción con la versión antigua y necesitas migrar:

### Preparación

```bash
# 1. Backup de configuración actual
cp -r /etc/rtpengine /etc/rtpengine.backup.$(date +%Y%m%d)

# 2. Documentar configuración de iptables actual
iptables-save > /root/iptables-backup-$(date +%Y%m%d).txt
ip6tables-save > /root/ip6tables-backup-$(date +%Y%m%d).txt

# 3. Documentar versión de RTPEngine actual
rtpengine --version > /root/rtpengine-version-old.txt
```

### Procedimiento de migración (mantenimiento programado)

```bash
# 1. Detener RTPEngine
systemctl stop rtpengine

# 2. Descargar módulo antiguo
rmmod xt_RTPENGINE

# 3. Remover configuración de iptables
iptables -D INPUT -p udp -j RTPENGINE --id 0
ip6tables -D INPUT -p udp -j RTPENGINE --id 0

# 4. Instalar nueva versión (seguir sección "Instalación de RTPEngine")
# ...compilación y configuración según secciones anteriores...

# 5. Configurar nftables (seguir sección "Configuración de nftables")
# ...

# 6. Actualizar configuración de rtpengine.conf
# Agregar líneas nftables:
# nftables-chain = rtpengine
# nftables-base-chain = input

# 7. Iniciar nuevo RTPEngine
systemctl start rtpengine

# 8. Verificar
nft list ruleset | grep rtpengine
lsmod | grep nft_rtpengine
```

### Rollback en caso de problemas

```bash
# 1. Detener nueva versión
systemctl stop rtpengine

# 2. Restaurar iptables
iptables-restore < /root/iptables-backup-[fecha].txt
ip6tables-restore < /root/ip6tables-backup-[fecha].txt

# 3. Cargar módulo antiguo
modprobe xt_RTPENGINE

# 4. Restaurar configuración antigua
cp -r /etc/rtpengine.backup.[fecha]/* /etc/rtpengine/

# 5. Iniciar versión antigua
systemctl start rtpengine
```

---

## Conclusiones

La migración de `xt_RTPENGINE` a `nft_rtpengine` representa más que un simple cambio de nombre. Es una adaptación necesaria al futuro del networking en Linux, donde nftables reemplaza gradualmente a iptables.

### Ventajas del método moderno:

1. **Gestión automática**: RTPEngine controla directamente las reglas de nftables
2. **Menos pasos manuales**: No se requiere compilar extensiones de iptables
3. **Sintaxis unificada**: nftables ofrece una interfaz consistente para IPv4 e IPv6
4. **Mejor performance**: nftables tiene mejor rendimiento en reglas complejas
5. **Preparado para el futuro**: Alineado con la dirección del kernel Linux

### Consideraciones filosóficas:

Desde una perspectiva heracliteana, este cambio ilustra la naturaleza fundamental del software: la permanencia es una ilusión. Los sistemas deben adaptarse o volverse obsoletos. RTPEngine, al evolucionar con el ecosistema Linux en lugar de resistirse, asegura su relevancia continua.

Sin embargo, como Aristóteles nos recordaría, el cambio debe ser teleológico - dirigido hacia un fin. El fin aquí es claro: mejor mantenibilidad, mejor integración con el sistema operativo moderno, y preparación para futuras evoluciones del netfilter framework.

### Recomendaciones finales:

1. **Sistemas nuevos**: Usar siempre el método moderno (nft_rtpengine + nftables)
2. **Sistemas legacy**: Planificar migración durante ventana de mantenimiento
3. **Documentación**: Mantener documentación actualizada con referencias a versiones específicas
4. **Monitoreo**: Implementar monitoreo robusto para detectar problemas temprano
5. **Testing**: Probar exhaustivamente en entorno no productivo antes de migrar producción

---

## Referencias

- **RTPEngine GitHub**: https://github.com/sipwise/rtpengine
- **Documentación oficial**: https://rtpengine.readthedocs.io/
- **nftables Wiki**: https://wiki.nftables.org/
- **AlmaLinux Documentation**: https://wiki.almalinux.org/

---

**Autor**: Documentación técnica para administradores VoIP  
**Fecha**: Diciembre 2024  
**Versión**: 1.0  
**Licencia**: Creative Commons BY-SA 4.0
