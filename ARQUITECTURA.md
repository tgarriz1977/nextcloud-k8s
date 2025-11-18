# Arquitectura Detallada - Nextcloud + Collabora + ClamAV

## Diagrama de Red y Comunicación

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            INTERNET                                      │
│                                                                          │
│  Usuario accede a:                                                       │
│  - https://nubed2.tecnicos.org.ar        (Nextcloud)                    │
│  - https://collabora.tecnicos.org.ar     (Collabora)                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ HTTPS (443)
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    AWS Application Load Balancer                         │
│                    (Managed by Nginx Ingress Controller)                 │
└───────────────┬──────────────────────────────────┬──────────────────────┘
                │                                   │
    ┌───────────┘                                   └──────────────┐
    │                                                               │
    │ HTTP (8080)                                       HTTP (9980) │
    ▼                                                               ▼
┌─────────────────────────────────┐              ┌──────────────────────────┐
│   Nextcloud Ingress             │              │  Collabora Ingress        │
│   - SSL Termination             │              │  - SSL Termination        │
│   - cert-manager (Let's Encrypt)│              │  - cert-manager (LE)      │
└────────────────┬────────────────┘              └──────────┬───────────────┘
                 │                                           │
                 │                                           │
                 ▼                                           ▼
        ┌─────────────────┐                         ┌───────────────────┐
        │ Nextcloud SVC   │                         │ Collabora SVC     │
        │ ClusterIP:8080  │                         │ ClusterIP:9980    │
        └────────┬────────┘                         └─────────┬─────────┘
                 │                                             │
                 │                                             │
                 ▼                                             ▼
        ┌─────────────────┐                         ┌───────────────────┐
        │ Nextcloud POD   │◄────────────────────────┤ Collabora POD 1   │
        │ (1 replica)     │  WOPI Protocol          │ (2-5 replicas)    │
        │                 │  HTTP Internal          ├───────────────────┤
        │ - Apache/PHP    │                         │ Collabora POD 2   │
        │ - Nextcloud App │                         └───────────────────┘
        └────┬────────┬───┘
             │        │
             │        └────────────────────┐
             │                             │
             │ TCP:3310                    │ TCP:6379
             │                             │
             ▼                             ▼
    ┌────────────────┐            ┌──────────────────┐
    │  ClamAV SVC    │            │   Redis SVC      │
    │ ClusterIP:3310 │            │ ClusterIP:6379   │
    └────────┬───────┘            └────────┬─────────┘
             │                             │
             │                             │
             ▼                             ▼
    ┌────────────────┐            ┌──────────────────┐
    │   ClamAV POD   │            │   Redis POD      │
    │  (1 replica)   │            │  (1 replica)     │
    │                │            │                  │
    │ Container 1:   │            │ - Master Mode    │
    │  clamd         │            │ - No Auth        │
    │                │            │ - No Persistence │
    │ Container 2:   │            └──────────────────┘
    │  freshclam     │
    └────────┬───────┘
             │
             │ TCP:5432
             │
             ▼
    ┌────────────────────┐
    │  PostgreSQL SVC    │
    │  ClusterIP:5432    │
    └────────┬───────────┘
             │
             │
             ▼
    ┌────────────────────┐
    │  PostgreSQL POD    │
    │  (StatefulSet)     │
    │  - 1 replica       │
    │  - Primary Mode    │
    └────────┬───────────┘
             │
             │
             ▼
    ┌────────────────────┐
    │  EBS Volume (gp2)  │
    │  50Gi - Database   │
    └────────────────────┘


┌─────────────────────────────────────────────────────────────────────────┐
│                      ALMACENAMIENTO                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Nextcloud POD tiene acceso a:                                          │
│                                                                          │
│  1. EBS Volume (gp2) - 20Gi                                             │
│     └─> /var/www/html/config                                            │
│         /var/www/html/custom_apps                                       │
│         /var/www/html/data/nextcloud.log                                │
│                                                                          │
│  2. S3 Bucket (via IRSA)                                                │
│     Bucket: nextcloud-colegio-staging-data                              │
│     Region: us-east-2                                                   │
│     └─> Todos los archivos de usuarios (objectstore)                   │
│                                                                          │
│  ClamAV POD tiene acceso a:                                             │
│                                                                          │
│  3. EBS Volume (gp2) - 10Gi                                             │
│     └─> /var/lib/clamav (base de datos de virus)                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Flujo de Datos

### 1. Usuario sube un archivo a Nextcloud

```
Usuario → Nextcloud (https) → Nextcloud POD
                                    │
                                    ├─→ Guarda en S3 (via IRSA)
                                    │
                                    └─→ Envía a ClamAV para escaneo
                                         │
                                         └─→ ClamAV POD (puerto 3310)
                                              │
                                              ├─→ Escanea archivo
                                              │
                                              └─→ Responde: Limpio / Infectado
                                                   │
                                                   └─→ Si infectado: Borrar archivo
```

### 2. Usuario edita un documento con Collabora

```
Usuario abre documento → Nextcloud
                            │
                            ├─→ Lee archivo de S3
                            │
                            └─→ Envía a Collabora via WOPI
                                 │
                                 └─→ Collabora POD carga documento
                                      │
                                      └─→ Usuario edita en navegador
                                           │
                                           └─→ Collabora guarda cambios
                                                │
                                                └─→ Nextcloud guarda en S3
```

### 3. Actualización de definiciones de virus

```
ClamAV POD
  │
  ├─→ Container: clamd (escucha en puerto 3310)
  │    └─→ Usa base de datos de /var/lib/clamav
  │
  └─→ Container: freshclam (actualizador)
       │
       ├─→ Cada hora consulta database.clamav.net
       │
       └─→ Descarga nuevas definiciones a /var/lib/clamav
            │
            └─→ clamd recarga automáticamente
```

## Componentes y Responsabilidades

### Nextcloud POD
- **Propósito**: Aplicación principal
- **Expone**: Puerto 8080 (HTTP)
- **Almacenamiento**: 
  - Configs/logs en EBS (20Gi)
  - Archivos de usuarios en S3
- **Conecta a**:
  - PostgreSQL (base de datos)
  - Redis (caché y file locking)
  - ClamAV (antivirus)
  - Collabora (edición documentos)
  - S3 (almacenamiento archivos)

### Collabora POD (x2-5)
- **Propósito**: Edición de documentos online
- **Expone**: Puerto 9980 (HTTP)
- **Protocolo**: WOPI (Web Application Open Platform Interface)
- **Réplicas**: 2 mínimo, hasta 5 con HPA
- **Recursos**: 500m-2000m CPU, 1-3Gi RAM por pod

### ClamAV POD
- **Propósito**: Antivirus
- **Containers**:
  1. `clamd`: Daemon principal (puerto 3310)
  2. `freshclam`: Actualizador de definiciones
- **Almacenamiento**: EBS 10Gi para base de datos
- **Actualización**: Cada hora automáticamente

### PostgreSQL StatefulSet
- **Propósito**: Base de datos de Nextcloud
- **Almacenamiento**: EBS 50Gi
- **Puerto**: 5432
- **Réplicas**: 1 (single primary)

### Redis POD
- **Propósito**: Caché y file locking
- **Modo**: Standalone (no cluster)
- **Persistencia**: Deshabilitada (memoria)
- **Puerto**: 6379

## Seguridad y Acceso

### SSL/TLS
```
Internet → AWS ALB (HTTPS:443) → Nginx Ingress → Nextcloud/Collabora (HTTP interno)
                │
                └─→ Cert-manager auto-genera certificados Let's Encrypt
```

### Acceso a S3 (IRSA - IAM Roles for Service Accounts)
```
Nextcloud POD usa ServiceAccount: nextcloud-sa
    │
    └─→ ServiceAccount tiene anotación con IAM Role ARN
         │
         └─→ AWS STS asume el rol
              │
              └─→ Nextcloud accede a S3 sin credenciales hardcodeadas
```

### Network Policies (Recomendado - no incluido)
```
Nextcloud POD puede comunicarse con:
  ✓ PostgreSQL
  ✓ Redis
  ✓ ClamAV
  ✓ Collabora
  ✓ Internet (para apps, etc)
  ✗ Otros pods del cluster

Collabora POD puede comunicarse con:
  ✓ Nextcloud
  ✗ Otros pods del cluster

ClamAV POD puede comunicarse con:
  ✓ Internet (para updates)
  ✓ Nextcloud
  ✗ Otros pods del cluster
```

## Recursos y Escalado

### Recursos por Componente

| Componente | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage |
|------------|-------------|-----------|----------------|--------------|---------|
| Nextcloud | 1000m | 3000m | 2Gi | 4Gi | 20Gi EBS + S3 |
| Collabora (x2) | 500m | 2000m | 1Gi | 3Gi | - |
| ClamAV | 500m | 2000m | 2Gi | 4Gi | 10Gi EBS |
| PostgreSQL | 250m | 1000m | 256Mi | 1Gi | 50Gi EBS |
| Redis | 100m | 500m | 128Mi | 512Mi | - |
| **TOTAL** | **~3.35 cores** | **~13.5 cores** | **~7.5Gi** | **~19.5Gi** | **80Gi EBS + S3** |

### Auto-Escalado (HPA)

Solo **Collabora** tiene HPA configurado:

```yaml
Min Replicas: 2
Max Replicas: 5
Trigger:
  - CPU > 70%
  - Memory > 80%
```

**Comportamiento esperado para 80 usuarios**:
- Carga baja: 2 réplicas
- Carga media: 2-3 réplicas
- Carga alta: 3-5 réplicas

## Alta Disponibilidad

### Componentes con HA
✅ **Collabora**: 2+ réplicas
✅ **Nextcloud**: Podría escalarse a 2+ con EFS (actualmente 1 con EBS)

### Componentes sin HA (Single Point of Failure)
❌ **ClamAV**: 1 réplica (EBS no permite ReadWriteMany)
❌ **PostgreSQL**: 1 réplica (configuración actual)
❌ **Redis**: 1 réplica (configuración actual)

### Recomendaciones para mejorar HA

1. **PostgreSQL**:
   ```yaml
   postgresql:
     architecture: replication
     replication:
       enabled: true
   ```

2. **Redis**:
   ```yaml
   redis:
     architecture: replication
     replica:
       replicaCount: 2
   ```

3. **Nextcloud**:
   - Migrar a EFS para permitir múltiples réplicas
   - Actualizar strategy a RollingUpdate

4. **ClamAV**:
   - Difícil mejorar HA sin EFS
   - Alternativa: Tolerar downtime temporal (no crítico)

## Monitoring y Observabilidad

### Logs Importantes

```bash
# Nextcloud
kubectl logs -f -n nextcloud -l app.kubernetes.io/name=nextcloud

# Collabora
kubectl logs -f -n nextcloud -l app=collabora

# ClamAV Daemon
kubectl logs -f -n nextcloud -l app=clamav -c clamav

# ClamAV Updater
kubectl logs -f -n nextcloud -l app=clamav -c freshclam

# PostgreSQL
kubectl logs -f -n nextcloud nextcloud-postgresql-0

# Redis
kubectl logs -f -n nextcloud nextcloud-redis-master-0
```

### Métricas Clave

```bash
# CPU y Memory
kubectl top pods -n nextcloud

# Estado de HPA
kubectl get hpa -n nextcloud

# Eventos del cluster
kubectl get events -n nextcloud --sort-by='.lastTimestamp'
```

### Health Checks

Todos los componentes tienen:
- **Liveness Probe**: Reinicia el pod si falla
- **Readiness Probe**: Quita el pod del balanceo si no está listo

## Backup y Recuperación

### Qué necesitas respaldar

1. **Configuración de Nextcloud**:
   ```bash
   kubectl exec -n nextcloud $POD -- tar czf - /var/www/html/config > nextcloud-config.tar.gz
   ```

2. **Base de datos PostgreSQL**:
   ```bash
   kubectl exec -n nextcloud nextcloud-postgresql-0 -- \
     pg_dump -U nextcloud nextcloud > nextcloud-db-backup.sql
   ```

3. **Archivos de usuarios** (S3):
   - Ya tiene versionado y durabilidad de S3
   - Opcional: Configurar lifecycle policies

4. **Base de datos de ClamAV**:
   - Se regenera automáticamente
   - No crítico respaldar

### Restauración

En caso de desastre:

1. Reinstalar cluster
2. Aplicar manifiestos
3. Restaurar configuración de Nextcloud
4. Restaurar base de datos
5. Los archivos ya están en S3
6. ClamAV descargará definiciones automáticamente

## Costos Mensuales Estimados (AWS us-east-2)

### Compute (EKS)
- **Worker Nodes** (t3.medium x 3): ~$90/mes
- **Control Plane**: $72/mes
- **Subtotal Compute**: ~$162/mes

### Storage
- **EBS gp2**:
  - Nextcloud: 20Gi = $2/mes
  - PostgreSQL: 50Gi = $5/mes
  - ClamAV: 10Gi = $1/mes
- **S3** (ejemplo 100Gi): $2.30/mes
- **Subtotal Storage**: ~$10.30/mes

### Networking
- **Data Transfer**: Variable (estimado $5-20/mes)
- **Load Balancer**: ~$16/mes

### Total Estimado
**~$188-203/mes** para 80 usuarios

### Optimizaciones de Costo

1. **Usar Spot Instances** para worker nodes: -50% a -70%
2. **Reserved Instances** (1 año): -30% a -40%
3. **S3 Intelligent-Tiering**: Ahorros automáticos
4. **Lifecycle policies** en S3: Mover archivos viejos a Glacier

Con optimizaciones: **~$70-100/mes**
