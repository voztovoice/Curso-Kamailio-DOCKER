## 📋 CONTENIDO DEL CURSO

Este paquete contiene el material completo del curso de Kakamailio Docker/Contenedores, organizado en 4 videoconferencias:

### 📁 Archivos incluidos:

1. **videoconf1_docker_kamailio.md** (3 horas)
   - Introducción a Docker para VoIP
   - Dockerfile para Kamailio
   - Docker Compose básico
   - Ejercicios prácticos

2. **videoconf2_docker_rtpengine.md** (2 horas)
   - RTPEngine en Docker
   - Multi-instancia
   - Integración Kamailio + RTPEngine
   - Troubleshooting

3. **videoconf3_docker_microservices.md** (2 horas)
   - Arquitectura de microservicios
   - Stack completo VoIP
   - Balanceo de carga
   - Monitoreo

4. **videoconf4_docker_ha.md** (2 horas)
   - Docker Swarm
   - Kubernetes (introducción)
   - Alta disponibilidad
   - Comparativas

---

## 🎯 OBJETIVOS DE APRENDIZAJE

Al completar este módulo de Docker, los participantes podrán:

✅ Entender cuándo y por qué usar contenedores para VoIP  
✅ Crear Dockerfiles optimizados para Kamailio y RTPEngine  
✅ Desplegar stacks VoIP completos con docker-compose  
✅ Configurar múltiples instancias de RTPEngine balanceadas  
✅ Implementar arquitecturas de microservicios  
✅ Configurar alta disponibilidad con Docker Swarm  
✅ Comprender los fundamentos de Kubernetes para VoIP  

---

## 📚 ESTRUCTURA PEDAGÓGICA

Cada videoconferencia sigue esta estructura:

```
1. Teoría (30-40%)
   - Conceptos fundamentales
   - Casos de uso
   - Mejores prácticas

2. Demos en vivo (30-40%)
   - Construcción de Dockerfiles
   - Configuración de compose files
   - Deployment y troubleshooting

3. Ejercicios prácticos (30-40%)
   - Hands-on con VPS provisto
   - Resolución de problemas
   - Testing de configuraciones
```

---

## 🛠️ REQUISITOS TÉCNICOS

### Para seguir el curso necesitas:

**Software:**
- Docker Engine 24.0+
- Docker Compose 2.20+
- Cliente SSH
- Editor de texto (VS Code recomendado)

**Conocimientos previos:**
- Comandos básicos de Linux
- Conceptos de redes (IP, puertos, NAT)
- Protocolo SIP (nivel básico)
- Opcional: Git básico

**Hardware (VPS provisto por el curso):**
- CPU: 4 cores
- RAM: 8 GB
- Disco: 40 GB SSD
- Network: 1 Gbps
- OS: AlmaLinux 9

---

## 📖 CÓMO USAR ESTE MATERIAL

### Opción 1: Instructor-led (videoconferencias en vivo)

1. **Antes de cada sesión:**
   - Leer el documento correspondiente
   - Preparar entorno de práctica
   - Revisar prerequisitos

2. **Durante la sesión:**
   - Seguir demos del instructor
   - Hacer preguntas
   - Tomar notas de configuraciones específicas

3. **Después de la sesión:**
   - Completar ejercicios prácticos
   - Experimentar con variaciones
   - Acceder a grabación si es necesario

### Opción 2: Auto-estudio (self-paced)

1. **Seguir en orden:**
   - Videoconf 1 → 2 → 3 → 4
   - No saltar secciones (son progresivas)

2. **Práctica obligatoria:**
   - Cada ejercicio debe completarse
   - Verificar funcionamiento antes de continuar

3. **Uso del foro:**
   - Consultar dudas en el campus virtual
   - Compartir hallazgos con otros estudiantes

---

## 💡 EJEMPLOS INCLUIDOS

### Dockerfiles:
- ✅ Kamailio 6.0 (básico y optimizado)
- ✅ RTPEngine (multi-stage build)
- ✅ Asterisk (app server)

### Docker Compose files:
- ✅ Stack básico (Kamailio + DB)
- ✅ Multi-instancia RTPEngine
- ✅ Stack completo VoIP (10+ servicios)
- ✅ Configuración para producción

### Kubernetes manifests:
- ✅ Deployment de Kamailio
- ✅ StatefulSet para MariaDB
- ✅ DaemonSet para RTPEngine
- ✅ Services y ConfigMaps

### Scripts de utilidad:
- ✅ Carga de módulo kernel RTPEngine
- ✅ Monitoreo de cluster
- ✅ Health checks
- ✅ Troubleshooting

---

## 🔧 LABORATORIOS PRÁCTICOS

### Videoconferencia 1:
**Lab 1:** Desplegar Kamailio básico en Docker  
**Lab 2:** Agregar MariaDB con persistencia  
**Lab 3:** Registrar softphones y hacer llamadas  

### Videoconferencia 2:
**Lab 4:** Deploy RTPEngine single instance  
**Lab 5:** Multi-instancia RTPEngine (2 instancias)  
**Lab 6:** Integración completa Kamailio + RTPEngine  

### Videoconferencia 3:
**Lab 7:** Stack microservicios (Kamailio + RTPEngine + Asterisk)  
**Lab 8:** Configurar balanceo con dispatcher  
**Lab 9:** Monitoreo del cluster con scripts  

### Videoconferencia 4:
**Lab 10:** Crear Docker Swarm de 3 nodos  
**Lab 11:** Deploy stack HA  
**Lab 12:** Test de failover  

---

## 📊 ARQUITECTURAS DE REFERENCIA

