# VIDEOCONFERENCIA 4 - CONTENIDO DOCKER
## Alta Disponibilidad con Docker Swarm y Kubernetes

---

## PARTE 1: ALTA DISPONIBILIDAD CONTAINERIZADA - INTRODUCCIÓN (10 minutos)

### 1.1 HA Tradicional vs HA Containerizada

**HA Tradicional (Keepalived/Pacemaker):**

```
┌───────────────────────────────────────────┐
│         Virtual IP (Floating)             │
│              192.168.1.100                │
└──────────────┬────────────────────────────┘
               │
        ┌──────┴───────┐
        │              │
   ┌────▼────┐    ┌───▼─────┐
   │ Server1 │    │ Server2 │
   │ ACTIVE  │    │ STANDBY │
   │Kamailio │    │Kamailio │
   │ + DB    │◄──►│  + DB   │
   └─────────┘    └─────────┘
   
Características:
✓ IP flotante automática
✓ Configuración compleja
✗ Desperdicio de recursos (standby)
✗ Split-brain posible
✗ Failover no instantáneo
```

**HA Containerizada (Docker Swarm / Kubernetes):**

```
┌──────────────────────────────────────────────────┐
│              Load Balancer / Ingress              │
│                 192.168.1.100                     │
└─────────┬────────────────────────┬────────────────┘
          │                        │
    ┌─────▼──────┐           ┌─────▼──────┐
    │  Node 1    │           │  Node 2    │
    │            │           │            │
    │ Kamailio-1 │           │ Kamailio-2 │
    │ Kamailio-3 │           │ RTPEng-1   │
    │ RTPEng-2   │           │ Asterisk-1 │
    └────────────┘           └────────────┘
          │                        │
          └──────────┬─────────────┘
                     │
              ┌──────▼──────┐
              │   Node 3    │
              │  (Manager)  │
              │             │
              │ MariaDB     │
              │ Redis       │
              └─────────────┘

Características:
✓ Distribución activa de carga
✓ Auto-healing (restart automático)
✓ Rolling updates sin downtime
✓ Escalado horizontal automático
✓ Service discovery automático
✗ Complejidad inicial mayor
```

### 1.2 Comparativa de soluciones HA containerizadas

| Característica | Docker Swarm | Kubernetes | Tradicional (Keepalived) |
|----------------|--------------|------------|--------------------------|
| **Complejidad** | Baja | Alta | Media |
| **Learning Curve** | 1-2 días | 1-2 semanas | 3-5 días |
| **Escalabilidad** | Buena (100 nodos) | Excelente (5000+ nodos) | Limitada (2-4 nodos) |
| **Service Discovery** | Integrado | Integrado | Manual (DNS) |
| **Load Balancing** | Integrado | Integrado | Externo (HAProxy) |
| **Auto-healing** | Sí | Sí | Limitado |
| **Rolling Updates** | Sí | Sí | Manual |
| **Ecosistema** | Docker | Muy amplio | Limitado |
| **Uso de recursos** | Bajo | Medio | Bajo |
| **Para VoIP** | ✅ Bueno | ✅ Excelente | ✅ Probado |

### 1.3 Cuándo usar cada solución

**Usar Docker Swarm cuando:**
- Cluster pequeño (3-20 nodos)
- Equipo con experiencia Docker
- Necesitas deployar rápido
- Infraestructura simple
- Recursos limitados

**Usar Kubernetes cuando:**
- Cluster grande (20+ nodos)
- Multi-cloud / hybrid cloud
- Microservicios complejos
- CI/CD avanzado
- Compliance estricto

**Usar Keepalived cuando:**
- Solo 2 servidores
- No puedes usar contenedores
- Legacy system migration
- Máximo rendimiento crítico

---

## PARTE 2: DOCKER SWARM PARA VOIP (20 minutos)

### 2.1 Introducción a Docker Swarm

**Conceptos clave:**

```
SWARM: Cluster de Docker engines
MANAGER: Nodo que controla el cluster
WORKER: Nodo que ejecuta contenedores
SERVICE: Aplicación distribuida en el cluster
STACK: Conjunto de servicios (docker-compose.yml)
OVERLAY NETWORK: Red que conecta contenedores entre nodos
```

### 2.2 Inicializar Swarm cluster

**En el nodo manager (Node1):**

```bash
# Inicializar swarm
docker swarm init --advertise-addr 192.168.1.10

# Output mostrará comando para unir workers:
# docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377

# Ver nodos
docker node ls

# Promover worker a manager (para HA)
docker node promote node2
docker node promote node3
```

