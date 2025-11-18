# Nextcloud + Collabora Office + ClamAV

Paquete completo para agregar **Collabora Office** (editor de documentos) y **ClamAV** (antivirus) a tu instalación existente de Nextcloud en Kubernetes (AWS EKS).

## 📦 Archivos Incluidos

### 📄 Manifiestos YAML

1. **nextcloud-complete-values.yaml** (7.9 KB)
   - Values actualizado de Helm para Nextcloud
   - Incluye configuraciones para Collabora y ClamAV
   - Adaptado para tu setup (S3 + EBS + IRSA)

2. **collabora-deployment.yaml** (5.3 KB)
   - Deployment completo de Collabora Office
   - 2 réplicas con HPA (auto-escalado 2-5)
   - Ingress configurado con SSL
   - Service y Secret incluidos

3. **clamav-deployment.yaml** (7.5 KB)
   - Deployment completo de ClamAV
   - Init container para descarga inicial de definiciones
   - Sidecar container (freshclam) para actualización automática
   - PVC de 10Gi para base de datos de virus
   - Service incluido

### 🚀 Script de Instalación

4. **install-collabora-clamav.sh** (8.9 KB)
   - Script automatizado que instala todo
   - Incluye verificaciones y validaciones
   - Colorized output con progreso
   - Manejo de errores robusto

### 📚 Documentación

5. **INSTALACION.md** (12 KB)
   - Guía completa paso a paso
   - Dos opciones: automatizada y manual
   - Sección de troubleshooting
   - Verificación y pruebas
   - Información de recursos y costos

6. **REFERENCIA-RAPIDA.md** (5.5 KB)
   - Comandos útiles para el día a día
   - Troubleshooting rápido
   - Comandos de logs, escalado, verificación
   - Pruebas y diagnósticos

## 🎯 ¿Qué Obtienes?

### Collabora Office (CODE)
- ✅ Editor de documentos online (Word, Excel, PowerPoint)
- ✅ Edición colaborativa en tiempo real
- ✅ Compatible con formatos Microsoft Office
- ✅ 2 réplicas con auto-escalado (hasta 5)
- ✅ SSL/TLS configurado
- ✅ Diccionarios español e inglés

### ClamAV Antivirus
- ✅ Escaneo automático de archivos subidos
- ✅ Base de datos actualizada cada hora
- ✅ Acción configurable (borrar/cuarentena)
- ✅ Límite de 100MB por archivo
- ✅ Logging y monitoreo

## 🚀 Inicio Rápido

### Instalación en 3 pasos:

```bash
# 1. Copiar archivos al servidor
scp *.yaml *.sh *.md admin@tu-servidor:~/nextcloud/

# 2. Conectarse al servidor
ssh admin@tu-servidor

# 3. Ejecutar instalación
cd ~/nextcloud
chmod +x install-collabora-clamav.sh
./install-collabora-clamav.sh
```

### ¿Prefieres instalación manual?

Consulta **INSTALACION.md** para instrucciones detalladas paso a paso.

## 📋 Prerrequisitos

- ✅ Cluster EKS funcionando
- ✅ Nextcloud ya instalado (versión 32.0.1)
- ✅ Nginx Ingress Controller
- ✅ Cert-manager para SSL
- ✅ kubectl configurado

## 🌐 DNS Requerido

Necesitas configurar este registro DNS adicional:

```
collabora.tecnicos.org.ar → LoadBalancer del Ingress Controller
```

El LoadBalancer es el mismo que usas para Nextcloud.

## 💰 Costos Estimados (AWS us-east-2)

### Recursos Adicionales:
- **CPU**: ~2 cores
- **Memory**: ~6Gi
- **Storage**: 10Gi EBS (gp2)

### Costo Mensual:
- EBS 10Gi: **~$1/mes**
- Compute: **~$0-30/mes** (depende de si necesitas nodo adicional)

