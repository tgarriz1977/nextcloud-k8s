# Instalación de Collabora Office y ClamAV para Nextcloud

## Resumen

Esta guía te ayudará a agregar dos componentes importantes a tu instalación de Nextcloud:

1. **Collabora Office (CODE)** - Editor de documentos online (similar a Google Docs)
2. **ClamAV** - Antivirus para escanear archivos subidos

## Arquitectura

```
┌─────────────────────────────────────────────────────────┐
│                    Internet                              │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │  AWS ALB/NLB   │
         │  (Ingress)     │
         └────┬───────┬───┘
              │       │
      ┌───────┘       └────────┐
      ▼                        ▼
┌──────────┐          ┌──────────────┐
│Nextcloud │          │  Collabora   │
│  Pods    │◄────────►│    Pods      │
└────┬─────┘          └──────────────┘
     │
     ▼
┌──────────┐          ┌──────────────┐
│ ClamAV   │          │  PostgreSQL  │
│  Pod     │          │     Pod      │
└──────────┘          └──────────────┘
     │                       │
     ▼                       ▼
┌──────────┐          ┌──────────────┐
│  Redis   │          │  S3 Bucket   │
│  Pod     │          │ (archivos)   │
└──────────┘          └──────────────┘
```

## Archivos proporcionados

1. **nextcloud-complete-values.yaml** - Values de Helm actualizado con todas las configuraciones
2. **collabora-deployment.yaml** - Deployment completo de Collabora Office
3. **clamav-deployment.yaml** - Deployment completo de ClamAV
4. **install-collabora-clamav.sh** - Script automatizado de instalación

## Prerrequisitos

✅ Cluster EKS funcionando
✅ Nextcloud ya instalado y funcionando
✅ Nginx Ingress Controller instalado
✅ Cert-manager instalado para certificados SSL
✅ kubectl configurado y con acceso al cluster

## DNS Requerido

Necesitas configurar un registro DNS adicional:

- `collabora.tecnicos.org.ar` → Apuntar al LoadBalancer del Ingress Controller

Puedes obtener la IP/hostname del LoadBalancer con:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

## Instalación Paso a Paso

### Opción 1: Instalación Automatizada (Recomendada)

```bash
# 1. Asegúrate de estar en el directorio correcto
cd ~/nextcloud

# 2. Copiar los archivos proporcionados al servidor
# (nextcloud-complete-values.yaml, collabora-deployment.yaml, 
#  clamav-deployment.yaml, install-collabora-clamav.sh)

# 3. Dar permisos de ejecución al script
chmod +x install-collabora-clamav.sh

# 4. Ejecutar el script
./install-collabora-clamav.sh
```

El script realizará automáticamente:
- ✅ Instalación de ClamAV
- ✅ Descarga de base de datos de virus
- ✅ Instalación de Collabora Office
- ✅ Actualización de Nextcloud
- ✅ Instalación de apps en Nextcloud
- ✅ Configuración completa
- ✅ Verificación de estado

### Opción 2: Instalación Manual

#### Paso 1: Instalar ClamAV

```bash
# Aplicar el deployment de ClamAV
kubectl apply -f clamav-deployment.yaml

# Monitorear el progreso (la descarga de definiciones tarda ~5-10 min)
kubectl get pods -n nextcloud -l app=clamav -w

# Ver logs del init container
kubectl logs -n nextcloud -l app=clamav -c clamav-init -f

# Esperar a que esté Ready
kubectl wait --for=condition=ready pod -l app=clamav -n nextcloud --timeout=600s
```

#### Paso 2: Instalar Collabora Office

```bash
# Aplicar el deployment de Collabora
kubectl apply -f collabora-deployment.yaml

# Monitorear el progreso
kubectl get pods -n nextcloud -l app=collabora -w

# Esperar a que esté Ready
kubectl wait --for=condition=ready pod -l app=collabora -n nextcloud --timeout=300s
```

#### Paso 3: Actualizar Nextcloud

```bash
# Hacer backup del values actual
cp nextcloud-values.yaml nextcloud-values.yaml.backup

# Actualizar con Helm usando el nuevo values
helm upgrade nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --values nextcloud-complete-values.yaml \
  --timeout 10m

# Esperar a que complete
kubectl rollout status deployment/nextcloud -n nextcloud
```

#### Paso 4: Instalar apps en Nextcloud

```bash
# Obtener el nombre del pod de Nextcloud
NEXTCLOUD_POD=$(kubectl get pod -n nextcloud -l app.kubernetes.io/name=nextcloud -o jsonpath='{.items[0].metadata.name}')

# Instalar y configurar Collabora
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ app:install richdocuments || php occ app:enable richdocuments"

kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:set richdocuments wopi_url --value='https://collabora.tecnicos.org.ar'"

# Instalar y configurar ClamAV
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ app:install files_antivirus || php occ app:enable files_antivirus"

kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:set files_antivirus av_mode --value='daemon'"

kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:set files_antivirus av_host --value='clamav-service'"

kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:set files_antivirus av_port --value='3310'"
```

## Verificación

### 1. Verificar estado de los pods

```bash
kubectl get pods -n nextcloud
```

Deberías ver algo como:

```
NAME                                 READY   STATUS    RESTARTS   AGE
clamav-xxxxxxxxxx-xxxxx              2/2     Running   0          10m
collabora-xxxxxxxxxx-xxxxx           1/1     Running   0          8m
collabora-xxxxxxxxxx-xxxxx           1/1     Running   0          8m
nextcloud-xxxxxxxxxx-xxxxx           1/1     Running   0          5m
nextcloud-postgresql-0               1/1     Running   0          2d
nextcloud-redis-master-0             1/1     Running   0          2d
```

