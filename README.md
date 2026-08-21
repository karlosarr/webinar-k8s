# Webinar Kubernetes 30 Minutos
## Gestión de Aplicaciones, Escalabilidad, Almacenamiento, Monitorización y Optimización

### 👨‍💻 Instructor: Master's in DevOps | Semana 4 | Contenedores

---
## 📋 Índice rápido

| Minuto | Tema | Recurso K8s |
|--------|------|-------------|
| 0-5 | Setup y Deployments | deployment.yml |
| 5-10 | StatefulSets y Jobs | statefulset.yml, job.yml |
| 10-15 | CronJobs y Escalabilidad | cronjob.yml, hpa.yml |
| 15-20 | Almacenamiento y Config | pv.yml, pvc.yml, configmap.yml, secret.yml |
| 20-25 | Monitorización EFK | elasticsearch.yml, kibana.yml, fluentd.yml |
| 25-30 | Optimización + Limpieza | README + clean |
| **30 min total** | **Webinar completo** | **19 archivos** |

---
## 0️⃣ Prerrequisitos (min 0:00 - 0:30)

```bash
# Verificar que kubectl está configurado
kubectl version --client

# Si es Docker Desktop en local:
minikube status  # o simplemente tener Docker Desktop corriendo
```

---
## 1️⃣ Gestión de Aplicaciones: Deployments (min 0:30 - 1:30)

### 1.1 Deployments - Deployment básico
```bash
kubectl apply -f deployment.yml
```
**Qué hace:** Crear `servicio-deployment` con 3 réplicas de `nginx:latest` en puerto 80.
**Key part:** `resources.requests: cpu: "100m"` ← **Obligatorio para que el HPA funcione**.

### 1.2 Accede a los pods
```bash
kubectl get pods -l app=servicio-app
kubectl get pods -o wide
```

### 1.3 StatefulSets - Aplicaciones stateful
```bash
kubectl apply -f statefulset.yml
```
**Qué hace:** `servicio-statefulset` con 3 réplicas que garante ordenamiento y estabilidad. Cada réplica tiene su propio PVC persistente.

### 1.3 Jobs - Tareas únicas
```bash
kubectl apply -f job.yml
```
**Qué hace:** `servicio-job` ejecuta una tarea y termina (restartPolicy: Never). Útil para procesamiento por lotes, backups, migraciones.

### 1.4 CronJobs - Tareas programadas
```bash
kubectl apply -f cronjob.yml
```
**Qué hace:** `servicio-cronjob` se ejecuta cada 5 minutos (`*/5 * * * *`). Para reportes, limpieza, envios de email.

---
## 2️⃣ Escalabilidad (min 1:30 - 2:30)

### 2.1 HorizontalPodAutoscaler (HPA)
```bash
kubectl apply -f hpa.yml
```
**Qué hace:** `servicio-hpa` escala automáticamente de 2 a 10 réplicas cuando el uso de CPU >= 50%.

### 2.2 Otras técnicas de escalabilidad
```bash
# Scaling manual
kubectl scale deployment servicio-deployment --replicas=5

# Cluster Autoscaler (añade nodos si no hay espacio)
# Custom Metrics HPA (escalado por memoria, QPS, etc.)
```

---
## 3️⃣ Almacenamiento y Configuración (min 2:30 - 3:30)

### 3.1 PersistentVolumes y Claims
```bash
kubectl apply -f pv.yml
kubectl apply -f pvc.yml
```
**Qué hace:** `servicio-pv` (5Gi, hostPath `/mnt/data`, ReadWriteOnce) + `servicio-pvc` (solicita 5Gi). Buena práctica para desarrollo local.

### 3.2 ConfigMaps - Configuración no sensible
```bash
kubectl apply -f configmap.yml
```
**Qué hace:** `servicio-config` almacena `database_host`, `database_port`, `log_level`. Disponible como variables de entorno o montajes de volumen.

### 3.3 Secrets - Credenciales sensibles
```bash
kubectl apply -f secret.yml
```
**Qué hace:** `servicio-secret` tipo Opaque con `username`/`password` codificados en base64. Nunca hardcodear creds en YAML.

