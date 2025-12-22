# CI/CD Docker, Kubernetes

Mantap—output kamu sudah cukup buat aku **final-kan pipeline** tanpa nebak-nebak lagi.

Dari hasil cek kamu:

* Struktur repo ✅: `laravel/`, `go/`, `frontend/` ada di root
* Dockerfile ✅: ada di `devops/docker/` dan memang **konteks build yang benar adalah `.`** (karena Dockerfile `COPY laravel/...`, `COPY go/...`, `COPY frontend/...`)
* Runner ✅: **executor = shell** (jadi job jalan langsung di VM2, bukan container)
* `.gitlab-ci.yml` kamu sekarang ✅: sudah ada build+push Harbor dan deploy dev via docker compose

Sekarang kita upgrade `.gitlab-ci.yml` kamu supaya:\
✅ build → push Harbor\
✅ deploy **staging (K8s)**\
✅ smoke test + migrate staging\
✅ **promote prod TANPA rebuild** (manual approval)\
✅ deploy prod (K8s) pakai TAG yang sama\
✅ smoke test + migrate prod

***

## 0) PENTING dulu: soal `tags: ["deploy"]` di pipeline kamu

Di config runner lokal memang tidak kelihatan tags (itu normal karena tags disimpan di sisi GitLab), tapi **kalau pipeline kamu pernah “pending”** biasanya penyebabnya: job minta tag `deploy` tapi runner tidak punya tag itu.

Cara pastikan di GitLab UI:

* Project → **Settings → CI/CD → Runners**
* Lihat runner `devops-ci`
* Pastikan tag `deploy` ada **atau** runner di-set “Run untagged jobs”

Kalau kamu mau aman total (biar nggak pending), opsi terbaik:

* **hapus `default: tags: ["deploy"]`** dari `.gitlab-ci.yml`\
  Tapi kalau kamu sengaja pakai tag untuk “mengunci” job agar hanya jalan di runner VM2, biarkan `deploy` dan pastikan runner punya tag itu.

***

## 1) Pastikan kubectl ada di VM2 (karena runner shell)

Karena job K8s akan jalan di VM2, `kubectl` harus ada di VM2 (dan bisa dipanggil user `gitlab-runner`).

Di VM2:

```bash
which kubectl || echo "kubectl belum ada"
kubectl version --client || true
```

Kalau belum ada, install (system-wide):

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubectl
```

Test sebagai runner user:

```bash
sudo -u gitlab-runner kubectl version --client
```

***

## 2) Tambah GitLab CI/CD Variables (wajib)

Kamu **sudah punya**:

* `HARBOR_URL`, `HARBOR_PROJECT`, `HARBOR_USERNAME`, `HARBOR_PASSWORD`
* `DEV_MYSQL_ROOT_PASSWORD`, `DEV_MYSQL_PASSWORD`, `DEV_APP_KEY` (untuk deploy dev compose)

Sekarang tambah ini:

### 2.1 Kubeconfig

Di VM1:

```bash
sudo base64 -w0 /etc/kubernetes/admin.conf
```

Copy outputnya ke GitLab variable:

* `KUBECONFIG_B64` = (base64 admin.conf)

### 2.2 Staging secrets

* `STAGING_MYSQL_ROOT_PASSWORD`
* `STAGING_MYSQL_PASSWORD`
* `STAGING_APP_KEY` (format **harus**: `base64:xxxxx`)

### 2.3 Prod secrets (jangan berubah-ubah)

* `PROD_MYSQL_ROOT_PASSWORD`
* `PROD_MYSQL_PASSWORD`
* `PROD_APP_KEY` (format `base64:xxxxx`)

Opsional (kalau mau bisa diubah):

* `STAGING_HOST` = `staging.local`
* `PROD_HOST` = `prod.local`

***

## 3) .gitlab-ci.yml FINAL (upgrade dari file kamu, tetap gaya kamu, + K8s staging/prod)

Di bawah ini versi **FULL** (kamu tinggal replace `.gitlab-ci.yml` kamu).\
Bagian build jobs + deploy\_dev **aku pertahankan** (cuma aku tambah stage list & job K8s).

> NOTE: aku pakai namespace tetap:\
> staging = `threebody-staging`\
> prod = `threebody-prod`

```yaml
stages:
  - build
  - deploy_dev
  - deploy_staging
  - smoke_staging
  - promote
  - deploy_prod
  - smoke_prod

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # DEV only: kalau root mismatch, auto wipe volume mysql_data (DATA HILANG)
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

  # K8s namespaces
  NS_STAGING: "threebody-staging"
  NS_PROD: "threebody-prod"

  # default hosts
  STAGING_HOST: "staging.local"
  PROD_HOST: "prod.local"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version

