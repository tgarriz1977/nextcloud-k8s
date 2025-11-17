# nextcloud-k8s

## Instalacion de repos
```bash
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update
kubectl create ns nextcloud
```

## Upgrades
```bash
helm upgrade … -set image.tag=29.0.5-apache
```

## Values

```yaml
image:
  repository: nextcloud
  tag: 29.0.4-apache          # pin a la última estable
  pullPolicy: IfNotPresent

replicaCount: 2               # 2 pods detrás de un svc ClusterIP
hpa:
  enabled: true
  cputhreshold: 60
  minPods: 2
  maxPods: 6

cronjob:
  enabled: true
  schedule: "*/5 * * * *"     # mantenimiento cada 5 min

internalDatabase:
  enabled: false              # importante: no uses SQLite

mariadb:
  enabled: true
  auth:
    rootPassword: "<fuerte>"
    database: nextcloud
    username: nextcloud
    password: "<fuerte>"
  primary:
    persistence:
      enabled: true
      size: 30Gi
      storageClass: gp3

redis:
  enabled: true
  auth:
    enabled: true
    password: "<fuerte>"

persistence:
  enabled: true
  storageClass: gp3           # EBS rápido para código/config
  size: 15Gi

nextcloud:
  host: nube.tu-dominio.com
  username: admin
  password: "<fuerte>"
  phpConfigs:
    uploadLimit.ini: |
      upload_max_filesize = 16G
      post_max_size = 16G
      max_input_time = 3600
      max_execution_time = 3600
    memory.ini: |
      memory_limit = 512M

  configs:
    s3.config.php: |
      <?php
      $CONFIG = array(
        'objectstore' => array(
          'class' => '\\OC\\Files\\ObjectStore\\S3',
          'arguments' => array(
            'bucket'     => 'nextcloud-assets-<uid>',
            'autocreate' => true,
            'key'        => getenv('AWS_ACCESS_KEY_ID'),
            'secret'     => getenv('AWS_SECRET_ACCESS_KEY'),
            'region'     => 'eu-central-1',
            'use_ssl'    => true
          )
        )
      );
    proxy.config.php: |-
      <?php
      $CONFIG = array (
        'trusted_proxies' => array(
          0 => '10.0.0.0/8',
        ),
        'forwarded_for_headers' => array('HTTP_X_FORWARDED_FOR'),
      );
    custom.config.php: |-
      <?php
      $CONFIG = array (
        'default_phone_region' => 'AR',
        'memcache.local' => '\OC\Memcache\APCu',
        'memcache.distributed' => '\OC\Memcache\Redis',
        'memcache.locking' => '\OC\Memcache\Redis',
        'redis' => array(
          'host' => 'nextcloud-redis-master',
          'port' => 6379,
        ),
      );

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: 1g
  tls:
    - secretName: nube-tls
      hosts:
        - nube.tu-dominio.com

metrics:
  enabled: false

## Configuración de inicio (livenessProbe y readinessProbe)
livenessProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

readinessProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
EOF