### 3.4 El Deployment usa ambos
El `deployment.yml` ahora inyecta automáticamente:
- `DB_HOST`, `DB_PORT`, `LOG_LEVEL` desde el ConfigMap
- `USERNAME`, `PASSWORD` desde el Secret

```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: servicio-config
      key: database_host
- name: USERNAME
  valueFrom:
    secretKeyRef:
      name: servicio-secret
      key: username
```

---
## 4️⃣ Monitorización: Pila EFK en Kubernetes (min 3:30 - 4:30)

**Estructura:**
- **Elasticsearch:** Almacena e indexa los logs (`elasticsearch.yml`)
- **Kibana:** Visualiza y busca (`kibana.yml`)
- **Fluentd:** DaemonSet en todos los nodos, lee `/var/log/containers/*.log` (logs reales de todos los pods del clúster) y los envía a Elasticsearch (`fluentd.yml`)

### 4.1 Desplegar Elasticsearch y Kibana
```bash
kubectl apply -f elasticsearch.yml
kubectl apply -f kibana.yml

# Esperar a que estén listos (Elasticsearch tarda ~30-60s en arrancar)
kubectl rollout status deployment/elasticsearch
kubectl rollout status deployment/kibana
```

### 4.2 Desplegar Fluentd
```bash
kubectl apply -f fluentd.yml
```
> ⚠️ **Mac Apple Silicon (arm64):** el manifiesto usa la imagen `fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch7-arm64`. Si el clúster corre en `amd64` (Intel/Docker Desktop en Windows/Linux, EC2, etc.), cambia el tag a `v1-debian-elasticsearch7` en `fluentd.yml`.

**Qué hace:** crea el `ServiceAccount`/`ClusterRole`/`ClusterRoleBinding` `fluentd` (necesarios para que Fluentd pueda consultar metadata de pods vía la API de K8s) y el DaemonSet `fluentd-logging`, que usa la imagen oficial `fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch7`. Esta imagen ya trae preconfigurado el tail de `/var/log/containers/*.log` con enriquecimiento de metadata de Kubernetes (namespace, pod, contenedor, labels) y el output hacia Elasticsearch, apuntando al Service `elasticsearch:9200` vía variables de entorno.

```bash
kubectl get pods -l k8s-app=fluentd-logging -o wide
kubectl logs -l k8s-app=fluentd-logging -f
```

### 4.3 Conectarte a Kibana y ver los logs
```bash
# Port-forward de Kibana a tu máquina local
kubectl port-forward svc/kibana 5601:5601
```
Abre `http://localhost:5601` en el navegador:
1. Ve a **☰ Menu → Management → Stack Management → Index Patterns** (o "Data Views" en versiones nuevas).
2. Crea un index pattern `logstash-*` (o `fluentd*`, según el índice que veas listado).
3. Selecciona `@timestamp` como campo de tiempo.
4. Ve a **☰ Menu → Analytics → Discover** para ver los logs de los pods del clúster en tiempo real (deployment, statefulset, jobs, etc.).

Para generar tráfico de logs vistoso durante la demo:
```bash
kubectl scale deployment servicio-deployment --replicas=5
kubectl logs deployment/servicio-deployment --tail=5
```

### 4.4 Ver los logs (sin Kibana, por si el port-forward falla)
```bash
# Port-forward directo a Elasticsearch
kubectl port-forward svc/elasticsearch 9200:9200

# Ver índices en Elasticsearch
curl -X GET "http://localhost:9200/_cat/indices?v"

# Ver documentos recientes
curl -X GET "http://localhost:9200/logstash-*/_search?pretty" -H 'Content-Type: application/json' -d'{ "query": {"match_all": true}, "size": 5, "sort": [{"@timestamp":"desc"}] }'
```

---
## 5️⃣ Optimización y Eficiencia (min 4:30 - 5:00)

### 5.1 Recursos: Requests y Limits
- **Requests:** Garantiza CPU/Memory mínima para el pod (`cpu: "100m"` en deployment.yml)
- **Limits:** Máximo que un container puede consumir (previene "noisy neighbors")

