# Part 87: Container Orchestration
## ขั้นตอนที่ 2471-2500: Kubernetes, Helm Charts และ GitOps

Kubernetes namespaces, RBAC, ConfigMaps, Persistent Volumes,
Ingress Controllers, Helm Charts และ ArgoCD

---

## ขั้นตอนที่ 2471: Kubernetes Namespace และ RBAC

```yaml
# k8s/namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: php-app-production
  labels:
    environment: production
    app: php-app
---
apiVersion: v1
kind: Namespace
metadata:
  name: php-app-staging
  labels:
    environment: staging
    app: php-app
```

```yaml
# k8s/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: php-app-sa
  namespace: php-app-production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: php-app-role
  namespace: php-app-production
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: php-app-rolebinding
  namespace: php-app-production
subjects:
  - kind: ServiceAccount
    name: php-app-sa
    namespace: php-app-production
roleRef:
  kind: Role
  name: php-app-role
  apiGroup: rbac.authorization.k8s.io
```

---

## ขั้นตอนที่ 2472: ConfigMaps และ Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: php-app-config
  namespace: php-app-production
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  LOG_CHANNEL: "stderr"
  CACHE_DRIVER: "redis"
  SESSION_DRIVER: "redis"
  QUEUE_CONNECTION: "redis"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
  DB_HOST: "mysql-service"
  DB_PORT: "3306"
  DB_DATABASE: "app_production"
  php.ini: |
    memory_limit = 256M
    max_execution_time = 60
    upload_max_filesize = 50M
    post_max_size = 50M
    opcache.enable = 1
    opcache.memory_consumption = 128
    opcache.jit_buffer_size = 100M
```

```yaml
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: php-app-secrets
  namespace: php-app-production
type: Opaque
stringData:
  APP_KEY: "base64:your-app-key-here"
  DB_PASSWORD: "your-secure-db-password"
  REDIS_PASSWORD: "your-redis-password"
  MAIL_PASSWORD: "your-mail-password"
  AWS_SECRET_ACCESS_KEY: "your-aws-secret"
```

---

## ขั้นตอนที่ 2473: PHP App Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app
  namespace: php-app-production
  labels:
    app: php-app
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: php-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: php-app
        version: v1
    spec:
      serviceAccountName: php-app-sa
      containers:
        - name: php-fpm
          image: registry.example.com/php-app:v1.2.3
          ports:
            - containerPort: 9000
          envFrom:
            - configMapRef:
                name: php-app-config
            - secretRef:
                name: php-app-secrets
          volumeMounts:
            - name: storage
              mountPath: /var/www/html/storage
            - name: php-config
              mountPath: /usr/local/etc/php/conf.d/custom.ini
              subPath: php.ini
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            exec:
              command: ["php-fpm-healthcheck"]
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            exec:
              command: ["php-fpm-healthcheck", "--accepted-conn=500"]
            initialDelaySeconds: 15
            periodSeconds: 10

        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
            - name: storage
              mountPath: /var/www/html/storage
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"

      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: php-app-storage-pvc
        - name: php-config
          configMap:
            name: php-app-config
        - name: nginx-config
          configMap:
            name: nginx-config
```

---

## ขั้นตอนที่ 2474: Persistent Volumes

```yaml
# k8s/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: php-app-storage-pvc
  namespace: php-app-production
spec:
  accessModes:
    - ReadWriteMany  # NFS หรือ EFS
  storageClassName: efs-sc
  resources:
    requests:
      storage: 10Gi
---
# MySQL PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: php-app-production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
```

---

## ขั้นตอนที่ 2475: Ingress Controller

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: php-app-ingress
  namespace: php-app-production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/rate-limit-connections: "20"
    nginx.ingress.kubernetes.io/rate-limit-rps: "10"
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-modsecurity-crs: "true"
spec:
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: php-app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: php-app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: php-app-service
                port:
                  number: 80
```

---

## ขั้นตอนที่ 2476: Helm Chart

```yaml
# helm/php-app/Chart.yaml
apiVersion: v2
name: php-app
description: A Helm chart for PHP Laravel application
type: application
version: 1.0.0
appVersion: "1.0.0"
dependencies:
  - name: mysql
    version: "9.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: mysql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# helm/php-app/values.yaml
replicaCount: 3

image:
  repository: registry.example.com/php-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hostname: app.example.com
  tls: true
  certManager: true

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

env:
  APP_ENV: production
  APP_DEBUG: "false"
  CACHE_DRIVER: redis
  SESSION_DRIVER: redis
  QUEUE_CONNECTION: redis

secrets:
  APP_KEY: ""
  DB_PASSWORD: ""
  REDIS_PASSWORD: ""

mysql:
  enabled: true
  auth:
    database: app
    username: app
  primary:
    persistence:
      size: 20Gi

redis:
  enabled: true
  auth:
    enabled: true
  master:
    persistence:
      size: 5Gi

jobs:
  migrate:
    enabled: true
    command: ["php", "artisan", "migrate", "--force"]
  seeders:
    enabled: false
    command: ["php", "artisan", "db:seed", "--force"]
```

---

## ขั้นตอนที่ 2477: ArgoCD สำหรับ GitOps

```yaml
# argocd/app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: php-app-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/company/php-app-helm
    targetRevision: main
    path: helm/php-app
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"

  destination:
    server: https://kubernetes.default.svc
    namespace: php-app-production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 5
```

---

## ขั้นตอนที่ 2478: CI/CD Pipeline กับ GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Update ArgoCD Application
        run: |
          curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          chmod +x argocd
          ./argocd app set php-app-production \
            --parameter image.tag=${{ needs.build-and-push.outputs.version }} \
            --server ${{ secrets.ARGOCD_SERVER }} \
            --auth-token ${{ secrets.ARGOCD_TOKEN }}
```

---

## สรุปบทที่ 87

| Component | เครื่องมือ | ประโยชน์ |
|-----------|----------|---------|
| Orchestration | Kubernetes | Auto scaling, self-healing |
| Package Manager | Helm | Reusable configs |
| GitOps | ArgoCD | Declarative deployments |
| Ingress | nginx-ingress | Load balancing, TLS |
| Storage | PVC + EFS | Persistent data |
| CI/CD | GitHub Actions | Automated pipeline |

ถัดไป → Part 88: AI Integration กับ PHP