**En los nodos workers (Node2, Node3):**

```bash
# Unirse al swarm (usar token del init)
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377

# Verificar (desde manager)
docker node ls
```

### 2.3 Stack VoIP para Swarm

**Archivo: `docker-stack-voip.yml`**

```yaml
version: '3.8'

services:
  #==========================================
  # BASE DE DATOS (con replicación)
  #==========================================
  
  mariadb:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: kamailio
      MYSQL_USER: kamailio
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G

  #==========================================
  # REDIS (para estado compartido)
  #==========================================
  
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure

  #==========================================
  # KAMAILIO (múltiples réplicas)
  #==========================================
  
  kamailio:
    image: kamailio:6.0
    environment:
      KAMAILIO_LOG_LEVEL: 3
      DB_HOST: mariadb
      DB_USER: kamailio
      DB_PASS: ${DB_PASSWORD}
      REDIS_HOST: redis
    configs:
      - source: kamailio_cfg
        target: /etc/kamailio/kamailio.cfg
    networks:
      - frontend
      - backend
    ports:
      - target: 5060
        published: 5060
        protocol: udp
        mode: host  # IMPORTANTE: mode host para SIP/UDP
      - target: 5060
        published: 5060
        protocol: tcp
        mode: host
    deploy:
      # 3 réplicas distribuidas
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        max_replicas_per_node: 2
        constraints:
          - node.role == worker
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

  #==========================================
  # RTPENGINE (host network mode)
  #==========================================
  
  # Nota: RTPEngine requiere kernel module, 
  # mejor deployment como servicio global en nodos específicos
  
  rtpengine:
    image: rtpengine:latest
    environment:
      RTPE_INTERFACE: enp1s0
      RTPE_LISTEN_NG: 22222
      RTPE_TABLE: 0
      RTPE_PORT_MIN: 10000
      RTPE_PORT_MAX: 30000
      RTPE_REDIS: redis:6379
    networks:
      - backend
    deploy:
      # Global = una instancia en cada worker node
      mode: global
      placement:
        constraints:
          - node.labels.rtpengine == true
      restart_policy:
        condition: on-failure
    # Host network para mejor rendimiento RTP
    network_mode: host
    cap_add:
      - NET_ADMIN

  #==========================================
  # ASTERISK (app servers)
  #==========================================
  
  asterisk:
    image: asterisk:20
    configs:
      - source: asterisk_conf
        target: /etc/asterisk/asterisk.conf
      - source: sip_conf
        target: /etc/asterisk/sip.conf
    volumes:
      - voicemail:/var/spool/asterisk/voicemail
      - recordings:/var/spool/asterisk/monitor
    networks:
      - backend
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 30s
      placement:
        constraints:
          - node.labels.asterisk == true
      resources:
        limits:
          cpus: '2'
          memory: 1G

#==========================================
# CONFIGS (archivos de configuración)
#==========================================

configs:
  kamailio_cfg:
    file: ./config/kamailio.cfg
  asterisk_conf:
    file: ./config/asterisk.conf
  sip_conf:
    file: ./config/sip.conf

#==========================================
# VOLUMES
#==========================================

volumes:
  mariadb_data:
  redis_data:
  voicemail:
  recordings:

#==========================================
# NETWORKS
#==========================================

networks:
  frontend:
    driver: overlay
    attachable: true
  backend:
    driver: overlay
    attachable: true
```

### 2.4 Deployment del stack

```bash
# Deploy stack
docker stack deploy -c docker-stack-voip.yml voip

# Ver servicios
docker stack services voip

# Ver tareas (contenedores) del stack
docker stack ps voip

# Logs de un servicio
docker service logs -f voip_kamailio

# Escalar servicio
docker service scale voip_kamailio=5

# Update servicio (rolling update)
docker service update --image kamailio:6.0.1 voip_kamailio

# Remover stack
docker stack rm voip
```

### 2.5 Labels en nodos para placement

```bash
# Etiquetar nodos para RTPEngine
docker node update --label-add rtpengine=true node1
docker node update --label-add rtpengine=true node2

# Etiquetar nodos para Asterisk
docker node update --label-add asterisk=true node2
docker node update --label-add asterisk=true node3

# Ver labels
docker node inspect node1 --format '{{.Spec.Labels}}'

# Remover label
docker node update --label-rm rtpengine node1
```

### 2.6 Health checks en Swarm

```yaml
services:
  kamailio:
    # ... config ...
    healthcheck:
      test: ["CMD-SHELL", "kamctl fifo get_statistics all || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    deploy:
      # Auto-replace unhealthy containers
      update_config:
        monitor: 30s
        failure_action: rollback
```