### 5.2 Estrategias adicionales
```bash
# Node Selectors - Asignar pods a nodos específicos
kubectl label node node1 env=production

# Pod Disruption Budgets (PDBs) - Garantizar disponibilidad mínima
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-web
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app

# Resource quotas y LimitRanges - A nivel de namespace

# Liveness y Readiness Probes - Detectar congelados
```

---
## 6️⃣ Docker Compose - Desarrollo Local (opcional)

Para probar sin Kubernetes:

```bash
docker-compose up -d
```

Accesos:
- **Kibana:** http://localhost:5601
- **Elasticsearch:** http://localhost:9200
- **Fluentd API:** http://localhost:24224

Genera un log de ejemplo:
```bash
echo '192.168.1.1 - - [01/Jan/2024:10:00:00 +0000] "GET /api HTTP/1.1" 200 123 "-" "Mozilla/5.0"' >> nginx.log
```

Logs fluyen: `nginx.log → Fluentd → Elasticsearch → Kibana`

---
## 🧹 Limpieza final (después del webinar)

```bash
# Eliminar todos los recursos del cluster
kubectl delete -f deployment.yml
kubectl delete -f service.yml
kubectl delete -f ingress.yml
kubectl delete -f pv.yml
kubectl delete -f pvc.yml
kubectl delete -f hpa.yml
kubectl delete -f statefulset.yml
kubectl delete -f job.yml
kubectl delete -f cronjob.yml
kubectl delete -f configmap.yml
kubectl delete -f secret.yml
kubectl delete -f fluentd.yml
kubectl delete -f elasticsearch.yml
kubectl delete -f kibana.yml

# O eliminar todo de una vez (todos los yaml de la carpeta):
kubectl delete -f ./

# En Docker Desktop + docker-compose:
docker-compose down

# Limpiar volúmenes locales generados:
rm -rf ./pos
rm -f ./nginx.log
```

---
## 📎 Recursos incluidos (19 archivos)

| Archivo | Propósito |
|---------|-----------|
| `deployment.yml` | Deployment 3 réplicas con CPU requests |
| `service.yml` | Service ClusterIP |
| `ingress.yml` | Ingress con nginx rewrite |
| `pv.yml` | PersistentVolume 5Gi hostPath |
| `pvc.yml` | PersistentVolumeClaim 5Gi |
| `hpa.yml` | HorizontalPodAutoscaler CPU 50% |
| `statefulset.yml` | StatefulSet con PVC templates |
| `job.yml` | Job unicidad (busybox) |
| `cronjob.yml` | CronJob cada 5 min |
| `configmap.yml` | ConfigMap db_host/port/log_level |
| `secret.yml` | Secret Opaque username/password |
| `elasticsearch.yml` | Deployment + Service Elasticsearch (single-node, sin auth) |
| `kibana.yml` | Deployment + Service Kibana |
| `fluentd.yml` | ServiceAccount/RBAC + DaemonSet EFK logging (tail de `/var/log/containers/*.log`) |
| `deployment.yml` | Ahora usa ConfigMap + Secret via env |
| `fluentd.conf` | Config Fluentd tail + elasticsearch output |
| `docker-compose.yml` | Pila ELK en local |
| `nginx.log` | Archivo de logs de ejemplo |
| `README.md` | Esta guía webinar |

---
## ⏱️ Timeline del webinar de 30 minutos

| Segmento | Tiempo | Enfoque |
|----------|--------|---------|
| **Intro y prerrequisitos** | 0-5 min | kubectl version, contexto |
| **Gestión de Apps** | 5-10 min | Deployments + StatefulSets + Jobs + CronJobs |
| **Escalabilidad** | 10-15 min | HPA + otras técnicas |
| **Almacenamiento/Config** | 15-20 min | PV/PVC + ConfigMap + Secret |
| **Monitorización** | 20-25 min | Fluentd + EKF stack + ver logs |
| **Optimización** | 25-30 min | Resources + PDBs + Probes + Limpieza |

---
**¡Fin del webinar!** Gracias por participar. Los 19 archivos de este repositorio cubren todos los conceptos clave de Kubernetes para gestión de aplicaciones en producción.