Para 80 usuarios, probablemente quepa en tus nodos existentes.

## 📊 Arquitectura

```
Internet
   │
   ▼
AWS ALB (Ingress)
   │
   ├─────────────────────────────┐
   ▼                             ▼
Nextcloud ◄────────────► Collabora Office
   │                      (2-5 pods)
   ├──────┐
   ▼      ▼
ClamAV  PostgreSQL
   │      │
   ▼      ▼
Redis   S3 Bucket
```

## 🔍 Verificación Post-Instalación

### 1. Verificar Pods
```bash
kubectl get pods -n nextcloud
```

### 2. Probar Collabora
- Accede a: https://nubed2.tecnicos.org.ar
- Ve a: Configuración → Administración → Collabora Online
- Verifica: ✅ "Collabora Online server is reachable"

### 3. Probar ClamAV
- Descarga: `wget https://secure.eicar.org/eicar.com.txt`
- Intenta subirlo a Nextcloud
- Resultado esperado: ❌ Bloqueado/Eliminado

## 📖 Documentación

| Archivo | Propósito |
|---------|-----------|
| **INSTALACION.md** | Guía completa de instalación y configuración |
| **REFERENCIA-RAPIDA.md** | Comandos útiles para operaciones diarias |

## 🆘 Soporte y Troubleshooting

Consulta la sección de **Troubleshooting** en **INSTALACION.md** para:
- Collabora no conecta
- ClamAV no escanea
- Problemas de SSL
- Logs y diagnósticos

### Comandos Útiles Rápidos:

```bash
# Ver logs de Collabora
kubectl logs -f -n nextcloud -l app=collabora

# Ver logs de ClamAV
kubectl logs -f -n nextcloud -l app=clamav

# Reiniciar servicio
kubectl rollout restart deployment/collabora -n nextcloud
kubectl rollout restart deployment/clamav -n nextcloud
```

## 🔄 Mantenimiento

### Actualización Automática
- **ClamAV**: Definiciones actualizadas automáticamente cada hora
- **Collabora**: Usa image `latest` (actualizar con rollout restart)

### Escalado
```bash
# Manual
kubectl scale deployment/collabora -n nextcloud --replicas=3

# Automático (ya configurado)
# HPA escala entre 2-5 réplicas basado en CPU/Memory
```

## ⚙️ Configuración

### Colabora
- **URL WOPI**: https://collabora.tecnicos.org.ar
- **Réplicas**: 2 (auto-scale hasta 5)
- **Recursos**: 500m-2000m CPU, 1-3Gi RAM por pod

### ClamAV
- **Modo**: Daemon
- **Host**: clamav-service
- **Puerto**: 3310
- **Max File Size**: 100MB
- **Acción**: Delete (borrar archivos infectados)

## 📞 Enlaces Útiles

- [Collabora Documentation](https://www.collaboraoffice.com/code/)
- [ClamAV Documentation](https://www.clamav.net/)
- [Nextcloud Admin Manual](https://docs.nextcloud.com/server/latest/admin_manual/)

## ✅ Checklist de Instalación

- [ ] DNS configurado para collabora.tecnicos.org.ar
- [ ] Archivos copiados al servidor
- [ ] Script ejecutado exitosamente
- [ ] Pods en estado Running
- [ ] Certificados SSL emitidos
- [ ] Collabora conecta desde Nextcloud
- [ ] ClamAV escanea archivos (prueba con EICAR)
- [ ] Documentos se editan correctamente

## 🎉 ¡Listo!

Una vez completada la instalación, tendrás:
- 📝 Edición de documentos online con Collabora
- 🛡️ Protección antivirus con ClamAV
- 🔄 Actualizaciones automáticas
- 📈 Auto-escalado configurado
- 🔒 SSL/TLS en todos los servicios

---

**¿Necesitas ayuda?** Revisa **INSTALACION.md** o **REFERENCIA-RAPIDA.md**