# -------------------------
# BUILD + PUSH (Harbor)
# -------------------------
.harbor_login: &harbor_login |
  set -euo pipefail
  : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
  REG="$HARBOR_URL/$HARBOR_PROJECT"
  export TAG REG

.kube_setup: &kube_setup |
  set -euo pipefail
  : "${KUBECONFIG_B64:?}"
  umask 077
  echo "$KUBECONFIG_B64" | base64 -d > kubeconfig
  export KUBECONFIG="$CI_PROJECT_DIR/kubeconfig"
  kubectl version --client

  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
  REG="$HARBOR_URL/$HARBOR_PROJECT"
  export TAG REG

  # tunggu ingress punya external ip (MetalLB)
  for i in $(seq 1 60); do
    INGRESS_IP="$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)"
    if [ -n "${INGRESS_IP:-}" ]; then break; fi
    sleep 2
  done
  : "${INGRESS_IP:?Ingress EXTERNAL-IP masih kosong. Pastikan ingress-nginx + MetalLB sudah OK.}"
  export INGRESS_IP

# -------------------------
# BUILD images
# -------------------------
build_goapi:
  stage: build
  script:
    - *harbor_login
    - docker build -t "$REG/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
    - docker push "$REG/goapi:$TAG"

build_laravel:
  stage: build
  retry: 2
  script:
    - *harbor_login
    - docker build -t "$REG/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
    - docker push "$REG/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - *harbor_login
    - docker build -t "$REG/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
    - docker push "$REG/frontend:$TAG"

# -------------------------
# DEPLOY DEV (Docker Compose) - main only
# -------------------------
deploy_dev:
  stage: deploy_dev
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail

      : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      mkdir -p "$DEPLOY_DIR"

      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -s "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -s "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          set -e
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      umask 077
      {
        echo "MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}"
        echo "MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}"
        echo "MYSQL_USER=threebody"
        echo "MYSQL_DATABASE=threebody"
        echo "APP_KEY=${DEV_APP_KEY}"
        echo "HARBOR_URL=${HARBOR_URL}"
        echo "HARBOR_PROJECT=${HARBOR_PROJECT}"
        echo "CI_COMMIT_SHA=${TAG}"
      } > "$DEPLOY_DIR/.env.dev.compose"

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      dc() { docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; }
      dc config >/dev/null

      wait_mysql_root_tcp() {
        dc exec -T mysql sh -lc '
          for i in $(seq 1 90); do
            if MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent >/dev/null 2>&1; then
              exit 0
            fi
            sleep 2
          done
          exit 1
        '
      }

      wipe_mysql_volume() {
        echo "AUTO RESET: wipe mysql_data volume (DEV ONLY, DATA HILANG)"
        dc down -v --remove-orphans || true
        for v in $(docker volume ls -q \
          --filter label=com.docker.compose.project="$COMPOSE_PROJECT_NAME" \
          --filter label=com.docker.compose.volume=mysql_data); do
          docker volume rm -f "$v" || true
        done
      }

      ensure_db_user() {
        dc exec -T mysql sh -lc '
          set -eu
          SQL="
            CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
            CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'' ;
            ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'' ;
            GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
            FLUSH PRIVILEGES;
          "
          MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "$SQL"
        '
      }

      dc down --remove-orphans || true
      dc pull
      dc up -d --remove-orphans mysql

      if ! wait_mysql_root_tcp; then
        echo "WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password)."
        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          wipe_mysql_volume
          dc up -d --remove-orphans mysql
          wait_mysql_root_tcp || { dc logs --no-color mysql | tail -n 200 || true; exit 1; }
        else
          dc logs --no-color mysql | tail -n 200 || true
          exit 1
        fi
      fi

      ensure_db_user
      dc up -d --remove-orphans
      dc restart laravel goapi || true
      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force
      dc ps