---

## PARTE 3: KUBERNETES PARA VOIP (INTRODUCCIÓN) (15 minutos)

### 3.1 Conceptos básicos de Kubernetes

**Arquitectura:**

```
┌────────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER                     │
│                                                     │
│  ┌──────────────────────────────────────────────┐ │
│  │         CONTROL PLANE (Master Node)           │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │ │
│  │  │API Server│  │Scheduler │  │Controller│   │ │
│  │  └──────────┘  └──────────┘  └──────────┘   │ │
│  │  ┌──────────┐                                │ │
│  │  │  etcd    │  (cluster state)               │ │
│  │  └──────────┘                                │ │
│  └──────────────────────────────────────────────┘ │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ Worker Node1 │  │ Worker Node2 │              │
│  │              │  │              │              │
│  │ ┌──────────┐ │  │ ┌──────────┐ │              │
│  │ │  kubelet │ │  │ │  kubelet │ │              │
│  │ └──────────┘ │  │ └──────────┘ │              │
│  │ ┌──────────┐ │  │ ┌──────────┐ │              │
│  │ │  Pods    │ │  │ │  Pods    │ │              │
│  │ │ Kamailio │ │  │ │RTPEngine │ │              │
│  │ │ Asterisk │ │  │ │ Asterisk │ │              │
│  │ └──────────┘ │  │ └──────────┘ │              │
│  └──────────────┘  └──────────────┘              │
└────────────────────────────────────────────────────┘
```

**Objetos principales:**

```
POD: Unidad mínima (1+ contenedores)
DEPLOYMENT: Gestiona réplicas de Pods
SERVICE: Punto de acceso a Pods (load balancing)
CONFIGMAP: Configuración
SECRET: Datos sensibles
STATEFULSET: Para apps stateful (DB)
DAEMONSET: Un pod por nodo (como RTPEngine)
PERSISTENTVOLUME: Almacenamiento
INGRESS: Punto de entrada HTTP/S
```

### 3.2 Instalación de Kubernetes (K3s - lightweight)

**En todos los nodos:**

```bash
# Instalar K3s (lightweight Kubernetes)
# En Master node:
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --flannel-backend=vxlan

# Obtener token para workers
sudo cat /var/lib/rancher/k3s/server/node-token

# En Worker nodes:
curl -sfL https://get.k3s.io | K3S_URL=https://MASTER_IP:6443 \
  K3S_TOKEN=node-token sh -

# Verificar cluster
kubectl get nodes
```

### 3.3 Deployment de Kamailio en Kubernetes

**Archivo: `k8s-kamailio-deployment.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: voip

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kamailio-config
  namespace: voip
data:
  kamailio.cfg: |
    #!KAMAILIO
    # Configuración completa aquí
    # (truncada para brevedad)

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kamailio
  namespace: voip
  labels:
    app: kamailio
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kamailio
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: kamailio
    spec:
      containers:
      - name: kamailio
        image: kamailio:6.0
        ports:
        - containerPort: 5060
          protocol: UDP
          name: sip-udp
        - containerPort: 5060
          protocol: TCP
          name: sip-tcp
        - containerPort: 5061
          protocol: TCP
          name: sips
        env:
        - name: KAMAILIO_LOG_LEVEL
          value: "3"
        - name: DB_HOST
          value: "mariadb.voip.svc.cluster.local"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: config
          mountPath: /etc/kamailio
          readOnly: true
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - kamctl fifo get_statistics all
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - kamctl fifo get_statistics all
          initialDelaySeconds: 20
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: kamailio-config

---
apiVersion: v1
kind: Service
metadata:
  name: kamailio
  namespace: voip
spec:
  type: NodePort
  selector:
    app: kamailio
  ports:
  - port: 5060
    targetPort: 5060
    protocol: UDP
    name: sip-udp
    nodePort: 30060
  - port: 5060
    targetPort: 5060
    protocol: TCP
    name: sip-tcp
    nodePort: 30061
```

**Archivo: `k8s-secrets.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: voip
type: Opaque
stringData:
  username: kamailio
  password: SecurePassword123!
```