### 2. Verificar servicios

```bash
kubectl get svc -n nextcloud | grep -E "(clamav|collabora)"
```

### 3. Verificar ingress

```bash
kubectl get ingress -n nextcloud
```

Deberías ver dos ingress:
- `nextcloud-ingress` para Nextcloud
- `collabora-ingress` para Collabora

### 4. Probar Collabora Office

1. Accede a Nextcloud: https://nubed2.tecnicos.org.ar
2. Ve a **Configuración** → **Administración** → **Collabora Online**
3. Verifica que aparezca: ✅ "Collabora Online server is reachable"
4. Crea un nuevo documento:
   - Archivos → Nuevo → Documento
   - Debería abrirse el editor de Collabora

### 5. Probar ClamAV

1. Ve a **Configuración** → **Administración** → **Basic settings** → **Antivirus**
2. Verifica que el modo sea "Daemon" y el host "clamav-service"
3. Prueba con el archivo de prueba EICAR:

```bash
# Descargar archivo de prueba (inofensivo pero detectado como virus)
wget https://secure.eicar.org/eicar.com.txt

# Intentar subirlo a Nextcloud
# Debería ser bloqueado/eliminado automáticamente
```

## Troubleshooting

### Collabora no conecta

```bash
# Ver logs de Collabora
kubectl logs -n nextcloud -l app=collabora --tail=50

# Verificar que el dominio esté bien configurado
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:get richdocuments wopi_url"

# Probar conectividad desde Nextcloud a Collabora
kubectl exec -n nextcloud $NEXTCLOUD_POD -- curl -k https://collabora-service:9980
```

### ClamAV no escanea archivos

```bash
# Ver logs de ClamAV
kubectl logs -n nextcloud -l app=clamav --tail=50

# Ver logs del actualizador (freshclam)
kubectl logs -n nextcloud -l app=clamav -c freshclam --tail=50

# Verificar conexión desde Nextcloud
kubectl exec -n nextcloud $NEXTCLOUD_POD -- nc -zv clamav-service 3310

# Ver configuración
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:list | grep antivirus"
```

### Colabora muestra error de SSL

Asegúrate de que:
1. El cert-manager haya generado el certificado
2. El DNS apunte correctamente
3. El ingress tenga las anotaciones correctas

```bash
# Ver estado del certificado
kubectl get certificate -n nextcloud collabora-tls

# Ver detalle
kubectl describe certificate -n nextcloud collabora-tls
```

### ClamAV se queda en Init

Si el init container tarda mucho o falla:

```bash
# Ver logs del init
kubectl logs -n nextcloud -l app=clamav -c clamav-init

# Posibles causas:
# - Problemas de red para descargar definiciones
# - Poco espacio en disco
# - Timeout de red

# Verificar espacio en PVC
kubectl exec -n nextcloud -c clamav $(kubectl get pod -n nextcloud -l app=clamav -o name) \
  -- df -h /var/lib/clamav
```

## Recursos Utilizados

### ClamAV
- **CPU Request**: 500m
- **CPU Limit**: 2000m
- **Memory Request**: 2Gi
- **Memory Limit**: 4Gi
- **Storage**: 10Gi (PVC para base de datos de virus)

### Collabora Office
- **CPU Request**: 500m por pod (2 pods = 1000m total)
- **CPU Limit**: 2000m por pod (2 pods = 4000m total)
- **Memory Request**: 1Gi por pod (2 pods = 2Gi total)
- **Memory Limit**: 3Gi por pod (2 pods = 6Gi total)

### Total adicional requerido
- **CPU**: ~2 cores adicionales
- **Memory**: ~6Gi adicionales
- **Storage**: 10Gi adicionales (EBS)

## Costos Estimados (AWS us-east-2)

- **EBS 10Gi (gp2)**: ~$1/mes
- **CPU/Memory**: Depende del tipo de nodos EKS
  - Si usas t3.medium: ~1 nodo adicional (~$30/mes)
  - Si usas t3.large: Puede caber en nodos existentes

## Mantenimiento

### Actualizar definiciones de ClamAV

Las definiciones se actualizan automáticamente cada hora gracias al contenedor `freshclam`.

Para forzar una actualización manual:

```bash
kubectl exec -n nextcloud -c freshclam $(kubectl get pod -n nextcloud -l app=clamav -o name) \
  -- freshclam --config-file=/etc/clamav/freshclam.conf
```

### Actualizar Collabora

```bash
# Actualizar a la última versión
kubectl set image deployment/collabora -n nextcloud \
  collabora=collabora/code:latest

kubectl rollout status deployment/collabora -n nextcloud
```

### Escalar Collabora

Para más usuarios, puedes aumentar réplicas:

```bash
kubectl scale deployment/collabora -n nextcloud --replicas=3
```

O usar el HPA que ya está configurado (escala automáticamente entre 2-5 réplicas).

## Desinstalación

Si necesitas remover estos componentes:

```bash
# Remover ClamAV
kubectl delete -f clamav-deployment.yaml

# Remover Collabora
kubectl delete -f collabora-deployment.yaml

# Desinstalar apps de Nextcloud
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ app:disable files_antivirus"
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ app:disable richdocuments"

# Restaurar values.yaml anterior
helm upgrade nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --values nextcloud-values.yaml.backup \
  --timeout 10m
```

## Soporte

Para más información:

- **Collabora**: https://www.collaboraoffice.com/code/
- **ClamAV**: https://www.clamav.net/
- **Nextcloud Antivirus**: https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/antivirus_configuration.html
- **Nextcloud Collabora**: https://docs.nextcloud.com/server/latest/admin_manual/office/index.html