# -------------------------
# DEPLOY STAGING (K8s)
# -------------------------
deploy_staging:
  stage: deploy_staging
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *harbor_login
    - *kube_setup
    - |
      : "${STAGING_MYSQL_ROOT_PASSWORD:?}" "${STAGING_MYSQL_PASSWORD:?}" "${STAGING_APP_KEY:?}"

      kubectl get ns "$NS_STAGING" >/dev/null 2>&1 || kubectl create ns "$NS_STAGING"

      # image pull secret
      kubectl -n "$NS_STAGING" create secret docker-registry harbor-creds \
        --docker-server="$HARBOR_URL" \
        --docker-username="$HARBOR_USERNAME" \
        --docker-password="$HARBOR_PASSWORD" \
        --dry-run=client -o yaml | kubectl apply -f -
      kubectl -n "$NS_STAGING" patch serviceaccount default -p '{"imagePullSecrets":[{"name":"harbor-creds"}]}' || true

      # mysql secret + app key secret (idempotent)
      kubectl -n "$NS_STAGING" create secret generic mysql-secret \
        --from-literal=MYSQL_ROOT_PASSWORD="$STAGING_MYSQL_ROOT_PASSWORD" \
        --from-literal=MYSQL_DATABASE="threebody" \
        --from-literal=MYSQL_USER="threebody" \
        --from-literal=MYSQL_PASSWORD="$STAGING_MYSQL_PASSWORD" \
        --dry-run=client -o yaml | kubectl apply -f -

      kubectl -n "$NS_STAGING" create secret generic app-secret \
        --from-literal=APP_KEY="$STAGING_APP_KEY" \
        --dry-run=client -o yaml | kubectl apply -f -

      # MySQL (StatefulSet)
      cat <<'EOF' | kubectl -n "$NS_STAGING" apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels: {app: mysql}
  template:
    metadata:
      labels: {app: mysql}
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        envFrom:
        - secretRef: {name: mysql-secret}
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        readinessProbe:
          exec:
            command: ["sh","-lc","mysqladmin ping -h 127.0.0.1 -uroot -p$MYSQL_ROOT_PASSWORD --silent"]
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
EOF

      kubectl -n "$NS_STAGING" rollout status statefulset/mysql --timeout=300s

      # App Deployments + Services
      cat <<EOF | kubectl -n "$NS_STAGING" apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: laravel}