**Archivo: `k8s-mariadb-statefulset.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  namespace: voip
spec:
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: mariadb
  ports:
  - port: 3306
    name: mysql

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  namespace: voip
spec:
  serviceName: mariadb
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.11
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: root-password
        - name: MYSQL_DATABASE
          value: kamailio
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

**Archivo: `k8s-rtpengine-daemonset.yaml`**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rtpengine
  namespace: voip
  labels:
    app: rtpengine
spec:
  selector:
    matchLabels:
      app: rtpengine
  template:
    metadata:
      labels:
        app: rtpengine
    spec:
      hostNetwork: true  # IMPORTANTE: para rendimiento RTP
      nodeSelector:
        rtpengine: "true"  # Solo en nodos etiquetados
      containers:
      - name: rtpengine
        image: rtpengine:latest
        env:
        - name: RTPE_INTERFACE
          value: "enp1s0"
        - name: RTPE_LISTEN_NG
          value: "22222"
        - name: RTPE_TABLE
          valueFrom:
            fieldRef:
              fieldPath: status.podIP  # Unique table per node
        - name: RTPE_PORT_MIN
          value: "10000"
        - name: RTPE_PORT_MAX
          value: "30000"
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        resources:
          requests:
            memory: "512Mi"
            cpu: "1000m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
```

### 3.4 Deploy en Kubernetes

```bash
# Crear namespace
kubectl create namespace voip

# Aplicar secrets
kubectl apply -f k8s-secrets.yaml

# Deploy MariaDB
kubectl apply -f k8s-mariadb-statefulset.yaml

# Deploy Kamailio
kubectl apply -f k8s-kamailio-deployment.yaml

# Deploy RTPEngine (en nodos etiquetados)
kubectl label nodes node1 rtpengine=true
kubectl label nodes node2 rtpengine=true
kubectl apply -f k8s-rtpengine-daemonset.yaml

# Ver recursos
kubectl get all -n voip

# Ver pods
kubectl get pods -n voip -o wide

# Logs
kubectl logs -f deployment/kamailio -n voip

# Escalar
kubectl scale deployment kamailio --replicas=5 -n voip

# Update (rolling)
kubectl set image deployment/kamailio kamailio=kamailio:6.0.1 -n voip

# Rollback
kubectl rollout undo deployment/kamailio -n voip

# Ver eventos
kubectl get events -n voip --sort-by='.lastTimestamp'
```

---

## PARTE 4: COMPARATIVA Y DECISIONES (10 minutos)

### 4.1 Matriz de decisión

```
┌────────────────┬──────────────┬───────────────┬──────────────┐
│ Criterio       │ Keepalived   │ Docker Swarm  │ Kubernetes   │
├────────────────┼──────────────┼───────────────┼──────────────┤
│ Tamaño cluster │ 2-4 nodos    │ 3-50 nodos    │ 10-1000+     │
│ Complejidad    │ ⭐⭐         │ ⭐⭐⭐        │ ⭐⭐⭐⭐⭐   │
│ Tiempo setup   │ 4-8 horas    │ 2-4 horas     │ 1-3 días     │
│ HA Database    │ Manual       │ Constraints   │ StatefulSet  │
│ Service LB     │ Externo      │ Integrado     │ Integrado    │
│ Auto-scaling   │ No           │ Manual        │ Automático   │
│ Multi-tenant   │ Difícil      │ Medio         │ Fácil        │
│ Monitoring     │ Externo      │ Básico        │ Completo     │
│ Costo operativo│ Bajo         │ Medio         │ Alto         │
│ Skills needed  │ Linux + LB   │ Docker        │ Docker + K8s │
└────────────────┴──────────────┴───────────────┴──────────────┘
```

### 4.2 Recomendaciones por escenario

**Startup / SMB (< 10,000 usuarios):**
```
Recomendación: Docker Swarm
Razón: Balance entre simplicidad y features
Setup: 3 nodos (1 manager, 2 workers)
```

**Enterprise (10,000 - 100,000 usuarios):**
```
Recomendación: Kubernetes
Razón: Escalabilidad, auto-healing, ecosystem
Setup: 5+ nodos con autoscaling
```

**Telco / Service Provider (100,000+ usuarios):**
```
Recomendación: Kubernetes + Custom orchestration
Razón: Máxima escalabilidad y control
Setup: Multi-cluster, multi-region
```

**Legacy Migration:**
```
Recomendación: Keepalived → Docker → Swarm → K8s
Razón: Migración gradual sin downtime
```

---

## PARTE 5: EJERCICIO PRÁCTICO (15 minutos)

### Ejercicio: Deploy HA con Docker Swarm

**Objetivo:** Crear un cluster Swarm de 3 nodos con Kamailio HA

#### Paso 1: Preparar 3 máquinas virtuales

