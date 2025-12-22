# CI/CD Docker and Harbor V2

Sip — ini versi **lebih lengkap + runbook end-to-end** yang benar-benar ngikutin perjalanan kita dari awal (problem → percobaan → error → fix) sampai **final proven** (job `deploy_dev` sukses), lalu aku jadikan **langkah step-by-step untuk VM2 baru**.

> Catatan penting: kamu minta **file `.gitlab-ci.yml` dan `docker-compose.dev.yml` jangan diubah sedikit pun**. Jadi di runbook ini aku **tidak mengubah isi file**, hanya ngarahin **cara membuat/menyalin/mengecek** file final itu dan konfigurasi pendukungnya.

***

## 1) Timeline lengkap masalah yang kamu alami → akar masalah → fix yang akhirnya proven

### 1.1 Mulai: pipeline deploy\_dev awal “kelihatan sukses”

* Deploy jalan: `docker compose down/pull/up`, MySQL healthy, lalu `artisan optimize:clear` dan `migrate`.
* Tapi begitu pipeline diulang / variabel berubah, mulai muncul masalah.

### 1.2 Masalah #1 — `.env.dev.compose.example` dipakai nggak?

**Konteks awal:**

*   Kamu punya `.env.dev.compose.example` dan pernah dipakai untuk:

    ```bash
    docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    ```

**Kesimpulan:**

* `.env.dev.compose.example` **hanya untuk validasi lokal / sanity check** (`compose config`) supaya compose tidak error karena variabel kosong.
* Di pipeline final kamu, job `deploy_dev` **membuat `.env.dev.compose` sendiri** di `$DEPLOY_DIR`, jadi `.env.dev.compose.example` **tidak dipakai saat deploy**.

### 1.3 Masalah #2 — YAML/Herodoc/indentation bikin deploy error atau envfile kacau

**Gejalanya (yang sempat kita hadapi):**

* `.env.dev.compose` yang dibuat via heredoc bisa “kepasang spasi” / format kacau karena indentation YAML block `|-` → variabel jadi tidak kebaca.\
  **Fix yang akhirnya stabil:**
*   Ganti pembuatan envfile dari heredoc menjadi block:

    ```bash
    umask 077
    { echo "..."; echo "..."; } > "$DEPLOY_DIR/.env.dev.compose"
    ```

Ini yang kamu pakai sekarang.

### 1.4 Masalah #3 — ERROR 1045 Access denied user root (root mismatch)

**Error:**

* `ERROR 1045 (28000): Access denied for user 'root'@'localhost' / '127.0.0.1'`

**Akar penyebab utama:**

* Volume MySQL (`mysql_data`) **sudah pernah terbuat** dengan root password lama.
* Image MySQL **HANYA set root password saat init pertama** (data dir masih kosong).
* Jadi kalau kamu ganti `DEV_MYSQL_ROOT_PASSWORD` tapi volumenya masih ada → root password di server tetap yang lama → login root gagal.

**Fix yang kita bikin: self-heal + auto reset DEV**

* Di deploy job:
  * coba login root
  * kalau gagal dan `AUTO_RESET_DB_ON_ROOT_MISMATCH=1` → **wipe volume** dan start ulang
* Ini membuat deploy **tidak butuh reset manual**.

### 1.5 Masalah #4 — ERROR 2002 socket race setelah wipe volume

**Error:**

* `ERROR 2002 (HY000): Can't connect to local MySQL server through socket ...`

**Akar penyebab:**

* Setelah wipe volume, MySQL punya fase init:
  * “temporary server” (kadang port 0 / socket belum stabil),
  * lalu restart menjadi “final server”.
* Kalau check pakai socket di timing yang salah → socket belum ada → 2002.

**Fix final yang proven:**

1. Healthcheck MySQL **pakai TCP** ke `127.0.0.1:3306` agar “healthy” hanya saat mysqld final sudah listen.
2. Deploy flow diubah:
   * **naikkan mysql dulu**
   * **tunggu root bisa ping via TCP**
   * **ensure db/user/grant**
   * baru `up` service lain

Itu yang membuat pipeline kamu akhirnya stabil.

### 1.6 Masalah #5 — Laravel 500 (umumnya DB user/grant mismatch)

**Gejala:**

* Laravel log ada request `/api/products` tapi response 500.\
  **Akar penyebab:**
* DB user `threebody` / password / database belum match dengan env container.\
  **Fix:**