spec:
  replicas: 1
  selector: {matchLabels: {app: laravel}}
  template:
    metadata: {labels: {app: laravel}}
    spec:
      containers:
      - name: laravel
        image: ${REG}/laravel:${TAG}
        ports: [{containerPort: 80}]
        env:
        - {name: APP_ENV, value: "staging"}
        - {name: APP_DEBUG, value: "false"}
        - {name: LOG_CHANNEL, value: "stderr"}
        - name: APP_KEY
          valueFrom: {secretKeyRef: {name: app-secret, key: APP_KEY}}
        - {name: DB_CONNECTION, value: "mysql"}
        - {name: DB_HOST, value: "mysql"}
        - {name: DB_PORT, value: "3306"}
        - name: DB_DATABASE
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USERNAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata: {name: laravel}
spec:
  selector: {app: laravel}
  ports: [{port: 80, targetPort: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: goapi}
spec:
  replicas: 1
  selector: {matchLabels: {app: goapi}}
  template:
    metadata: {labels: {app: goapi}}
    spec:
      containers:
      - name: goapi
        image: ${REG}/goapi:${TAG}
        ports: [{containerPort: 8080}]
        env:
        - {name: DB_HOST, value: "mysql"}
        - {name: DB_PORT, value: "3306"}
        - name: DB_NAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USER
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata: {name: goapi}
spec:
  selector: {app: goapi}
  ports: [{port: 8080, targetPort: 8080}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: frontend}
spec:
  replicas: 1
  selector: {matchLabels: {app: frontend}}
  template:
    metadata: {labels: {app: frontend}}
    spec:
      containers:
      - name: frontend
        image: ${REG}/frontend:${TAG}
        ports: [{containerPort: 80}]
---
apiVersion: v1
kind: Service
metadata: {name: frontend}
spec:
  selector: {app: frontend}
  ports: [{port: 80, targetPort: 80}]
EOF

      kubectl -n "$NS_STAGING" rollout status deploy/laravel --timeout=300s
      kubectl -n "$NS_STAGING" rollout status deploy/goapi --timeout=300s
      kubectl -n "$NS_STAGING" rollout status deploy/frontend --timeout=300s

      # Ingress
      cat <<EOF | kubectl -n "$NS_STAGING" apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ing
spec:
  ingressClassName: nginx
  rules:
  - host: ${STAGING_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: frontend, port: {number: 80}}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ing
spec:
  ingressClassName: nginx
  rules:
  - host: ${STAGING_HOST}
    http:
      paths:
      - path: /api/laravel
        pathType: Prefix
        backend:
          service: {name: laravel, port: {number: 80}}
      - path: /api/go
        pathType: Prefix
        backend:
          service: {name: goapi, port: {number: 8080}}
EOF

smoke_staging:
  stage: smoke_staging
  needs: ["deploy_staging"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *harbor_login
    - *kube_setup
    - |
      kubectl -n "$NS_STAGING" exec deploy/laravel -- sh -lc 'php artisan optimize:clear && php artisan migrate --force'

      curl -sS -H "Host: ${STAGING_HOST}" "http://${INGRESS_IP}/" -I | head -n 1
      curl -sS -H "Host: ${STAGING_HOST}" "http://${INGRESS_IP}/api/go/api/products" | head -c 200; echo
      curl -sS -H "Host: ${STAGING_HOST}" "http://${INGRESS_IP}/api/laravel/api/products" | head -c 200; echo

# -------------------------
# PROMOTE (manual) - TANPA rebuild
# -------------------------
promote_to_prod:
  stage: promote
  needs: ["smoke_staging"]
  when: manual
  allow_failure: false
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      echo "PROMOTE_TAG=$TAG" > promote.env
  artifacts:
    reports:
      dotenv: promote.env

# -------------------------
# DEPLOY PROD (K8s) - pakai PROMOTE_TAG
# -------------------------
deploy_prod:
  stage: deploy_prod
  needs: ["promote_to_prod"]
  resource_group: prod
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *harbor_login
    - *kube_setup
    - |
      : "${PROD_MYSQL_ROOT_PASSWORD:?}" "${PROD_MYSQL_PASSWORD:?}" "${PROD_APP_KEY:?}"
      : "${PROMOTE_TAG:?}"
      export TAG="$PROMOTE_TAG"

      kubectl get ns "$NS_PROD" >/dev/null 2>&1 || kubectl create ns "$NS_PROD"

      kubectl -n "$NS_PROD" create secret docker-registry harbor-creds \
        --docker-server="$HARBOR_URL" \
        --docker-username="$HARBOR_USERNAME" \
        --docker-password="$HARBOR_PASSWORD" \
        --dry-run=client -o yaml | kubectl apply -f -
      kubectl -n "$NS_PROD" patch serviceaccount default -p '{"imagePullSecrets":[{"name":"harbor-creds"}]}' || true

      kubectl -n "$NS_PROD" create secret generic mysql-secret \
        --from-literal=MYSQL_ROOT_PASSWORD="$PROD_MYSQL_ROOT_PASSWORD" \
        --from-literal=MYSQL_DATABASE="threebody" \
        --from-literal=MYSQL_USER="threebody" \
        --from-literal=MYSQL_PASSWORD="$PROD_MYSQL_PASSWORD" \
        --dry-run=client -o yaml | kubectl apply -f -

      kubectl -n "$NS_PROD" create secret generic app-secret \
        --from-literal=APP_KEY="$PROD_APP_KEY" \
        --dry-run=client -o yaml | kubectl apply -f -

      cat <<'EOF' | kubectl -n "$NS_PROD" apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels: {app: mysql}
  template:
    metadata:
      labels: {app: mysql}
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        envFrom:
        - secretRef: {name: mysql-secret}
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        readinessProbe:
          exec:
            command: ["sh","-lc","mysqladmin ping -h 127.0.0.1 -uroot -p$MYSQL_ROOT_PASSWORD --silent"]
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
EOF

      kubectl -n "$NS_PROD" rollout status statefulset/mysql --timeout=300s

      cat <<EOF | kubectl -n "$NS_PROD" apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: laravel}
spec:
  replicas: 2
  selector: {matchLabels: {app: laravel}}
  template:
    metadata: {labels: {app: laravel}}
    spec:
      containers:
      - name: laravel
        image: ${REG}/laravel:${TAG}
        ports: [{containerPort: 80}]
        env:
        - {name: APP_ENV, value: "production"}
        - {name: APP_DEBUG, value: "false"}
        - {name: LOG_CHANNEL, value: "stderr"}
        - name: APP_KEY
          valueFrom: {secretKeyRef: {name: app-secret, key: APP_KEY}}
        - {name: DB_CONNECTION, value: "mysql"}
        - {name: DB_HOST, value: "mysql"}
        - {name: DB_PORT, value: "3306"}
        - name: DB_DATABASE
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USERNAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata: {name: laravel}
spec:
  selector: {app: laravel}
  ports: [{port: 80, targetPort: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: goapi}
spec:
  replicas: 2
  selector: {matchLabels: {app: goapi}}
  template:
    metadata: {labels: {app: goapi}}
    spec:
      containers:
      - name: goapi
        image: ${REG}/goapi:${TAG}
        ports: [{containerPort: 8080}]
        env:
        - {name: DB_HOST, value: "mysql"}
        - {name: DB_PORT, value: "3306"}
        - name: DB_NAME
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_DATABASE}}
        - name: DB_USER
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_USER}}
        - name: DB_PASSWORD
          valueFrom: {secretKeyRef: {name: mysql-secret, key: MYSQL_PASSWORD}}