```
Node1 (Manager): 192.168.1.10
Node2 (Worker):  192.168.1.11
Node3 (Worker):  192.168.1.12
```

#### Paso 2: Inicializar Swarm

```bash
# En Node1 (Manager)
docker swarm init --advertise-addr 192.168.1.10

# Copiar comando join que aparece
# En Node2 y Node3
docker swarm join --token SWMTKN-xxx 192.168.1.10:2377

# Verificar (en Manager)
docker node ls
```

#### Paso 3: Crear overlay network

```bash
docker network create --driver overlay --attachable voip-net
```

#### Paso 4: Deploy stack

```bash
# Copiar docker-stack-voip.yml al manager
# Deploy
docker stack deploy -c docker-stack-voip.yml voip

# Verificar
docker stack services voip
docker stack ps voip
```

#### Paso 5: Test de failover

```bash
# Ver donde está corriendo Kamailio
docker stack ps voip | grep kamailio

# Simular fallo de nodo (drenar node2)
docker node update --availability drain node2

# Ver redistribución
watch -n 1 'docker stack ps voip | grep kamailio'

# Restaurar node2
docker node update --availability active node2
```

#### Paso 6: Rolling update

```bash
# Update imagen
docker service update --image kamailio:6.0.1 voip_kamailio

# Ver progreso
docker service ps voip_kamailio

# Rollback si hay problemas
docker service update --rollback voip_kamailio
```

---

## MONITOREO Y TROUBLESHOOTING

### Script de monitoreo Swarm

**Archivo: `monitor-swarm.sh`**

```bash
#!/bin/bash

while true; do
    clear
    echo "=== DOCKER SWARM STATUS ==="
    echo ""
    echo "NODES:"
    docker node ls
    echo ""
    echo "SERVICES:"
    docker stack services voip
    echo ""
    echo "TASKS:"
    docker stack ps voip --no-trunc
    echo ""
    echo "Updated: $(date)"
    sleep 5
done
```

### Comandos útiles de troubleshooting

```bash
# Ver logs de servicio
docker service logs -f voip_kamailio

# Inspeccionar servicio
docker service inspect voip_kamailio --pretty

# Ver tareas fallidas
docker stack ps voip --filter "desired-state=shutdown"

# Conectar a contenedor específico
docker exec -it <container-id> bash

# Ver uso de recursos
docker stats

# Ver networks
docker network ls
docker network inspect voip-net

# Cleanup
docker system prune -a
docker volume prune
```

---

## RESUMEN Y MEJORES PRÁCTICAS

### ✅ Checklist HA con contenedores:

```
□ Mínimo 3 nodos (quorum)
□ Managers en nodos separados
□ Overlay networks configuradas
□ Persistent volumes para datos
□ Health checks en todos los servicios
□ Resource limits definidos
□ Secrets para credenciales
□ Backup strategy para volumes
□ Monitoring configurado
□ Logging centralizado
□ Update strategy (rolling)
□ Rollback plan documentado
```

### 🔥 Errores comunes:

```
❌ Solo 1 manager node (sin redundancia)
❌ No usar persistent volumes para DB
❌ No configurar health checks
❌ No limitar recursos (OOM kills)
❌ Credenciales en código
❌ No probar failover antes de producción
❌ Update de todos los servicios a la vez
❌ No monitorear métricas
```

### 📚 Recursos adicionales:

```
Docker Swarm:
- https://docs.docker.com/engine/swarm/
- https://docs.docker.com/engine/swarm/stack-deploy/

Kubernetes:
- https://kubernetes.io/docs/
- https://k3s.io/ (lightweight K8s)

VoIP específico:
- Kamailio on K8s: https://github.com/kamailio/kamailio/wiki/Kubernetes
```

---

## CONCLUSIÓN DEL CURSO DOCKER

Has aprendido:

✅ **Videoconf 1:** Fundamentos Docker, Dockerfiles, Kamailio containerizado  
✅ **Videoconf 2:** RTPEngine multi-instancia, networking avanzado  
✅ **Videoconf 3:** Microservicios, docker-compose complejo, balanceo  
✅ **Videoconf 4:** Alta disponibilidad con Swarm y Kubernetes  

**Próximos pasos:**

1. Implementar en entorno de testing
2. Probar failover scenarios
3. Optimizar configuraciones
4. Configurar monitoreo completo
5. Documentar arquitectura
6. Crear runbooks operacionales
7. Capacitar equipo operativo

**¡Éxito con tus deployments VoIP containerizados!**

---

**FIN VIDEOCONFERENCIA 4 - FIN CONTENIDO DOCKER**