* Tambah step **idempotent** di deploy:
  * `CREATE DATABASE IF NOT EXISTS`
  * `CREATE USER IF NOT EXISTS`
  * `ALTER USER ... IDENTIFIED BY ...`
  * `GRANT ALL PRIVILEGES`
  * `FLUSH PRIVILEGES`
* Restart `laravel` dan `goapi`, lalu migrate.

***

## 2) “FINAL DESIGN” yang terbukti sukses (yang kamu pakai sekarang)

Prinsip suksesnya:

* Jangan mengandalkan “mysql container sudah jalan” → harus **mysqld final siap TCP:3306**
* Deploy urutan wajib:
  1. up mysql
  2. wait root ping TCP
  3. ensure DB user/grants
  4. up semua service
  5. restart app
  6. migrate
* Root mismatch (dev only) diselesaikan dengan wipe volume otomatis.

***

## 3) RUNBOOK VM2 BARU — step-by-step dari nol sampai sebelum `push`

Di bagian ini aku bikin seperti prosedur yang kamu ikuti dulu: **setup VM → siapkan repo → buat file → cek → baru push**.

> Asumsi: VM2 ini akan jadi host deploy (container “menetap” di VM2).\
> Pipeline kamu pakai tag runner `deploy`, jadi **runner di VM2 harus punya tag `deploy`**.

***

### STEP 0 — Catat parameter yang kamu pakai

* Harbor: `192.168.56.43:8081`
* Project Harbor: `threebody`
* Deploy dir: `/home/gitlab-runner/threebody-deploy`
* Compose project: `threebody-dev`
* Domain: `dev.local`

***

### STEP 1 — Install Docker & Compose (di VM2)

Cek:

```bash
docker version
docker compose version
```

Kalau belum ada: install sesuai distro VM kamu (Ubuntu/Debian biasanya via repo Docker).

***

### STEP 2 — Set Docker “insecure registry” untuk Harbor HTTP (WAJIB)

Edit:

```bash
sudo nano /etc/docker/daemon.json
```

Isi:

```json
{
  "insecure-registries": ["192.168.56.43:8081"]
}
```

Restart + cek:

```bash
sudo systemctl restart docker
docker info | sed -n '/Insecure Registries/,+10p'
```

Harus muncul `192.168.56.43:8081` di list insecure registries.

***

### STEP 3 — Install & register GitLab Runner (shell executor) dengan tag deploy

1. Pastikan runner terinstall dan running:

```bash
gitlab-runner --version
sudo systemctl status gitlab-runner --no-pager
```

2. Register:

```bash
sudo gitlab-runner register
# executor: shell
# tags: deploy
```

3. Pastikan user runner bisa akses docker:

```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner

sudo -u gitlab-runner docker ps
sudo -u gitlab-runner docker compose version
```

> Ini penting supaya job build/push/deploy di pipeline kamu bisa jalan (karena semua stage pakai tag `deploy`).

***

### STEP 4 — Siapkan folder deploy sesuai pipeline

```bash
sudo mkdir -p /home/gitlab-runner/threebody-deploy
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/threebody-deploy
ls -la /home/gitlab-runner/threebody-deploy
```

***

### STEP 5 — Pastikan Harbor siap & test login dari VM2

```bash
echo 'HARBOR_PASSWORD' | docker login 192.168.56.43:8081 -u HARBOR_USERNAME --password-stdin
```

Kalau gagal:

* pastikan daemon.json insecure registry benar
* pastikan Harbor reachable dari VM2
* pastikan user punya permission push ke project `threebody`

***

### STEP 6 — Siapkan CI/CD Variables di GitLab (ini yang dulu sering jadi sumber error)

Project → Settings → CI/CD → Variables:

**Harbor**

* `HARBOR_URL` = `192.168.56.43:8081`
* `HARBOR_PROJECT` = `threebody`
* `HARBOR_USERNAME` = `...`
* `HARBOR_PASSWORD` = `...` (Masked)

**MySQL**

* `DEV_MYSQL_ROOT_PASSWORD` = `...` (Masked)
* `DEV_MYSQL_PASSWORD` = `...` (Masked)

**Laravel**

* `DEV_APP_KEY` = `base64:...` (Masked)

**Opsional**

* `AUTO_RESET_DB_ON_ROOT_MISMATCH` = `1`

> Ini wajib karena `.gitlab-ci.yml` kamu pakai `: "${VAR:?}"` sehingga kalau ada yang kosong job langsung stop.

***

### STEP 7 — Clone repo di VM2 (atau di laptop kamu) dan siapkan file final

Kalau VM2 fresh:

```bash
git clone https://gitlab.com/cikalfarid/three-body-problem.git
cd three-body-problem
git checkout main
```

#### 7A) Pastikan struktur file yang dibutuhkan ada

Harus ada:

* `.gitlab-ci.yml`
* `docker-compose.dev.yml`
* folder `devops/` (nginx configs, dockerfiles)
* `devops/docker/Dockerfile.goapi`, `Dockerfile.laravel`, `Dockerfile.frontend`
* `devops/nginx/dev.conf` dan `frontend.conf` sesuai kebutuhan

Cek cepat:

```bash
ls -la
find devops -maxdepth 3 -type f | sort
```

#### 7B) Membuat / mengganti isi file **(tanpa mengubah isi final kamu)**

Karena kamu minta **jangan diubah sedikit pun**, maka caranya:

* pakai file dari repo kamu yang sudah proven (commit terakhir yang sukses),
* jangan “ketik ulang manual” kalau bisa.

Cek isi file:

```bash
cat .gitlab-ci.yml
cat docker-compose.dev.yml
```

Kalau kamu memang ingin recreate file via terminal sebelum push (seperti yang kamu lakukan dulu), gunakan `nano`:

```bash
nano .gitlab-ci.yml
nano docker-compose.dev.yml
```

Lalu **paste persis** versi final kamu.

Siap — ini aku **tambahkan isi file** `docker-compose.dev.yml` dan `.gitlab-ci.yml` **persis seperti yang kamu kirim** (tanpa mengubah satu karakter pun) untuk kamu taruh di runbook **sebelum push**.

***

### A) Isi final `docker-compose.dev.yml` (copy-paste persis)

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_ROOT_HOST: "%"
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      # TCP: hanya sehat kalau mysqld FINAL sudah listen di 3306 (temporary server port 0 tidak akan lolos)
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent"]
      interval: 5s
      timeout: 5s
      retries: 40
      start_period: 20s
    restart: unless-stopped

  laravel:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/laravel:${CI_COMMIT_SHA}
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      APP_ENV: local
      APP_DEBUG: "true"
      APP_KEY: ${APP_KEY}
      APP_URL: https://dev.local/api/laravel
      LOG_CHANNEL: stderr

      DB_CONNECTION: mysql
      DB_HOST: mysql
      DB_PORT: 3306
      DB_DATABASE: ${MYSQL_DATABASE}
      DB_USERNAME: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - devnet
    restart: unless-stopped

  goapi:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/goapi:${CI_COMMIT_SHA}
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: ${MYSQL_DATABASE}
    networks:
      - devnet
    restart: unless-stopped

  frontend:
    image: ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${CI_COMMIT_SHA}
    depends_on:
      - laravel
      - goapi
    networks:
      - devnet
    restart: unless-stopped

  nginx-dev:
    image: nginx:alpine
    depends_on:
      - frontend
      - laravel
      - goapi
    volumes:
      - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro
      - ./certs/dev:/etc/nginx/certs:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - devnet
    restart: unless-stopped

volumes:
  mysql_data:

networks:
  devnet:
```

***

### B) Isi final `.gitlab-ci.yml` (copy-paste persis)

```yaml
stages:
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # DEV only: kalau root mismatch, auto wipe volume mysql_data (DATA HILANG)
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

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

build_goapi:
  stage: build
  script:
    - *harbor_login
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  stage: build
  retry: 2
  script:
    - *harbor_login
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - *harbor_login
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# -------------------------
# DEPLOY DEV (main only) - self heal + auto reset volume jika root mismatch
# -------------------------
deploy_dev:
  stage: deploy
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

      # sync deploy files
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      # cert (sekali saja)
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

      # env compose
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
        # extra safety: hapus volume mysql_data berdasarkan label compose
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
            CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
            ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
            GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
            FLUSH PRIVILEGES;
          "
          MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "$SQL"
        '
      }

      # bersih + pull
      dc down --remove-orphans || true
      dc pull

      # 1) naikkan mysql dulu saja
      dc up -d --remove-orphans mysql

      # 2) tunggu mysql FINAL siap (TCP 3306 + root auth)
      if ! wait_mysql_root_tcp; then
        echo "WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password)."
        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          wipe_mysql_volume
          dc up -d --remove-orphans mysql
          wait_mysql_root_tcp || {
            echo "ERROR: Setelah auto reset, root MySQL masih gagal login."
            dc logs --no-color mysql | tail -n 200 || true
            exit 1
          }
        else
          echo "ERROR: AUTO_RESET_DB_ON_ROOT_MISMATCH=0 -> stop."
          dc logs --no-color mysql | tail -n 200 || true
          exit 1
        fi
      fi

      # 3) pastikan db/user app selalu benar (ini yang bikin Laravel tidak Access denied)
      ensure_db_user

      # 4) baru naikkan service lainnya
      dc up -d --remove-orphans

      # 5) restart service app biar pasti pick up koneksi DB
      dc restart laravel goapi || true

      # 6) migrate
      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force

      dc ps
