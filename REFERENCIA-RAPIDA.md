# Referencia Rápida - Collabora + ClamAV

## Comandos de Instalación

```bash
# Instalación automatizada (recomendado)
chmod +x install-collabora-clamav.sh
./install-collabora-clamav.sh
```

## Comandos de Verificación

```bash
# Ver todos los pods
kubectl get pods -n nextcloud

# Ver servicios
kubectl get svc -n nextcloud

# Ver ingress
kubectl get ingress -n nextcloud

# Ver certificados SSL
kubectl get certificate -n nextcloud
```

## Logs

```bash
# ClamAV
kubectl logs -f -n nextcloud -l app=clamav -c clamav
kubectl logs -f -n nextcloud -l app=clamav -c freshclam

# Collabora
kubectl logs -f -n nextcloud -l app=collabora

# Nextcloud
kubectl logs -f -n nextcloud -l app.kubernetes.io/name=nextcloud
```

## Comandos occ de Nextcloud

```bash
# Variable con el nombre del pod
NEXTCLOUD_POD=$(kubectl get pod -n nextcloud -l app.kubernetes.io/name=nextcloud -o jsonpath='{.items[0].metadata.name}')

# Ver configuración de Collabora
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:get richdocuments wopi_url"

# Ver configuración de ClamAV
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:app:get files_antivirus av_mode"

# Ver todas las apps instaladas
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ app:list"

# Estado general del sistema
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ status"
```

## Pruebas

```bash
# Probar conectividad a Collabora desde Nextcloud
kubectl exec -n nextcloud $NEXTCLOUD_POD -- curl -k https://collabora-service:9980

# Probar conectividad a ClamAV desde Nextcloud
kubectl exec -n nextcloud $NEXTCLOUD_POD -- nc -zv clamav-service 3310

# Descargar archivo de prueba EICAR (para probar antivirus)
wget https://secure.eicar.org/eicar.com.txt
# Luego intenta subirlo a Nextcloud - debe ser bloqueado
```

## Escalado

```bash
# Escalar Collabora manualmente
kubectl scale deployment/collabora -n nextcloud --replicas=3

# Ver HPA de Collabora
kubectl get hpa -n nextcloud

# Ver métricas
kubectl top pods -n nextcloud
```

## Troubleshooting Rápido

```bash
# Reiniciar ClamAV
kubectl rollout restart deployment/clamav -n nextcloud

# Reiniciar Collabora
kubectl rollout restart deployment/collabora -n nextcloud

# Reiniciar Nextcloud
kubectl rollout restart deployment/nextcloud -n nextcloud

# Ver eventos recientes
kubectl get events -n nextcloud --sort-by='.lastTimestamp' | tail -20

# Describir un pod con problemas
kubectl describe pod -n nextcloud <POD_NAME>
```

## Actualización de Definiciones de Virus

```bash
# Forzar actualización inmediata
kubectl exec -n nextcloud -c freshclam $(kubectl get pod -n nextcloud -l app=clamav -o name) \
  -- freshclam --config-file=/etc/clamav/freshclam.conf

# Ver última actualización
kubectl logs -n nextcloud -l app=clamav -c freshclam --tail=20
```

## Verificar Certificados SSL

```bash
# Estado del certificado de Collabora
kubectl describe certificate -n nextcloud collabora-tls

# Ver el secret generado
kubectl get secret -n nextcloud collabora-tls -o yaml
```

## Recursos y Consumo

```bash
# Ver uso de recursos en tiempo real
kubectl top pods -n nextcloud

# Ver detalles de recursos asignados
kubectl describe deployment -n nextcloud clamav
kubectl describe deployment -n nextcloud collabora

# Ver uso de storage
kubectl get pvc -n nextcloud
```

## Backup y Restauración

```bash
# Hacer backup de configuración
kubectl get deployment -n nextcloud clamav -o yaml > clamav-backup.yaml
kubectl get deployment -n nextcloud collabora -o yaml > collabora-backup.yaml

# Exportar configuración de apps de Nextcloud
kubectl exec -n nextcloud $NEXTCLOUD_POD -- su -s /bin/bash www-data -c \
  "php occ config:list" > nextcloud-config-backup.json
```

## Métricas y Monitoreo

```bash
# Ver réplicas actuales y deseadas
kubectl get deployment -n nextcloud

# Ver estado del HPA
kubectl describe hpa -n nextcloud collabora-hpa

# Ver métricas de CPU y memoria
kubectl top pods -n nextcloud -l app=collabora
kubectl top pods -n nextcloud -l app=clamav
```

## DNS y Conectividad

```bash
# Probar resolución DNS dentro del cluster
kubectl run -it --rm debug --image=busybox --restart=Never -n nextcloud -- nslookup collabora-service
kubectl run -it --rm debug --image=busybox --restart=Never -n nextcloud -- nslookup clamav-service

# Probar conectividad HTTP
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n nextcloud -- \
  curl -v http://collabora-service:9980

# Probar puerto de ClamAV
kubectl run -it --rm debug --image=busybox --restart=Never -n nextcloud -- \
  nc -zv clamav-service 3310
```

## Limpieza

```bash
# Limpiar pods en estado Failed/Error
kubectl delete pods -n nextcloud --field-selector status.phase=Failed
kubectl delete pods -n nextcloud --field-selector status.phase=Error

# Ver pods que se están reiniciando mucho
kubectl get pods -n nextcloud --sort-by='.status.containerStatuses[0].restartCount'
```

## URLs Importantes

- Nextcloud: https://nubed2.tecnicos.org.ar
- Collabora: https://collabora.tecnicos.org.ar
- Configuración Collabora en Nextcloud: Settings → Administration → Collabora Online
- Configuración Antivirus en Nextcloud: Settings → Administration → Basic settings → Antivirus

## Contactos de Soporte

- Documentación Collabora: https://sdk.collaboraonline.com/docs/index.html
- Documentación ClamAV: https://docs.clamav.net/
- Nextcloud Admin Manual: https://docs.nextcloud.com/server/latest/admin_manual/
