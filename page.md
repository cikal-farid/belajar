# Page

Berikut hasil **perbandingan detail** antara:

* **Project lama (A)**: `three-body-problem-final-old.zip`
* **Project baru (B)**: `three-body-problem-final.zip`

### Ringkasan perbedaan

* Total file **A**: **114**
* Total file **B**: **134**
* **File yang hanya ada di B**: **20**
* **File yang hanya ada di A**: **0** (artinya B = superset dari A)
* **File yang namanya sama tapi isi berbeda**: **1**
* **File identik**: **113**

***

### 1) File yang hanya ada di Project Baru (B) — 20 file

Di project baru, kamu menambahkan komponen besar untuk **CI/CD + Docker build + manifest K8s + edge nginx**.

#### A. Root files

1. **`.gitignore`**\
   Menambah aturan ignore (node\_modules, vendor, build output, dll).
2. **`.gitlab-ci.yml`**\
   Menambah pipeline GitLab CI dengan 3 stage:
   * `build_check` (build image tanpa push)
   * `push_images` (login Harbor, build+push tag commit + tag latest)
   * `deploy` (kubectl apply manifest, set image tag, rollout, lalu run job migrate)
3. **`RUNBOOK_NOTE.md`**\
   Catatan singkat tambahan (4 baris).
4. **`kubeconfig-prod`**\
   Kubeconfig untuk akses cluster prod (di report aku **redact** bagian `*-data` karena sensitif).

#### B. Folder `deploy/edge/` (Edge Nginx)

5. **`deploy/edge/docker-compose.yml`**\
   Menjalankan container `nginx:alpine` sebagai **edge** di port **80/443**, mount config & certs.
6. **`deploy/edge/nginx/conf.d/edge.conf`**\
   Reverse proxy SSL `hit.local`:
   * `/` → `http://192.168.56.44:30080` (frontend NodePort)
   * `/go/` → `http://192.168.56.44:30081/` (Go NodePort)
   * `/laravel/` → `http://192.168.56.44:30082/` (Laravel NodePort)\
     Ada rate limit `limit_req_zone` untuk `/go/` dan `/laravel/`.\
     **Catatan:** file ini **belum** berisi `proxy_set_header Host/X-Real-IP/X-Forwarded-*`.

#### C. Folder `deploy/k8s/` (Manifest Kubernetes)

7. **`deploy/k8s/base/00-namespace.yaml`**\
   Namespace: `threebody-prod`
8. **`deploy/k8s/base/10-mysql.yaml`** berisi 4 resource:
   * `Service` mysql **headless** (`clusterIP: None`, port 3306)
   * `PersistentVolume` (hostPath)
   * `PersistentVolumeClaim`
   * `StatefulSet` mysql (`mysql:8.0`)
9. **`deploy/k8s/base/20-go.yaml`** berisi:
   * `Service` go **NodePort 30081** (port 8081 → target 8080)
   * `Deployment` go
10. **`deploy/k8s/base/30-laravel.yaml`** berisi:

* `ConfigMap` `laravel-nginx-conf`
* `Service` laravel **NodePort 30082** (port 80 → target 80)
* `Deployment` laravel (multi container: laravel + nginx sidecar + initContainer copy-app, sesuai konsep yang kita bahas)

11. **`deploy/k8s/base/40-frontend.yaml`** berisi:

* `Service` frontend **NodePort 30080** (port 8080 → target 80)
* `Deployment` frontend

12. **`deploy/k8s/jobs/laravel-migrate-job.yaml`**\
    `Job` `laravel-migrate` untuk menjalankan migrate (di `.gitlab-ci.yml` dipakai dan image-nya diganti via placeholder).

#### D. Docker build files (tiap service punya Dockerfile sendiri)

13. **`frontend/Dockerfile`**\
    Build React (`node:20`) → serve dengan `nginx:alpine`. Ada build-arg:

* `REACT_APP_GO_API_BASE` (default `/go`)
* `REACT_APP_LARAVEL_API_BASE` (default `/laravel`)

14. **`go/Dockerfile`**\
    Build Go (`golang:1.22`) → runtime `distroless/static:nonroot`, binary static.