```

***

### C) Checklist “sebelum push” (yang wajib kamu jalankan)

1. Pastikan file sudah benar:

```bash
cat docker-compose.dev.yml
cat .gitlab-ci.yml
```

2. Pastikan compose file valid (pakai env dummy):

```bash
cat > /tmp/.env.dev.compose.check <<'EOF'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_USER=threebody
MYSQL_DATABASE=threebody
APP_KEY=base64:dummy_dummy_dummy_dummy_dummy_dummy=
HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody
CI_COMMIT_SHA=local
EOF

docker compose --env-file /tmp/.env.dev.compose.check -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"
```

3. Pastikan repo bersih dan siap commit:

```bash
git status
```

> Kunci sukses dari pengalaman kita: jangan ada spasi aneh di envfile generation dan jangan mengubah urutan deploy mysql-first di deploy job (yang sekarang sudah ada di file kamu).

***

### STEP 8 — Sanity check sebelum push (ini “pencegah error” yang dulu kita lakukan berulang)

1. Cek YAML bisa terbaca:

```bash
gitlab-ci-lint (kalau pakai UI GitLab CI Lint)  # opsional via web
```

2. Cek docker compose config (butuh env dummy).\
   Karena file compose kamu pakai env var, untuk cek config lokal kamu bisa buat file dummy (opsional):

```bash
cat > /tmp/.env.dev.compose.check <<'EOF'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_USER=threebody
MYSQL_DATABASE=threebody
APP_KEY=base64:dummy_dummy_dummy_dummy_dummy_dummy=
HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody
CI_COMMIT_SHA=local
EOF

docker compose --env-file /tmp/.env.dev.compose.check -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"
```

> Ini persis fungsi `.env.dev.compose.example` dulu: **buat validasi**, bukan untuk deploy.

***

### STEP 9 — Baru lakukan commit & push (trigger pipeline)

```bash
git status
git add .gitlab-ci.yml docker-compose.dev.yml
git commit -m "CI: final stable build+deploy (mysql-first + tcp healthcheck)"
git push origin main
```

Pipeline akan otomatis:

* Build `goapi`, `laravel`, `frontend` → push ke Harbor dengan tag SHA
* Deploy:
  * sync file ke `/home/gitlab-runner/threebody-deploy`
  * generate cert kalau belum ada
  * generate `.env.dev.compose` dari CI variables
  * `dc pull`
  * `dc up mysql`
  * wait root TCP OK
  * ensure DB user/grant
  * `dc up` lainnya
  * restart laravel/goapi
  * migrate

***

## 4) Setelah pipeline sukses — checklist verifikasi di VM2 (yang kamu pakai saat troubleshooting)

Masuk ke deploy dir:

```bash
cd /home/gitlab-runner/threebody-deploy
```

Cek container:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml ps
```

Cek log mysql (kalau ada masalah root/port):

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color mysql | tail -n 200
```

Cek laravel (kalau 500):

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color laravel | tail -n 200
```

Cek user/grants (yang dulu kamu pakai):

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql \
  sh -lc 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "SELECT user,host FROM mysql.user; SHOW GRANTS FOR '\''threebody'\''@'\''%'\'';"'
```

***

## 5) Kenapa final kamu sekarang “anti drama” (inti pelajaran dari semua masalah)

* **Root mismatch 1045**: terjadi karena volume lama → kamu atasi dengan auto wipe dev.
* **Socket race 2002**: terjadi karena timing init mysql → kamu atasi dengan healthcheck TCP + mysql-first.
* **Laravel 500**: terjadi karena user/db/grant mismatch → kamu atasi dengan ensure\_db\_user idempotent + restart + migrate.
* `.env.dev.compose.example`: hanya buat validasi config; deploy pakai `.env.dev.compose` yang dibuat job.

***

Kalau kamu mau, aku bisa bikin versi **“Runbook VM2 (printable)”** dalam format yang lebih rapih (judul per langkah + checkbox ✅ yang bisa kamu centang), tapi tetap **tanpa mengubah isi file final kamu**.