---
apiVersion: v1
kind: Service
metadata: {name: goapi}
spec:
  selector: {app: goapi}
  ports: [{port: 8080, targetPort: 8080}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: frontend}
spec:
  replicas: 2
  selector: {matchLabels: {app: frontend}}
  template:
    metadata: {labels: {app: frontend}}
    spec:
      containers:
      - name: frontend
        image: ${REG}/frontend:${TAG}
        ports: [{containerPort: 80}]
---
apiVersion: v1
kind: Service
metadata: {name: frontend}
spec:
  selector: {app: frontend}
  ports: [{port: 80, targetPort: 80}]
EOF

      kubectl -n "$NS_PROD" rollout status deploy/laravel --timeout=300s
      kubectl -n "$NS_PROD" rollout status deploy/goapi --timeout=300s
      kubectl -n "$NS_PROD" rollout status deploy/frontend --timeout=300s

      cat <<EOF | kubectl -n "$NS_PROD" apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ing
spec:
  ingressClassName: nginx
  rules:
  - host: ${PROD_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: frontend, port: {number: 80}}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ing
spec:
  ingressClassName: nginx
  rules:
  - host: ${PROD_HOST}
    http:
      paths:
      - path: /api/laravel
        pathType: Prefix
        backend:
          service: {name: laravel, port: {number: 80}}
      - path: /api/go
        pathType: Prefix
        backend:
          service: {name: goapi, port: {number: 8080}}
EOF

smoke_prod:
  stage: smoke_prod
  needs: ["deploy_prod"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *harbor_login
    - *kube_setup
    - |
      kubectl -n "$NS_PROD" exec deploy/laravel -- sh -lc 'php artisan optimize:clear && php artisan migrate --force'

      curl -sS -H "Host: ${PROD_HOST}" "http://${INGRESS_IP}/" -I | head -n 1
      curl -sS -H "Host: ${PROD_HOST}" "http://${INGRESS_IP}/api/go/api/products" | head -c 200; echo
      curl -sS -H "Host: ${PROD_HOST}" "http://${INGRESS_IP}/api/laravel/api/products" | head -c 200; echo
```

***

## 4) Cara jalaninnya (alur yang akan kamu lihat)

1. Push ke `main`
2. Pipeline otomatis:
   * `build_*` push ke Harbor tag `${CI_COMMIT_SHA}`
   * `deploy_dev` (docker compose)
   * `deploy_staging` (K8s)
   * `smoke_staging` (migrate + curl)
3. Lalu muncul tombol manual: **promote\_to\_prod**
4. Klik promote → lanjut:
   * `deploy_prod` (pakai TAG yang sama → tanpa rebuild)
   * `smoke_prod`

***

## 5) Kenapa ini “anti-kendala” sesuai masalah kamu kemarin

* `kubectl not found` ✅ → sekarang dipastikan di VM2 (runner shell)
* “Ingress external ip belum siap” ✅ → ada loop `wait` sampai IP ada
* “tag/runner mismatch pending” ✅ → aku kasih warning & opsi fix
* “migrate pakai -it” ✅ → di CI pakai `kubectl exec` tanpa TTY

***

Kalau kamu mau, berikutnya aku bisa rapihin lebih “enterprise-style”:

* pakai **robot account Harbor** (lebih aman daripada admin),
* pakai **ServiceAccount + kubeconfig minimal** (bukan admin.conf),
* dan opsional: promote prod dengan **retag tanpa rebuild** (mis. tag `prod` / `release-xxxx`) tapi tetap digest sama.