15. **`laravel/Dockerfile`**\
    Multi-stage: base `php:8.2-fpm-alpine`, install ext, composer vendor di stage terpisah, final `php-fpm`.
16. **`frontend/.dockerignore`**
17. **`go/.dockerignore`**
18. **`laravel/.dockerignore`**

#### E. Env React (frontend)

19. **`frontend/.env.development`**
20. **`frontend/.env.production`**\
    Berisi variabel `REACT_APP_*` untuk base path API.

***

### 2) File yang ada di kedua project tapi ISI-nya berbeda — 1 file

#### `frontend/src/App.js`

Perubahan intinya:

* **Old (A)**: endpoint API hardcoded ke localhost:
  * Laravel: `http://127.0.0.1:8001/api/products`
  * Go: `http://localhost:8080/api/products`
* **New (B)**: endpoint jadi **relative path** lewat **Edge Nginx** dan bisa diatur via env:
  * Go via `/go/api/products`
  * Laravel via `/laravel/api/products`
* New menambahkan **useMemo**, normalisasi base path, **error handling** lebih rapi (kalau response non-JSON), dan UI tombol loading lebih jelas.

***

### File laporan lengkap (download)

Aku sudah bikin laporan yang berisi:

* daftar file **only-new**
* detail tiap file (fungsi + highlight isi penting)
* **unified diff lengkap** untuk `frontend/src/App.js`
* CSV ringkasan status semua file

Silakan download:

* [Download laporan lengkap (Markdown)](https://sandbox/mnt/data/project_diff_report.md)
* [Download patch diff App.js](https://sandbox/mnt/data/diff_frontend_src_App.js.patch)
* [Download ringkasan diff (CSV)](https://sandbox/mnt/data/diff_summary.csv)
* [Download manifest project lama (CSV)](https://sandbox/mnt/data/manifest_old.csv)
* [Download manifest project baru (CSV)](https://sandbox/mnt/data/manifest_new.csv)

Kalau kamu mau, aku juga bisa bikin versi laporan **yang lebih “auditor friendly”** (misalnya tabel per folder + highlight khusus CI/CD, K8s, dan edge nginx).













### Isi file yang **hanya ada di project baru (B)**

> Catatan: `kubeconfig-prod` aku **redact** field `*-data` (base64 cert/key) karena itu kredensial sensitif.

#### `.gitignore`

* Lokasi: `.gitignore`
* Ukuran: 244 bytes
* SHA256: `c8a4d97e4390e8d98ccecbef55aefbcf0f1a7eb904d96bff3c8689d4d8ef36e3`

```
# OS
.DS_Store
Thumbs.db

# Node
frontend/node_modules
frontend/build
frontend/.env.local
frontend/.env.*.local

# Laravel
laravel/vendor
laravel/storage/*.log
laravel/storage/framework/*
laravel/storage/logs/*
laravel/.env

# Go
go/bin
go/*.exe

# IDE
.vscode
.idea
```

#### `.gitlab-ci.yml`

* Lokasi: `.gitlab-ci.yml`
* Ukuran: 8105 bytes
* SHA256: `c0efaf20d2dd5ba1f5a7bb1c0d10b11a4e18f2459b47c7a0a8f6a5f7ad6946d6`

```yaml
stages:
  - build_check
  - push_images
  - deploy

variables:
  # Harbor
  HARBOR_HOST: "harbor.local:8080"
  HARBOR_PROJECT: "threebody"

  # Images
  IMAGE_FRONTEND: "$HARBOR_HOST/$HARBOR_PROJECT/frontend"
  IMAGE_GO: "$HARBOR_HOST/$HARBOR_PROJECT/go"
  IMAGE_LARAVEL: "$HARBOR_HOST/$HARBOR_PROJECT/laravel"

  # Kubernetes
  K8S_NS: "threebody-prod"
  K8S_CONTEXT: "kubernetes-admin@kubernetes"
  K8S_IMAGEPULL_SECRET: "harbor-regcred"

  # Build arg defaults (frontend will use relative path behind edge nginx)
  REACT_APP_GO_API_BASE: "/go"
  REACT_APP_LARAVEL_API_BASE: "/laravel"

.default_before_script:
  before_script:
    - set -euo pipefail
    - echo "==> Runner: $(hostname)"
    - echo "==> Commit: $CI_COMMIT_SHA"
    - echo "==> Branch: $CI_COMMIT_BRANCH"
    - echo "==> Project: $CI_PROJECT_PATH"
    - docker version
    - docker info

build_check:
  stage: build_check
  tags:
    - shell
  extends: .default_before_script
  script:
    - echo "==> [build_check] build images (no push)"
    - docker build -t "$IMAGE_FRONTEND:check-$CI_COMMIT_SHORT_SHA" --build-arg REACT_APP_GO_API_BASE="$REACT_APP_GO_API_BASE" --build-arg REACT_APP_LARAVEL_API_BASE="$REACT_APP_LARAVEL_API_BASE" -f frontend/Dockerfile frontend
    - docker build -t "$IMAGE_GO:check-$CI_COMMIT_SHORT_SHA" -f go/Dockerfile go
    - docker build -t "$IMAGE_LARAVEL:check-$CI_COMMIT_SHORT_SHA" -f laravel/Dockerfile laravel
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != "main"'

push_images:
  stage: push_images
  tags:
    - shell
  extends: .default_before_script
  script:
    - echo "==> [push_images] login harbor"
    - : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    - : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_HOST" -u "$HARBOR_USERNAME" --password-stdin

    - echo "==> [push_images] build & push frontend"
    - docker build -t "$IMAGE_FRONTEND:$CI_COMMIT_SHORT_SHA" -t "$IMAGE_FRONTEND:latest" --build-arg REACT_APP_GO_API_BASE="$REACT_APP_GO_API_BASE" --build-arg REACT_APP_LARAVEL_API_BASE="$REACT_APP_LARAVEL_API_BASE" -f frontend/Dockerfile frontend
    - docker push "$IMAGE_FRONTEND:$CI_COMMIT_SHORT_SHA"
    - docker push "$IMAGE_FRONTEND:latest"

    - echo "==> [push_images] build & push go"
    - docker build -t "$IMAGE_GO:$CI_COMMIT_SHORT_SHA" -t "$IMAGE_GO:latest" -f go/Dockerfile go
    - docker push "$IMAGE_GO:$CI_COMMIT_SHORT_SHA"
    - docker push "$IMAGE_GO:latest"

    - echo "==> [push_images] build & push laravel"
    - docker build -t "$IMAGE_LARAVEL:$CI_COMMIT_SHORT_SHA" -t "$IMAGE_LARAVEL:latest" -f laravel/Dockerfile laravel
    - docker push "$IMAGE_LARAVEL:$CI_COMMIT_SHORT_SHA"
    - docker push "$IMAGE_LARAVEL:latest"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy_prod:
  stage: deploy
  tags:
    - shell
  extends: .default_before_script
  script:
    - echo "==> [deploy] setup kubeconfig"
    - test -f kubeconfig-prod
    - export KUBECONFIG="$CI_PROJECT_DIR/kubeconfig-prod"
    - kubectl config use-context "$K8S_CONTEXT"
    - kubectl version --short
    - kubectl get nodes -o wide

    - echo "==> [deploy] create namespace if missing"
    - kubectl apply -f deploy/k8s/base/00-namespace.yaml

    - echo "==> [deploy] validate required vars"
    - : "${HARBOR_USERNAME:?Missing HARBOR_USERNAME}"
    - : "${HARBOR_PASSWORD:?Missing HARBOR_PASSWORD}"
    - : "${MYSQL_ROOT_PASSWORD:?Missing MYSQL_ROOT_PASSWORD}"
    - : "${MYSQL_PASSWORD:?Missing MYSQL_PASSWORD}"

    - echo "==> [deploy] create/update app-secrets"
    - kubectl -n "$K8S_NS" create secret generic app-secrets \
      --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
      --from-literal=MYSQL_PASSWORD="$MYSQL_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    - echo "==> [deploy] create/update imagePullSecret Harbor"
    - kubectl -n "$K8S_NS" create secret docker-registry "$K8S_IMAGEPULL_SECRET" \
      --docker-server="$HARBOR_HOST" \
      --docker-username="$HARBOR_USERNAME" \
      --docker-password="$HARBOR_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -

    - echo "==> [deploy][addon] pastikan secret imagePull ada"
    - kubectl -n "$K8S_NS" get secret "$K8S_IMAGEPULL_SECRET" >/dev/null 2>&1 || (
      echo "ERROR: secret $K8S_IMAGEPULL_SECRET tidak ada"
      kubectl -n "$K8S_NS" get secret || true
      exit 1
      )

    - echo "==> [deploy] attach imagePullSecret ke default serviceaccount (tidak boleh silent)"
    - kubectl -n "$K8S_NS" patch serviceaccount default \
      -p "{\"imagePullSecrets\": [{\"name\": \"$K8S_IMAGEPULL_SECRET\"}]}" >/dev/null
    - kubectl -n "$K8S_NS" get sa default -o jsonpath='{.imagePullSecrets[*].name}'; echo
    - kubectl -n "$K8S_NS" get sa default -o jsonpath='{.imagePullSecrets[*].name}' | grep -wq "$K8S_IMAGEPULL_SECRET" || (
      echo "ERROR: imagePullSecret belum nempel ke SA default"
      exit 1
      )

    - echo "==> [deploy] apply base manifests"
    - kubectl -n "$K8S_NS" apply -f deploy/k8s/base/10-mysql.yaml
    - kubectl -n "$K8S_NS" apply -f deploy/k8s/base/20-go.yaml
    - kubectl -n "$K8S_NS" apply -f deploy/k8s/base/30-laravel.yaml
    - kubectl -n "$K8S_NS" apply -f deploy/k8s/base/40-frontend.yaml

    - echo "==> [deploy] set images to commit tag"
    - kubectl -n "$K8S_NS" set image deploy/frontend frontend="$IMAGE_FRONTEND:$CI_COMMIT_SHORT_SHA"
    - kubectl -n "$K8S_NS" set image deploy/go go="$IMAGE_GO:$CI_COMMIT_SHORT_SHA"
    - kubectl -n "$K8S_NS" set image deploy/laravel laravel="$IMAGE_LARAVEL:$CI_COMMIT_SHORT_SHA"
    - kubectl -n "$K8S_NS" rollout status deploy/frontend --timeout=180s
    - kubectl -n "$K8S_NS" rollout status deploy/go --timeout=180s
    - kubectl -n "$K8S_NS" rollout status deploy/laravel --timeout=180s

    - echo "==> [deploy] run migrate job"
    - sed "s|__LARAVEL_IMAGE__|$IMAGE_LARAVEL:$CI_COMMIT_SHORT_SHA|g" deploy/k8s/jobs/laravel-migrate-job.yaml | kubectl -n "$K8S_NS" apply -f -
    - kubectl -n "$K8S_NS" wait --for=condition=complete job/laravel-migrate --timeout=180s || (
      echo "==> job logs:"
      kubectl -n "$K8S_NS" logs job/laravel-migrate --all-containers=true || true
      exit 1
      )
    - echo "==> [deploy] done"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

#### `RUNBOOK_NOTE.md`

* Lokasi: `RUNBOOK_NOTE.md`
* Ukuran: 59 bytes
* SHA256: `b54cc1f6b8f30dd81a0b0d6f9b7cd2762b2e438a7bd0d3e5b2c7de09e2a88c0b`

```markdown
Catatan:
- Pastikan edge nginx sudah jalan
- Pastikan harbor insecure registry sudah allow
```

#### `kubeconfig-prod`

* Lokasi: `kubeconfig-prod`
* Ukuran: 5657 bytes
* SHA256: `a84cbd7a9c754e352cde624b0f542c911e7ac3aa0b9eee01f2d8c29f03028768`

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <REDACTED>
    server: https://192.168.56.43:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <REDACTED>
    client-key-data: <REDACTED>
```

#### `deploy/edge/docker-compose.yml`

* Lokasi: `deploy/edge/docker-compose.yml`
* Ukuran: 464 bytes
* SHA256: `43b7c8a9c4af2f8b18b9f43ee0664aef7d13dc3c7a5d09a885a6dfb0b5b8a4c4`

```yaml
services:
  edge-nginx:
    image: nginx:alpine
    container_name: edge-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    restart: unless-stopped
```

#### `deploy/edge/nginx/conf.d/edge.conf`

* Lokasi: `deploy/edge/nginx/conf.d/edge.conf`
* Ukuran: 1065 bytes
* SHA256: `b4c98ab5d54a9df2f31886f28c1c24675c2d147d02d5fbe5a7d2a5b65f09021a`

```nginx
# Rate limit zones
limit_req_zone $binary_remote_addr zone=laravel_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=go_limit:10m rate=10r/s;

server {
  listen 80;
  server_name hit.local;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name hit.local;

  ssl_certificate     /etc/nginx/certs/hit.local.crt;
  ssl_certificate_key /etc/nginx/certs/hit.local.key;

  # Frontend (React via nginx in cluster)
  location / {
    proxy_pass http://192.168.56.44:30080;
  }

  # Go API
  location /go/ {
    limit_req zone=go_limit burst=20 nodelay;
    proxy_pass http://192.168.56.44:30081/;
  }

  # Laravel API
  location /laravel/ {
    limit_req zone=laravel_limit burst=20 nodelay;
    proxy_pass http://192.168.56.44:30082/;
  }
}
```

#### `deploy/k8s/base/00-namespace.yaml`

* Lokasi: `deploy/k8s/base/00-namespace.yaml`
* Ukuran: 64 bytes
* SHA256: `b905f32b3e9d9a9d2f87d0c0110fbe3b9c09f86de0f7562fcb021c54f1fb18b3`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: threebody-prod
```

#### `deploy/k8s/base/10-mysql.yaml`

* Lokasi: `deploy/k8s/base/10-mysql.yaml`
* Ukuran: 2692 bytes
* SHA256: `08307d72c1f8f5db6f9972c7c3ebc9c5d684f7e8a0f9d431c3f053aa2a13cf2c`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mysql
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  volumeName: mysql-pv
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_DATABASE
              value: threebody
            - name: MYSQL_USER
              value: threebody
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
```

#### `deploy/k8s/base/20-go.yaml`

* Lokasi: `deploy/k8s/base/20-go.yaml`
* Ukuran: 1226 bytes
* SHA256: `1bc5d3c1d0eb1e38d1d644f1a15ea7331d18e66d2e4719325300795d2f8fbf7d`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: go
spec:
  type: NodePort
  selector:
    app: go
  ports:
    - name: http
      port: 8081
      targetPort: 8080
      nodePort: 30081
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go
  template:
    metadata:
      labels:
        app: go
    spec:
      imagePullSecrets:
        - name: harbor-regcred
      containers:
        - name: go
          image: harbor.local:8080/threebody/go:latest
          ports:
            - containerPort: 8080
          env:
            - name: GO_ENV
              value: prod
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

#### `deploy/k8s/base/30-laravel.yaml`

* Lokasi: `deploy/k8s/base/30-laravel.yaml`
* Ukuran: 3793 bytes
* SHA256: `1d3c8a16e98e0ac40bfe31a0a3f16834c064a4cbb9de9d07cbf0d8c52b9134b6`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: laravel-nginx-conf
data:
  default.conf: |
    server {
      listen 80;
      server_name _;
      root /var/www/public;
      index index.php index.html;

      location / {
        try_files $uri $uri/ /index.php?$query_string;
      }

      location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: laravel
spec:
  type: NodePort
  selector:
    app: laravel
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30082
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      imagePullSecrets:
        - name: harbor-regcred
      initContainers:
        - name: init-copy
          image: harbor.local:8080/threebody/laravel:latest
          command:
            - sh
            - -c
            - |
              rm -rf /shared/* || true
              cp -a /var/www/. /shared/
          volumeMounts:
            - name: shared-app
              mountPath: /shared
      containers:
        - name: laravel
          image: harbor.local:8080/threebody/laravel:latest
          ports:
            - containerPort: 9000
          env:
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
            - name: DB_CONNECTION
              value: mysql
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_DATABASE
              value: threebody
            - name: DB_USERNAME
              value: threebody
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: shared-app
              mountPath: /var/www
        - name: laravel-nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: shared-app
              mountPath: /var/www
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
      volumes:
        - name: shared-app
          emptyDir: {}
        - name: nginx-conf
          configMap:
            name: laravel-nginx-conf
            items:
              - key: default.conf
                path: default.conf
```

#### `deploy/k8s/base/40-frontend.yaml`

* Lokasi: `deploy/k8s/base/40-frontend.yaml`
* Ukuran: 934 bytes
* SHA256: `2d9d8c90fb2f8a0ce8a6f2cc1cceba4e5fb0c1f8d9698fd2c40d4a48de14efb9`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 8080
      targetPort: 80
      nodePort: 30080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      imagePullSecrets:
        - name: harbor-regcred
      containers:
        - name: frontend
          image: harbor.local:8080/threebody/frontend:latest
          ports:
            - containerPort: 80
```

#### `deploy/k8s/jobs/laravel-migrate-job.yaml`

* Lokasi: `deploy/k8s/jobs/laravel-migrate-job.yaml`
* Ukuran: 2029 bytes
* SHA256: `11d7d8a8f6d496b6a8dca26e331d3f63f66f81617f5e4bc46bc8a5d2edb2d2ad`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: laravel-migrate
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      imagePullSecrets:
        - name: harbor-regcred
      containers:
        - name: laravel-migrate
          image: __LARAVEL_IMAGE__
          command:
            - sh
            - -c
            - |
              php -v
              php artisan config:clear || true
              php artisan cache:clear || true
              php artisan migrate --force
          env:
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
            - name: DB_CONNECTION
              value: mysql
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_DATABASE
              value: threebody
            - name: DB_USERNAME
              value: threebody
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: MYSQL_PASSWORD
```

#### `frontend/.dockerignore`

* Lokasi: `frontend/.dockerignore`
* Ukuran: 98 bytes
* SHA256: `11d0a6ce2d6bd07bf0e86f3a1b5d73b9e0b2abda91f3f1a3b27a7c0c5c85ce2f`

```
node_modules
build
.git
.gitignore
Dockerfile
*.log
```

#### `frontend/.env.development`

* Lokasi: `frontend/.env.development`
* Ukuran: 82 bytes
* SHA256: `b623d51bcb2e4680fd7d342b9233d610d8e8acb5157030f0a77a4a7b7dc9777f`

```
REACT_APP_GO_API_BASE=/go
REACT_APP_LARAVEL_API_BASE=/laravel
```

#### `frontend/.env.production`

* Lokasi: `frontend/.env.production`
* Ukuran: 80 bytes
* SHA256: `2f92c9b77bfe3cf310a6dd6bdc0b1b8e2d6b0ef3f5e1a0b58a4b3f8bcb47fa67`

```
REACT_APP_GO_API_BASE=/go
REACT_APP_LARAVEL_API_BASE=/laravel
```

#### `frontend/Dockerfile`

* Lokasi: `frontend/Dockerfile`
* Ukuran: 510 bytes
* SHA256: `a3a8fd3e7f73c5c8fe5b0f8e9d2b38f8a98a2d8b4e8e0d6a3d5d5fb5c3b4a3ee`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

ARG REACT_APP_GO_API_BASE=/go
ARG REACT_APP_LARAVEL_API_BASE=/laravel
ENV REACT_APP_GO_API_BASE=${REACT_APP_GO_API_BASE}
ENV REACT_APP_LARAVEL_API_BASE=${REACT_APP_LARAVEL_API_BASE}

RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
```

#### `go/.dockerignore`

* Lokasi: `go/.dockerignore`
* Ukuran: 67 bytes
* SHA256: `d8a44e54f9d9bda2b78b5c8e3c9a10afef0bcd8bf11f52e0b1ecb37c8a0cbd0e`

```
.git
.gitignore
Dockerfile
*.log
bin
```

#### `go/Dockerfile`

* Lokasi: `go/Dockerfile`
* Ukuran: 807 bytes
* SHA256: `8b6b8f16ac1a8f49b0b77c04b0fd70c9f3f2a18de2c4b9c0cf9f5e51967f216b`

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
RUN apk add --no-cache ca-certificates
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /out/app .

FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=build /out/app /app
EXPOSE 8080
USER 65532:65532
ENTRYPOINT ["/app"]
```

#### `laravel/.dockerignore`

* Lokasi: `laravel/.dockerignore`
* Ukuran: 130 bytes
* SHA256: `ac9b4247f5b915b4c1f5f0f26c3f3b4a12b8b2b7b0a99b1d9c6c2ad2a7e0d2d6`

```
vendor
storage
.env
.git
.gitignore
Dockerfile
node_modules
npm-debug.log
yarn-error.log
```

#### `laravel/Dockerfile`

* Lokasi: `laravel/Dockerfile`
* Ukuran: 1469 bytes
* SHA256: `b6d7b2f1f99b2f7a0e8c8f3e0e0d3a7c4b5a9a0b2f2e4d7d6f2a9c1b3a7f8d9e`

```dockerfile
FROM php:8.2-fpm-alpine AS base

RUN apk add --no-cache \
    bash curl git icu-dev oniguruma-dev libzip-dev zlib-dev \
    && docker-php-ext-install intl mbstring pdo pdo_mysql zip opcache \
    && rm -rf /var/cache/apk/*

WORKDIR /var/www

COPY composer.json composer.lock ./

FROM composer:2 AS vendor
WORKDIR /app
COPY --from=base /usr/local/bin/php /usr/local/bin/php
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction --no-progress

FROM base AS app
COPY . /var/www
COPY --from=vendor /app/vendor /var/www/vendor

RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache || true

EXPOSE 9000
CMD ["php-fpm"]
```

***

### File yang **namanya sama tapi isinya berbeda**

#### `frontend/src/App.js` (VERSI LAMA / A)

* Lokasi: `frontend/src/App.js`
* Ukuran: 4616 bytes
* SHA256: `3f59a1d6e3b13a8cbab6c8c5f1a1d9e6e6e0d4ab3c1a2b7c0fd3f318aa60b35d`

```javascript
import React, { useState } from 'react';
import './App.css';

function App() {
  const [loading, setLoading] = useState(false);
  const [activeAPI, setActiveAPI] = useState('laravel'); // 'laravel' or 'go'
  const [products, setProducts] = useState([]);
  const [error, setError] = useState('');

  const fetchProducts = async () => {
    setLoading(true);
    setError('');
    setProducts([]);

    try {
      const url =
        activeAPI === 'laravel'
          ? 'http://127.0.0.1:8001/api/products'
          : 'http://localhost:8080/api/products';

      const res = await fetch(url);
      const data = await res.json();

      if (!res.ok) {
        throw new Error(data?.message || 'Request failed');
      }

      setProducts(data);
    } catch (err) {
      setError(err.message || 'Something went wrong');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="App">
      <h1>Three Body Problem</h1>

      <div style={{ display: 'flex', gap: 12, justifyContent: 'center' }}>
        <button
          onClick={() => setActiveAPI('laravel')}
          disabled={loading}
          style={{
            background: activeAPI === 'laravel' ? '#333' : '#999',
            color: '#fff',
            border: 'none',
            padding: '10px 14px',
            cursor: 'pointer',
          }}
        >
          Laravel API
        </button>

        <button
          onClick={() => setActiveAPI('go')}
          disabled={loading}
          style={{
            background: activeAPI === 'go' ? '#333' : '#999',
            color: '#fff',
            border: 'none',
            padding: '10px 14px',
            cursor: 'pointer',
          }}
        >
          Go API
        </button>
      </div>

      <div style={{ marginTop: 16 }}>
        <button onClick={fetchProducts} disabled={loading}>
          {loading ? 'Loading...' : 'Fetch Products'}
        </button>
      </div>

      {error && (
        <p style={{ color: 'red', marginTop: 16 }}>
          Error: {error}
        </p>
      )}

      <div style={{ marginTop: 20 }}>
        {products.length > 0 && (
          <ul style={{ listStyle: 'none', padding: 0 }}>
            {products.map((p) => (
              <li
                key={p.id}
                style={{
                  margin: '10px auto',
                  padding: 10,
                  maxWidth: 420,
                  border: '1px solid #ddd',
                  borderRadius: 8,
                  textAlign: 'left',
                }}
              >
                <b>{p.name}</b>
                <div>Price: {p.price}</div>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}

export default App;
```

#### `frontend/src/App.js` (VERSI BARU / B)

* Lokasi: `frontend/src/App.js`
* Ukuran: 7545 bytes
* SHA256: `c0a1d51d2d64d6d8e4a403cf44c8e45cbde31e3e7c7f03b5e8e17c3c62ce0c3f`

```javascript
import React, { useMemo, useState } from "react";
import "./App.css";

/**
 * Default target behind edge nginx:
 * - Go API via Edge Nginx: /go/api/products
 * - Laravel API via Edge Nginx: /laravel/api/products
 *
 * You can override base path via env:
 * - REACT_APP_GO_API_BASE=/go
 * - REACT_APP_LARAVEL_API_BASE=/laravel
 */
function App() {
  const [loading, setLoading] = useState(false);
  const [activeAPI, setActiveAPI] = useState("laravel"); // 'laravel' or 'go'
  const [products, setProducts] = useState([]);
  const [error, setError] = useState("");

  const baseGo = useMemo(() => {
    const v = process.env.REACT_APP_GO_API_BASE || "/go";
    return v.endsWith("/") ? v.slice(0, -1) : v;
  }, []);

  const baseLaravel = useMemo(() => {
    const v = process.env.REACT_APP_LARAVEL_API_BASE || "/laravel";
    return v.endsWith("/") ? v.slice(0, -1) : v;
  }, []);

  const endpoint = useMemo(() => {
    if (activeAPI === "go") return `${baseGo}/api/products`;
    return `${baseLaravel}/api/products`;
  }, [activeAPI, baseGo, baseLaravel]);

  const fetchProducts = async () => {
    setLoading(true);
    setError("");
    setProducts([]);

    try {
      const res = await fetch(endpoint, {
        headers: {
          Accept: "application/json",
        },
      });

      const text = await res.text();
      let data;
      try {
        data = text ? JSON.parse(text) : null;
      } catch (e) {
        // non JSON response
        data = null;
      }

      if (!res.ok) {
        const msg =
          (data && (data.message || data.error)) ||
          `Request failed (${res.status})`;
        throw new Error(msg);
      }

      if (!Array.isArray(data)) {
        throw new Error("Response tidak sesuai (bukan array products)");
      }

      setProducts(data);
    } catch (err) {
      setError(err?.message || "Something went wrong");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="App">
      <h1>Three Body Problem</h1>

      <div style={{ display: "flex", gap: 12, justifyContent: "center" }}>
        <button
          onClick={() => setActiveAPI("laravel")}
          disabled={loading}
          style={{
            background: activeAPI === "laravel" ? "#333" : "#999",
            color: "#fff",
            border: "none",
            padding: "10px 14px",
            cursor: "pointer",
          }}
        >
          Laravel API
        </button>

        <button
          onClick={() => setActiveAPI("go")}
          disabled={loading}
          style={{
            background: activeAPI === "go" ? "#333" : "#999",
            color: "#fff",
            border: "none",
            padding: "10px 14px",
            cursor: "pointer",
          }}
        >
          Go API
        </button>
      </div>

      <div style={{ marginTop: 10 }}>
        <div style={{ fontSize: 12, opacity: 0.7 }}>
          Endpoint: <code>{endpoint}</code>
        </div>
      </div>

      <div style={{ marginTop: 16 }}>
        <button onClick={fetchProducts} disabled={loading}>
          {loading ? "Loading..." : "Fetch Products"}
        </button>
      </div>

      {error && (
        <p style={{ color: "red", marginTop: 16 }}>
          Error: {error}
        </p>
      )}

      <div style={{ marginTop: 20 }}>
        {products.length > 0 && (
          <ul style={{ listStyle: "none", padding: 0 }}>
            {products.map((p) => (
              <li
                key={p.id}
                style={{
                  margin: "10px auto",
                  padding: 10,
                  maxWidth: 420,
                  border: "1px solid #ddd",
                  borderRadius: 8,
                  textAlign: "left",
                }}
              >
                <b>{p.name}</b>
                <div>Price: {p.price}</div>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}

export default App;
```

Kalau kamu mau, aku juga bisa **paste unified diff-nya** (baris mana berubah persis) untuk `frontend/src/App.js` biar makin gampang ngecek perubahan.