### Arquitectura 1: Desarrollo / Testing
```
┌─────────────────────────────────┐
│     Docker Host (Laptop)        │
│  ┌──────────┐  ┌──────────┐     │
│  │ Kamailio │  │RTPEngine │     │
│  └──────────┘  └──────────┘     │
│  ┌──────────┐                   │
│  │ MariaDB  │                   │
│  └──────────┘                   │
└─────────────────────────────────┘
Archivos: docker-compose básico
Videoconferencia: 1
```

### Arquitectura 2: Producción Pequeña
```
┌─────────────────────────────────┐
│      Docker Swarm (3 nodos)     │
│  ┌──────────┐  ┌──────────┐     │
│  │Kamailio×3│  │RTPEng ×2 │     │
│  └──────────┘  └──────────┘     │
│  ┌──────────┐  ┌──────────┐     │ 
│  │Asterisk×2│  │MariaDB HA│     │ 
│  └──────────┘  └──────────┘     │
└─────────────────────────────────┘
Archivos: docker-stack-voip.yml
Videoconferencia: 4
```

### Arquitectura 3: Enterprise
```
┌─────────────────────────────────┐
│    Kubernetes Cluster (10+)     │
│  ┌──────────┐  ┌──────────┐     │
│  │Kamailio  │  │RTPEngine │     │
│  │AutoScale │  │DaemonSet │     │
│  └──────────┘  └──────────┘     │
│  ┌──────────┐  ┌──────────┐     │
│  │Asterisk  │  │ MariaDB  │     │
│  │  Pods    │  │StatefulS.│     │
│  └──────────┘  └──────────┘     │
│  ┌────────────────────────┐     │
│  │  Prometheus + Grafana  │     │
│  └────────────────────────┘     │
└─────────────────────────────────┘
Archivos: k8s-*.yaml
Videoconferencia: 4
```

---

## 🔍 TROUBLESHOOTING GUIDE

### Problema: Contenedor no inicia
```bash
# Ver logs
docker logs <container_id>

# Verificar healthcheck
docker inspect <container_id> | grep Health

# Entrar al contenedor (si está running)
docker exec -it <container_id> bash
```

### Problema: RTPEngine no funciona
```bash
# Verificar módulo kernel
lsmod | grep xt_RTPENGINE

# Cargar módulo
./scripts/load-rtpengine-module.sh

# Verificar /proc
ls -la /proc/rtpengine/
```

### Problema: No hay audio en llamadas
```bash
# Verificar sesiones RTP
docker exec rtpengine-1 rtpengine-ctl list

# Ver puertos
netstat -unlp | grep rtpengine

# Verificar firewall
iptables -L -n | grep RTP
```

### Problema: Kamailio no conecta a DB
```bash
# Test conexión
docker exec kamailio ping mariadb

# Verificar DNS interno
docker exec kamailio nslookup mariadb

# Ver logs de DB
docker logs mariadb
```

---

## 📈 PRÓXIMOS PASOS DESPUÉS DEL CURSO

### Nivel Intermedio:
1. Implementar monitoreo con Prometheus + Grafana
2. Configurar logging centralizado (ELK Stack)
3. Implementar CI/CD con GitLab/Jenkins
4. Configurar backups automatizados

### Nivel Avanzado:
1. Multi-region deployment
2. Service Mesh (Istio/Linkerd)
3. Auto-scaling basado en métricas
4. Disaster Recovery plan
5. Performance tuning avanzado

---

## 🤝 SOPORTE Y RECURSOS

### Durante el curso:
- **Foro:** Campus Virtual Mesa Proyectos
- **Email:** soporte@mesaproyectos.com
- **Chat:** Disponible durante videoconferencias

### Recursos adicionales:
- **Docker Docs:** https://docs.docker.com
- **Kamailio Wiki:** https://www.kamailio.org/wiki/
- **RTPEngine GitHub:** https://github.com/sipwise/rtpengine
- **K8s Docs:** https://kubernetes.io/docs/

### Comunidad:
- **Kamailio Mailing List:** sr-users@lists.kamailio.org
- **Docker Community:** https://forums.docker.com
- **VoIP subreddit:** r/VOIP

---

## ✅ CHECKLIST DE FINALIZACIÓN

Marca cuando completes cada sección:

**Videoconferencia 1:**
- [ ] Entiendo conceptos de Docker para VoIP
- [ ] Puedo crear Dockerfiles para Kamailio
- [ ] Manejo docker-compose básico
- [ ] Completé Lab 1-3

**Videoconferencia 2:**
- [ ] Puedo desplegar RTPEngine en Docker
- [ ] Configuré multi-instancia correctamente
- [ ] Entiendo networking Docker para RTP
- [ ] Completé Lab 4-6

**Videoconferencia 3:**
- [ ] Entiendo arquitectura de microservicios
- [ ] Puedo configurar balanceo de carga
- [ ] Implementé stack completo VoIP
- [ ] Completé Lab 7-9

**Videoconferencia 4:**
- [ ] Configuré Docker Swarm
- [ ] Entiendo conceptos de Kubernetes
- [ ] Implementé HA correctamente
- [ ] Completé Lab 10-12

---

## 🎓 CERTIFICACIÓN

Al completar exitosamente:
- Todos los laboratorios prácticos
- Proyecto final (deployment HA completo)
- Examen teórico-práctico

Recibirás:
**Certificado de especialización en VoIP Containerizado**
*Emitido por Mesa Proyectos SAS*

---

## 📞 CONTACTO

**Instructor:** Andrea Sannucci  
**Email:** campus@mesaproyectos.com  
**Empresa:** Mesa Proyectos SAS  
**WhatsAPP:** +573182409064  

---

**¡Éxito en tu aprendizaje de VoIP con Docker!** 🚀

*Última actualización: Noviembre 2025*
*Versión: 1.0*

