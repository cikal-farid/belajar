# CI/CD Docker and Harbor V1

Oke, kita **lanjut bikin `.gitlab-ci.yml` final** untuk build + push 3 image (**goapi, laravel, frontend**) ke **Harbor** `192.168.56.43:8081/threebody` pakai **Docker-in-Docker** dan **insecure registry (HTTP)**.

> Catatan penting biar nggak error:
>
> 1. **GitLab Runner Docker executor harus `privileged = true`** (wajib untuk dind).
> 2. Project Harbor **`threebody` harus sudah dibuat** dan user CI punya akses push.

***

### 1) Isi `.gitlab-ci.yml` (FINAL)

Buat/replace file di root repo:

```yaml
stages:
  - build

workflow:
  rules:
    - if: $CI_COMMIT_TAG
    - if: $CI_COMMIT_BRANCH

default:
  image: docker:27.3.1
  services:
    - name: docker:27.3.1-dind
      # Harbor kamu HTTP (insecure registry)
      command: ["--insecure-registry=192.168.56.43:8081"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    DOCKER_DRIVER: overlay2
    DOCKER_BUILDKIT: "1"

    # Default (boleh override di CI/CD Variables)
    HARBOR_URL: "192.168.56.43:8081"
    HARBOR_PROJECT: "threebody"

  before_script:
    - docker info
    - |
      if [ -z "${HARBOR_USERNAME:-}" ] || [ -z "${HARBOR_PASSWORD:-}" ]; then
        echo "ERROR: set CI/CD Variables: HARBOR_USERNAME dan HARBOR_PASSWORD"
        exit 1
      fi
    - echo "$HARBOR_PASSWORD" | docker login -u "$HARBOR_USERNAME" --password-stdin "$HARBOR_URL"

.build_and_push_template: &build_and_push_template
  stage: build
  script:
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - IMAGE="${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${TAG}"
    - echo "Building $IMAGE from $DOCKERFILE"
    - docker build --pull -t "$IMAGE" -f "$DOCKERFILE" .
    - docker push "$IMAGE"
    # Push tag "latest" hanya untuk branch main
    - |
      if [ "${CI_COMMIT_BRANCH:-}" = "main" ]; then
        LATEST="${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest"
        docker tag "$IMAGE" "$LATEST"
        docker push "$LATEST"
      fi
  rules:
    - if: $CI_COMMIT_TAG
    - if: $CI_COMMIT_BRANCH

build:goapi:
  <<: *build_and_push_template
  variables:
    IMAGE_NAME: "goapi"
    DOCKERFILE: "devops/docker/Dockerfile.goapi"

build:laravel:
  <<: *build_and_push_template
  variables:
    IMAGE_NAME: "laravel"
    DOCKERFILE: "devops/docker/Dockerfile.laravel"

build:frontend:
  <<: *build_and_push_template
  variables:
    IMAGE_NAME: "frontend"
    DOCKERFILE: "devops/docker/Dockerfile.frontend"
```

***

### 2) Set CI/CD Variables di GitLab (wajib)

Di GitLab project → **Settings → CI/CD → Variables**:

* `HARBOR_USERNAME` = (mis. `admin` atau user harbor kamu)
* `HARBOR_PASSWORD` = passwordnya (Masked + Protected sesuai kebutuhan)
* (opsional) `HARBOR_URL` = `192.168.56.43:8081`
* (opsional) `HARBOR_PROJECT` = `threebody`

***

### 3) Cek runner biar fix anti error (wajib untuk dind)

Di mesin runner (GitLab Runner config), pastikan docker executor **privileged**:

`/etc/gitlab-runner/config.toml`:

```toml
[runners.docker]
  privileged = true
```

Lalu restart runner:

```bash
sudo systemctl restart gitlab-runner
```

***

### 4) Commit perubahan repo (biar rapi)

Dari output `git status` kamu, ini yang biasanya di-commit:

```bash
cd ~/three-body-problem-main
git add .gitlab-ci.yml docker-compose.dev.yml devops/ frontend/src/App.js .gitignore .env.dev.compose.example
git status
git commit -m "Add final CI pipeline and docker/dev configs"
git push
```

***

Kalau sudah di-push, kirimkan 2 output ini (biar aku cek pipeline-nya bener-bener aman):

1. Screenshot/log pipeline job yang jalan
2. Output error kalau ada (kalau hijau semua juga oke)

Setelah itu kita bisa lanjut tahap berikutnya: **deploy otomatis (optional) dari CI ke VM2 pakai docker compose pull+up**.

cikal@devops-ci:\~$ cd \~/three-body-problem-main git add .gitlab-ci.yml docker-compose.dev.yml devops/ frontend/src/App.js .gitignore .env.dev.compose.example git status git commit -m "Add final CI pipeline and docker/dev configs" git push fatal: pathspec '.gitlab-ci.yml' did not match any files On branch main Your branch is up to date with 'origin/main'. Changes not staged for commit: (use "git add \<file>..." to update what will be committed) (use "git restore \<file>..." to discard changes in working directory) modified: frontend/src/App.js Untracked files: (use "git add \<file>..." to include in what will be committed) .env.dev.compose certs/ devops/ docker-compose.dev.yml no changes added to commit (use "git add" and/or "git commit -a") Author identity unknown \*\*\* Please tell me who you are. Run git config --global user.email "you@example.com" git config --global user.name "Your Name" to set your account's default identity. Omit --global to set the identity only in this repository. fatal: unable to auto-detect email address (got 'cikal@devops-ci.(none)') Username for 'https://github.com': cikalfarid Password for 'https://cikalfarid@github.com': remote: Invalid username or token. Password authentication is not supported for Git operations. fatal: Authentication failed for 'https://github.com/Rafianda/three-body-problem.git/' cikal@devops-ci:\~/three-body-problem-main$ git push Username for 'https://github.com': cikalfarid Password for 'https://cikalfarid@github.com': remote: Invalid username or token. Password authentication is not supported for Git operations. fatal: Authentication failed for 'https://github.com/Rafianda/three-body-problem.git/' cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 1m 2s

Sip, error kamu sekarang **bukan di Docker/Harbor lagi**, tapi murni di **Git**. Ada 3 masalah utama:

1. **`.gitlab-ci.yml` belum ada** → makanya `pathspec ... did not match any files`
2. **Git identity belum diset** → `Please tell me who you are`
3. **Push ke GitHub pakai password sudah tidak bisa** → harus pakai **PAT (token)** atau **SSH key**

Di bawah ini langkah yang “aman dan beres” sampai bisa push.

***

### 1) Buat file `.gitlab-ci.yml` dulu (karena memang belum ada)

Di root repo:

```bash
cd ~/three-body-problem-main

cat > .gitlab-ci.yml <<'YAML'
stages:
  - build

workflow:
  rules:
    - if: $CI_COMMIT_TAG
    - if: $CI_COMMIT_BRANCH

default:
  image: docker:27.3.1
  services:
    - name: docker:27.3.1-dind
      command: ["--insecure-registry=192.168.56.43:8081"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    DOCKER_DRIVER: overlay2
    DOCKER_BUILDKIT: "1"
    HARBOR_URL: "192.168.56.43:8081"
    HARBOR_PROJECT: "threebody"

  before_script:
    - docker info
    - |
      if [ -z "${HARBOR_USERNAME:-}" ] || [ -z "${HARBOR_PASSWORD:-}" ]; then
        echo "ERROR: set CI/CD Variables: HARBOR_USERNAME dan HARBOR_PASSWORD"
        exit 1
      fi
    - echo "$HARBOR_PASSWORD" | docker login -u "$HARBOR_USERNAME" --password-stdin "$HARBOR_URL"

.build_and_push_template: &build_and_push_template
  stage: build
  script:
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - IMAGE="${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${TAG}"
    - docker build --pull -t "$IMAGE" -f "$DOCKERFILE" .
    - docker push "$IMAGE"
    - |
      if [ "${CI_COMMIT_BRANCH:-}" = "main" ]; then
        LATEST="${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest"
        docker tag "$IMAGE" "$LATEST"
        docker push "$LATEST"
      fi
  rules:
    - if: $CI_COMMIT_TAG
    - if: $CI_COMMIT_BRANCH

build:goapi:
  <<: *build_and_push_template
  variables:
    IMAGE_NAME: "goapi"
    DOCKERFILE: "devops/docker/Dockerfile.goapi"

build:laravel:
  <<: *build_and_push_template
  variables:
    IMAGE_NAME: "laravel"
    DOCKERFILE: "devops/docker/Dockerfile.laravel"

build:frontend:
  <<: *build_and_push_template
  variables:
    IMAGE_NAME: "frontend"
    DOCKERFILE: "devops/docker/Dockerfile.frontend"
YAML
```

***

### 2) Pastikan `.gitignore` bener (biar `.env.dev.compose` & certs gak ikut)

```bash
cd ~/three-body-problem-main

# pastikan ignore .env.dev.compose dan certs/dev
grep -qE '^\.env\.dev\.compose$' .gitignore || echo ".env.dev.compose" >> .gitignore
grep -qE '^certs/dev/$' .gitignore || printf "\n# dev certs\ncerts/dev/\n" >> .gitignore
```

> Kalau mau lebih aman, ignore semua certs:\
> `echo "certs/" >> .gitignore`

***

### 3) Set identity Git (wajib supaya bisa commit)

```bash
git config --global user.name "Cikal"
git config --global user.email "cikal@example.com"
```

(Ganti email sesuai email kamu.)

***

### 4) Stage & commit (pastikan certs dan .env.dev.compose tidak ikut)

```bash
cd ~/three-body-problem-main

git add .gitlab-ci.yml docker-compose.dev.yml devops/ frontend/src/App.js .gitignore .env.dev.compose.example
git status
git commit -m "Add final CI pipeline and docker/dev configs"
```

***

### 5) Push ke GitHub: wajib pakai PAT (Token), bukan password

#### Opsi A (paling gampang): pakai PAT saat diminta password

1. Buat **GitHub Personal Access Token** (scope minimal: `repo`)
2.  Jalankan:

    ```bash
    git push
    ```
3. Saat prompt:
   * **Username**: username GitHub kamu
   * **Password**: **paste TOKEN**, bukan password akun

#### Opsi B (lebih enak): set remote URL pakai token (sekali aja)

```bash
git remote -v
git remote set-url origin https://cikalfarid:<PASTE_TOKEN_DI_SINI>@github.com/Rafianda/three-body-problem.git
git push
```

> Kalau repo `Rafianda/three-body-problem` bukan punya kamu, pastikan akun kamu **punya akses write** (collaborator). Kalau tidak, tetap akan gagal walau token benar.

#### Opsi C (recommended jangka panjang): pakai SSH

```bash
ssh-keygen -t ed25519 -C "cikal@devops-ci"
cat ~/.ssh/id_ed25519.pub
```

Copy output pubkey → pasang di GitHub (SSH Keys), lalu:

```bash
git remote set-url origin git@github.com:Rafianda/three-body-problem.git
git push
```

***

Kalau kamu jalankan langkah di atas, harusnya **commit beres** dan **push sukses**.

Kirim output ini ya setelah kamu coba:

```bash
git status
git remote -v
git push
```

Habis itu baru kita lanjut ke tahap berikutnya (biar VM2 restart tetap “semua auto up”: docker service, harbor compose, dan stack threebody).

ini mau push ke gitlab cikal@devops-ci:\~/three-body-problem-main$ git remote add origin https://gitlab.com/cikalfarid/three-body-problem.git error: remote origin already exists. cikal@devops-ci:\~/three-body-problem-main$ git branch -M main cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 14s

Oke, itu wajar: **`origin` sudah ada** (kemungkinan masih pointing ke GitHub). Kamu punya 2 opsi yang rapi:

### Opsi 1 (paling simpel): ganti URL `origin` jadi GitLab

```bash
cd ~/three-body-problem-main

git remote -v
git remote set-url origin https://gitlab.com/cikalfarid/three-body-problem.git
git remote -v

git push -u origin main
```

### Opsi 2 (lebih rapi kalau mau tetap simpan remote GitHub juga): rename origin lama → `github`, lalu `origin` untuk GitLab

```bash
cd ~/three-body-problem-main

git remote -v
git remote rename origin github
git remote add origin https://gitlab.com/cikalfarid/three-body-problem.git
git remote -v

git push -u origin main
```

### Catatan penting soal login GitLab

Kalau kamu push via **HTTPS**, GitLab biasanya **tidak pakai password akun**. Saat diminta:

* **Username**: username GitLab kamu (mis. `cikalfarid`)
* **Password**: **Personal Access Token (PAT)** dari GitLab (scope minimal: `write_repository`)

***

Kalau setelah ini masih gagal, kirim output:

```bash
git remote -v
git status --porcelain
git push -u origin main
```

Biar kelihatan apakah masalahnya di remote URL, belum ada commit, atau autentikasi.

cikal@devops-ci:\~/three-body-problem-main$ git remote -v git status --porcelain git push -u origin main github https://github.com/Rafianda/three-body-problem.git (fetch) github https://github.com/Rafianda/three-body-problem.git (push) origin https://gitlab.com/cikalfarid/three-body-problem.git (fetch) origin https://gitlab.com/cikalfarid/three-body-problem.git (push) M frontend/src/App.js ?? .gitignore ?? .gitlab-ci.yml ?? devops/ ?? docker-compose.dev.yml Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': To https://gitlab.com/cikalfarid/three-body-problem.git ! \[rejected] main -> main (fetch first) error: failed to push some refs to 'https://gitlab.com/cikalfarid/three-body-problem.git' hint: Updates were rejected because the remote contains work that you do not hint: have locally. This is usually caused by another repository pushing to hint: the same ref. If you want to integrate the remote changes, use hint: 'git pull' before pushing again. hint: See the 'Note about fast-forwards' in 'git push --help' for details. cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 28s

Sip, statusnya jelas: **GitLab repo kamu sudah ada commit di remote** (biasanya karena waktu bikin project di GitLab kamu centang “Initialize with README” atau ada default file), jadi push kamu ketolak “fetch first”.

Karena kamu juga **punya perubahan lokal yang belum di-commit**, jalan paling aman adalah: **commit dulu → pull/rebase dari GitLab → push**.

Jalankan ini persis:

#### 1) Set identitas git (wajib, biar bisa commit)

```bash
cd ~/three-body-problem-main

git config --global user.name "Cikal Farid"
git config --global user.email "cikalfarid@example.com"   # ganti email kamu
```

#### 2) Add & commit (jangan ikutkan `.env.dev.compose` dan `certs/`)

```bash
git add .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml frontend/src/App.js .env.dev.compose.example
git commit -m "Add dev docker/nginx configs and CI pipeline"
```

> Pastikan `.env.dev.compose` **tidak** ikut `git add` (biarkan lokal saja).

#### 3) Ambil commit dari GitLab, lalu rebase ke atasnya

Karena kemungkinan historinya beda (unrelated), pakai flag ini:

```bash
git pull --rebase origin main --allow-unrelated-histories
```

Kalau muncul konflik:

```bash
git status
# edit file yang konflik, beresin
git add <file-yang-konflik>
git rebase --continue
```

Kalau kamu pengen batalin rebase:

```bash
git rebase --abort
```

#### 4) Push ke GitLab

```bash
git push -u origin main
```

#### Catatan login GitLab (HTTPS)

Waktu prompt password, **pakai Personal Access Token (PAT)** GitLab (scope minimal `write_repository`), bukan password akun.

***

Kalau setelah `git pull --rebase ...` ada error yang spesifik (mis. “refusing to merge unrelated histories” atau konflik aneh), paste output dari:

```bash
git pull --rebase origin main --allow-unrelated-histories
git status
```

biar aku tuntun sampai bersih dan bisa push.

cikal@devops-ci:\~/three-body-problem-main$ cd \~/three-body-problem-main git config --global user.name "Cikal Farid" git config --global user.email "cikalfarid@gmail.com" cikal@devops-ci:\~/three-body-problem-main$ git add .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml frontend/src/App.js .env.dev.compose.example git commit -m "Add dev docker/nginx configs and CI pipeline" fatal: pathspec '.env.dev.compose.example' did not match any files On branch main Your branch is up to date with 'github/main'. Changes not staged for commit: (use "git add \<file>..." to update what will be committed) (use "git restore \<file>..." to discard changes in working directory) modified: frontend/src/App.js Untracked files: (use "git add \<file>..." to include in what will be committed) .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml no changes added to commit (use "git add" and/or "git commit -a") cikal@devops-ci:\~/three-body-problem-main$ git pull --rebase origin main --allow-unrelated-histories error: cannot pull with rebase: You have unstaged changes. error: Please commit or stash them. cikal@devops-ci:\~/three-body-problem-main$ git status # edit file yang konflik, beresin git add \<file-yang-konflik> git rebase --continue On branch main Your branch is up to date with 'github/main'. Changes not staged for commit: (use "git add \<file>..." to update what will be committed) (use "git restore \<file>..." to discard changes in working directory) modified: frontend/src/App.js Untracked files: (use "git add \<file>..." to include in what will be committed) .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml no changes added to commit (use "git add" and/or "git commit -a") -bash: syntax error near unexpected token `newline'` fatal: No rebase in progress? cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 36s

Masalahnya sekarang simpel:

1. `git add` kamu **gagal total** karena ada 1 file yang **nggak ada**: `.env.dev.compose.example`\
   → kalau 1 pathspec error, **semua file lain jadi tidak ke-add**.
2. Karena belum ada commit, `git pull --rebase` ditolak: **“You have unstaged changes.”**

Kita beresin dengan langkah ini.

***

### 1) Pastikan `.env.dev.compose.example` benar-benar ada

Cek dulu:

```bash
cd ~/three-body-problem-main
ls -la .env.dev.compose.example || true
```

Kalau file itu memang belum ada, buat:

```bash
cp .env.dev.compose .env.dev.compose.example
sed -i 's/RootPassDev123!/CHANGE_ME/g; s/UserPassDev123!/CHANGE_ME/g' .env.dev.compose.example
sed -i '/^APP_KEY=/d' .env.dev.compose.example
```

***

### 2) Add file yang benar (jangan ikutkan `.env.dev.compose`)

Sekarang add ulang (tanpa error):

```bash
git add .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml frontend/src/App.js .env.dev.compose.example
git status
```

Harusnya sekarang sudah “staged” (bukan cuma modified/untracked).

Commit:

```bash
git commit -m "Add dev docker/nginx configs and CI pipeline"
```

***

### 3) Pull dari GitLab dulu (karena remote sudah ada isi)

```bash
git pull --rebase origin main --allow-unrelated-histories
```

Kalau ada konflik:

```bash
git status
# edit file yang konflik sampai beres
git add <file_yang_konflik>
git rebase --continue
```

***

### 4) Push ke GitLab

```bash
git push -u origin main
```

Saat diminta password di GitLab: **pakai Personal Access Token (PAT)**, bukan password akun.

***

Kalau kamu jalankan dan masih mentok, paste output dari:

```bash
git status
git pull --rebase origin main --allow-unrelated-histories
```

biar aku tunjukin langkah conflict-resolve yang tepat.

cikal@devops-ci:\~/three-body-problem-main$ cd \~/three-body-problem-main ls -la .env.dev.compose.example || true ls: cannot access '.env.dev.compose.example': No such file or directory cikal@devops-ci:\~/three-body-problem-main$ cp .env.dev.compose .env.dev.compose.example sed -i 's/RootPassDev123!/CHANGE\_ME/g; s/UserPassDev123!/CHANGE\_ME/g' .env.dev.compose.example sed -i '/^APP\_KEY=/d' .env.dev.compose.example cikal@devops-ci:\~/three-body-problem-main$ git add .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml frontend/src/App.js .env.dev.compose.example git status On branch main Your branch is up to date with 'github/main'. Changes to be committed: (use "git restore --staged \<file>..." to unstage) new file: .env.dev.compose.example new file: .gitignore new file: .gitlab-ci.yml new file: devops/docker/Dockerfile.frontend new file: devops/docker/Dockerfile.goapi new file: devops/docker/Dockerfile.laravel new file: devops/nginx/dev.conf new file: devops/nginx/frontend.conf new file: docker-compose.dev.yml modified: frontend/src/App.js cikal@devops-ci:\~/three-body-problem-main$ git commit -m "Add dev docker/nginx configs and CI pipeline" \[main 3abd724] Add dev docker/nginx configs and CI pipeline 10 files changed, 280 insertions(+), 2 deletions(-) create mode 100644 .env.dev.compose.example create mode 100644 .gitignore create mode 100644 .gitlab-ci.yml create mode 100644 devops/docker/Dockerfile.frontend create mode 100644 devops/docker/Dockerfile.goapi create mode 100644 devops/docker/Dockerfile.laravel create mode 100644 devops/nginx/dev.conf create mode 100644 devops/nginx/frontend.conf create mode 100644 docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ git pull --rebase origin main --allow-unrelated-histories fatal: unable to access 'https://gitlab.com/cikalfarid/three-body-problem.git/': Could not resolve host: gitlab.com cikal@devops-ci:\~/three-body-problem-main$ cikal@devops-ci:\~/three-body-problem-main$ git status git pull --rebase origin main --allow-unrelated-histories On branch main Your branch is ahead of 'github/main' by 1 commit. (use "git push" to publish your local commits) nothing to commit, working tree clean remote: Enumerating objects: 225, done. remote: Counting objects: 100% (49/49), done. remote: Compressing objects: 100% (49/49), done. remote: Total 225 (delta 28), reused 0 (delta 0), pack-reused 176 (from 2) Receiving objects: 100% (225/225), 281.70 KiB | 2.00 MiB/s, done. Resolving deltas: 100% (45/45), done. From https://gitlab.com/cikalfarid/three-body-problem \* branch main -> FETCH\_HEAD \* \[new branch] main -> origin/main Auto-merging README.md CONFLICT (add/add): Merge conflict in README.md Auto-merging frontend/src/App.js CONFLICT (add/add): Merge conflict in frontend/src/App.js error: could not apply 892eebb... Initial commit hint: Resolve all conflicts manually, mark them as resolved with hint: "git add/rm \<conflicted\_files>", then run "git rebase --continue". hint: You can instead skip this commit: run "git rebase --skip". hint: To abort and get back to the state before "git rebase", run "git rebase --abort". Could not apply 892eebb... Initial commit cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 1m 11s

Sip—sekarang kamu lagi **di tengah proses rebase** dan kena conflict di:

* `README.md`
* `frontend/src/App.js`

Karena kasusnya “unrelated histories”, cara **paling aman & gampang** adalah: **abort rebase** lalu lakukan **merge biasa** (lebih jelas “ours/theirs”-nya).

### 1) Abort rebase yang lagi jalan

```bash
cd ~/three-body-problem-main
git rebase --abort
```

Cek status harusnya normal lagi:

```bash
git status
```

### 2) Tarik state GitLab lalu merge (bukan rebase)

```bash
git fetch origin
git merge origin/main --allow-unrelated-histories
```

Nanti akan conflict lagi di 2 file itu (normal).

### 3) Resolve conflict dengan pilihan yang benar

Karena `App.js` punyamu sudah benar (pakai `/api/laravel/...` & `/api/go/...`), kita **pertahankan versi lokal** untuk `App.js`.

Untuk `README.md`, biasanya enaknya **ambil versi GitLab** (kalau GitLab sudah punya README bawaan). Kalau mau gabung dua-duanya, edit manual juga boleh.

Jalankan ini:

```bash
# ambil versi LOCAL (repo kamu sekarang) untuk App.js
git checkout --ours frontend/src/App.js

# ambil versi REMOTE (GitLab origin/main) untuk README
git checkout --theirs README.md
```

Lalu stage & commit merge:

```bash
git add README.md frontend/src/App.js
git commit -m "Merge GitLab main into local history"
```

### 4) Push ke GitLab

```bash
git push -u origin main
```

Kalau diminta password saat push ke GitLab: pakai **Personal Access Token (PAT)** sebagai password (bukan password akun).

***

#### Catatan soal error “Could not resolve host: gitlab.com”

Itu tadi sempat kejadian tapi kemudian berhasil fetch, jadi kemungkinan DNS sempat ngadat. Kalau muncul lagi:

```bash
getent hosts gitlab.com
ping -c 1 gitlab.com
```

***

Kalau kamu sudah jalanin langkah di atas dan masih mentok, paste output dari:

```bash
git status
git merge origin/main --allow-unrelated-histories
```

cikal@devops-ci:\~/three-body-problem-main$ cd \~/three-body-problem-main ls -la .env.dev.compose.example || true ls: cannot access '.env.dev.compose.example': No such file or directory cikal@devops-ci:\~/three-body-problem-main$ cp .env.dev.compose .env.dev.compose.example sed -i 's/RootPassDev123!/CHANGE\_ME/g; s/UserPassDev123!/CHANGE\_ME/g' .env.dev.compose.example sed -i '/^APP\_KEY=/d' .env.dev.compose.example cikal@devops-ci:\~/three-body-problem-main$ git add .gitignore .gitlab-ci.yml devops/ docker-compose.dev.yml frontend/src/App.js .env.dev.compose.example git status On branch main Your branch is up to date with 'github/main'. Changes to be committed: (use "git restore --staged \<file>..." to unstage) new file: .env.dev.compose.example new file: .gitignore new file: .gitlab-ci.yml new file: devops/docker/Dockerfile.frontend new file: devops/docker/Dockerfile.goapi new file: devops/docker/Dockerfile.laravel new file: devops/nginx/dev.conf new file: devops/nginx/frontend.conf new file: docker-compose.dev.yml modified: frontend/src/App.js cikal@devops-ci:\~/three-body-problem-main$ git commit -m "Add dev docker/nginx configs and CI pipeline" \[main 3abd724] Add dev docker/nginx configs and CI pipeline 10 files changed, 280 insertions(+), 2 deletions(-) create mode 100644 .env.dev.compose.example create mode 100644 .gitignore create mode 100644 .gitlab-ci.yml create mode 100644 devops/docker/Dockerfile.frontend create mode 100644 devops/docker/Dockerfile.goapi create mode 100644 devops/docker/Dockerfile.laravel create mode 100644 devops/nginx/dev.conf create mode 100644 devops/nginx/frontend.conf create mode 100644 docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ git pull --rebase origin main --allow-unrelated-histories fatal: unable to access 'https://gitlab.com/cikalfarid/three-body-problem.git/': Could not resolve host: gitlab.com cikal@devops-ci:\~/three-body-problem-main$ cikal@devops-ci:\~/three-body-problem-main$ git status git pull --rebase origin main --allow-unrelated-histories On branch main Your branch is ahead of 'github/main' by 1 commit. (use "git push" to publish your local commits) nothing to commit, working tree clean remote: Enumerating objects: 225, done. remote: Counting objects: 100% (49/49), done. remote: Compressing objects: 100% (49/49), done. remote: Total 225 (delta 28), reused 0 (delta 0), pack-reused 176 (from 2) Receiving objects: 100% (225/225), 281.70 KiB | 2.00 MiB/s, done. Resolving deltas: 100% (45/45), done. From https://gitlab.com/cikalfarid/three-body-problem \* branch main -> FETCH\_HEAD \* \[new branch] main -> origin/main Auto-merging README.md CONFLICT (add/add): Merge conflict in README.md Auto-merging frontend/src/App.js CONFLICT (add/add): Merge conflict in frontend/src/App.js error: could not apply 892eebb... Initial commit hint: Resolve all conflicts manually, mark them as resolved with hint: "git add/rm \<conflicted\_files>", then run "git rebase --continue". hint: You can instead skip this commit: run "git rebase --skip". hint: To abort and get back to the state before "git rebase", run "git rebase --abort". Could not apply 892eebb... Initial commit cikal@devops-ci:\~/three-body-problem-main$ cd \~/three-body-problem-main git rebase --abort cikal@devops-ci:\~/three-body-problem-main$ git status On branch main Your branch is ahead of 'github/main' by 1 commit. (use "git push" to publish your local commits) nothing to commit, working tree clean cikal@devops-ci:\~/three-body-problem-main$ git fetch origin git merge origin/main --allow-unrelated-histories Auto-merging .gitignore CONFLICT (add/add): Merge conflict in .gitignore Auto-merging .gitlab-ci.yml CONFLICT (add/add): Merge conflict in .gitlab-ci.yml Auto-merging README.md CONFLICT (add/add): Merge conflict in README.md Auto-merging devops/nginx/dev.conf CONFLICT (add/add): Merge conflict in devops/nginx/dev.conf Auto-merging docker-compose.dev.yml CONFLICT (add/add): Merge conflict in docker-compose.dev.yml Auto-merging frontend/src/App.js CONFLICT (add/add): Merge conflict in frontend/src/App.js Automatic merge failed; fix conflicts and then commit the result. cikal@devops-ci:\~/three-body-problem-main$ # ambil versi LOCAL (repo kamu sekarang) untuk App.js git checkout --ours frontend/src/App.js # ambil versi REMOTE (GitLab origin/main) untuk README git checkout --theirs README.md Updated 1 path from the index Updated 1 path from the index cikal@devops-ci:\~/three-body-problem-main$ git add README.md frontend/src/App.js git commit -m "Merge GitLab main into local history" U .gitignore U .gitlab-ci.yml U devops/nginx/dev.conf U docker-compose.dev.yml error: Committing is not possible because you have unmerged files. hint: Fix them up in the work tree, and then use 'git add/rm \<file>' hint: as appropriate to mark resolution and make a commit. fatal: Exiting because of an unresolved conflict. cikal@devops-ci:\~/three-body-problem-main$ getent hosts gitlab.com ping -c 1 gitlab.com 2606:4700:90:0:f22e:fbec:5bed:a9b9 gitlab.com PING gitlab.com (172.65.251.78) 56(84) bytes of data. 64 bytes from 172.65.251.78: icmp\_seq=1 ttl=255 time=11.9 ms --- gitlab.com ping statistics --- 1 packets transmitted, 1 received, 0% packet loss, time 0ms rtt min/avg/max/mdev = 11.925/11.925/11.925/0.000 ms cikal@devops-ci:\~/three-body-problem-main$ git status git merge origin/main --allow-unrelated-histories On branch main Your branch is ahead of 'github/main' by 1 commit. (use "git push" to publish your local commits) You have unmerged paths. (fix conflicts and run "git commit") (use "git merge --abort" to abort the merge) Changes to be committed: modified: README.md new file: certs/dev/dev.crt new file: certs/dev/dev.key new file: frontend/Dockerfile new file: go/Dockerfile new file: laravel/Dockerfile Unmerged paths: (use "git add \<file>..." to mark resolution) both added: .gitignore both added: .gitlab-ci.yml both added: devops/nginx/dev.conf both added: docker-compose.dev.yml error: Merging is not possible because you have unmerged files. hint: Fix them up in the work tree, and then use 'git add/rm \<file>' hint: as appropriate to mark resolution and make a commit. fatal: Exiting because of an unresolved conflict. cikal@devops-ci:\~/three-body-problem-main$ git push -u origin main Username for 'https://gitlab.com': Cikal Farid Password for 'https://Cikal%20Farid@gitlab.com': To https://gitlab.com/cikalfarid/three-body-problem.git ! \[rejected] main -> main (non-fast-forward) error: failed to push some refs to 'https://gitlab.com/cikalfarid/three-body-problem.git' hint: Updates were rejected because the tip of your current branch is behind hint: its remote counterpart. If you want to integrate the remote changes, hint: use 'git pull' before pushing again. hint: See the 'Note about fast-forwards' in 'git push --help' for details. cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 43s

Mantap—sekarang kamu **sudah benar pindah ke mode MERGE**, tapi masih ada **4 file konflik** jadi wajar belum bisa `commit`/`push`.

Yang kita lakukan: **ambil versi LOCAL** untuk semua file konflik (karena itu “final config” kamu), **buang cert/key dari staging** (jangan sampai kepush), lalu **commit merge**, baru **push**.

***

### 1) Selesaikan 4 konflik dengan pakai versi LOCAL (ours)

```bash
cd ~/three-body-problem-main

# ambil versi lokal untuk semua file konflik
git checkout --ours .gitignore .gitlab-ci.yml devops/nginx/dev.conf docker-compose.dev.yml
```

Pastikan `.gitignore` benar (abaikan env & cert):

```bash
grep -q '^\.env\.dev\.compose$' .gitignore || echo '.env.dev.compose' >> .gitignore
grep -q '^certs/dev/$' .gitignore || printf '\n# dev certs\ncerts/dev/\n' >> .gitignore
```

Stage file konflik yang sudah resolved:

```bash
git add .gitignore .gitlab-ci.yml devops/nginx/dev.conf docker-compose.dev.yml
```

***

### 2) Pastikan cert/key **tidak ikut ter-commit**

Di status kamu tadi, `certs/dev/dev.crt` dan `certs/dev/dev.key` sudah “Changes to be committed” — itu harus dibatalkan.

```bash
# keluarkan dari staging (jika masih staged)
git restore --staged certs/dev/dev.crt certs/dev/dev.key 2>/dev/null || true

# kalau tetap ke-track, paksa untrack dari index
git rm --cached -f certs/dev/dev.crt certs/dev/dev.key 2>/dev/null || true
```

***

### 3) Commit MERGE (baru bisa push)

```bash
git status
# pastikan: tidak ada "unmerged paths" lagi

git commit -m "Merge GitLab main into local history"
```

Kalau `git commit` masih nolak, paste output `git status`—artinya masih ada file konflik yang belum ke-add.

***

### 4) Push ke GitLab (pakai PAT, bukan password)

GitLab **tidak terima password akun** untuk Git over HTTPS. Harus pakai **Personal Access Token** sebagai password.

```bash
git push -u origin main
```

Saat prompt:

* **Username**: `cikalfarid` (bukan “Cikal Farid”)
* **Password**: isi **PAT GitLab** kamu

***

### 5) Quick sanity check

```bash
git status
git log --oneline --decorate -5
```

Kalau ini sudah beres, baru kita lanjut step berikutnya (CI pipeline jalan + deploy compose, dll).

cikal@devops-ci:\~/three-body-problem-main$ cd \~/three-body-problem-main # ambil versi lokal untuk semua file konflik git checkout --ours .gitignore .gitlab-ci.yml devops/nginx/dev.conf docker-compose.dev.yml Updated 4 paths from the index cikal@devops-ci:\~/three-body-problem-main$ grep -q '^\\.env\\.dev\\.compose$' .gitignore || echo '.env.dev.compose' >> .gitignore grep -q '^certs/dev/$' .gitignore || printf '\n# dev certs\ncerts/dev/\n' >> .gitignore cikal@devops-ci:\~/three-body-problem-main$ git add .gitignore .gitlab-ci.yml devops/nginx/dev.conf docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ # keluarkan dari staging (jika masih staged) git restore --staged certs/dev/dev.crt certs/dev/dev.key 2>/dev/null || true # kalau tetap ke-track, paksa untrack dari index git rm --cached -f certs/dev/dev.crt certs/dev/dev.key 2>/dev/null || true cikal@devops-ci:\~/three-body-problem-main$ git status # pastikan: tidak ada "unmerged paths" lagi git commit -m "Merge GitLab main into local history" On branch main Your branch is ahead of 'github/main' by 1 commit. (use "git push" to publish your local commits) All conflicts fixed but you are still merging. (use "git commit" to conclude merge) Changes to be committed: modified: README.md new file: frontend/Dockerfile new file: go/Dockerfile new file: laravel/Dockerfile \[main 5e93aff] Merge GitLab main into local history cikal@devops-ci:\~/three-body-problem-main$ git push -u origin main Username for 'https://gitlab.com': Cikal Farid Password for 'https://Cikal%20Farid@gitlab.com': Enumerating objects: 41, done. Counting objects: 100% (41/41), done. Delta compression using up to 2 threads Compressing objects: 100% (27/27), done. Writing objects: 100% (28/28), 4.52 KiB | 243.00 KiB/s, done. Total 28 (delta 14), reused 1 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git f941f38..5e93aff main -> main branch 'main' set up to track 'origin/main'. cikal@devops-ci:\~/three-body-problem-main$ git status git log --oneline --decorate -5 On branch main Your branch is up to date with 'origin/main'. nothing to commit, working tree clean 5e93aff (HEAD -> main, origin/main) Merge GitLab main into local history 3abd724 Add dev docker/nginx configs and CI pipeline f941f38 Reset CI pipeline (build/test/push/deploy) 2891da2 Reset CI pipeline (build/test/push/deploy) e385fdc Reset CI pipeline (build/test/push/deploy) cikal@devops-ci:\~/three-body-problem-main$

Mantap ✅ Push ke GitLab sudah **berhasil** dan repo kamu sekarang **clean + sudah track `origin/main`**.

Tinggal 2 hal penting yang perlu kamu rapikan biar “final” dan aman:

***

### 1) Pastikan cert/key benar-benar tidak ikut ke repo

Kamu tadi sudah `git rm --cached` (bagus), tapi **cek dulu di GitLab apakah `certs/dev/dev.key` sempat kepush sebelumnya**.

Di VM, jalankan:

```bash
cd ~/three-body-problem-main
git ls-files | grep -E '^certs/' || echo "OK: tidak ada certs yang ke-track"
```

Kalau masih muncul `certs/dev/dev.key` / `.crt`, jalankan:

```bash
git rm -r --cached certs
git commit -m "Stop tracking dev certs"
git push
```

***

### 2) File “extra” yang ikut dari remote GitLab

Di commit merge kamu, ikut masuk:

* `frontend/Dockerfile`
* `go/Dockerfile`
* `laravel/Dockerfile`

Kalau kamu **tidak mau** pakai yang itu (karena sudah pakai `devops/docker/*`), kamu bisa hapus biar tidak bikin bingung CI.

Cek dulu isinya:

```bash
sed -n '1,160p' frontend/Dockerfile
sed -n '1,160p' go/Dockerfile
sed -n '1,160p' laravel/Dockerfile
```

Kalau memang tidak dipakai, hapus:

```bash
git rm frontend/Dockerfile go/Dockerfile laravel/Dockerfile
git commit -m "Remove unused Dockerfiles (use devops/docker/*)"
git push
```

***

Kalau 2 poin di atas sudah beres, kita bisa **lanjut ke tahap berikutnya**: memastikan `.gitlab-ci.yml final` kamu jalan mulus (build → push ke Harbor → deploy compose) dan juga memastikan **Harbor + Docker auto-start** saat VM2 reboot supaya semua tetap aktif.

sudah beres dan lanjutSelesai berpikir selama 3m 27s

Oke, kita lanjut ke **finalisasi `.gitlab-ci.yml` + deploy otomatis**, dan sekalian beresin 1 hal kecil yang kemarin bikin `nginx -t` gagal saat dites “ephemeral”.

### 1) Fix `devops/nginx/dev.conf` biar `nginx -t` lolos di CI

Masalahnya: `nginx -t` di container “standalone” tidak bisa resolve hostname `frontend/laravel/goapi` (karena tidak ada Docker network). Solusi paling aman: **pakai variable di `proxy_pass`** supaya Nginx tidak resolve saat start/test.

Ganti file ini:

```bash
cd ~/three-body-problem-main

cat > devops/nginx/dev.conf <<'EOF'
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
  listen 80;
  server_name dev.local;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name dev.local;

  ssl_certificate     /etc/nginx/certs/dev.crt;
  ssl_certificate_key /etc/nginx/certs/dev.key;

  proxy_set_header Host              $host;
  proxy_set_header X-Real-IP         $remote_addr;
  proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;

  # Docker DNS (di network docker compose)
  resolver 127.0.0.11 ipv6=off valid=10s;

  location / {
    set $up_frontend "frontend:80";
    proxy_pass http://$up_frontend;
  }

  location ^~ /api/laravel {
    limit_req zone=api_limit burst=20 nodelay;
    rewrite ^/api/laravel/?(.*)$ /$1 break;

    set $up_laravel "laravel:80";
    proxy_pass http://$up_laravel;
  }

  location ^~ /api/go {
    limit_req zone=api_limit burst=20 nodelay;
    rewrite ^/api/go/?(.*)$ /$1 break;

    set $up_go "goapi:8080";
    proxy_pass http://$up_go;
  }
}
EOF

git add devops/nginx/dev.conf
git commit -m "Fix nginx dev.conf to pass nginx -t in CI"
git push
```

***

### 2) `.gitlab-ci.yml` FINAL (build → push Harbor → deploy compose)

Ini asumsi kamu pakai **self-hosted GitLab Runner di VM2 (shell executor)** yang bisa akses:

* Docker daemon
* Harbor `192.168.56.43:8081`

Buat/replace `.gitlab-ci.yml`:

```bash
cd ~/three-body-problem-main

cat > .gitlab-ci.yml <<'EOF'
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

default:
  # Kalau runner kamu ada tag tertentu, isi di sini (opsional)
  # tags: ["vm2"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# --- VALIDATE ---
validate:
  stage: validate
  script:
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    # nginx config test (harus lolos)
    - docker run --rm
      -v "$CI_PROJECT_DIR/devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro"
      -v "$CI_PROJECT_DIR/devops/nginx/frontend.conf:/etc/nginx/conf.d/default.conf:ro"
      nginx:alpine nginx -t

# helper: login Harbor + set TAG
.build_common:
  stage: build
  script:
    - set -euo pipefail
    - test -n "${HARBOR_URL:-}" && test -n "${HARBOR_PROJECT:-}" && test -n "${HARBOR_USERNAME:-}" && test -n "${HARBOR_PASSWORD:-}"
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - echo "TAG=$TAG" > tag.env
  artifacts:
    reports:
      dotenv: tag.env

build_goapi:
  extends: .build_common
  script:
    - set -euo pipefail
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  extends: .build_common
  script:
    - set -euo pipefail
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  extends: .build_common
  script:
    - set -euo pipefail
    - echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# --- DEPLOY DEV (main only) ---
deploy_dev:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - set -euo pipefail
    - TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"

    # deploy dir stabil supaya compose project stabil dan tidak dobel container
    - DEPLOY_DIR="$HOME/threebody-deploy"
    - mkdir -p "$DEPLOY_DIR"
    - rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"

    # generate cert di deploy dir (tanpa commit cert ke repo)
    - mkdir -p "$DEPLOY_DIR/certs/dev"
    - |
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

    # write .env.dev.compose from CI variables (WAJIB diset di GitLab CI Variables)
    - |
      cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
      MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
      MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}

      HARBOR_URL=${HARBOR_URL}
      HARBOR_PROJECT=${HARBOR_PROJECT}

      CI_COMMIT_SHA=${TAG}
      APP_KEY=${DEV_APP_KEY}
      ENV

    - cd "$DEPLOY_DIR"
    - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml pull
    - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

    # migrate otomatis
    - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force

    - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps
EOF

git add .gitlab-ci.yml
git commit -m "Finalize GitLab CI: build/push to Harbor and deploy dev compose"
git push
```

***

### 3) WAJIB set GitLab CI Variables (di Project → Settings → CI/CD → Variables)

Buat variabel berikut (Masked + Protected kalau perlu):

**Harbor**

* `HARBOR_URL` = `192.168.56.43:8081`
* `HARBOR_PROJECT` = `threebody`
* `HARBOR_USERNAME` = _(pakai robot account Harbor atau admin)_
* `HARBOR_PASSWORD` = _(secret)_

**Dev env**

* `DEV_MYSQL_ROOT_PASSWORD`
* `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY` = `base64:...` (yang stabil, jangan gonta-ganti)

***

### 4) Biar setelah VM2 reboot semuanya tetap “aktif”

Ini kuncinya:

1. Docker service auto-start:

```bash
sudo systemctl enable --now docker
```

2. Pastikan container Harbor & stack kamu punya restart policy (biasanya Harbor sudah `always`, app kamu sudah `unless-stopped`). Setelah sekali `docker compose up -d`, saat reboot container akan hidup lagi otomatis.
3. GitLab Runner juga auto-start:

```bash
sudo systemctl enable --now gitlab-runner
```

***

Kalau kamu sudah push `.gitlab-ci.yml` final di atas, bilang hasil pipeline-nya di GitLab (job mana yang fail kalau ada). Kalau fail, biasanya 2 penyebab utama: **runner bukan self-hosted** atau **variable Harbor/robot belum benar**—nanti kita beresin cepat.

Running with gitlab-runner 18.4.0\~pre.246.g71914659 (71914659) on green-3.saas-linux-small-amd64.runners-manager.gitlab.com/default Jhc\_Jxvh8, system ID: s\_0e6850b2bce1 Preparing the "docker+machine" executor 00:20 Using default image Using Docker executor with image ruby:3.1 ... Using default image Using effective pull policy of \[always] for container ruby:3.1 Pulling docker image ruby:3.1 ... Using docker image sha256:9981df1d883b246c27c62f8ccb9b57d3e07d14cee8092299e102b4a69c35ea61 for ruby:3.1 with digest ruby@sha256:91627f55e8969006aab67d15c92fb930500ff73948803da1330b8a853fecebb5 ... Preparing environment 00:05 Using effective pull policy of \[always] for container sha256:23a975904c78ebfebd4d1310c52ae3507d6d4d9e68e9222ff79454f6b7108034 Running on runner-jhcjxvh8-project-76926856-concurrent-0 via runner-jhcjxvh8-s-l-s-amd64-1765658336-f47cb92c... Getting source from Git repository 00:01 Gitaly correlation ID: ed7810bbb8c74d83b1bd7a7864806b09 Fetching changes with git depth set to 20... Initialized empty Git repository in /builds/cikalfarid/three-body-problem/.git/ Created fresh repository. Checking out d7a447dc as detached HEAD (ref is main)... Skipping Git submodules setup $ git remote set-url origin "${CI\_REPOSITORY\_URL}" || echo 'Not a git repository; skipping' Executing "step\_script" stage of the job script 00:00 Using default image Using effective pull policy of \[always] for container ruby:3.1 Using docker image sha256:9981df1d883b246c27c62f8ccb9b57d3e07d14cee8092299e102b4a69c35ea61 for ruby:3.1 with digest ruby@sha256:91627f55e8969006aab67d15c92fb930500ff73948803da1330b8a853fecebb5 ... $ set -euo pipefail $ docker version /usr/bin/bash: line 172: docker: command not found Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit code 1Selesai berpikir selama 52s

Log itu jelas banget: **job kamu jalan di GitLab.com Shared Runner (docker+machine)** pakai image `ruby:3.1`, jadi **nggak ada binary `docker`** di dalam container. Dan yang lebih penting: **Shared Runner GitLab.com juga tidak bisa akses IP private kamu `192.168.56.43:8081` (Harbor di VM)**, jadi walaupun docker-nya ada pun, push ke Harbor lokal bakal mentok network.

Jadi fix yang benar: **pakai Self-Hosted Runner di VM2** (yang satu network dengan Harbor), lalu “paksa” pipeline pakai runner itu (via `tags`) dan **disable shared runner**.

Di bawah ini step yang rapi sampai beres.

***

### 1) Daftarkan GitLab Runner di VM2 (rekomendasi: shell executor)

Jalankan di VM2:

```bash
# pastikan docker sudah aktif
sudo systemctl enable --now docker

# install gitlab-runner (Ubuntu/Debian)
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install -y gitlab-runner

# biar gitlab-runner bisa akses docker tanpa sudo
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
sudo systemctl enable --now gitlab-runner
```

Sekarang register runner (token ambil dari GitLab: Project → Settings → CI/CD → Runners → **Registration token** / “Set up a specific runner”):

```bash
sudo gitlab-runner register \
  --url "https://gitlab.com" \
  --registration-token "ISI_TOKEN_DARI_GITLAB" \
  --executor "shell" \
  --description "vm2-shell-runner" \
  --tag-list "vm2" \
  --run-untagged="false" \
  --locked="false"
```

Cek:

```bash
sudo gitlab-runner verify
sudo gitlab-runner status
```

***

### 2) Matikan Shared Runner, pakai runner VM2 saja

Di GitLab project kamu:

* Settings → CI/CD → Runners
* **Disable shared runners**
* Pastikan runner “vm2-shell-runner” statusnya **online** dan aktif.

***

### 3) Paksa `.gitlab-ci.yml` pakai tag `vm2` (ini wajib)

Edit `.gitlab-ci.yml` kamu, pastikan ada ini:

```yaml
default:
  tags: ["vm2"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker info | sed -n '/Insecure Registries/,+10p' || true
```

Kalau file kamu masih ada `image: ruby:3.1` di atas (top-level), **hapus** (shell executor nggak butuh image).

Setelah itu commit & push:

```bash
git add .gitlab-ci.yml
git commit -m "Force pipeline to run on VM2 runner"
git push
```

***

### 4) Quick sanity check di VM2: Harbor & insecure registry

Kamu sudah set `/etc/docker/daemon.json`. Pastikan benar:

```bash
docker info | sed -n '/Insecure Registries/,+10p'
```

Harus kelihatan `192.168.56.43:8081`.

Dan pastikan Harbor project `threebody` memang ada (UI Harbor).

***

### 5) Kenapa sebelumnya fail?

* Runner yang dipakai: `green-3.saas-linux-small-amd64...` = **shared runner GitLab.com**
* Jalan dalam container `ruby:3.1` → `docker: command not found`
* Dan walaupun kamu ganti jadi image docker pun, shared runner **nggak bisa reach** `192.168.56.43` (private IP).

***

Kalau kamu sudah lakukan step 1–3, kirimkan log pipeline berikutnya (minimal job validate/build pertama). Kalau runner sudah benar, harus terlihat di log: **runner description “vm2-shell-runner”** atau tag `vm2`, dan `docker version` sudah jalan.

cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner register \ --url "https://gitlab.com" \ --registration-token "glrt-P5qLStHlfkb9wfPeQ1orxW86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1r0u5v0" \ --executor "shell" \ --description "vm2-shell-runner" \ --tag-list "vm2" \ --run-untagged="false" \ --locked="false" Runtime platform arch=amd64 os=linux pid=45509 revision=5908bc19 version=18.6.5 Running in system-mode. Enter the GitLab instance URL (for example, https://gitlab.com/): \[https://gitlab.com]: Enter the registration token: \[glrt-P5qLStHlfkb9wfPeQ1orxW86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1r0u5v0]: WARNING: A runner with this system ID and token has already been registered. WARNING: You have specified an authentication token in the legacy parameter --registration-token. This has triggered the 'legacy-compatible registration process' which has resulted in the following command line parameters being ignored: --locked, --access-level, --run-untagged, --maximum-timeout, --paused, --tag-list, and --maintenance-note. For more information, see https://docs.gitlab.com/ci/runners/new\_creation\_workflow/#changes-to-the-gitlab-runner-register-command-syntaxThese parameters and the legacy-compatible registration process will be removed in a future GitLab Runner release. Verifying runner... is valid correlation\_id=9ad84f0353f59b07-ATL runner=P5qLStHlf Enter a name for the runner. This is stored only in the local config.toml file: \[vm2-shell-runner]: Enter an executor: custom, shell, parallels, virtualbox, instance, ssh, docker, docker-windows, docker+machine, kubernetes, docker-autoscaler: \[shell]: Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml" cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner verify sudo gitlab-runner status Runtime platform arch=amd64 os=linux pid=45879 revision=5908bc19 version=18.6.5 Running in system-mode. Verifying runner... is valid correlation\_id=9ad84f6b2475a5b9-ATL runner=P5qLStHlf Verifying runner... is valid correlation\_id=9ad84f6e035db58d-ATL runner=P5qLStHlf Runtime platform arch=amd64 os=linux pid=45899 revision=5908bc19 version=18.6.5 gitlab-runner: Service is running cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: # Kalau runner kamu ada tag tertentu, isi di sini (opsional) # tags: \["vm2"] before\_script: - set -euo pipefail - docker version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null # nginx config test (harus lolos) - docker run --rm -v "$CI\_PROJECT\_DIR/devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro" -v "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf:/etc/nginx/conf.d/default.conf:ro" nginx:alpine nginx -t # helper: login Harbor + set TAG .build\_common: stage: build script: - set -euo pipefail - test -n "${HARBOR\_URL:-}" && test -n "${HARBOR\_PROJECT:-}" && test -n "${HARBOR\_USERNAME:-}" && test -n "${HARBOR\_PASSWORD:-}" - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" - echo "TAG=$TAG" > tag.env artifacts: reports: dotenv: tag.env build\_goapi: extends: .build\_common script: - set -euo pipefail - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: extends: .build\_common script: - set -euo pipefail - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: extends: .build\_common script: - set -euo pipefail - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - set -euo pipefail - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" # deploy dir stabil supaya compose project stabil dan tidak dobel container - DEPLOY\_DIR="$HOME/threebody-deploy" - mkdir -p "$DEPLOY\_DIR" - rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" # generate cert di deploy dir (tanpa commit cert ke repo) - mkdir -p "$DEPLOY\_DIR/certs/dev" - | if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # write .env.dev.compose from CI variables (WAJIB diset di GitLab CI Variables) - | cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV - cd "$DEPLOY\_DIR" - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml pull - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # migrate otomatis - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force - docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$ bantu updateSelesai berpikir selama 2m 29s

Siap — `.gitlab-ci.yml` kamu perlu **2 update penting**:

1. **Wajib pakai runner VM2** → `tags: ["vm2"]` harus diaktifkan (dan runner di GitLab harus punya tag `vm2`, karena tadi `--tag-list` kamu _di-ignore_ oleh mode legacy register).
2. Job **validate nginx** di CI kamu bakal gagal kalau `nginx -t` dijalankan tanpa:
   * DNS upstream `frontend/laravel/goapi` (host not found), dan
   * file SSL cert (`/etc/nginx/certs/dev.crt` & `.key`) (tidak ada di repo)

Di bawah ini versi `.gitlab-ci.yml` yang sudah aku rapikan + aman buat kasus di atas.

***

### ✅ Replace isi `.gitlab-ci.yml` dengan ini

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

default:
  tags: ["vm2"]   # WAJIB: paksa jalan di runner VM2
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# --- VALIDATE ---
validate:
  stage: validate
  script:
    # validasi compose (pakai env example biar ga butuh secret)
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null

    # nginx -t butuh:
    # - upstream hostname frontend/laravel/goapi biar resolve
    # - cert ssl biar ga error "cannot load certificate"
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      CERT_DIR="$(mktemp -d)"

      # generate cert sementara untuk test nginx
      docker run --rm -v "$CERT_DIR:/out" alpine:3.19 sh -lc '
        apk add --no-cache openssl >/dev/null
        openssl req -newkey rsa:2048 -nodes \
          -keyout /out/dev.key \
          -x509 -days 365 \
          -out /out/dev.crt \
          -subj "/CN=dev.local"
      '

      docker network create "$NET" >/dev/null

      # dummy containers agar hostname upstream resolve di nginx -t
      docker run -d --rm --name frontend --network "$NET" nginx:alpine >/dev/null
      docker run -d --rm --name laravel  --network "$NET" nginx:alpine >/dev/null
      docker run -d --rm --name goapi    --network "$NET" nginx:alpine >/dev/null

      # test config
      docker run --rm --network "$NET" \
        -v "$CI_PROJECT_DIR/devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro" \
        -v "$CI_PROJECT_DIR/devops/nginx/frontend.conf:/etc/nginx/conf.d/default.conf:ro" \
        -v "$CERT_DIR:/etc/nginx/certs:ro" \
        nginx:alpine nginx -t

      # cleanup
      docker rm -f frontend laravel goapi >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true
      rm -rf "$CERT_DIR" || true

# --- BUILD + PUSH ---
build_goapi:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# --- DEPLOY DEV (main only) ---
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"

      DEPLOY_DIR="$HOME/threebody-deploy"
      mkdir -p "$DEPLOY_DIR"

      # sync file deploy
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      # generate cert (persist di host, bukan di repo)
      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      # tulis env deploy dari CI Variables
      cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
      MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
      MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}

      HARBOR_URL=${HARBOR_URL}
      HARBOR_PROJECT=${HARBOR_PROJECT}

      CI_COMMIT_SHA=${TAG}
      APP_KEY=${DEV_APP_KEY}
      ENV

      cd "$DEPLOY_DIR"
      docker compose --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      # tunggu laravel siap, lalu migrate
      for i in $(seq 1 30); do
        if docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php -v >/dev/null 2>&1; then
          break
        fi
        sleep 2
      done
      docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps
```

***

### 🔧 2 hal yang WAJIB kamu set di GitLab (biar tidak pending)

#### A) Set tag runner jadi `vm2`

Karena tadi `--tag-list "vm2"` kamu **diabaikan** (legacy flow), jadi:

* GitLab → Project → **Settings → CI/CD → Runners**
* Klik runner VM2 kamu → set **Tags = vm2**
* Pastikan **“Run untagged jobs” OFF** (opsional), karena kita pakai tag.

#### B) Isi CI/CD Variables (Project → Settings → CI/CD → Variables)

Minimal:

* `HARBOR_URL` = `192.168.56.43:8081`
* `HARBOR_PROJECT` = `threebody`
* `HARBOR_USERNAME` = `admin` (atau user lain)
* `HARBOR_PASSWORD` = (password)
* `DEV_MYSQL_ROOT_PASSWORD`
* `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY` (yang base64:...)

***

### Setelah update, jalankan ini

```bash
git add .gitlab-ci.yml
git commit -m "Fix CI: enforce vm2 runner + nginx validate with dummy upstream + ssl"
git push
```

Kalau sudah push, kirim 30–50 baris log pipeline bagian atas (runner info + job validate) — harusnya sekarang:

* runner yang kepakai **vm2**
* `docker version` **jalan**
* validate **tidak lagi error host not found / cert missing**

DEV\_MYSQL\_ROOT\_PASSWORD DEV\_MYSQL\_PASSWORD DEV\_APP\_KEY (yang base64:...) value nya apa di gitlab ini?Selesai berpikir selama 18s

Itu **isi (value) yang kamu set di GitLab CI/CD Variables** supaya job `deploy_dev` bisa bikin file `.env.dev.compose` di server (VM2) secara otomatis.

#### 1) `DEV_MYSQL_ROOT_PASSWORD`

Ini password **user `root` MySQL** di container.

* Kalau kamu sebelumnya pakai ini di `.env.dev.compose`:
  * `MYSQL_ROOT_PASSWORD=RootPassDev123!`\
    maka di GitLab isi:
  * `DEV_MYSQL_ROOT_PASSWORD = RootPassDev123!`

> Catatan penting: kalau volume MySQL (`mysql_data`) **sudah pernah dibuat**, mengganti password di variable **tidak akan mengubah password root yang sudah tersimpan**. Jadi isinya harus **sama dengan password saat pertama kali volume dibuat**, kecuali kamu reset DB (down -v).

***

#### 2) `DEV_MYSQL_PASSWORD`

Ini password untuk **user aplikasi** (`threebody`) di MySQL.

* Kalau kamu sebelumnya pakai:
  * `MYSQL_PASSWORD=UserPassDev123!`\
    maka isi GitLab:
  * `DEV_MYSQL_PASSWORD = UserPassDev123!`

***

#### 3) `DEV_APP_KEY`

Ini **APP\_KEY Laravel**. Formatnya harus seperti ini:

* `base64:<random_base64_32_bytes>`

Cara bikin (pilih salah satu):

**Opsi A (paling gampang, dari host/VM):**

```bash
echo "base64:$(openssl rand -base64 32)"
```

**Opsi B (kalau sudah bisa masuk container laravel):**

```bash
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan key:generate --show
```

Lalu copy hasilnya ke GitLab:

* `DEV_APP_KEY = base64:...`

> Penting: `DEV_APP_KEY` sebaiknya **stabil** (jangan berubah-ubah) supaya data yang dienkripsi Laravel (kalau ada) tidak rusak.

***

### Cara set di GitLab

Project → **Settings → CI/CD → Variables** → Add variable:

* Key: `DEV_MYSQL_ROOT_PASSWORD` Value: (password root kamu)
* Key: `DEV_MYSQL_PASSWORD` Value: (password user `threebody`)
* Key: `DEV_APP_KEY` Value: `base64:...`

Untuk password Harbor (`HARBOR_PASSWORD`) sebaiknya centang **Masked**.\
Untuk `DEV_APP_KEY`, kalau GitLab menolak “Masked” (kadang regex ketat), cukup **jangan dimasked** (tetap aman kalau project private), atau pakai Protected sesuai kebutuhan.

Kalau kamu mau, kirim isi `.env.dev.compose` kamu yang sekarang (tanpa password kalau mau disensor), aku cocokin persis variable mana saja yang wajib ada dan nilainya harus apa.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000013e872079fcee5e0085cd433\&ts=490475\&p=fs\&cid=1\&sig=174f0b744390fb92fc0f0b8da0086c0252227495d4ff94d2241664bea6501a5a\&v=0)cikal@devops-ci:\~/three-body-problem-main$ echo "base64:$(openssl rand -base64 32)" base64:LUtk3bmVqbELgudTMT0us8hv5JFGlZP5cVAMQBHAkPM= cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["vm2"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # validasi compose (pakai env example biar ga butuh secret) - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null # nginx -t butuh: # - upstream hostname frontend/laravel/goapi biar resolve # - cert ssl biar ga error "cannot load certificate" - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" CERT\_DIR="$(mktemp -d)" # generate cert sementara untuk test nginx docker run --rm -v "$CERT\_DIR:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' docker network create "$NET" >/dev/null # dummy containers agar hostname upstream resolve di nginx -t docker run -d --rm --name frontend --network "$NET" nginx:alpine >/dev/null docker run -d --rm --name laravel --network "$NET" nginx:alpine >/dev/null docker run -d --rm --name goapi --network "$NET" nginx:alpine >/dev/null # test config docker run --rm --network "$NET" \ -v "$CI\_PROJECT\_DIR/devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro" \ -v "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf:/etc/nginx/conf.d/default.conf:ro" \ -v "$CERT\_DIR:/etc/nginx/certs:ro" \ nginx:alpine nginx -t # cleanup docker rm -f frontend laravel goapi >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true rm -rf "$CERT\_DIR" || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" DEPLOY\_DIR="$HOME/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # generate cert (persist di host, bukan di repo) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # tulis env deploy dari CI Variables cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" docker compose --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # tunggu laravel siap, lalu migrate for i in $(seq 1 30); do if docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php -v >/dev/null 2>&1; then break fi sleep 2 done docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$ oke lanjut push yaa?Selesai berpikir selama 43s

Iya, **boleh lanjut push** — tapi cek 2 hal ini dulu biar pipeline nggak “misterius” fail:

1. **Protected variables vs branch**

* Di screenshot, variable kamu banyak yang **Protected**.
* Protected variables **hanya kebaca kalau pipeline jalan di protected branch/tag**.\
  ✅ Jadi pilih salah satu:
* **Protect branch `main`** di GitLab (Settings → Repository → Protected branches), **atau**
* **Uncheck Protected** untuk variable yang dipakai pipeline (`DEV_*` dan `HARBOR_*`) kalau kamu nggak mau protect branch.

2. **Runner shell bisa akses Docker**\
   Karena runner kamu `shell`, job akan jalan sebagai user `gitlab-runner`. Pastikan dia bisa akses `/var/run/docker.sock`.\
   Kalau nanti error “permission denied”, fix cepat:

```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
# login ulang / reboot kalau perlu
```

***

### Kalau 2 hal di atas sudah OK → push aja

Di repo:

```bash
cd ~/three-body-problem-main
git add .gitlab-ci.yml
git commit -m "Update CI to use vm2 runner and deploy" || true
git push origin main
```

Setelah push, jalankan pipeline. Kalau `validate` lolos tapi `build_*` gagal login/push, biasanya karena:

* `HARBOR_PROJECT` salah (harus `threebody`)
* project di Harbor belum dibuat / permission user kurang
* insecure registry belum kebaca di VM2 Docker daemon (tapi kamu sudah set)

Kalau kamu mau, kirim screenshot **Protected branch setting** + satu log fail berikutnya (kalau ada), biar aku tune sampai green semua.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000007f6472078e6ab5e37131a46c\&ts=490475\&p=fs\&cid=1\&sig=c622c406dbe15859a7677d4c3e030f032f40cfee6d991318e151ef634a9c56fc\&v=0)cikal@devops-ci:\~/three-body-problem-main$ cd \~/three-body-problem-main git add .gitlab-ci.yml git commit -m "Update CI to use vm2 runner and deploy" || true git push origin main \[main 468b51c] Update CI to use vm2 runner and deploy 1 file changed, 98 insertions(+), 58 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.34 KiB | 91.00 KiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git d7a447d..468b51c main -> main cikal@devops-ci:\~/three-body-problem-main$ sudo systemctl status gitlab-runner ● gitlab-runner.service - GitLab Runner Loaded: loaded (/etc/systemd/system/gitlab-runner.service; enabled; preset: enabled) Active: active (running) since Sat 2025-12-13 21:04:57 UTC; 1min 11s ago Main PID: 62323 (gitlab-runner) Tasks: 7 (limit: 2265) Memory: 23.4M (peak: 23.9M) CPU: 1.095s CGroup: /system.slice/gitlab-runner.service └─62323 /usr/bin/gitlab-runner run --config /etc/gitlab-runner/config.toml --working-directory /home/gitlab-runner> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: Recommended solutions: Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: 1. Increase 'concurrent' to at least 3 (current: 1) Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: 2. Increase 'request\_concurrency' to 2-4 for 2 runners currently using reques> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: Note: The 'FF\_USE\_ADAPTIVE\_REQUEST\_CONCURRENCY' feature flag can help automatic> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: This message will be printed each time the configuration is reloaded if the iss> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: See documentation: https://docs.gitlab.com/runner/configuration/advanced-config> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: listen\_address not defined, metrics & debug endpoints disabled builds=0 max\_bu> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: \[session\_server].listen\_address not defined, session endpoints disabled builds> Dec 13 21:04:58 devops-ci gitlab-runner\[62323]: Initializing executor providers builds=0 max\_builds=1 Dec 13 21:05:10 devops-ci gitlab-runner\[62323]: Waiting before making the next call context=ratelimit-requester> lines 1-20/20 (END) cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner verify Runtime platform arch=amd64 os=linux pid=63654 revision=5908bc19 version=18.6.5 Running in system-mode. Verifying runner... is valid correlation\_id=9ad86cd46715b581-ATL runner=P5qLStHlf Verifying runner... is valid correlation\_id=9ad86cd7b2823541-ATL runner=P5qLStHlf cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 1m 13s

Sip, push kamu **sudah benar** — sekarang masalahnya ada di **pipeline “Pending / stuck”** itu.

Dari screenshot pipeline: semua job **nunggu runner**. Ini hampir pasti karena **job kamu pakai tag `vm2`**, tapi runner VM2 kamu **belum punya tag `vm2`** (waktu `gitlab-runner register` kemarin ada warning: `--tag-list ... ignored` karena pakai legacy `--registration-token`). Akibatnya GitLab bilang “nggak ada runner yang match tag vm2” → job stuck.

### 1) Fix utama: set tag runner jadi `vm2`

Paling gampang via UI:

**Project → Settings → CI/CD → Runners → Project runners → klik runner “vm2-shell-runner” → Edit**

* **Tags**: isi `vm2`
* Pastikan runner **Active**
* “Run untagged jobs”: boleh OFF (karena kita memang pakai tag)
* “Protected”: **OFF dulu** kalau branch `main` belum kamu protect (biar job bisa jalan di main)

Setelah itu, **Retry pipeline** (atau push commit kecil lagi).

> Alternatif (kalau mau sekalian rapi): protect branch `main`, lalu boleh set runner jadi protected & variables tetap protected.

***

### 2) Setelah job mulai jalan, pastikan user `gitlab-runner` bisa akses Docker

Karena executor kamu `shell`, job jalan sebagai user `gitlab-runner`. Jalankan ini di VM2:

```bash
sudo -u gitlab-runner docker ps
```

Kalau keluar error permission, fix:

```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
# lalu coba lagi:
sudo -u gitlab-runner docker ps
```

***

### 3) Soal DEV\_\* variables (nilai apa?)

Isi sama seperti yang kamu pakai di VM2 (dev env):

* `DEV_MYSQL_ROOT_PASSWORD` = password root mysql kamu (contoh yang kamu pakai: `RootPassDev123!`)
* `DEV_MYSQL_PASSWORD` = password user mysql `threebody` (contoh: `UserPassDev123!`)
* `DEV_APP_KEY` = string Laravel APP\_KEY **yang lengkap** diawali `base64:...`
  * kamu bisa generate sekali (seperti yang kamu lakukan):
    * contoh: `base64:LUtk3bmVqbELgudTMT0us8hv5JFGlZP5cVAMQBHAkPM=`

Kalau variables kamu **Protected**, pastikan `main` juga **Protected branch**, kalau tidak variables itu **tidak akan kebaca** saat deploy.

***

Kalau sudah set tag runner `vm2`, pipeline harus langsung jalan.\
Kalau setelah itu job gagal, kirim **log job validate/build pertama yang gagal**, aku beresin sampai green.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000026cc72079db85f74e554a8d5\&ts=490475\&p=fs\&cid=1\&sig=60d4b87f1f6337e0d2d6f5755d1ed58970098583200a543545aa173a332b21c4\&v=0)cikal@devops-ci:\~/three-body-problem-main$ sudo -u gitlab-runner docker ps CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES d18016f82438 nginx:alpine "/docker-entrypoint.…" 2 hours ago Up About an hour 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp three-body-problem-main-nginx-dev-1 136539596e73 192.168.56.43:8081/threebody/frontend:local "/docker-entrypoint.…" 2 hours ago Up About an hour 80/tcp three-body-problem-main-frontend-1 677ae78c6fe4 192.168.56.43:8081/threebody/laravel:local "docker-php-entrypoi…" 2 hours ago Up About an hour 80/tcp three-body-problem-main-laravel-1 aac74363478e 192.168.56.43:8081/threebody/goapi:local "/app/goapi" 2 hours ago Up About an hour 8080/tcp three-body-problem-main-goapi-1 1515109d83de mysql:8.0 "docker-entrypoint.s…" 2 hours ago Up About an hour (healthy) 3306/tcp, 33060/tcp three-body-problem-main-mysql-1 82885c6c7e15 goharbor/nginx-photon:v2.10.0 "nginx -g 'daemon of…" 4 hours ago Up About an hour (healthy) 0.0.0.0:8081->8080/tcp, \[::]:8081->8080/tcp nginx e501f5d63061 goharbor/harbor-jobservice:v2.10.0 "/harbor/entrypoint.…" 4 hours ago Up About an hour (healthy) harbor-jobservice 9d071bb8a1df goharbor/harbor-core:v2.10.0 "/harbor/entrypoint.…" 4 hours ago Up About an hour (healthy) harbor-core 43a07e2e2d41 goharbor/registry-photon:v2.10.0 "/home/harbor/entryp…" 4 hours ago Up About an hour (healthy) registry d198b314fd34 goharbor/harbor-portal:v2.10.0 "nginx -g 'daemon of…" 4 hours ago Up About an hour (healthy) harbor-portal b6be3be19741 goharbor/redis-photon:v2.10.0 "redis-server /etc/r…" 4 hours ago Up About an hour (healthy) redis 0ead292610dd goharbor/harbor-registryctl:v2.10.0 "/home/harbor/start.…" 4 hours ago Up About an hour (healthy) registryctl 7bcf866625c6 goharbor/harbor-db:v2.10.0 "/docker-entrypoint.…" 4 hours ago Up About an hour (healthy) harbor-db 147c6797fc06 goharbor/harbor-log:v2.10.0 "/bin/sh -c /usr/loc…" 4 hours ago Up About an hour (healthy) 127.0.0.1:1514->10514/tcp harbor-log cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 2m 24s

Mantap — dari screenshot runner kamu **Online**, dan dari `sudo -u gitlab-runner docker ps` artinya **shell-runner di VM2 sudah bisa akses Docker** ✅

Tapi ada 2 hal penting yang kalau nggak dibenerin, pipeline bakal **stuck “Pending”** atau nanti **deploy gagal**.

***

### 1) Pipeline “Pending” karena TAG runner tidak cocok

Di `.gitlab-ci.yml` kamu pakai:

```yml
default:
  tags: ["vm2"]
```

Tapi di UI GitLab runner kamu kelihatan tag-nya **`docker`** (bukan `vm2`). Jadi job **nggak nemu runner** yang matching.

**Solusi (pilih salah satu):**

#### Opsi A (disarankan): Tambah tag `vm2` di runner

GitLab → **Settings → CI/CD → Runners → klik ikon pensil (Edit)** → bagian **Tags**:

* tambahkan: `vm2`
* boleh tetap simpan `docker` juga
* save

> Setelah itu, pipeline pending akan langsung ke-pick runner.

#### Opsi B: Ubah `.gitlab-ci.yml` ikut tag runner (`docker`)

Kalau mau cepat:

```yml
default:
  tags: ["docker"]
```

***

### 2) Value GitLab Variables: DEV\_MYSQL\_ROOT\_PASSWORD, DEV\_MYSQL\_PASSWORD, DEV\_APP\_KEY

Isi **harus sama konsepnya** dengan `.env.dev.compose` di VM2.

Karena kamu sebelumnya pakai:

* `MYSQL_ROOT_PASSWORD=RootPassDev123!`
* `MYSQL_PASSWORD=UserPassDev123!`

Maka di GitLab CI/CD Variables isi seperti ini:

* `DEV_MYSQL_ROOT_PASSWORD` = `RootPassDev123!`
* `DEV_MYSQL_PASSWORD` = `UserPassDev123!`
* `DEV_APP_KEY` = **string base64 Laravel**, contoh yang kamu generate:
  * `base64:LUtk3bmVqbELgudTMT0us8hv5JFGlZP5cVAMQBHAkPM=`

> Lebih “Laravel-native” kalau generate pakai:\
> `php laravel/artisan key:generate --show`\
> tapi `openssl rand -base64 32` + prefix `base64:` juga aman untuk kebutuhan ini.

#### Penting: “Protected” Variables

Di screenshot variables kamu banyak yang **Protected**.\
Kalau branch `main` kamu **tidak** protected, job `main` bisa **nggak dapat variable** (jadinya deploy/build gagal).

**Solusinya pilih salah satu:**

* Protect branch `main` (Settings → Repository → Protected branches), **atau**
* Uncheck “Protected” pada variables yang dipakai pipeline main (DEV\_\* dan HARBOR\_\*).

***

### 3) Sebelum deploy pertama: stop stack yang sudah jalan di port 80/443

Di `docker ps` kamu ada:\
`three-body-problem-main-nginx-dev-1` yang binding **80 & 443**.

Pipeline deploy kamu akan bikin stack baru di folder `/home/gitlab-runner/threebody-deploy` yang juga map **80 & 443** → bakal error: **address already in use**.

Jadi sebelum kamu biarkan pipeline deploy jalan, jalankan di VM2:

```bash
cd /home/cikal/three-body-problem-main
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml down
```

(Atau alternatifnya ubah port mapping di `docker-compose.dev.yml`, tapi lebih rapi: biarkan pipeline yang jadi “source of truth”.)

***

### 4) Biar setelah VM2 reboot semuanya tetap aktif

Ini sebenarnya sudah aman **kalau Docker service start otomatis** dan container punya restart policy (punya kamu: `restart: unless-stopped`, Harbor juga default restart).

Jalankan sekali saja:

```bash
sudo systemctl enable --now docker
sudo systemctl enable --now gitlab-runner
```

Setelah itu saat VM reboot, Docker naik → container yang restart policy akan ikut hidup lagi.

***

### 5) Satu update kecil yang saya sarankan di `.gitlab-ci.yml` (biar pull Harbor nggak 401)

Kalau project Harbor kamu **private**, deploy job perlu login dulu sebelum `docker compose pull`.

Tambahkan di `deploy_dev` sebelum `docker compose pull`:

```bash
echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
```

***

Kalau kamu sudah:

1. tambah tag `vm2` di runner **atau** ubah tag di CI,
2. pastikan Protected variables sesuai branch,
3. stop stack lama yang pegang 80/443,

langsung **retry pipeline** — harusnya job `validate → build_* → deploy_dev` jalan mulus.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000ca9471fa9991051fe9bf65da\&ts=490475\&p=fs\&cid=1\&sig=578b0565d9500aee640c561ca45c9aae9cf6c81448a619f78b3026e9ee5bd331\&v=0)tidak bisa di retry karena pending, dan mungkin harus di pushSelesai berpikir selama 2m 9s

Penyebab **pipeline “Pending”** itu hampir pasti karena **tidak ada runner yang “match” tag job**.

Di `.gitlab-ci.yml` kamu sudah pakai:

```yaml
default:
  tags: ["vm2"]
```

Tapi di screenshot **Settings → CI/CD → Runners**, runner yang assigned ke project kamu **tag-nya “docker”** (badge biru “docker”). Jadi GitLab _tidak menemukan runner bertag `vm2`_ → job jadi pending terus.

### Perbaikan (tanpa push lagi)

#### Opsi A (paling bagus): Tambahkan tag `vm2` ke runner yang sudah ada

1. Masuk **Project → Settings → CI/CD → Runners**
2. Di runner yang “Online”, klik **ikon pensil (Edit)**
3. Pada **Tags**, tambahkan `vm2` (biarkan `docker` boleh tetap ada)
4. **Save changes**

Kalau runner sudah punya tag `vm2`, **job yang pending itu biasanya langsung jalan otomatis** (nggak perlu retry / push).

> Catatan: kalau di `.gitlab-ci.yml` tags cuma `["vm2"]`, runner minimal harus punya tag `vm2`. Tag lain boleh ada.

#### Opsi B: Ubah `.gitlab-ci.yml` ikut tag runner (butuh push)

Kalau kamu nggak mau ubah tag runner, ubah tags jadi `["docker"]`, commit, push.

***

### Satu hal penting biar next step nggak gagal: “Protected variables”

Di screenshot Variables kamu banyak yang **Protected + Masked** (DEV\_APP\_KEY, DEV\_MYSQL\__, HARBOR\__).

Kalau branch `main` kamu **belum “Protected branch”**, maka job di `main` **TIDAK akan dapat value variable protected** → nanti job build/deploy gagal dengan error variabel kosong.

Solusi (pilih salah satu):

* **Recommended:** Protect branch `main`\
  Settings → Repository → **Protected branches** → protect `main`
* Atau: **uncheck “Protected”** untuk variable yang dibutuhkan pipeline

***

### Jawaban value GitLab Variables kamu

Isi pakai value real yang mau dipakai di environment DEV:

* **DEV\_MYSQL\_ROOT\_PASSWORD** = password root MySQL untuk container `mysql`\
  Contoh: `RootPassDev123!` (lebih baik ganti yang lebih kuat)
* **DEV\_MYSQL\_PASSWORD** = password user `threebody` (MYSQL\_PASSWORD)\
  Contoh: `UserPassDev123!` (lebih baik ganti)
* **DEV\_APP\_KEY** = APP\_KEY Laravel format `base64:...` **(copy persis)**\
  Kamu sudah generate dan valid, misalnya:\
  `base64:LUtk3bmVqbELgudTMT0us8hv5JFGlZP5cVAMQBHAkPM=`

***

Kalau kamu sudah **edit tag runner jadi ada `vm2`**, refresh halaman pipeline: harusnya status pending berubah jadi running. Jika masih pending, kirim screenshot halaman **job validate** (yang pending) biar aku cek “Tags required” / “protected branch only” / runner paused.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000003b5871fa9b11fa4d5115d7ea\&ts=490475\&p=fs\&cid=1\&sig=6e0b6119a3e75d671a0002e7a20864d0cf5435cf6881f8d3bd2106099b54a385\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000004a0072098d7bad14169c809e\&ts=490475\&p=fs\&cid=1\&sig=57f11a6db6ee34a7662b999b15b2092f107bb3f16ee9322ef40dab24a44d2891\&v=0)Running with gitlab-runner 18.6.5 (5908bc19) on DevOps\_43\_runner P5qLStHlf, system ID: s\_7bd6fa72a2b0 Preparing the "docker" executor 02:55 Using default image Using Docker executor with image docker:latest ... Using default image Using effective pull policy of \[always] for container docker:latest Pulling docker image docker:latest ... Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... Preparing environment 00:04 Using effective pull policy of \[always] for container sha256:54e6bb7e83797064b15abaf06223c9083f9e7b807d7046c13920150573a66d97 Running on runner-p5qlsthlf-project-76926856-concurrent-0 via devops-ci... Getting source from Git repository 00:09 Gitaly correlation ID: 9adb786221b6aa09-ATL Fetching changes with git depth set to 20... Initialized empty Git repository in /builds/cikalfarid/three-body-problem/.git/ Created fresh repository. Checking out 468b51ce as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:36 Using default image Using effective pull policy of \[always] for container docker:latest Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... $ set -euo pipefail $ docker version Client: Version: 29.1.3 API version: 1.52 Go version: go1.25.5 Git commit: f52814d Built: Fri Dec 12 14:48:47 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null time="2025-12-14T06:01:47Z" level=warning msg="The \\"APP\_KEY\\" variable is not set. Defaulting to a blank string." $ set -euo pipefail # collapsed multi-line command Unable to find image 'alpine:3.19' locally 3.19: Pulling from library/alpine Digest: sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1 Status: Downloaded newer image for alpine:3.19 ......+.+.........+..+.......+...+..................+..+...+.+......+..+.+.........+..+.+........+.+........+...+...+....+..+.+.....+...+.+++++++++++++++++++++++++++++++++++++++\*..........+++++++++++++++++++++++++++++++++++++++\*...+.......+......+..+..........+.....+.......+..+.+........+......+....+..+.......+..+..........+...+....................+...+.......+........+...................+...+...+..+......+.+...............+..............+.+.....+.+...+.........+..+.+..............+...+.+.........+...+........+....+......+...............+..+.........+.+..+...+.............+.....+......+...............+.........+.......+..++++++ ........+...+.+......+........+.+......+......+++++++++++++++++++++++++++++++++++++++\*..+....+.....+.......+++++++++++++++++++++++++++++++++++++++\*................+.....+......+.............+..+.......+......++++++ ----- docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error mounting "/builds/cikalfarid/three-body-problem/devops/nginx/frontend.conf" to rootfs at "/etc/nginx/conf.d/default.conf": mount src=/builds/cikalfarid/three-body-problem/devops/nginx/frontend.conf, dst=/etc/nginx/conf.d/default.conf, dstFd=/proc/thread-self/fd/11, flags=MS\_BIND|MS\_REC: not a directory: Are you trying to mount a directory onto a file (or vice-versa)? Check if the specified host path exists and is the expected type Run 'docker run --help' for more information Cleaning up project directory and file based variables 00:03 ERROR: Job failed: exit code 125 sekarang sudah tidak pending tapi error bantu perbaikiSelesai berpikir selama 4m 54s

Error itu karena **`docker run -v $CI_PROJECT_DIR/...` bind-mount** dilakukan ke **Docker daemon** yang dipakai runner. Di runner “docker executor”, daemon sering **tidak melihat file** di path itu (atau dianggap tidak ada), lalu Docker **membuat folder** bernama `frontend.conf` → akhirnya waktu mau di-mount ke file `/etc/nginx/conf.d/default.conf` keluarlah error **“not a directory”**.

Solusi paling aman: **jangan bind-mount file repo untuk test nginx**. Pakai `docker cp` + `docker exec` (nggak butuh host path sama sekali). Update job `validate` jadi seperti ini:

```yaml
validate:
  stage: validate
  script:
    # biar tidak warning APP_KEY kosong saat compose config
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="

    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null

    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      # dummy upstream biar hostname frontend/laravel/goapi resolve
      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      # container untuk test nginx
      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      # siapkan folder, copy config TANPA bind-mount
      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      # generate cert di dalam container (tanpa mount)
      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      # test nginx config
      docker exec "$NG" nginx -t

      # cleanup
      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true
```

Lalu push:

```bash
git add .gitlab-ci.yml
git commit -m "Fix validate nginx test (no bind mounts)"
git push origin main
```

Catatan cepat (biar nggak ketipu nanti):

* Karena CI Variables kamu **Protected**, pastikan branch `main` kamu **Protected branch** juga, kalau tidak job `build/deploy` bisa gagal karena variabelnya “kosong”.

Kalau sudah kamu push, kirim log job `validate` terbaru—kalau lolos, kita lanjut beresin build/push ke Harbor + deploy-nya.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000008fb471fa91014f8424b75974\&ts=490475\&p=fs\&cid=1\&sig=15e517783eb9036bf7f162ec0591bda8b6622bda49fa31a24d9aedecdac7c16e\&v=0)Running with gitlab-runner 18.6.5 (5908bc19) on DevOps\_43\_runner P5qLStHlf, system ID: s\_7bd6fa72a2b0 Preparing the "docker" executor 00:16 Using default image Using Docker executor with image docker:latest ... Using default image Using effective pull policy of \[always] for container docker:latest Pulling docker image docker:latest ... Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... Preparing environment 00:04 Using effective pull policy of \[always] for container sha256:54e6bb7e83797064b15abaf06223c9083f9e7b807d7046c13920150573a66d97 Running on runner-p5qlsthlf-project-76926856-concurrent-0 via devops-ci... Getting source from Git repository 00:09 Gitaly correlation ID: 9adb8d2467b68740-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /builds/cikalfarid/three-body-problem/.git/ Created fresh repository. Checking out 3c6218ba as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:52 Using default image Using effective pull policy of \[always] for container docker:latest Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... $ set -euo pipefail $ docker version Client: Version: 29.1.3 API version: 1.52 Go version: go1.25.5 Git commit: f52814d Built: Fri Dec 12 14:48:47 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'. Configure a credential helper to remove this warning. See https://docs.docker.com/go/credential-store/ Login Succeeded #0 building with "default" instance using docker driver #1 \[internal] load build definition from Dockerfile.laravel #1 transferring dockerfile: #1 transferring dockerfile: 1.32kB 0.1s done #1 DONE 0.7s #2 resolve image config for docker-image://docker.io/docker/dockerfile:1 #2 DONE 26.1s #3 docker-image://docker.io/docker/dockerfile:1@sha256:b6afd42430b15f2d2a4c5a02b919e98a525b785b1aaff16747d2f623364e39b6 #3 resolve docker.io/docker/dockerfile:1@sha256:b6afd42430b15f2d2a4c5a02b919e98a525b785b1aaff16747d2f623364e39b6 #3 resolve docker.io/docker/dockerfile:1@sha256:b6afd42430b15f2d2a4c5a02b919e98a525b785b1aaff16747d2f623364e39b6 0.6s done #3 DONE 0.6s #3 docker-image://docker.io/docker/dockerfile:1@sha256:b6afd42430b15f2d2a4c5a02b919e98a525b785b1aaff16747d2f623364e39b6 #3 CACHED #4 \[internal] load metadata for docker.io/library/php:8.2-apache #4 ... #5 \[internal] load metadata for docker.io/library/php:8.2-cli #5 ... #6 \[internal] load metadata for docker.io/library/composer:2 #6 ERROR: failed to do request: Head "https://registry-1.docker.io/v2/library/composer/manifests/2": net/http: TLS handshake timeout #4 \[internal] load metadata for docker.io/library/php:8.2-apache #4 CANCELED #5 \[internal] load metadata for docker.io/library/php:8.2-cli #5 CANCELED ------ > \[internal] load metadata for docker.io/library/composer:2: ------ Dockerfile.laravel:12 -------------------- 10 | && rm -rf /var/lib/apt/lists/\* 11 | 12 | >>> COPY --from=composer:2 /usr/bin/composer /usr/bin/composer 13 | 14 | COPY laravel/composer.json laravel/composer.lock ./ -------------------- ERROR: failed to build: failed to solve: failed to do request: Head "https://registry-1.docker.io/v2/library/composer/manifests/2": net/http: TLS handshake timeout Cleaning up project directory and file based variables 00:04 ERROR: Job failed: exit code 1 peningkatan yang luar biasa, namun bantu perbaiki yaaSelesai berpikir selama 1m 44s

Mantap — **validate sudah hijau**, build `goapi` & `frontend` juga lolos. Sekarang yang gagal **khusus build\_laravel** karena:

> `failed to do request ... composer:2 ... TLS handshake timeout`

Itu artinya runner kamu **lagi timeout akses Docker Hub** untuk image `composer:2` (bukan error Dockerfile/CI logic). Cara paling stabil: **hilangkan dependency `composer:2`** dari Dockerfile laravel, supaya build laravel **nggak perlu pull image composer**.

***

### Fix paling stabil: ubah `devops/docker/Dockerfile.laravel`

Ganti bagian vendor stage jadi seperti ini (intinya: install composer via `curl` di dalam image php, bukan `COPY --from=composer:2`):

```dockerfile
# syntax=docker/dockerfile:1

FROM php:8.2-cli AS vendor
WORKDIR /app
ENV COMPOSER_ALLOW_SUPERUSER=1

RUN apt-get update \
 && apt-get install -y --no-install-recommends git unzip libzip-dev curl ca-certificates \
 && docker-php-ext-install zip \
 && rm -rf /var/lib/apt/lists/*

# install composer TANPA image composer:2
RUN curl -sS https://getcomposer.org/installer | php -- \
    --install-dir=/usr/local/bin --filename=composer

COPY laravel/composer.json laravel/composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction --no-progress --no-scripts

FROM php:8.2-apache

RUN apt-get update \
 && apt-get install -y --no-install-recommends libzip-dev unzip \
 && docker-php-ext-install pdo pdo_mysql zip \
 && rm -rf /var/lib/apt/lists/* \
 && a2enmod rewrite \
 && echo "ServerName localhost" > /etc/apache2/conf-available/servername.conf \
 && a2enconf servername

WORKDIR /var/www/html
COPY laravel/ ./
COPY --from=vendor /app/vendor ./vendor

RUN sed -i 's|DocumentRoot /var/www/html|DocumentRoot /var/www/html/public|g' /etc/apache2/sites-available/000-default.conf \
 && printf '\n<Directory "/var/www/html/public">\n\tAllowOverride All\n\tRequire all granted\n</Directory>\n' >> /etc/apache2/apache2.conf \
 && chown -R www-data:www-data storage bootstrap/cache || true \
 && chmod -R 775 storage bootstrap/cache || true

EXPOSE 80
CMD ["apache2-foreground"]
```

Lalu push:

```bash
git add devops/docker/Dockerfile.laravel
git commit -m "Fix laravel build: install composer without composer:2 image"
git push origin main
```

***

### Tambahan biar makin tahan banting (opsional tapi recommended)

Tambahkan retry di job `build_laravel` (kadang download dependency juga bisa flaky):

```yaml
build_laravel:
  stage: build
  retry: 2
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"
```

***

Kalau sudah kamu push, pipeline harus lanjut sampai `deploy_dev` jalan.\
Kalau masih gagal, kirim **log job build\_laravel** yang terbaru (bagian error paling bawah).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000b3807209b38376f730a47e49\&ts=490475\&p=fs\&cid=1\&sig=9dac007704e2f88f84d47553223012d4a99d50c737c60c805b400a8b910fbcf7\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000dcc07209bf0e0360501fca1b\&ts=490475\&p=fs\&cid=1\&sig=5f708fd0803ee249ccf1345235a4b59fe3b3638e2c800369b7735225db94bc05\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000ce1c7209a06af3dc6d1ba2ea\&ts=490475\&p=fs\&cid=1\&sig=b718f1c2542798b95fcf59b539cfa23173397425e46bcc38f71a8b48843f081d\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000011647209be338170c06ed507\&ts=490475\&p=fs\&cid=1\&sig=0b6c2c75f25e5b873bbf7c0235af2380f6bd6ad39730da0a0e68bef8869fb348\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000072e07209a7796c739d79a556\&ts=490475\&p=fs\&cid=1\&sig=23c5a4f73addfaee6e435431e1b24e8e20c77e066d18ffbc224e4e8bddaea445\&v=0)mantap, tapi bantu perbaiki ya deploy\_dev nya dan ketika di hit masih error juga seperti di gambar Running with gitlab-runner 18.6.5 (5908bc19) on DevOps\_43\_runner P5qLStHlf, system ID: s\_7bd6fa72a2b0 Preparing the "docker" executor 00:12 Using default image Using Docker executor with image docker:latest ... Using default image Using effective pull policy of \[always] for container docker:latest Pulling docker image docker:latest ... Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... Preparing environment 00:02 Using effective pull policy of \[always] for container sha256:54e6bb7e83797064b15abaf06223c9083f9e7b807d7046c13920150573a66d97 Running on runner-p5qlsthlf-project-76926856-concurrent-0 via devops-ci... Getting source from Git repository 00:08 Gitaly correlation ID: 9adbd069d6cc7d66-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /builds/cikalfarid/three-body-problem/.git/ Created fresh repository. Checking out 10aa9f11 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:18 Using default image Using effective pull policy of \[always] for container docker:latest Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... $ set -euo pipefail $ docker version Client: Version: 29.1.3 API version: 1.52 Go version: go1.25.5 Git commit: f52814d Built: Fri Dec 12 14:48:47 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command ..+...+....+.....+...+..........+........+.......+..+.......+.....+...+...+...............+.............+......+.....+.+++++++++++++++++++++++++++++++++++++++\*....+.....+......+....+.....+......+...+++++++++++++++++++++++++++++++++++++++\*.....+.+......+......+..+......+............+.........+......+...+.+.....+.........+.......+...+.........+..+......+............+...+.........+.+...++++++ ......+........+....+...+++++++++++++++++++++++++++++++++++++++\*..+...+......+++++++++++++++++++++++++++++++++++++++\*...+..+.+.................+.+...............+.....+........................+.+..+.......+...+..+......+......................+.....+....+.........+.....+....+...........+....+............+.....+...+..........+........+...+.+...+...+...+..+............................+.....+....+..................+...+............+..+.+..+.........+.+.....+...+.+...........+.........+..........+.....+...+...+..........+..............+....+......+.....................+..+...+...+....+..+.........+....+...+..+.+...........+...+......................+...+.....+.+.....+.......+...+..+....+...+.....+...+....+...........+.......+.....+.+..+....+.....+....+......+...+......+.....+....+..+.+..+......+.++++++ ----- WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'. Configure a credential helper to remove this warning. See https://docs.docker.com/go/credential-store/ Login Succeeded Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/laravel:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348 Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348 Pulling Image \[MASKED]/\[MASKED]/frontend:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348 Pulling Image \[MASKED]/\[MASKED]/goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348 Error failed to resolve reference "\[MASKED]/\[MASKED]/goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348": \[MASKED]/\[MASKED]/goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348: not found Image nginx:alpine Interrupted Image mysql:8.0 Interrupted Image \[MASKED]/\[MASKED]/frontend:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348 Interrupted Image \[MASKED]/\[MASKED]/laravel:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348 Interrupted Error response from daemon: failed to resolve reference "\[MASKED]/\[MASKED]/goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348": \[MASKED]/\[MASKED]/goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348: not found Cleaning up project directory and file based variables 00:02 ERROR: Job failed: exit code 1Selesai berpikir selama 6m 20s

Sip, sekarang problem utamanya **deploy\_dev gagal pull image** karena **TAG yang dipakai build ≠ TAG yang dipakai saat deploy/compose**.

Di log deploy dia nyari:

`goapi:10aa9f1175ed67edf1fb4dc9e85aa1aa4ad27348` _(full SHA)_

tapi build kamu sebelumnya nge-push pakai:

`$CI_COMMIT_SHORT_SHA` _(8 char)_

Jadi image “full SHA” memang **tidak ada** di Harbor → `not found`.

### 1) Fix deploy\_dev “image not found” (paling simpel & rapi)

Samakan semuanya pakai **full commit SHA** (biar nggak konflik sama variable built-in GitLab juga).

Ubah di `.gitlab-ci.yml`:

#### Build jobs (goapi/laravel/frontend)

Ganti:

```bash
TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
```

menjadi:

```bash
TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
```

#### deploy\_dev job

Ganti juga:

```bash
TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
```

menjadi:

```bash
TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
```

Setelah itu commit & push. Pipeline berikutnya akan build+push image dengan tag full SHA, dan deploy bisa pull.

> Alternatif lain (kalau kamu mau tetap short SHA): di deploy sebelum `docker compose pull` lakukan `export CI_COMMIT_SHA="$CI_COMMIT_SHORT_SHA"` supaya compose pakai short, tapi opsi “full SHA everywhere” lebih aman.

***

### 2) “Hit API 500” di dev.local (setelah deploy jalan)

Dari screenshot, itu halaman default **nginx 500**, biasanya karena **upstream (laravel/goapi) error runtime** (DB nggak siap / env nggak kebaca / table belum ada).

Langkah cek cepat di VM2 (host tempat container jalan):

```bash
cd ~/threebody-deploy
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 laravel
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 goapi
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 nginx-dev
```

Kalau Laravel biasanya paling jelas di:

```bash
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel ls -la storage/logs
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel tail -n 200 storage/logs/laravel.log
```

#### Penyebab paling sering (dan fix yang biasanya perlu)

Pastikan `docker-compose.dev.yml` benar-benar **ngasih env ke container**, bukan cuma buat substitusi tag image.

Minimal untuk `mysql` + `laravel` (contoh pola yang benar):

```yaml
mysql:
  environment:
    MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    MYSQL_DATABASE: threebody
    MYSQL_USER: threebody
    MYSQL_PASSWORD: ${MYSQL_PASSWORD}

laravel:
  environment:
    APP_KEY: ${APP_KEY}
    DB_CONNECTION: mysql
    DB_HOST: mysql
    DB_PORT: 3306
    DB_DATABASE: threebody
    DB_USERNAME: threebody
    DB_PASSWORD: ${MYSQL_PASSWORD}
```

Untuk `goapi`, cek variabel env yang dipakai di source (di repo):

```bash
grep -R "Getenv" -n go/ || true
grep -R "DB_" -n go/ || true
```

Lalu mapping-kan ke service `goapi:` di compose.

***

### 3) Pastikan CI/CD Variables bener (Protected/Masked)

Di screenshot kamu, `DEV_*` kamu **Protected**. Kalau branch `main` kamu **belum “protected branch”**, maka variable Protected **nggak akan kebawa ke job** → bisa bikin APP\_KEY/DB password kosong → backend 500.

Jadi pastikan salah satu:

* **Protect branch `main`** di repo GitLab, **atau**
* **Uncheck “Protected”** di `DEV_APP_KEY`, `DEV_MYSQL_ROOT_PASSWORD`, `DEV_MYSQL_PASSWORD` (biar jalan di main biasa)

Dan untuk value:

* `DEV_APP_KEY`: pakai yang kamu generate itu ✅\
  `base64:LUtk3bmVqbELgudTMT0us8hv5JFGlZP5cVAMQBHAkPM=`
* Password kalau mau **Masked**, hindari karakter aneh (misal `!`). Paling aman:
  * `openssl rand -hex 24` untuk password root & user.

***

Kalau kamu sudah commit perubahan TAG (full SHA) dan pipeline deploy sudah sukses, kirim **log `docker compose logs --tail=200 laravel` dan `goapi`** pas kamu hit endpoint—nanti aku tunjukin fix tepatnya untuk 500-nya (biasanya tinggal beresin DB env/migrate/seed).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000000f447209b266855c2199cefd\&ts=490475\&p=fs\&cid=1\&sig=ac926c73865843be610b2fee4ae99b367fd678792feb83199ea866c91ff999ea\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000005c6c72099e8fcc55d207d48e\&ts=490475\&p=fs\&cid=1\&sig=4cfc0e4050149e30eac5b2207335efa35afeb7fe5defd4eda047c1e7042cad7b\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000d3187209816a27a8d2af2e67\&ts=490475\&p=fs\&cid=1\&sig=0cda082367c90547e991bfb0d2f0000fb5fdb322d3c3ebdc82889c1a82d4508a\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000005f07209907fd353365f43dd\&ts=490475\&p=fs\&cid=1\&sig=3bfbe7ea9c613545ef17f9ca69ad29663e9ec4abc625a5355adfabed9d455e4c\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000002cf4720987e20721da79a8c2\&ts=490475\&p=fs\&cid=1\&sig=bcfba500e9f11181b3b67e86f842f1a01100012b8a036a108730522b7f805e74\&v=0)Running with gitlab-runner 18.6.5 (5908bc19) on vm2-shell-runner P5qLStHlf, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on devops-ci... Getting source from Git repository 00:02 Gitaly correlation ID: 9adbf202312cea82-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/P5qLStHlf/0/cikalfarid/three-body-problem/.git/ Checking out 5a95919b as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 01:03 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command ......+...+.........+...+...+.+...+......+...........+...+...+...+.+....................+......+.......+..+.+...+..+............+......+++++++++++++++++++++++++++++++++++++++\*.+....+........+++++++++++++++++++++++++++++++++++++++\*.......+.+..............+...+.......+......+..+...+...+.+.....+..............................................+.........+......+...+.....+......+...+......+....+...+..+......+...................+...+..+.+.........+.....+.+.................+....+...+..................+..+.+..+...............+...+.......+...+..+........................+...+.+......+.........+..+.............+......+........+.......+...+..+.........+.+.....+.+...+.............................+.+.........+...+..+.+..++++++ .....+.........+......+++++++++++++++++++++++++++++++++++++++\*..+..+..........+........+.+......+........+....+...........+.+...+..+.........+....+..+++++++++++++++++++++++++++++++++++++++\*....+.+.....+....+.........+...+.........+...........+.......+...+...+...............+.....+.+...........+.........+.........+.......+..+.+..+...+.+...........+...............+...+......+...................+.....+.......+..+.............+..+......+......+.........+.+...........+...+.+....................+.+...+......+...+..+...+.+..++++++ ----- Login Succeeded Image \[MASKED]/\[MASKED]/goapi:5a95919b855ffdb4012bfbfcf682a2d4f19091b6 Pulling Image \[MASKED]/\[MASKED]/frontend:5a95919b855ffdb4012bfbfcf682a2d4f19091b6 Pulling Image mysql:8.0 Pulling Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/laravel:5a95919b855ffdb4012bfbfcf682a2d4f19091b6 Pulling Image \[MASKED]/\[MASKED]/goapi:5a95919b855ffdb4012bfbfcf682a2d4f19091b6 Pulled Image \[MASKED]/\[MASKED]/frontend:5a95919b855ffdb4012bfbfcf682a2d4f19091b6 Pulled Image \[MASKED]/\[MASKED]/laravel:5a95919b855ffdb4012bfbfcf682a2d4f19091b6 Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-deploy\_devnet Creating Network \[MASKED]-deploy\_devnet Created Volume \[MASKED]-deploy\_mysql\_data Creating Volume \[MASKED]-deploy\_mysql\_data Created Container \[MASKED]-deploy-mysql-1 Creating Container \[MASKED]-deploy-mysql-1 Created Container \[MASKED]-deploy-goapi-1 Creating Container \[MASKED]-deploy-laravel-1 Creating Container \[MASKED]-deploy-goapi-1 Created Container \[MASKED]-deploy-laravel-1 Created Container \[MASKED]-deploy-frontend-1 Creating Container \[MASKED]-deploy-frontend-1 Created Container \[MASKED]-deploy-nginx-dev-1 Creating Container \[MASKED]-deploy-nginx-dev-1 Created Container \[MASKED]-deploy-mysql-1 Starting Container \[MASKED]-deploy-mysql-1 Started Container \[MASKED]-deploy-mysql-1 Waiting Container \[MASKED]-deploy-mysql-1 Waiting Container \[MASKED]-deploy-mysql-1 Healthy Container \[MASKED]-deploy-laravel-1 Starting Container \[MASKED]-deploy-mysql-1 Healthy Container \[MASKED]-deploy-goapi-1 Starting Container \[MASKED]-deploy-goapi-1 Started Container \[MASKED]-deploy-laravel-1 Started Container \[MASKED]-deploy-frontend-1 Starting Container \[MASKED]-deploy-frontend-1 Started Container \[MASKED]-deploy-nginx-dev-1 Starting Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint \[MASKED]-deploy-nginx-dev-1 (b9ea0d1e6e263dca901a0a7439f321329a8dae284b4ce6a409cafcffd2988769): Bind for 0.0.0.0:80 failed: port is already allocated Cleaning up project directory and file based variables 00:00 ERROR: Job failed: exit status 1 cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ git add . cikal@devops-ci:\~/three-body-problem-main$ git commit -m "Fix laravel build: install composer without composer:2 image" \[main 5a95919] Fix laravel build: install composer without composer:2 image 1 file changed, 3 insertions(+), 3 deletions(-) cikal@devops-ci:\~/three-body-problem-main$ git push Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 351 bytes | 351.00 KiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 10aa9f1..5a95919 main -> main cikal@devops-ci:\~/three-body-problem-main$ cikal@devops-ci:\~/three-body-problem-main$ cikal@devops-ci:\~/three-body-problem-main$ cd \~/threebody-deploy docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps docker compose --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 laravel docker compose --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 goapi docker compose --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 nginx-dev -bash: cd: /home/cikal/threebody-deploy: No such file or directory NAME IMAGE COMMAND SERVICE CREATED STATUS PORTS three-body-problem-main-frontend-1 192.168.56.43:8081/threebody/frontend:local "/docker-entrypoint.…" frontend 2 hours ago Up 2 hours 80/tcp three-body-problem-main-goapi-1 192.168.56.43:8081/threebody/goapi:local "/app/goapi" goapi 2 hours ago Up 2 hours 8080/tcp three-body-problem-main-laravel-1 192.168.56.43:8081/threebody/laravel:local "docker-php-entrypoi…" laravel 2 hours ago Up 2 hours 80/tcp three-body-problem-main-mysql-1 mysql:8.0 "docker-entrypoint.s…" mysql 2 hours ago Up 2 hours (healthy) 3306/tcp, 33060/tcp three-body-problem-main-nginx-dev-1 sha256:289decab414250121a93c3f1b8316b9c69906de3a4993757c424cb964169ad42 "/docker-entrypoint.…" nginx-dev 2 hours ago Up 2 hours 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp laravel-1 | \[Sun Dec 14 05:55:09.194930 2025] \[mpm\_prefork:notice] \[pid 1:tid 1] AH00163: Apache/2.4.65 (Debian) PHP/8.2.29 configured -- resuming normal operations laravel-1 | \[Sun Dec 14 05:55:09.195684 2025] \[core:notice] \[pid 1:tid 1] AH00094: Command line: 'apache2 -D FOREGROUND' goapi-1 | 2025/12/14 05:55:05 No .env file found, using default environment variables goapi-1 | 2025/12/14 05:55:05 Connecting to database: threebody@mysql:3306/threebody goapi-1 | 2025/12/14 05:55:05 Successfully connected to MySQL database goapi-1 | 2025/12/14 05:55:06 Products already exist in database goapi-1 | 2025/12/14 05:55:06 Go Products API Server is running on http://localhost:8080 goapi-1 | 2025/12/14 05:55:06 Endpoints: goapi-1 | 2025/12/14 05:55:06 GET /api/products - Get all products goapi-1 | 2025/12/14 05:55:06 GET /api/products/{id} - Get product by ID nginx-dev-1 | /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration nginx-dev-1 | /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/ nginx-dev-1 | /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh nginx-dev-1 | 10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf nginx-dev-1 | 10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf nginx-dev-1 | /docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh nginx-dev-1 | /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh nginx-dev-1 | /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh nginx-dev-1 | /docker-entrypoint.sh: Configuration complete; ready for start up nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: using the "epoll" event method nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: nginx/1.29.4 nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: built by gcc 15.2.0 (Alpine 15.2.0) nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: OS: Linux 6.8.0-88-generic nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: getrlimit(RLIMIT\_NOFILE): 1024:524288 nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: start worker processes nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: start worker process 31 nginx-dev-1 | 2025/12/14 05:55:12 \[notice] 1#1: start worker process 32 nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:05:55:33 +0000] "GET / HTTP/1.1" 200 644 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:05:55:33 +0000] "GET /static/css/main.fe6da703.css HTTP/1.1" 200 2950 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:05:55:33 +0000] "GET /static/js/main.c5b29507.js HTTP/1.1" 200 189571 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:05:55:33 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:02 +0000] "GET / HTTP/1.1" 200 644 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:02 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:49:03 \[warn] 32#32: \*12 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local", referrer: "https://dev.local/" nginx-dev-1 | 2025/12/14 06:49:03 \[error] 32#32: \*12 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local", referrer: "https://dev.local/" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:03 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:06 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:49:06 \[warn] 32#32: \*16 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:49:06 \[error] 32#32: \*16 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:06 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:08 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:49:08 \[warn] 31#31: \*18 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:49:08 \[error] 31#31: \*18 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:08 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:16 +0000] "GET / HTTP/1.1" 200 615 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:49:16 \[error] 31#31: \*23 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.56.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "192.168.56.43", referrer: "http://192.168.56.43/" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:49:16 +0000] "GET /favicon.ico HTTP/1.1" 404 555 "http://192.168.56.43/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:54:05 \[warn] 32#32: \*26 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:54:05 \[error] 32#32: \*26 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:05 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:06 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:54:08 \[warn] 32#32: \*28 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:54:08 \[error] 32#32: \*28 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:08 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:08 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:54:10 \[warn] 32#32: \*31 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:54:10 \[error] 32#32: \*31 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:10 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:10 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:11 +0000] "GET / HTTP/1.1" 200 644 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:11 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:54:13 \[warn] 31#31: \*34 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:54:13 \[error] 31#31: \*34 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:13 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:13 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:14 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:54:14 \[warn] 31#31: \*39 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:54:14 \[error] 31#31: \*39 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:54:14 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:43 \[warn] 32#32: \*45 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:43 \[error] 32#32: \*45 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:43 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:43 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:44 \[warn] 32#32: \*47 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:44 \[error] 32#32: \*47 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:44 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:44 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:45 \[warn] 31#31: \*50 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:45 \[error] 31#31: \*50 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:45 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:45 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:46 \[warn] 31#31: \*53 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:46 \[error] 31#31: \*53 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:46 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:46 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:46 \[warn] 31#31: \*56 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:46 \[error] 31#31: \*56 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:46 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:46 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:47 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:47 \[warn] 31#31: \*59 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:47 \[error] 31#31: \*59 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:47 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:47 \[warn] 31#31: \*62 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:47 \[error] 31#31: \*62 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:47 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:47 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:48 \[warn] 31#31: \*65 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:48 \[error] 31#31: \*65 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:48 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:48 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:48 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:48 \[warn] 32#32: \*68 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:48 \[error] 32#32: \*68 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:48 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:48 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:56:48 \[warn] 32#32: \*71 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:56:48 \[error] 32#32: \*71 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:48 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:50 +0000] "GET / HTTP/1.1" 200 644 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:50 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:50 +0000] "GET / HTTP/1.1" 200 644 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:56:50 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:27 \[warn] 32#32: \*81 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:27 \[error] 32#32: \*81 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:27 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:27 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:27 \[warn] 31#31: \*83 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:27 \[error] 31#31: \*83 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:27 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:27 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:28 \[warn] 31#31: \*86 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:28 \[error] 31#31: \*86 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:28 \[warn] 32#32: \*89 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:28 \[error] 32#32: \*89 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:28 \[warn] 31#31: \*92 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:28 \[error] 31#31: \*92 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:28 \[warn] 32#32: \*95 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:28 \[error] 32#32: \*95 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:28 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 06:58:29 \[warn] 32#32: \*98 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 06:58:29 \[error] 32#32: \*98 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:29 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:29 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/laravel/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:30 +0000] "GET / HTTP/1.1" 200 644 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:06:58:30 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:07:00:46 +0000] "GET /api/laravel/api/products HTTP/1.1" 500 579 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 07:00:46 \[warn] 32#32: \*106 using uninitialized "up\_laravel" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local", referrer: "https://dev.local/" nginx-dev-1 | 2025/12/14 07:00:46 \[error] 32#32: \*106 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/laravel/api/products HTTP/1.1", host: "dev.local", referrer: "https://dev.local/" nginx-dev-1 | 2025/12/14 07:01:03 \[warn] 31#31: \*108 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 2025/12/14 07:01:03 \[error] 31#31: \*108 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:07:01:03 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:07:01:03 +0000] "GET /favicon.ico HTTP/1.1" 200 3870 "https://dev.local/api/go/api/products" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" nginx-dev-1 | 2025/12/14 07:01:06 \[warn] 31#31: \*110 using uninitialized "up\_go" variable, client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local", referrer: "https://dev.local/" nginx-dev-1 | 2025/12/14 07:01:06 \[error] 31#31: \*110 invalid URL prefix in "http://", client: 192.168.56.1, server: dev.local, request: "GET /api/go/api/products HTTP/1.1", host: "dev.local", referrer: "https://dev.local/" nginx-dev-1 | 192.168.56.1 - - \[14/Dec/2025:07:01:06 +0000] "GET /api/go/api/products HTTP/1.1" 500 579 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" "-" cikal@devops-ci:\~/three-body-problem-main$ docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel ls -la storage/logs docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel tail -n 200 storage/logs/laravel.log total 12 drwxrwxr-x 1 www-data www-data 4096 Dec 9 04:22 . drwxrwxr-x 1 www-data www-data 4096 Dec 9 04:22 .. -rwxrwxr-x 1 www-data www-data 14 Dec 9 04:22 .gitignore tail: cannot open 'storage/logs/laravel.log' for reading: No such file or directory cikal@devops-ci:\~/three-body-problem-main$ grep -R "Getenv" -n go/ || true grep -R "DB\_" -n go/ || true go/main.go:44: if value := os.Getenv(key); value != "" { go/main.go:51: driver = getEnv("DB\_DRIVER", "mysql") go/main.go:52: user = getEnv("DB\_USER", "root") go/main.go:53: password = getEnv("DB\_PASSWORD", "") go/main.go:54: name = getEnv("DB\_NAME", "laravel") go/main.go:55: host = getEnv("DB\_HOST", "127.0.0.1") go/main.go:56: port = getEnv("DB\_PORT", "3306") go/.env.example:2:DB\_DRIVER=mysql go/.env.example:3:DB\_HOST=127.0.0.1 go/.env.example:4:DB\_PORT=3306 go/.env.example:5:DB\_USER=root go/.env.example:6:DB\_PASSWORD= go/.env.example:7:DB\_NAME=laravel cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["docker"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' # test nginx config docker exec "$NG" nginx -t # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHORT\_SHA}" DEPLOY\_DIR="$HOME/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # generate cert (persist di host, bukan di repo) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # tulis env deploy dari CI Variables cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin docker compose --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # tunggu laravel siap, lalu migrate for i in $(seq 1 30); do if docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php -v >/dev/null 2>&1; then break fi sleep 2 done docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 7m 10s

Siap—sekarang masalahnya **2** dan keduanya sudah kelihatan jelas dari log yang kamu kirim:

1. **`deploy_dev` gagal** karena **port 80 sudah dipakai**\
   `Bind for 0.0.0.0:80 failed: port is already allocated` → di VM kamu sudah ada container nginx lama yang publish `80->80` (yang `three-body-problem-main-nginx-dev-1`), jadi ketika job deploy bikin stack baru, dia bentrok.
2. **Hit API dari `https://dev.local/...` dapat 500** karena **nginx config salah**\
   Log nginx kamu: `using uninitialized "up_laravel" variable` dan `invalid URL prefix in "http://"` → artinya `proxy_pass http://$up_laravel` / `http://$up_go` tapi variabelnya kosong, jadi nginx mencoba proxy ke `http://` (invalid) → 500.

***

### A) Fix 500 error (nginx proxy salah)

Edit file nginx yang dipakai untuk `server_name dev.local` (biasanya `devops/nginx/dev.conf` atau `devops/nginx/frontend.conf`). Targetnya: **hapus pemakaian `$up_laravel` / `$up_go`** dan langsung proxy ke service name docker-compose.

Pakai contoh config ini (aman, simpel, dan cocok dengan path React kamu):

```nginx
# devops/nginx/dev.conf (atau file yang berisi server dev.local)

upstream laravel_upstream { server laravel:80; }
upstream go_upstream      { server goapi:8080; }
upstream fe_upstream      { server frontend:80; }

server {
  listen 80;
  listen 443 ssl;
  server_name dev.local;

  ssl_certificate     /etc/nginx/certs/dev.crt;
  ssl_certificate_key /etc/nginx/certs/dev.key;

  # Frontend
  location / {
    proxy_pass http://fe_upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # React memanggil: /api/laravel/api/products  -> laravel menerima /api/products
  location /api/laravel/ {
    proxy_pass http://laravel_upstream/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # React memanggil: /api/go/api/products -> goapi menerima /api/products
  location /api/go/ {
    proxy_pass http://go_upstream/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

Lalu apply ke container yang lagi jalan:

```bash
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T nginx-dev nginx -t
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml restart nginx-dev
```

Tes cepat (kalau ini sukses, 500 akan hilang):

```bash
curl -k https://dev.local/api/go/api/products
curl -k https://dev.local/api/laravel/api/products
```

> Kalau setelah ini berubah jadi **502/504**, baru berarti backend-nya tidak reachable. Tes dari dalam nginx:\
> `docker compose ... exec -T nginx-dev sh -lc "apk add --no-cache curl >/dev/null; curl -sS http://goapi:8080/api/products; curl -sS http://laravel/api/products"`

***

### B) Fix `deploy_dev` (port 80 bentrok)

#### Penyebab

Job deploy kamu bikin stack baru (prefix container `...-deploy-...`) tapi stack lama (`three-body-problem-main-nginx-dev-1`) masih pakai port 80/443 → bentrok.

#### Solusi yang paling rapi

1. Paksa semua deploy pakai **project name yang tetap** (biar “stack”-nya sama, bukan bikin stack baru)
2. Jalankan `down` sebelum `up` (tanpa `-v`, jadi volume DB tetap aman)

Patch yang disarankan untuk bagian `deploy_dev` di `.gitlab-ci.yml`:

```yml
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail

      # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port)
      export COMPOSE_PROJECT_NAME="threebody-dev"

      # samakan TAG dengan build job (paling aman pakai FULL SHA)
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
      mkdir -p "$DEPLOY_DIR"

      # sync file deploy
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
      MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
      MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
      HARBOR_URL=${HARBOR_URL}
      HARBOR_PROJECT=${HARBOR_PROJECT}
      CI_COMMIT_SHA=${TAG}
      APP_KEY=${DEV_APP_KEY}
      ENV

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      # hentikan stack yang sama dulu (volume DB tetap aman)
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps
```

#### Yang perlu kamu lakukan sekali saja (biar port 80 kosong)

Karena saat ini stack lama masih jalan, stop dulu supaya pipeline berikutnya tidak bentrok:

```bash
# dari folder yang dulu kamu pakai (yang menghasilkan prefix three-body-problem-main-*)
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml down
```

***

### C) Kenapa kamu “nggak ketemu” folder deploy

Di job deploy kamu pakai `DEPLOY_DIR="$HOME/threebody-deploy"`. Karena runner shell jalan sebagai user `gitlab-runner`, foldernya ada di:

```bash
ls -la /home/gitlab-runner/threebody-deploy
```

Kalau kamu mau supaya gampang dicek dari user `cikal`, pindahkan DEPLOY\_DIR ke `/home/cikal/threebody-deploy` dan pastikan permission-nya:

```bash
sudo mkdir -p /home/cikal/threebody-deploy
sudo chown -R gitlab-runner:gitlab-runner /home/cikal/threebody-deploy
```

***

Kalau kamu lakukan **A (nginx config)** dulu, biasanya error 500 di UI langsung hilang. Setelah itu lanjut **B (deploy\_dev)** supaya pipeline deploy bisa jalan tanpa bentrok port.

cikal@devops-ci:\~/three-body-problem-main$ nano devops/nginx/dev.conf cikal@devops-ci:\~/three-body-problem-main$ cat devops/nginx/dev.conf limit\_req\_zone $binary\_remote\_addr zone=api\_limit:10m rate=10r/s; server { listen 80; server\_name dev.local; return 301 https://$host$request\_uri; } server { listen 443 ssl; server\_name dev.local; ssl\_certificate /etc/nginx/certs/dev.crt; ssl\_certificate\_key /etc/nginx/certs/dev.key; proxy\_set\_header Host $host; proxy\_set\_header X-Real-IP $remote\_addr; proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for; proxy\_set\_header X-Forwarded-Proto $scheme; # Docker DNS (di network docker compose) resolver 127.0.0.11 ipv6=off valid=10s; location / { set $up\_frontend "frontend:80"; proxy\_pass http://$up\_frontend; } location ^\~ /api/laravel { limit\_req zone=api\_limit burst=20 nodelay; rewrite ^/api/laravel/?(.\*)$ /$1 break; set $up\_laravel "laravel:80"; proxy\_pass http://$up\_laravel; } location ^\~ /api/go { limit\_req zone=api\_limit burst=20 nodelay; rewrite ^/api/go/?(.\*)$ /$1 break; set $up\_go "goapi:8080"; proxy\_pass http://$up\_go; } } cikal@devops-ci:\~/three-body-problem-main$ nano devops/nginx/dev.conf cikal@devops-ci:\~/three-body-problem-main$ docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T nginx-dev nginx -t docker compose --env-file .env.dev.compose -f docker-compose.dev.yml restart nginx-dev nginx: the configuration file /etc/nginx/nginx.conf syntax is ok nginx: configuration file /etc/nginx/nginx.conf test is successful \[+] restart 0/1 ⠸ Container three-body-problem-main-nginx-dev-1 Restarting 1.3s cikal@devops-ci:\~/three-body-problem-main$ curl -k https://dev.local/api/go/api/products curl -k https://dev.local/api/laravel/api/products curl: (6) Could not resolve host: dev.local curl: (6) Could not resolve host: dev.local cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ # dari folder yang dulu kamu pakai (yang menghasilkan prefix three-body-problem-main-\*) docker compose --env-file .env.dev.compose -f docker-compose.dev.yml down \[+] down 6/6 ✔ Container three-body-problem-main-nginx-dev-1 Removed 0.7s ✔ Container three-body-problem-main-frontend-1 Removed 0.5s ✔ Container three-body-problem-main-goapi-1 Removed 0.7s ✔ Container three-body-problem-main-laravel-1 Removed 1.5s ✔ Container three-body-problem-main-mysql-1 Removed 7.5s ✔ Network three-body-problem-main\_devnet Removed 0.2s cikal@devops-ci:\~/three-body-problem-main$ ls -la /home/gitlab-runner/threebody-deploy ls: cannot access '/home/gitlab-runner/threebody-deploy': Permission denied cikal@devops-ci:\~/three-body-problem-main$ sudo mkdir -p /home/cikal/threebody-deploy sudo chown -R gitlab-runner:gitlab-runner /home/cikal/threebody-deploy \[sudo] password for cikal: cikal@devops-ci:\~/three-body-problem-main$ ls -la /home/gitlab-runner/threebody-deploy ls: cannot access '/home/gitlab-runner/threebody-deploy': Permission denied cikal@devops-ci:\~/three-body-problem-main$ cat devops/nginx/dev.conf # devops/nginx/dev.conf (atau file yang berisi server dev.local) upstream laravel\_upstream { server laravel:80; } upstream go\_upstream { server goapi:8080; } upstream fe\_upstream { server frontend:80; } server { listen 80; listen 443 ssl; server\_name dev.local; ssl\_certificate /etc/nginx/certs/dev.crt; ssl\_certificate\_key /etc/nginx/certs/dev.key; # Frontend location / { proxy\_pass http://fe\_upstream; proxy\_set\_header Host $host; proxy\_set\_header X-Real-IP $remote\_addr; proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for; proxy\_set\_header X-Forwarded-Proto $scheme; } # React memanggil: /api/laravel/api/products -> laravel menerima /api/products location /api/laravel/ { proxy\_pass http://laravel\_upstream/; proxy\_set\_header Host $host; proxy\_set\_header X-Real-IP $remote\_addr; proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for; proxy\_set\_header X-Forwarded-Proto $scheme; } # React memanggil: /api/go/api/products -> goapi menerima /api/products location /api/go/ { proxy\_pass http://go\_upstream/; proxy\_set\_header Host $host; proxy\_set\_header X-Real-IP $remote\_addr; proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for; proxy\_set\_header X-Forwarded-Proto $scheme; } } cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["docker"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' # test nginx config docker exec "$NG" nginx -t # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port) export COMPOSE\_PROJECT\_NAME="threebody-dev" # samakan TAG dengan build job (paling aman pakai FULL SHA) TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # hentikan stack yang sama dulu (volume DB tetap aman) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 2m 55s

Mantap—progressnya sudah benar. Ada 3 hal yang perlu kamu bereskan dari output terbaru:

***

### 1) `curl: Could not resolve host: dev.local` itu **bukan error nginx**

Itu murni DNS/hosts di mesin **devops-ci** (Linux) yang belum tahu `dev.local`.

Kalau mau test dari server **tanpa ubah /etc/hosts**, pakai salah satu ini:

**Opsi A (paling gampang, pakai Host header):**

```bash
curl -k -H 'Host: dev.local' https://127.0.0.1/api/go/api/products
curl -k -H 'Host: dev.local' https://127.0.0.1/api/laravel/api/products
```

**Opsi B (`--resolve`):**

```bash
curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/api/go/api/products
curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/api/laravel/api/products
```

Kalau kamu memang mau `dev.local` resolve normal di server:

```bash
echo "127.0.0.1 dev.local" | sudo tee -a /etc/hosts
```

> Di Windows kamu kemungkinan sudah ada hosts `dev.local -> 192.168.56.43`, makanya browser bisa buka `https://dev.local/`.

***

### 2) Port 80 sudah beres ✅ (kamu sudah `down`)

Bagus, kamu sudah jalankan:

```bash
docker compose ... down
```

Jadi error `Bind for 0.0.0.0:80 failed` harusnya hilang di pipeline deploy berikutnya.

***

### 3) Permission `/home/gitlab-runner/threebody-deploy` wajar “Permission denied”

Karena folder **/home/gitlab-runner** biasanya mode `700`, user `cikal` nggak bisa masuk walaupun subfoldernya ada.

#### Solusi paling rapi: pindahkan DEPLOY\_DIR ke `/home/cikal/threebody-deploy`

Kamu sudah bikin folder itu dan sudah `chown` ke `gitlab-runner`, jadi tinggal ubah ini di `.gitlab-ci.yml`:

```diff
- DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
+ DEPLOY_DIR="/home/cikal/threebody-deploy"
```

Lalu untuk cek isi deploy folder:

```bash
ls -la /home/cikal/threebody-deploy
```

***

### 4) Soal nginx 500 “uninitialized up\_go / up\_laravel”

Dengan `dev.conf` versi baru (yang pakai `upstream` + `location /api/go/` dan `/api/laravel/`) itu harusnya **sudah fix**.

Tapi pastikan file yang benar-benar kepakai oleh container nginx-dev adalah yang baru. Setelah deploy up, jalankan:

```bash
docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T nginx-dev nginx -T | sed -n '1,200p'
```

Kalau masih muncul config lama, berarti yang termount bukan file yang kamu edit (atau deploy folder masih berisi versi lama).

***

### 5) Next step (wajib): commit & push perubahan

Kamu sudah edit `devops/nginx/dev.conf` dan `.gitlab-ci.yml`, tapi belum kamu commit/push.

```bash
git add devops/nginx/dev.conf .gitlab-ci.yml
git commit -m "Fix nginx upstream + stable deploy dir/project"
git push
```

Setelah pipeline selesai deploy, test dari devops-ci (pakai Host header):

```bash
curl -k -H 'Host: dev.local' https://127.0.0.1/api/go/api/products
curl -k -H 'Host: dev.local' https://127.0.0.1/api/laravel/api/products
```

Kalau kamu kirim **isi `docker-compose.dev.yml` bagian service `nginx-dev` (volumes + ports)** dan isi `devops/nginx/frontend.conf`, aku bisa pastikan 100% tidak ada config yang ketiban/konflik.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000df1c72098905e043e6564c67\&ts=490475\&p=fs\&cid=1\&sig=914f0b0ab7e863fe5fdd69d86b07282944aeea792f211355ae6428d3c49d992c\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000a2387209a74ae2a2dc390dea\&ts=490475\&p=fs\&cid=1\&sig=81f9f55683fe886b0724dbd0510e7f7d8563131865b6dc72f49bf1e39e1ecf71\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000a5d872099d0d476177517d44\&ts=490475\&p=fs\&cid=1\&sig=357cad2f6bfd0627878982fa4fd23b7e84fcc344fe5bc343cecfe1ba5173c7e3\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000068f8720998d4b51076bf1a61\&ts=490475\&p=fs\&cid=1\&sig=3d109895fa0659b7527161eed58743ca07ac96df47000ca50fccade1ebdf626d\&v=0)cikal@devops-ci:\~/three-body-problem-main$ echo "127.0.0.1 dev.local" | sudo tee -a /etc/hosts 127.0.0.1 dev.local cikal@devops-ci:\~/three-body-problem-main$ curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/api/go/api/products curl -k --resolve dev.local:443:127.0.0.1 https://dev.local/api/laravel/api/products curl: (7) Failed to connect to dev.local port 443 after 0 ms: Couldn't connect to server curl: (7) Failed to connect to dev.local port 443 after 0 ms: Couldn't connect to server cikal@devops-ci:\~/three-body-problem-main$ curl -k -H 'Host: dev.local' https://127.0.0.1/api/go/api/products curl -k -H 'Host: dev.local' https://127.0.0.1/api/laravel/api/products curl: (7) Failed to connect to 127.0.0.1 port 443 after 0 ms: Couldn't connect to server curl: (7) Failed to connect to 127.0.0.1 port 443 after 0 ms: Couldn't connect to server cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["docker"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' # test nginx config docker exec "$NG" nginx -t # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port) export COMPOSE\_PROJECT\_NAME="threebody-dev" # samakan TAG dengan build job (paling aman pakai FULL SHA) TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" #DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" DEPLOY\_DIR="/home/cikal/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # hentikan stack yang sama dulu (volume DB tetap aman) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$ ls -la /home/cikal/threebody-deploy total 8 drwxr-xr-x 2 gitlab-runner gitlab-runner 4096 Dec 14 07:40 . drwxr-x--- 8 cikal cikal 4096 Dec 14 07:40 .. cikal@devops-ci:\~/three-body-problem-main$ docker compose --env-file .env.dev.compose -f docker-compose.dev.yml exec -T nginx-dev nginx -T | sed -n '1,200p' service "nginx-dev" is not running cikal@devops-ci:\~/three-body-problem-main$ git add devops/nginx/dev.conf .gitlab-ci.yml git commit -m "Fix nginx upstream + stable deploy dir/project" git push \[main 712b90b] Fix nginx upstream + stable deploy dir/project 2 files changed, 42 insertions(+), 46 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 11, done. Counting objects: 100% (11/11), done. Delta compression using up to 2 threads Compressing objects: 100% (6/6), done. Writing objects: 100% (6/6), 1.20 KiB | 1.20 MiB/s, done. Total 6 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 5a95919..712b90b main -> main cikal@devops-ci:\~/three-body-problem-main$ curl -k -H 'Host: dev.local' https://127.0.0.1/api/go/api/products curl -k -H 'Host: dev.local' https://127.0.0.1/api/laravel/api/products curl: (7) Failed to connect to 127.0.0.1 port 443 after 0 ms: Couldn't connect to server curl: (7) Failed to connect to 127.0.0.1 port 443 after 0 ms: Couldn't connect to server cikal@devops-ci:\~/three-body-problem-main$ Running with gitlab-runner 18.6.5 (5908bc19) on DevOps\_43\_runner P5qLStHlf, system ID: s\_7bd6fa72a2b0 Preparing the "docker" executor 00:05 Using default image Using Docker executor with image docker:latest ... Using default image Using effective pull policy of \[always] for container docker:latest Pulling docker image docker:latest ... Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... Preparing environment 00:01 Using effective pull policy of \[always] for container sha256:54e6bb7e83797064b15abaf06223c9083f9e7b807d7046c13920150573a66d97 Running on runner-p5qlsthlf-project-76926856-concurrent-0 via devops-ci... Getting source from Git repository 00:05 Gitaly correlation ID: 9adc1c0cb62baa09-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /builds/cikalfarid/three-body-problem/.git/ Created fresh repository. Checking out 712b90bf as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:56 Using default image Using effective pull policy of \[always] for container docker:latest Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... $ set -euo pipefail $ docker version Client: Version: 29.1.3 API version: 1.52 Go version: go1.25.5 Git commit: f52814d Built: Fri Dec 12 14:48:47 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command ......+...+.......+...+..+++++++++++++++++++++++++++++++++++++++\*..+++++++++++++++++++++++++++++++++++++++\*.+...+....+...+.........+..............+.+.........+..++++++ .+.................+++++++++++++++++++++++++++++++++++++++\*........+........+++++++++++++++++++++++++++++++++++++++\*.........+.....+.............+..+..........+.....+...+...+...+.+.......................+......+.........+.+..+....+.....+.+.....+...+..........+.....+....+............+...+.....+......+.+........................+......+........+...+...............+....+..............+....+...+.....+.......+..+......+...+....+........+.............+............+..+..........+..+...+......+......+..................+.......+.....+...+......+.+........+.+.....+................+.........+......+.....+..........+...+..+.+..............+.+.....+.......+............+........+.......+......+..+.+............+...+...........+...+......+......+.............+...+.....+.......+......+.....+.......+..+..........+......+......++++++ ----- Login Succeeded WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'. Configure a credential helper to remove this warning. See https://docs.docker.com/go/credential-store/ Image \[MASKED]/\[MASKED]/goapi:712b90bf2ee2b60bf51dbc028aaf95c240d33e48 Pulling Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/frontend:712b90bf2ee2b60bf51dbc028aaf95c240d33e48 Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/laravel:712b90bf2ee2b60bf51dbc028aaf95c240d33e48 Pulling Image \[MASKED]/\[MASKED]/goapi:712b90bf2ee2b60bf51dbc028aaf95c240d33e48 Pulled Image \[MASKED]/\[MASKED]/frontend:712b90bf2ee2b60bf51dbc028aaf95c240d33e48 Pulled Image \[MASKED]/\[MASKED]/laravel:712b90bf2ee2b60bf51dbc028aaf95c240d33e48 Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Volume \[MASKED]-dev\_mysql\_data Creating Volume \[MASKED]-dev\_mysql\_data Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started Migration table created successfully. Migrating: 2014\_10\_12\_000000\_create\_users\_table Migrated: 2014\_10\_12\_000000\_create\_users\_table (486.79ms) Migrating: 2014\_10\_12\_100000\_create\_password\_resets\_table Migrated: 2014\_10\_12\_100000\_create\_password\_resets\_table (296.13ms) Migrating: 2019\_08\_19\_000000\_create\_failed\_jobs\_table Migrated: 2019\_08\_19\_000000\_create\_failed\_jobs\_table (245.49ms) Migrating: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table Migrated: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table (428.11ms) Migrating: 2025\_07\_25\_075615\_create\_products\_table \[2025-12-14 07:51:24] local.ERROR: SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' already exists (SQL: create table `products` (`id` bigint unsigned not null auto\_increment primary key, `name` varchar(255) not null, `description` text null, `price` decimal(10, 2) not null, `quantity` int not null default '0', `category` varchar(255) null, `created_at` timestamp null, `updated_at` timestamp null) default character set utf8mb4 collate 'utf8mb4\_unicode\_ci') {"exception":"\[object] (Illuminate\\\Database\\\QueryException(code: 42S01): SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' already exists (SQL: create table `products` (`id` bigint unsigned not null auto\_increment primary key, `name` varchar(255) not null, `description` text null, `price` decimal(10, 2) not null, `quantity` int not null default '0', `category` varchar(255) null, `created_at` timestamp null, `updated_at` timestamp null) default character set utf8mb4 collate 'utf8mb4\_unicode\_ci') at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:712) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('create table `p...', Array, Object(Closure))` #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(490): Illuminate\\\Database\\\Connection->run('create table `p...', Array, Object(Closure))` #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Blueprint.php(109): Illuminate\\\Database\\\Connection->statement('create table `p...')` #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(364): Illuminate\\\Database\\\Schema\\\Blueprint->build(Object(Illuminate\\\Database\\\MySqlConnection), Object(Illuminate\\\Database\\\Schema\\\Grammars\\\MySqlGrammar)) #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(223): Illuminate\\\Database\\\Schema\\\Builder->build(Object(Illuminate\\\Database\\\Schema\\\Blueprint)) #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Support/Facades/Facade.php(261): Illuminate\\\Database\\\Schema\\\Builder->create('products', Object(Closure)) #6 /var/www/html/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php(16): Illuminate\\\Support\\\Facades\\\Facade::\_\_callStatic('create', Array) #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(472): CreateProductsTable->up() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(394): Illuminate\\\Database\\\Migrations\\\Migrator->runMethod(Object(Illuminate\\\Database\\\MySqlConnection), Object(CreateProductsTable), 'up') #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(403): Illuminate\\\Database\\\Migrations\\\Migrator->Illuminate\\\Database\\\Migrations\\\\{closure}() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(202): Illuminate\\\Database\\\Migrations\\\Migrator->runMigration(Object(CreateProductsTable), 'up') #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(167): Illuminate\\\Database\\\Migrations\\\Migrator->runUp('/var/www/html/d...', 1, false) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(112): Illuminate\\\Database\\\Migrations\\\Migrator->runPending(Array, Array) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(84): Illuminate\\\Database\\\Migrations\\\Migrator->run(Array, Array) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #22 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #24 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #25 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #26 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #27 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 {main} \[previous exception] \[object] (PDOException(code: 42S01): SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' already exists at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:501) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(501): PDOStatement->execute() #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(705): Illuminate\\\Database\\\Connection->Illuminate\\\Database\\\\{closure}('create table `p...', Array)` #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('create table `p...', Array, Object(Closure))` #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(490): Illuminate\\\Database\\\Connection->run('create table `p...', Array, Object(Closure))` #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Blueprint.php(109): Illuminate\\\Database\\\Connection->statement('create table `p...')` #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(364): Illuminate\\\Database\\\Schema\\\Blueprint->build(Object(Illuminate\\\Database\\\MySqlConnection), Object(Illuminate\\\Database\\\Schema\\\Grammars\\\MySqlGrammar)) #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(223): Illuminate\\\Database\\\Schema\\\Builder->build(Object(Illuminate\\\Database\\\Schema\\\Blueprint)) #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Support/Facades/Facade.php(261): Illuminate\\\Database\\\Schema\\\Builder->create('products', Object(Closure)) #8 /var/www/html/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php(16): Illuminate\\\Support\\\Facades\\\Facade::\_\_callStatic('create', Array) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(472): CreateProductsTable->up() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(394): Illuminate\\\Database\\\Migrations\\\Migrator->runMethod(Object(Illuminate\\\Database\\\MySqlConnection), Object(CreateProductsTable), 'up') #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(403): Illuminate\\\Database\\\Migrations\\\Migrator->Illuminate\\\Database\\\Migrations\\\\{closure}() #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(202): Illuminate\\\Database\\\Migrations\\\Migrator->runMigration(Object(CreateProductsTable), 'up') #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(167): Illuminate\\\Database\\\Migrations\\\Migrator->runUp('/var/www/html/d...', 1, false) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(112): Illuminate\\\Database\\\Migrations\\\Migrator->runPending(Array, Array) #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(84): Illuminate\\\Database\\\Migrations\\\Migrator->run(Array, Array) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #22 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #24 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #25 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #26 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #27 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #31 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #32 {main} "} In Connection.php line 712: SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' a lready exists (SQL: create table `products` (`id` bigint unsigned not null auto\_increment primary key, `name` varchar(255) not null, `description` tex t null, `price` decimal(10, 2) not null, `quantity` int not null default '0 ', `category` varchar(255) null, `created_at` timestamp null, `updated_at` timestamp null) default character set utf8mb4 collate 'utf8mb4\_unicode\_ci') In Connection.php line 501: SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' a lready exists Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit code 1Selesai berpikir selama 4m 5s

Kamu sekarang kena **2 masalah terpisah**:

### 1) `ERR_CONNECTION_REFUSED` ke `https://dev.local` (nggak ada yang listen di 443/80)

Dari log job terakhir kelihatan **deploy jalan pakai&#x20;**_**docker executor**_ (“Preparing the docker executor”, image `docker:latest`). Kalau runner-mu pakai **DinD / Docker daemon terpisah**, container `nginx-dev` itu start **di dalam environment CI**, bukan di VM `192.168.56.43` yang diakses Windows — hasilnya di VM **tidak ada proses yang bind ke 0.0.0.0:443**, jadi browser dapet _connection refused_.

✅ Fix-nya: **deploy harus jalan di runner “shell” (di host VM), bukan docker executor** (atau pastikan docker executor mount `/var/run/docker.sock` ke host — tapi itu setting level runner, bukan di YAML).

**Paling simpel (recommended):**

* Pakai tag khusus untuk **shell-runner** dan paksa job `deploy_dev` jalan di situ.
* Build boleh tetap di runner docker kalau mau.

Contoh:

```yaml
default:
  tags: ["docker"]   # biarin build pakai docker-runner

deploy_dev:
  tags: ["shell"]    # <-- ganti ke tag runner shell-mu (mis. vm2-shell-runner)
  stage: deploy
  ...
```

> Kamu harus pastikan runner shell punya tag itu. Kalau runner shell & docker-runner sama-sama bertag “docker”, deploy-mu bisa “nyasar” lagi.

Setelah deploy benar-benar jalan di host VM, cek cepat dari VM:

```bash
sudo ss -lntp | egrep ':80|:443' || true
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep nginx
```

### 2) Pipeline gagal di `php artisan migrate` karena tabel `products` keburu dibuat

Log kamu jelas:

* “Migration table created successfully.” (berarti fresh)
* lalu gagal: **Table `products` already exists**\
  Ini hampir pasti karena **service Go (`goapi`) auto-create/auto-migrate tabel `products` saat container start**, _sebelum_ Laravel migrasi jalan. Di log goapi sebelumnya juga kelihatan dia seed/create data.

✅ Fix paling bersih: **jalankan migrasi Laravel sebelum menyalakan goapi**.

Ubah bagian deploy jadi 2 tahap:

```bash
# pull images dulu (ok)
docker compose ... pull

# naikkan hanya mysql + laravel dulu
docker compose ... up -d mysql laravel

# tunggu laravel siap lalu migrate
for i in $(seq 1 30); do
  docker compose ... exec -T laravel php -v >/dev/null 2>&1 && break
  sleep 2
done
docker compose ... exec -T laravel php artisan migrate --force

# baru naikkan semuanya (goapi, frontend, nginx, dll)
docker compose ... up -d --remove-orphans
```

Dengan ini, Go nggak sempat bikin tabel `products` duluan.

> Alternatif (quick & dirty) kalau kamu mau pipeline tetap hijau walau konflik: `php artisan migrate --force || true` — tapi ini nutupin problem sebenarnya.

### Catatan penting soal `DEPLOY_DIR`

Kalau deploy jalan sebagai user `gitlab-runner` di shell runner, **jangan taruh di `/home/cikal/...`** kecuali permission parent foldernya juga kebuka (karena `/home/cikal` biasanya 750). Paling aman:

* `DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"` **atau** `/opt/threebody-deploy`

Kalau kamu mau tetap bisa lihat sebagai user `cikal`, pakai `sudo ls` aja.

***

Kalau kamu terapkan 2 perubahan di atas:

1. `deploy_dev` dipaksa ke **shell runner di host VM**
2. **migrate Laravel sebelum start goapi**

…maka `dev.local` harusnya balik reachable dan endpoint `/api/go/api/products` & `/api/laravel/api/products` nggak 500 lagi karena Nginx upstream-mu sudah bener.

Kalau kamu kirim `docker-compose.dev.yml` (bagian service `nginx-dev` terutama ports/volume), aku bisa tunjukin juga apakah mapping 80/443 dan mount cert/conf sudah konsisten dengan `DEPLOY_DIR`.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000bc607207a089e10f7631b4a5\&ts=490475\&p=fs\&cid=1\&sig=46c05d2eefe5af319be0f075fed1b8d167043bbbae725e98489b3b634d3a2b7a\&v=0)cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_DATABASE: threebody MYSQL\_USER: threebody MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: test: \["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p\$${MYSQL\_ROOT\_PASSWORD} --silent"] interval: 5s timeout: 3s retries: 20 restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: threebody DB\_USERNAME: threebody DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: threebody DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: threebody networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ coba bantu dan beri arahan yang detail, karena masih kurang paham untuk kendala iniSelesai berpikir selama 3m 2s

Oke, dari yang kamu kirim sekarang, akar masalahnya jadi kelihatan jelas. Kita pecah jadi **3 “blok masalah”** dan aku kasih langkah yang benar-benar bisa kamu ikuti.

***

### A) Kenapa `dev.local` sekarang **ERR\_CONNECTION\_REFUSED**

Di screenshot **Settings → CI/CD → Runners**, project kamu cuma punya **1 runner**: `DevOps_44_runner` dengan tag **`vm2`** dan **`docker`**.

Artinya deploy kamu kemungkinan besar **jalan di Docker executor** (bukan shell di host VM). Kalau runner Docker executor itu pakai **DinD (docker-in-docker)**, maka:

* `docker compose up -d` memang sukses **di dalam engine Docker milik job/runner**,
* tapi **port 80/443 itu tidak kebuka di VM 192.168.56.43**,
* akhirnya Windows akses `https://dev.local` → **refused** (karena di VM tidak ada yang listen 80/443).

#### 1) Verifikasi cepat di VM (devops-ci)

Di VM, jalankan:

```bash
sudo ss -lntp | egrep ':80|:443' || true
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep -E "nginx|threebody|dev" || true
```

* Kalau **tidak ada yang listen** di 80/443 → fix harus di runner/deploy method (bagian B).
* Kalau ada listener tapi masih refused dari Windows → baru cek firewall/route (tapi kasusmu ini hampir pasti “tidak ada listener”).

***

### B) Solusi yang paling gampang dipahami: **buat runner Shell untuk deploy**

Kamu butuh **runner shell** di VM yang sama (192.168.56.43), supaya `docker compose` benar-benar jalan di host VM dan binding port 80/443 muncul.

#### 2) Buat Project Runner Shell (di VM2)

Di GitLab UI:

* Settings → CI/CD → Runners → **Create project runner**
* Copy token/command register yang diberikan GitLab

Di VM jalankan register (contoh bentuknya, ikuti token asli dari UI):

```bash
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token <TOKEN_DARI_UI> \
  --executor shell \
  --description "vm2-shell-runner" \
  --tag-list "vm2,shell" \
  --run-untagged="false"
```

> Setelah itu di halaman runners harus muncul runner baru dengan tag `shell`.

#### 3) Paksa job `deploy_dev` pakai runner shell

Di `.gitlab-ci.yml`:

```yaml
deploy_dev:
  tags: ["shell"]   # atau ["vm2","shell"] biar spesifik
  stage: deploy
  ...
```

**Build job** boleh tetap pakai tag `docker`.

***

### C) Deploy kamu gagal karena migrasi Laravel bentrok dengan Go (tabel `products` keburu dibuat)

Log kamu: `Table 'products' already exists` itu kejadian karena **goapi hidup duluan lalu bikin tabel**, baru Laravel migrasi → Laravel gagal.

#### 4) Perbaiki urutan start di deploy: naikkan mysql+laravel dulu, migrate, baru goapi+frontend+nginx

Ganti bagian deploy script jadi model 2 tahap seperti ini (ini yang paling aman):

```bash
# pull dulu
docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull

# naikkan DB + laravel dulu
docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d mysql laravel

# tunggu laravel siap
for i in $(seq 1 30); do
  if docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php -v >/dev/null 2>&1; then
    break
  fi
  sleep 2
done

# migrate
docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force

# baru naikkan sisanya
docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans
docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps
```

> Dengan ini Go tidak sempat bikin schema sebelum Laravel.

#### 5) Satu kali “bersih-bersih” kalau DB udah terlanjur kacau (opsional tapi sering perlu)

Kalau sebelumnya kamu sudah pernah bikin volume yang isinya “setengah jadi”, paling cepat untuk dev adalah reset volume:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
```

⚠️ Ini hapus data MySQL (dev aman biasanya).

***

### D) Saran penting soal `DEPLOY_DIR`

Di docker-compose kamu ada mount relative:

```yaml
volumes:
  - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro
  - ./certs/dev:/etc/nginx/certs:ro
```

Artinya: **folder kerja deploy harus benar** (DEPLOY\_DIR harus punya `devops/nginx/dev.conf` dan `certs/dev/*`).

Untuk menghindari permission “home user”, paling aman pakai:

* `/home/gitlab-runner/threebody-deploy` **atau**
* `/opt/threebody-deploy`

Kalau kamu pakai `/home/cikal/...` sering mentok permission traversing `/home/cikal`.

***

### E) Checklist setelah kamu terapkan fix

Setelah pipeline deploy sukses (runner shell + urutan start benar), di VM jalankan:

```bash
sudo ss -lntp | egrep ':80|:443' || true
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep nginx
curl -k -H 'Host: dev.local' https://127.0.0.1/
curl -k -H 'Host: dev.local' https://127.0.0.1/api/go/api/products
curl -k -H 'Host: dev.local' https://127.0.0.1/api/laravel/api/products
```

Di Windows pastikan `hosts` sudah:

```
192.168.56.43  dev.local
```

***

Kalau kamu mau, paste **bagian `deploy_dev:`** terbaru dari `.gitlab-ci.yml` kamu (setelah kamu coba ubah) — aku bisa rapihin jadi versi final yang clean + tidak gampang “nyasar runner” + tidak gagal migrate.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000d2c871fa8447834307ea8ed4\&ts=490475\&p=fs\&cid=1\&sig=f3cdb1736c6562b42867287a8d3ef45888ff903a04441e1e5056542f642f11a0\&v=0)cikal@devops-ci:\~/three-body-problem-main$ sudo ss -lntp | egrep ':80|:443' || true docker ps --format "table \{{.Names\}}\t\{{.Ports\}}" | grep -E "nginx|threebody|dev" || true \[sudo] password for cikal: LISTEN 0 4096 0.0.0.0:8081 0.0.0.0:\* users:(("docker-proxy",pid=1894,fd=7)) LISTEN 0 4096 \[::]:8081 \[::]:\* users:(("docker-proxy",pid=1900,fd=7)) threebody-dev-nginx-dev-1 threebody-dev-frontend-1 80/tcp threebody-dev-laravel-1 80/tcp threebody-dev-goapi-1 8080/tcp threebody-dev-mysql-1 3306/tcp, 33060/tcp threebody-deploy-frontend-1 80/tcp threebody-deploy-laravel-1 80/tcp threebody-deploy-goapi-1 8080/tcp threebody-deploy-mysql-1 3306/tcp, 33060/tcp nginx 0.0.0.0:8081->8080/tcp, \[::]:8081->8080/tcp cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner register \ --url https://gitlab.com \ --token glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7 \ --executor shell \ --description "vm2-shell-runner" \ --tag-list "vm2,shell" \ --run-untagged="false" Runtime platform arch=amd64 os=linux pid=147860 revision=5908bc19 version=18.6.5 Running in system-mode. Enter the GitLab instance URL (for example, https://gitlab.com/): \[https://gitlab.com]: FATAL: Runner configuration other than name and executor configuration is reserved (specifically --locked, --access-level, --run-untagged, --maximum-timeout, --paused, --tag-list, and --maintenance-note) and cannot be specified when registering with a runner authentication token. This configuration is specified on the GitLab server. Please try again without specifying any of those arguments. For more information, see https://docs.gitlab.com/ci/runners/new\_creation\_workflow/#changes-to-the-gitlab-runner-register-command-syntax cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner register \ --url https://gitlab.com \ --token glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7 \ --executor shell \ --description "vm2-shell-runner" \ --tag-list "vm2,shell" \ --run-untagged="false" Runtime platform arch=amd64 os=linux pid=148195 revision=5908bc19 version=18.6.5 Running in system-mode. Enter the GitLab instance URL (for example, https://gitlab.com/): \[https://gitlab.com]: https://gitlab.com FATAL: Runner configuration other than name and executor configuration is reserved (specifically --locked, --access-level, --run-untagged, --maximum-timeout, --paused, --tag-list, and --maintenance-note) and cannot be specified when registering with a runner authentication token. This configuration is specified on the GitLab server. Please try again without specifying any of those arguments. For more information, see https://docs.gitlab.com/ci/runners/new\_creation\_workflow/#changes-to-the-gitlab-runner-register-command-syntax cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["docker"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' # test nginx config docker exec "$NG" nginx -t # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: tags: \["shell"] stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port) export COMPOSE\_PROJECT\_NAME="threebody-dev" # samakan TAG dengan build job (paling aman pakai FULL SHA) TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" #DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" DEPLOY\_DIR="/home/cikal/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # hentikan stack yang sama dulu (volume DB tetap aman) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$ ll total 72 drwxrwxr-x 8 cikal cikal 4096 Dec 14 08:09 ./ drwxr-x--- 8 cikal cikal 4096 Dec 14 07:40 ../ -rw-rw-r-- 1 cikal cikal 204 Dec 13 17:17 .env.dev.compose -rw-rw-r-- 1 cikal cikal 132 Dec 13 20:26 .env.dev.compose.example drwxrwxr-x 8 cikal cikal 4096 Dec 14 07:48 .git/ -rw-rw-r-- 1 cikal cikal 41 Dec 13 20:29 .gitignore -rw-rw-r-- 1 cikal cikal 6202 Dec 14 08:09 .gitlab-ci.yml -rw-rw-r-- 1 cikal cikal 8220 Dec 13 20:27 README.md drwxrwxr-x 3 cikal cikal 4096 Dec 13 20:26 certs/ -rw-rw-r-- 1 cikal cikal 809 Dec 9 04:22 compare\_apis.sh drwxrwxr-x 4 cikal cikal 4096 Dec 13 20:26 devops/ -rw-rw-r-- 1 cikal cikal 1897 Dec 13 20:29 docker-compose.dev.yml drwxrwxr-x 4 cikal cikal 4096 Dec 13 20:33 frontend/ drwxrwxr-x 2 cikal cikal 4096 Dec 13 20:33 go/ drwxrwxr-x 11 cikal cikal 4096 Dec 13 20:33 laravel/ cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 4m 18s

Mantap, output kamu sekarang **sudah cukup untuk “mengunci” penyebabnya**.

### 1) Kenapa `dev.local` masih _connection refused_

Di VM kamu, hasil ini penting:

* `ss -lntp` **tidak menunjukkan port 80/443**
* `docker ps ...` menunjukkan container `threebody-dev-nginx-dev-1` **tapi kolom Ports kosong**
* Kamu juga masih punya **2 stack sekaligus**: `threebody-dev-*` dan `threebody-deploy-*`

Artinya: **nginx-dev yang harusnya publish `80:80` dan `443:443` itu tidak ter-publish ke host**, sehingga Windows akses `https://dev.local` → **refused**.

#### Aksi cepat: cek apakah nginx-dev benar-benar publish port

Jalankan ini di VM:

```bash
docker port threebody-dev-nginx-dev-1 || true
docker inspect -f 'Image={{.Config.Image}} NetworkMode={{.HostConfig.NetworkMode}} Ports={{json .NetworkSettings.Ports}}' threebody-dev-nginx-dev-1
```

* Kalau `docker port ...` **tidak keluar apa-apa** → berarti container itu memang **dibuat tanpa publish ports** (bukan masalah hosts Windows).
* Kalau keluar mapping `0.0.0.0:80->80/tcp` dan `0.0.0.0:443->443/tcp` → lanjut cek firewall, tapi dari kasusmu kemungkinan besar **tidak ada mapping**.

***

### 2) Rapihin dulu: kamu kebanyakan “stack nyangkut”

Di output kamu ada:

* `threebody-dev-*`
* `threebody-deploy-*`
* plus container `nginx` lain publish `8081->8080` (ini bikin makin bingung)

Biar bersih, stop & hapus dulu yang tidak dipakai.

#### Hapus stack `threebody-deploy` (yang nyangkut)

```bash
docker rm -f $(docker ps -aq --filter "name=threebody-deploy") 2>/dev/null || true
docker network rm $(docker network ls -q --filter "name=threebody-deploy") 2>/dev/null || true
```

#### Restart stack yang kamu mau pakai: **hanya `threebody-dev`**

Dari folder repo yang ada `docker-compose.dev.yml`, jalankan:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d
```

Lalu cek mappingnya:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep -E "threebody-dev|nginx-dev"
docker port threebody-dev-nginx-dev-1
```

Kalau setelah ini `docker port` masih kosong → berarti compose yang kamu jalankan **tidak membaca section `ports:` untuk nginx-dev** (biasanya karena file compose yang dipakai beda / salah path / salah project).

***

### 3) Tes lokal dari VM (harus tembus dulu sebelum Windows)

Kalau port sudah publish, ini harus berhasil:

```bash
curl -vk https://127.0.0.1/ -H "Host: dev.local"
curl -vk https://127.0.0.1/api/go/api/products -H "Host: dev.local"
curl -vk https://127.0.0.1/api/laravel/api/products -H "Host: dev.local"
```

Kalau dari VM **sudah bisa**, baru Windows akan ikut bisa (dengan hosts `192.168.56.43 dev.local` yang kamu sudah benar).

***

### 4) Masalah runner register: kamu pakai token model baru (glrt-…)

Error itu normal untuk GitLab Runner versi baru.

Dengan **runner authentication token (glrt-...)**, kamu **tidak boleh** set:

* `--tag-list`
* `--run-untagged`
* dll

Semua itu harus di-set **di GitLab UI saat create runner**, bukan lewat CLI.

#### Cara register yang benar

1. Di GitLab UI → Settings → CI/CD → Runners → **Create project runner**
   * set Tags: `shell,vm2`
   * Run untagged: **off** (kalau kamu mau)
2. Di VM jalankan **tanpa tag-list**:

```bash
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token <TOKEN_glrt_DARI_UI> \
  --executor shell
```

> Setelah ini, `deploy_dev: tags: ["shell"]` milikmu sudah benar — job deploy akan “nempel” ke runner shell.

⚠️ Penting: token `glrt-...` kamu sudah kamu paste di chat. Demi keamanan, **hapus / rotate token runner itu di GitLab** dan buat token baru.

***

### 5) Setelah deploy jalan, kamu tetap perlu fix urutan migrasi (biar gak bentrok Go vs Laravel)

Yang kemarin gagal itu karena tabel `products` sudah ada duluan.

Nanti di `deploy_dev` ubah jadi 2 tahap:

1. `up -d mysql laravel`
2. `migrate`
3. baru `up -d` sisanya (goapi, frontend, nginx)

Aku bisa tuliskan blok script finalnya kalau kamu mau.

***

#### Paling penting sekarang (biar dev.local hidup):

Kerjakan urutan ini:

1. `docker port threebody-dev-nginx-dev-1` (cek ada mapping atau tidak)
2. bersihkan stack `threebody-deploy-*`
3. `docker compose -p threebody-dev ... down` lalu `up -d`
4. tes `curl -vk https://127.0.0.1 -H "Host: dev.local"`

Kalau kamu paste output dari:

* `docker port threebody-dev-nginx-dev-1`
* `docker inspect ...Ports... threebody-dev-nginx-dev-1`

aku bisa langsung bilang persis bagian mana yang “nggak kebaca” sehingga ports 80/443 tidak kepublish.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000003fd87209a45352fbf036111f\&ts=490475\&p=fs\&cid=1\&sig=bbdb28305fef0234d2ce31ea839d3fb3f9ef71675edf29a8120b0dc049e49e52\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000320471faac402b999383fa63\&ts=490475\&p=fs\&cid=1\&sig=d90e05dbe36316d868586ed11194bcd3a4f8893ce8bcff0ba9eee49314f066a2\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000b4bc72079a711a1acaaccf10\&ts=490475\&p=fs\&cid=1\&sig=6fde5ff90847ef4e4393f573d19ab8fa8ccb8e6a7fa43da57877c4d46c84b8db\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000070cc720789f03f8881e3d710\&ts=490475\&p=fs\&cid=1\&sig=c4b155739fefaa1f78076eb7c32192515228b3fc0e0b8c0fc2396e2665c13c82\&v=0)cikal@devops-ci:\~/three-body-problem-main$ docker port threebody-dev-nginx-dev-1 || true docker inspect -f 'Image=\{{.Config.Image\}} NetworkMode=\{{.HostConfig.NetworkMode\}} Ports=\{{json .NetworkSettings.Ports\}}' threebody-dev-nginx-dev-1 80/tcp -> 0.0.0.0:80 80/tcp -> \[::]:80 443/tcp -> 0.0.0.0:443 443/tcp -> \[::]:443 Image=nginx:alpine NetworkMode=threebody-dev\_devnet Ports={} cikal@devops-ci:\~/three-body-problem-main$ docker rm -f $(docker ps -aq --filter "name=threebody-deploy") 2>/dev/null || true docker network rm $(docker network ls -q --filter "name=threebody-deploy") 2>/dev/null || true 4b46dbf457b3 d71ef7a6a676 7ad2e2bf20b4 6a6696451e7f 5896cec3f28b 60a3911e2f7d cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d \[+] down 5/6 ✔ Container threebody-dev-nginx-dev-1 Removed 0.2s ✔ Container threebody-dev-frontend-1 Removed 1.1s ✔ Container threebody-dev-laravel-1 Removed 2.4s ✔ Container threebody-dev-goapi-1 Removed 1.4s ✔ Container threebody-dev-mysql-1 Removed 3.4s ⠼ Network threebody-dev\_devnet Removing 0.4s \[+] up 6/6 ✔ Network threebody-dev\_devnet Created 0.5s ✔ Container threebody-dev-mysql-1 Healthy 26.8s ✔ Container threebody-dev-goapi-1 Created 1.3s ✔ Container threebody-dev-laravel-1 Created 1.6s ✔ Container threebody-dev-frontend-1 Created 1.2s ✔ Container threebody-dev-nginx-dev-1 Created 1.1s cikal@devops-ci:\~/three-body-problem-main$ docker ps --format "table \{{.Names\}}\t\{{.Ports\}}" | grep -E "threebody-dev|nginx-dev" docker port threebody-dev-nginx-dev-1 threebody-dev-nginx-dev-1 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp threebody-dev-frontend-1 80/tcp threebody-dev-laravel-1 80/tcp threebody-dev-goapi-1 8080/tcp threebody-dev-mysql-1 3306/tcp, 33060/tcp 80/tcp -> 0.0.0.0:80 80/tcp -> \[::]:80 443/tcp -> 0.0.0.0:443 443/tcp -> \[::]:443 cikal@devops-ci:\~/three-body-problem-main$ curl -vk https://127.0.0.1/ -H "Host: dev.local" curl -vk https://127.0.0.1/api/go/api/products -H "Host: dev.local" curl -vk https://127.0.0.1/api/laravel/api/products -H "Host: dev.local" \* Trying 127.0.0.1:443... \* Connected to 127.0.0.1 (127.0.0.1) port 443 \* ALPN: curl offers h2,http/1.1 \* TLSv1.3 (OUT), TLS handshake, Client hello (1): \* TLSv1.3 (IN), TLS handshake, Server hello (2): \* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8): \* TLSv1.3 (IN), TLS handshake, Certificate (11): \* TLSv1.3 (IN), TLS handshake, CERT verify (15): \* TLSv1.3 (IN), TLS handshake, Finished (20): \* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1): \* TLSv1.3 (OUT), TLS handshake, Finished (20): \* SSL connection using TLSv1.3 / TLS\_AES\_256\_GCM\_SHA384 / X25519 / RSASSA-PSS \* ALPN: server accepted http/1.1 \* Server certificate: \* subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* start date: Dec 7 05:39:06 2025 GMT \* expire date: Dec 7 05:39:06 2026 GMT \* issuer: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* SSL certificate verify result: self-signed certificate (18), continuing anyway. \* Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption \* using HTTP/1.x > GET / HTTP/1.1 > Host: dev.local > User-Agent: curl/8.5.0 > Accept: \*/\* > \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* old SSL session ID is stale, removing < HTTP/1.1 200 OK < Server: nginx/1.29.4 < Date: Sun, 14 Dec 2025 08:36:36 GMT < Content-Type: text/html < Content-Length: 644 < Connection: keep-alive < Last-Modified: Sat, 13 Dec 2025 17:24:34 GMT < ETag: "693da152-284" < Accept-Ranges: bytes < \* Connection #0 to host 127.0.0.1 left intact \<!doctype html>\<html lang="en">\<head>\<meta charset="utf-8"/>\<link rel="icon" href="/favicon.ico"/>\<meta name="viewport" content="width=device-width,initial-scale=1"/>\<meta name="theme-color" content="#000000"/>\<meta name="description" content="Web site created using create-react-app"/>\<link rel="apple-touch-icon" href="/logo192.png"/>\<link rel="manifest" href="/manifest.json"/>\<title>React App\</title>\<script defer="defer" src="/static/js/main.c5b29507.js">\</script>\<link href="/static/css/main.fe6da703.css" rel="stylesheet">\</head>\<body>\<noscript>You need to enable JavaScript to run this app.\</noscript>\<div id="root">\</div>\</body>\</html>\* Trying 127.0.0.1:443... \* Connected to 127.0.0.1 (127.0.0.1) port 443 \* ALPN: curl offers h2,http/1.1 \* TLSv1.3 (OUT), TLS handshake, Client hello (1): \* TLSv1.3 (IN), TLS handshake, Server hello (2): \* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8): \* TLSv1.3 (IN), TLS handshake, Certificate (11): \* TLSv1.3 (IN), TLS handshake, CERT verify (15): \* TLSv1.3 (IN), TLS handshake, Finished (20): \* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1): \* TLSv1.3 (OUT), TLS handshake, Finished (20): \* SSL connection using TLSv1.3 / TLS\_AES\_256\_GCM\_SHA384 / X25519 / RSASSA-PSS \* ALPN: server accepted http/1.1 \* Server certificate: \* subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* start date: Dec 7 05:39:06 2025 GMT \* expire date: Dec 7 05:39:06 2026 GMT \* issuer: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* SSL certificate verify result: self-signed certificate (18), continuing anyway. \* Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption \* using HTTP/1.x > GET /api/go/api/products HTTP/1.1 > Host: dev.local > User-Agent: curl/8.5.0 > Accept: \*/\* > \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* old SSL session ID is stale, removing < HTTP/1.1 200 OK < Server: nginx/1.29.4 < Date: Sun, 14 Dec 2025 08:36:37 GMT < Content-Type: application/json < Content-Length: 1306 < Connection: keep-alive < {"success":true,"message":"Products retrieved successfully","data":\[{"id":1,"name":"Laptop Pro 15","description":"High-performance laptop with 16GB RAM and 512GB SSD","price":1299.99,"quantity":25,"category":"Electronics","created\_at":"2025-07-25T08:01:39.000000Z","updated\_at":"2025-07-25T08:01:39.000000Z"},{"id":2,"name":"Wireless Headphones","description":"Noise-cancelling wireless headphones with 30h battery life","price":199.99,"quantity":50,"category":"Electronics","created\_at":"2025-07-25T08:01:39.000000Z","updated\_at":"2025-07-25T08:01:39.000000Z"},{"id":3,"name":"Coffee Maker","description":"Programmable coffee maker with 12-cup capacity","price":89.99,"quantity":15,"category":"Home \u0026 Kitchen","created\_at":"2025-07-25T08:01:39.000000Z","updated\_at":"2025-07-25T08:01:39.000000Z"},{"id":4,"name":"Running Shoes","description":"Lightweight running shoes with excellent cushioning","price":129.99,"quantity":30,"category":"Sports \u0026 Outdoors","created\_at":"2025-07-25T08:01:39.000000Z","updated\_at":"2025-07-25T08:01:39.000000Z"},{"id":5,"name":"Smartphone","description":"Latest smartphone with 128GB storage and triple camera","price":699.99,"quantity":40,"category":"Electronics","created\_at":"2025-07-25T08:01:39.000000Z","updated\_at":"2025-07-25T08:01:39.000000Z"}],"count":5} \* Connection #0 to host 127.0.0.1 left intact \* Trying 127.0.0.1:443... \* Connected to 127.0.0.1 (127.0.0.1) port 443 \* ALPN: curl offers h2,http/1.1 \* TLSv1.3 (OUT), TLS handshake, Client hello (1): \* TLSv1.3 (IN), TLS handshake, Server hello (2): \* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8): \* TLSv1.3 (IN), TLS handshake, Certificate (11): \* TLSv1.3 (IN), TLS handshake, CERT verify (15): \* TLSv1.3 (IN), TLS handshake, Finished (20): \* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1): \* TLSv1.3 (OUT), TLS handshake, Finished (20): \* SSL connection using TLSv1.3 / TLS\_AES\_256\_GCM\_SHA384 / X25519 / RSASSA-PSS \* ALPN: server accepted http/1.1 \* Server certificate: \* subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* start date: Dec 7 05:39:06 2025 GMT \* expire date: Dec 7 05:39:06 2026 GMT \* issuer: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* SSL certificate verify result: self-signed certificate (18), continuing anyway. \* Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption \* using HTTP/1.x > GET /api/laravel/api/products HTTP/1.1 > Host: dev.local > User-Agent: curl/8.5.0 > Accept: \*/\* > \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* old SSL session ID is stale, removing < HTTP/1.1 500 Internal Server Error < Server: nginx/1.29.4 < Date: Sun, 14 Dec 2025 08:36:38 GMT < Content-Type: application/json < Transfer-Encoding: chunked < Connection: keep-alive < X-Powered-By: PHP/8.2.29 < Cache-Control: private, must-revalidate < pragma: no-cache < expires: -1 < X-RateLimit-Limit: 60 < X-RateLimit-Remaining: 59 < Access-Control-Allow-Origin: \* < \* Connection #0 to host 127.0.0.1 left intact {"success":false,"message":"Error retrieving products","error":"SQLSTATE\[HY000] \[1045] Access denied for user 'threebody'@'172.19.0.4' (using password: YES) (SQL: select \* from `products`)"}cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner regicikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner register \ --url https://gitlab.com \ --token glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7 \ --executor shell \[sudo] password for cikal: Runtime platform arch=amd64 os=linux pid=180447 revision=5908bc19 version=18.6.5 Running in system-mode. Enter the GitLab instance URL (for example, https://gitlab.com/): \[https://gitlab.com]: Verifying runner... is valid correlation\_id=9adc61ba36b65f5c-ATL runner=ArpoA9OGu Enter a name for the runner. This is stored only in the local config.toml file: \[devops-ci]: Enter an executor: kubernetes, instance, shell, ssh, parallels, virtualbox, docker, docker-autoscaler, custom, docker-windows, docker+machine: \[shell]: Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml" cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 6m 5s

Oke, sekarang kondisinya sudah makin jelas:

* **Nginx + Frontend OK** (bisa buka `https://dev.local`)
* **Go API OK** (`/api/go/api/products` balik data)
* **Laravel API 500** dan errornya **MySQL “Access denied”**:

```
SQLSTATE[HY000] [1045] Access denied for user 'threebody'@'172.19.0.4'
```

Ini **hampir selalu** karena **password user MySQL yang tersimpan di volume `mysql_data` tidak sama** dengan `MYSQL_PASSWORD` yang sekarang dipakai container Laravel (env berubah, tapi user di MySQL nggak ikut berubah karena data MySQL sudah pernah di-init).

Di image `mysql:8`, variabel:

* `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DATABASE`\
  hanya dipakai **saat pertama kali /var/lib/mysql masih kosong**. Kalau volumenya sudah ada isinya, ubah env **tidak akan mengubah** password user yang sudah terbuat.

***

### 1) Pastikan kamu cuma pakai 1 stack (biar nggak ketuker DB/volume)

Di VM, jalankan:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep threebody
docker volume ls | grep threebody
```

Kalau masih ada stack lain yang nyangkut (misal `threebody-deploy-*`), hapus (kamu sudah sempat bersihin, ini sekadar memastikan):

```bash
docker rm -f $(docker ps -aq --filter "name=threebody-deploy") 2>/dev/null || true
```

***

### 2) Cek nilai env yang dipakai Laravel & MySQL (harus konsisten)

Jalankan ini dari folder yang ada `docker-compose.dev.yml` + `.env.dev.compose`:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel sh -lc 'printenv | egrep "DB_(HOST|DATABASE|USERNAME|PASSWORD)"'
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc 'printenv | egrep "MYSQL_(DATABASE|USER|PASSWORD|ROOT_PASSWORD)"'
```

Yang penting:

* Laravel: `DB_USERNAME=threebody`, `DB_PASSWORD=...`
* MySQL: `MYSQL_USER=threebody`, `MYSQL_PASSWORD=...`

Kalau **nilai passwordnya sama**, tapi Laravel tetap ditolak, berarti **password yang “nyata” di MySQL berbeda** (karena volume lama).

***

### 3) Fix utama (tanpa hapus data): reset password user `threebody` di MySQL agar sama dengan env sekarang

Copy-paste ini:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
ALTER USER '\''threebody'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_PASSWORD'\'';
FLUSH PRIVILEGES;
"
'
```

Kalau ternyata user `threebody` belum ada / host-nya beda, pakai versi yang lebih “aman” ini:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
CREATE USER IF NOT EXISTS '\''threebody'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_PASSWORD'\'';
GRANT ALL PRIVILEGES ON threebody.* TO '\''threebody'\''@'\''%'\'' ;
FLUSH PRIVILEGES;
"
'
```

Lalu restart laravel:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml restart laravel
```

Tes lagi:

```bash
curl -vk https://127.0.0.1/api/laravel/api/products -H "Host: dev.local"
```

Harusnya sudah **200**.

***

### 4) Opsi “nuklir” (paling gampang untuk DEV): hapus volume DB lalu init ulang

Kalau ini environment dev dan gapapa DB direset:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

Ini akan menghilangkan problem password & migration “nyangkut” karena DB fresh.

***

### 5) Kenapa pipeline kamu sempat gagal “products already exists”?

Log pipeline kamu sebelumnya gagal di migration karena:

* tabel `products` **sudah ada**
* tapi di tabel `migrations`, migration `create_products_table` dianggap **belum pernah jalan**
* jadi Laravel mencoba `create table products` lagi → error `1050 table exists`

Kalau kamu **tidak reset DB**, cara beresinnya ada 2:

#### A) Drop table `products` lalu migrate ulang

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -D threebody -e "DROP TABLE IF EXISTS products;"
'
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

#### B) Tandai migration-nya “sudah pernah jalan” (kalau kamu yakin tabelnya memang sama)

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -D threebody -e "
INSERT IGNORE INTO migrations(migration, batch)
VALUES (\"2025_07_25_075615_create_products_table\", (SELECT IFNULL(MAX(batch),0)+1 FROM migrations));
"
'
```

***

### 6) Catatan penting soal deploy job pakai Docker runner vs Shell runner

Kamu sempat bikin `DEPLOY_DIR=/home/cikal/threebody-deploy`. Kalau **deploy job masih jalan di Docker executor**, path itu **ada di container job**, bukan benar-benar “di host” (VM), dan bind-mount bisa bikin folder kosong/aneh di host.

Kalau kamu memang mau deploy “beneran di VM” pakai folder `/home/...`, maka deploy stage **wajib** jalan di **shell runner** (bukan docker runner). Dan karena GitLab sekarang melarang `--tag-list` saat register pakai _runner authentication token_, tag `shell` harus kamu set dari UI GitLab (Settings → CI/CD → Runners → edit runner → Tags).

***

Kalau kamu jalankan step **(3)** di atas (ALTER USER), biasanya Laravel langsung normal.\
Kalau setelah itu masih 500, kirim output ini (tanpa sensor juga boleh, tapi kalau mau aman boleh disamarkan passwordnya):

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --tail=200 laravel
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel sh -lc 'php -v && php artisan migrate:status || true'
```

cikal@devops-ci:\~/three-body-problem-main$ docker ps --format "table \{{.Names\}}\t\{{.Ports\}}" | grep threebody docker volume ls | grep threebody threebody-dev-nginx-dev-1 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp threebody-dev-frontend-1 80/tcp threebody-dev-laravel-1 80/tcp threebody-dev-goapi-1 8080/tcp threebody-dev-mysql-1 3306/tcp, 33060/tcp local threebody-deploy\_mysql\_data local threebody-dev\_mysql\_data cikal@devops-ci:\~/three-body-problem-main$ docker rm -f $(docker ps -aq --filter "name=threebody-deploy") 2>/dev/null || true cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel sh -lc 'printenv | egrep "DB\_(HOST|DATABASE|USERNAME|PASSWORD)"' docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc 'printenv | egrep "MYSQL\_(DATABASE|USER|PASSWORD|ROOT\_PASSWORD)"' DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=UserPassDev123! DB\_HOST=mysql MYSQL\_ROOT\_PASSWORD=RootPassDev123! MYSQL\_PASSWORD=UserPassDev123! MYSQL\_USER=threebody MYSQL\_DATABASE=threebody cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " ALTER USER '\\''threebody'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; FLUSH PRIVILEGES; " ' mysql: \[Warning] Using a password on the command line interface can be insecure. ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " CREATE USER IF NOT EXISTS '\\''threebody'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; GRANT ALL PRIVILEGES ON threebody.\* TO '\\''threebody'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' mysql: \[Warning] Using a password on the command line interface can be insecure. ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 4m 19s

Sip, outputmu **jelas banget**: env di container sudah benar, tapi kamu **nggak bisa login sebagai `root`** ke MySQL:

```
ERROR 1045: Access denied for user 'root'@'localhost'
```

Ini artinya **password root yang “sebenarnya” di MySQL (yang tersimpan di volume `threebody-dev_mysql_data`) TIDAK sama** dengan `MYSQL_ROOT_PASSWORD` yang sekarang kamu set di `.env.dev.compose`.

> Di image `mysql:8`, `MYSQL_ROOT_PASSWORD/MYSQL_PASSWORD` hanya dipakai **saat pertama kali volume masih kosong**. Kalau volume sudah berisi data, ubah env **tidak mengubah** password user di MySQL.

Selain itu, ada kemungkinan kecil root di MySQL kamu **tidak bisa login via socket `localhost`**, tapi bisa via TCP `127.0.0.1`. Jadi kita coba yang paling aman dulu.

***

## A) Coba login root lewat TCP (sering jadi penyebab “root@localhost” gagal)

Jalankan ini:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -h127.0.0.1 -e "SELECT 1;"
'
```

* Kalau keluar `1` → berarti root bisa login (via TCP), lanjut ke langkah **B**.
* Kalau tetap `ERROR 1045` → password root di volume memang beda → lompat ke opsi **C** (reset volume) atau **D** (reset password tanpa hapus data).

***

## B) Kalau root bisa login, perbaiki user `threebody` supaya Laravel nggak ditolak

Jalankan:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -h127.0.0.1 -e "
CREATE USER IF NOT EXISTS '\''threebody'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_PASSWORD'\'';
ALTER USER '\''threebody'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_PASSWORD'\'';
GRANT ALL PRIVILEGES ON threebody.* TO '\''threebody'\''@'\''%'\'' ;
FLUSH PRIVILEGES;
"
'
```

Lalu restart Laravel:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml restart laravel
```

Tes:

```bash
curl -vk https://127.0.0.1/api/laravel/api/products -H "Host: dev.local"
```

***

## C) Cara paling gampang untuk DEV: reset DB (hapus volume) ✅

Kalau ini environment dev dan **nggak masalah data hilang**, ini yang paling cepat & paling bersih:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
docker volume rm threebody-dev_mysql_data 2>/dev/null || true
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

Setelah itu coba lagi endpoint Laravel.

> Bonus: kamu juga punya volume lama `threebody-deploy_mysql_data` yang tidak kepakai. Biar nggak bikin bingung, boleh hapus:

```bash
docker volume rm threebody-deploy_mysql_data
```

***

## D) Kalau data mau dipertahankan: reset password root pakai `--skip-grant-tables`

Ini “jalan darurat” tanpa hapus data.

1. Stop MySQL (supaya volume tidak dipakai dua mysqld sekaligus):

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml stop mysql
```

2. Jalankan MySQL sementara tanpa autentikasi:

```bash
docker run -d --name mysql-reset \
  -v threebody-dev_mysql_data:/var/lib/mysql \
  mysql:8.0 mysqld --skip-grant-tables --skip-networking
```

3. Reset password root & threebody (pakai nilai dari `.env.dev.compose` yang kamu mau):

```bash
docker exec -i mysql-reset mysql -uroot <<'SQL'
FLUSH PRIVILEGES;
-- set root (buat jaga-jaga set keduanya)
ALTER USER 'root'@'localhost' IDENTIFIED BY 'RootPassDev123!';
ALTER USER 'root'@'%'       IDENTIFIED BY 'RootPassDev123!';

-- set user app
CREATE USER IF NOT EXISTS 'threebody'@'%' IDENTIFIED BY 'UserPassDev123!';
ALTER USER 'threebody'@'%' IDENTIFIED BY 'UserPassDev123!';
GRANT ALL PRIVILEGES ON threebody.* TO 'threebody'@'%';
FLUSH PRIVILEGES;
SQL
```

> Kalau mau, kamu ganti `RootPassDev123!` dan `UserPassDev123!` sesuai yang ada di `.env.dev.compose`.

4. Matikan container reset, lalu start MySQL normal lagi:

```bash
docker rm -f mysql-reset
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml start mysql
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml restart laravel
```

Tes lagi endpoint Laravel.

***

## E) Kalau setelah itu Laravel masih “Access denied”, kemungkinan Laravel pakai config cache lama

Coba bersihkan cache config Laravel:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml restart laravel
```

***

Kalau kamu mau yang paling cepat, **C (reset volume)** biasanya langsung beres.

Kalau kamu jalankan langkah **A** (root via TCP) dan masih gagal, kamu bisa langsung lompat ke **C** atau **D**.

cikal@devops-ci:\~/three-body-problem-main$ su - Password: root@devops-ci:\~# docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -h127.0.0.1 -e "SELECT 1;" ' couldn't find env file: /root/.env.dev.compose root@devops-ci:\~# su - cikal cikal@devops-ci:\~$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -h127.0.0.1 -e "SELECT 1;" ' couldn't find env file: /home/cikal/.env.dev.compose cikal@devops-ci:\~$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -h127.0.0.1 -e " CREATE USER IF NOT EXISTS '\\''threebody'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; ALTER USER '\\''threebody'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; GRANT ALL PRIVILEGES ON threebody.\* TO '\\''threebody'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' couldn't find env file: /home/cikal/.env.dev.compose cikal@devops-ci:\~$ su - Password: root@devops-ci:\~# docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -h127.0.0.1 -e " CREATE USER IF NOT EXISTS '\\''threebody'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; ALTER USER '\\''threebody'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; GRANT ALL PRIVILEGES ON threebody.\* TO '\\''threebody'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' couldn't find env file: /root/.env.dev.compose root@devops-ci:\~# docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v docker volume rm threebody-dev\_mysql\_data 2>/dev/null || true docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force couldn't find env file: /root/.env.dev.compose couldn't find env file: /root/.env.dev.compose couldn't find env file: /root/.env.dev.compose root@devops-ci:\~# su - cikal cikal@devops-ci:\~$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v docker volume rm threebody-dev\_mysql\_data 2>/dev/null || true docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force couldn't find env file: /home/cikal/.env.dev.compose couldn't find env file: /home/cikal/.env.dev.compose couldn't find env file: /home/cikal/.env.dev.compose cikal@devops-ci:\~$ docker volume rm threebody-deploy\_mysql\_data threebody-deploy\_mysql\_data cikal@devops-ci:\~$Selesai berpikir selama 42s

Itu error-nya **bukan MySQL lagi**, tapi karena kamu menjalankan `docker compose` dari folder yang **tidak punya** file `.env.dev.compose` dan `docker-compose.dev.yml`.

Sekarang prompt kamu ada di:

* `root@devops-ci:~#` → folder kerja: `/root`
* `cikal@devops-ci:~$` → folder kerja: `/home/cikal`

Padahal `.env.dev.compose` dan `docker-compose.dev.yml` ada di folder project kamu (contoh sebelumnya: `~/three-body-problem-main`), jadi ketika kamu tulis `--env-file .env.dev.compose`, compose nyari di folder saat ini → **ketemu “couldn't find env file”**.

***

### Cara paling aman: pakai path absolut (nggak peduli kamu lagi di folder mana)

> Ganti `APP_DIR` sesuai lokasi project kamu. Dari log kamu sebelumnya: `/home/cikal/three-body-problem-main`

Jalankan sebagai **cikal** (atau root juga boleh, karena path absolut):

```bash
APP_DIR=/home/cikal/three-body-problem-main

# pastikan filenya memang ada
ls -la "$APP_DIR/.env.dev.compose" "$APP_DIR/docker-compose.dev.yml"
```

Kalau file ketemu, lanjut reset DB (opsi C yang kamu coba tadi) dengan benar:

```bash
docker compose -p threebody-dev \
  --env-file "$APP_DIR/.env.dev.compose" \
  -f "$APP_DIR/docker-compose.dev.yml" down -v

docker volume rm -f threebody-dev_mysql_data 2>/dev/null || true

docker compose -p threebody-dev \
  --env-file "$APP_DIR/.env.dev.compose" \
  -f "$APP_DIR/docker-compose.dev.yml" up -d

docker compose -p threebody-dev \
  --env-file "$APP_DIR/.env.dev.compose" \
  -f "$APP_DIR/docker-compose.dev.yml" exec -T laravel php artisan migrate --force

docker compose -p threebody-dev \
  --env-file "$APP_DIR/.env.dev.compose" \
  -f "$APP_DIR/docker-compose.dev.yml" exec -T laravel php artisan optimize:clear
```

Tes endpoint Laravel:

```bash
curl -vk https://127.0.0.1/api/laravel/api/products -H "Host: dev.local"
```

Kalau ini sukses, error 1045 “Access denied” harusnya hilang karena DB dibuat ulang sesuai env terbaru.

***

### Alternatif: cukup `cd` ke folder yang benar (paling simpel)

```bash
cd /home/cikal/three-body-problem-main
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
docker volume rm -f threebody-dev_mysql_data 2>/dev/null || true
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

***

### Catatan penting (biar nggak keulang)

File compose kamu punya mount relatif:

```yml
volumes:
  - ./devops/nginx/dev.conf:...
  - ./certs/dev:...
```

Artinya `docker compose` **harus dijalankan dari folder yang benar** (yang punya `devops/` dan `certs/`). Kalau dijalankan dari `/home/cikal` atau `/root`, mount path jadi salah.

***

Kalau setelah reset volume Laravel masih 500, kirim output ini ya (cukup 20-30 baris):

```bash
APP_DIR=/home/cikal/three-body-problem-main
docker compose -p threebody-dev --env-file "$APP_DIR/.env.dev.compose" -f "$APP_DIR/docker-compose.dev.yml" logs --tail=80 laravel
```

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000006a307207a15cc653d3ce119f\&ts=490475\&p=fs\&cid=1\&sig=e6b492ac72776aecc09c8e31614d0cfa25d5b77011bbf6236b36be3de955faa9\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000bce87207b20d667b389299fa\&ts=490475\&p=fs\&cid=1\&sig=14b36d9e1003b68538756ffd18d23353e6675b863d6e9e0fa89c5d228908ff6a\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000002af071faa336dba0f71f490b\&ts=490475\&p=fs\&cid=1\&sig=42fa5d011533e6932fa9e44ddbef979f51b5d5ccd90922a9612f746d04a5f367\&v=0)cikal@devops-ci:\~$ APP\_DIR=/home/cikal/three-body-problem-main # pastikan filenya memang ada ls -la "$APP\_DIR/.env.dev.compose" "$APP\_DIR/docker-compose.dev.yml" -rw-rw-r-- 1 cikal cikal 204 Dec 13 17:17 /home/cikal/three-body-problem-main/.env.dev.compose -rw-rw-r-- 1 cikal cikal 1897 Dec 13 20:29 /home/cikal/three-body-problem-main/docker-compose.dev.yml cikal@devops-ci:\~$ docker compose -p threebody-dev \ --env-file "$APP\_DIR/.env.dev.compose" \ -f "$APP\_DIR/docker-compose.dev.yml" down -v docker volume rm -f threebody-dev\_mysql\_data 2>/dev/null || true docker compose -p threebody-dev \ --env-file "$APP\_DIR/.env.dev.compose" \ -f "$APP\_DIR/docker-compose.dev.yml" up -d docker compose -p threebody-dev \ --env-file "$APP\_DIR/.env.dev.compose" \ -f "$APP\_DIR/docker-compose.dev.yml" exec -T laravel php artisan migrate --force docker compose -p threebody-dev \ --env-file "$APP\_DIR/.env.dev.compose" \ -f "$APP\_DIR/docker-compose.dev.yml" exec -T laravel php artisan optimize:clear \[+] down 7/7 ✔ Container threebody-dev-nginx-dev-1 Removed 1.9s ✔ Container threebody-dev-frontend-1 Removed 1.1s ✔ Container threebody-dev-laravel-1 Removed 2.7s ✔ Container threebody-dev-goapi-1 Removed 2.1s ✔ Container threebody-dev-mysql-1 Removed 3.1s ✔ Volume threebody-dev\_mysql\_data Removed 0.3s ✔ Network threebody-dev\_devnet Removed 0.7s threebody-dev\_mysql\_data \[+] up 7/7 ✔ Network threebody-dev\_devnet Created 0.4s ✔ Volume threebody-dev\_mysql\_data Created 0.0s ✔ Container threebody-dev-mysql-1 Healthy 141.9s ✔ Container threebody-dev-goapi-1 Created 1.2s ✔ Container threebody-dev-laravel-1 Created 1.4s ✔ Container threebody-dev-frontend-1 Created 0.7s ✔ Container threebody-dev-nginx-dev-1 Created 0.6s Migration table created successfully. Migrating: 2014\_10\_12\_000000\_create\_users\_table Migrated: 2014\_10\_12\_000000\_create\_users\_table (820.36ms) Migrating: 2014\_10\_12\_100000\_create\_password\_resets\_table Migrated: 2014\_10\_12\_100000\_create\_password\_resets\_table (558.86ms) Migrating: 2019\_08\_19\_000000\_create\_failed\_jobs\_table Migrated: 2019\_08\_19\_000000\_create\_failed\_jobs\_table (386.44ms) Migrating: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table Migrated: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table (699.80ms) Migrating: 2025\_07\_25\_075615\_create\_products\_table \[2025-12-14 09:02:56] local.ERROR: SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' already exists (SQL: create table `products` (`id` bigint unsigned not null auto\_increment primary key, `name` varchar(255) not null, `description` text null, `price` decimal(10, 2) not null, `quantity` int not null default '0', `category` varchar(255) null, `created_at` timestamp null, `updated_at` timestamp null) default character set utf8mb4 collate 'utf8mb4\_unicode\_ci') {"exception":"\[object] (Illuminate\\\Database\\\QueryException(code: 42S01): SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' already exists (SQL: create table `products` (`id` bigint unsigned not null auto\_increment primary key, `name` varchar(255) not null, `description` text null, `price` decimal(10, 2) not null, `quantity` int not null default '0', `category` varchar(255) null, `created_at` timestamp null, `updated_at` timestamp null) default character set utf8mb4 collate 'utf8mb4\_unicode\_ci') at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:712) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('create table `p...', Array, Object(Closure))` #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(490): Illuminate\\\Database\\\Connection->run('create table `p...', Array, Object(Closure))` #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Blueprint.php(109): Illuminate\\\Database\\\Connection->statement('create table `p...')` #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(364): Illuminate\\\Database\\\Schema\\\Blueprint->build(Object(Illuminate\\\Database\\\MySqlConnection), Object(Illuminate\\\Database\\\Schema\\\Grammars\\\MySqlGrammar)) #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(223): Illuminate\\\Database\\\Schema\\\Builder->build(Object(Illuminate\\\Database\\\Schema\\\Blueprint)) #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Support/Facades/Facade.php(261): Illuminate\\\Database\\\Schema\\\Builder->create('products', Object(Closure)) #6 /var/www/html/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php(16): Illuminate\\\Support\\\Facades\\\Facade::\_\_callStatic('create', Array) #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(472): CreateProductsTable->up() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(394): Illuminate\\\Database\\\Migrations\\\Migrator->runMethod(Object(Illuminate\\\Database\\\MySqlConnection), Object(CreateProductsTable), 'up') #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(403): Illuminate\\\Database\\\Migrations\\\Migrator->Illuminate\\\Database\\\Migrations\\\\{closure}() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(202): Illuminate\\\Database\\\Migrations\\\Migrator->runMigration(Object(CreateProductsTable), 'up') #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(167): Illuminate\\\Database\\\Migrations\\\Migrator->runUp('/var/www/html/d...', 1, false) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(112): Illuminate\\\Database\\\Migrations\\\Migrator->runPending(Array, Array) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(84): Illuminate\\\Database\\\Migrations\\\Migrator->run(Array, Array) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #22 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #24 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #25 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #26 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #27 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 {main} \[previous exception] \[object] (PDOException(code: 42S01): SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' already exists at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:501) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(501): PDOStatement->execute() #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(705): Illuminate\\\Database\\\Connection->Illuminate\\\Database\\\\{closure}('create table `p...', Array)` #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('create table `p...', Array, Object(Closure))` #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(490): Illuminate\\\Database\\\Connection->run('create table `p...', Array, Object(Closure))` #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Blueprint.php(109): Illuminate\\\Database\\\Connection->statement('create table `p...')` #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(364): Illuminate\\\Database\\\Schema\\\Blueprint->build(Object(Illuminate\\\Database\\\MySqlConnection), Object(Illuminate\\\Database\\\Schema\\\Grammars\\\MySqlGrammar)) #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/Builder.php(223): Illuminate\\\Database\\\Schema\\\Builder->build(Object(Illuminate\\\Database\\\Schema\\\Blueprint)) #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Support/Facades/Facade.php(261): Illuminate\\\Database\\\Schema\\\Builder->create('products', Object(Closure)) #8 /var/www/html/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php(16): Illuminate\\\Support\\\Facades\\\Facade::\_\_callStatic('create', Array) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(472): CreateProductsTable->up() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(394): Illuminate\\\Database\\\Migrations\\\Migrator->runMethod(Object(Illuminate\\\Database\\\MySqlConnection), Object(CreateProductsTable), 'up') #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(403): Illuminate\\\Database\\\Migrations\\\Migrator->Illuminate\\\Database\\\Migrations\\\\{closure}() #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(202): Illuminate\\\Database\\\Migrations\\\Migrator->runMigration(Object(CreateProductsTable), 'up') #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(167): Illuminate\\\Database\\\Migrations\\\Migrator->runUp('/var/www/html/d...', 1, false) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(112): Illuminate\\\Database\\\Migrations\\\Migrator->runPending(Array, Array) #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(84): Illuminate\\\Database\\\Migrations\\\Migrator->run(Array, Array) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #22 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #24 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #25 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #26 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #27 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #31 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #32 {main} "} In Connection.php line 712: SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' a lready exists (SQL: create table `products` (`id` bigint unsigned not null auto\_increment primary key, `name` varchar(255) not null, `description` tex t null, `price` decimal(10, 2) not null, `quantity` int not null default '0 ', `category` varchar(255) null, `created_at` timestamp null, `updated_at` timestamp null) default character set utf8mb4 collate 'utf8mb4\_unicode\_ci') In Connection.php line 501: SQLSTATE\[42S01]: Base table or view already exists: 1050 Table 'products' a lready exists Cached events cleared! Compiled views cleared! Application cache cleared! Route cache cleared! Configuration cache cleared! Compiled services and packages files removed! Caches cleared successfully! cikal@devops-ci:\~$ curl -vk https://127.0.0.1/api/laravel/api/products -H "Host: dev.local" \* Trying 127.0.0.1:443... \* Connected to 127.0.0.1 (127.0.0.1) port 443 \* ALPN: curl offers h2,http/1.1 \* TLSv1.3 (OUT), TLS handshake, Client hello (1): \* TLSv1.3 (IN), TLS handshake, Server hello (2): \* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8): \* TLSv1.3 (IN), TLS handshake, Certificate (11): \* TLSv1.3 (IN), TLS handshake, CERT verify (15): \* TLSv1.3 (IN), TLS handshake, Finished (20): \* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1): \* TLSv1.3 (OUT), TLS handshake, Finished (20): \* SSL connection using TLSv1.3 / TLS\_AES\_256\_GCM\_SHA384 / X25519 / RSASSA-PSS \* ALPN: server accepted http/1.1 \* Server certificate: \* subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* start date: Dec 7 05:39:06 2025 GMT \* expire date: Dec 7 05:39:06 2026 GMT \* issuer: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd \* SSL certificate verify result: self-signed certificate (18), continuing anyway. \* Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption \* using HTTP/1.x > GET /api/laravel/api/products HTTP/1.1 > Host: dev.local > User-Agent: curl/8.5.0 > Accept: \*/\* > \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4): \* old SSL session ID is stale, removing < HTTP/1.1 200 OK < Server: nginx/1.29.4 < Date: Sun, 14 Dec 2025 09:03:38 GMT < Content-Type: application/json < Transfer-Encoding: chunked < Connection: keep-alive < X-Powered-By: PHP/8.2.29 < Cache-Control: private, must-revalidate < pragma: no-cache < expires: -1 < X-RateLimit-Limit: 60 < X-RateLimit-Remaining: 59 < Access-Control-Allow-Origin: \* < {"success":true,"message":"Products retrieved successfully","data":\[{"id":1,"name":"Laptop Pro 15","description":"High-performance laptop with 16GB RAM and 512GB SSD","price":"1299.99","quantity":25,"category":"Electronics","created\_at":"2025-12-14T09:02:45.000000Z","updated\_at":"2025-12-14T09:02:45.000000Z"},{"id":2,"name":"Wireless Headphones","description":"Noise-cancelling wireless headphones with 30h battery life","price":"199.99","quantity":50,"category":"Electronics","created\_at":"2025-12-14T09:02:45.000000Z","updated\_at":"2025-12-14T09:02:45.000000Z"},{"id":3,"name":"Coffee Maker","description":"Programmable coffee maker with 12-cup capacity","price":"89.99","quantity":15,"category":"Home & Kitchen","created\_at":"2025-12-14T09:02:45.000000Z","updated\_at":"2025-12-14T09:02:45.000000Z"},{"id":4,"name":"Running Shoes","description":"Lightweight running shoes with excellent cushioning","price":"129.99","quantity":30,"category":"Sports & Outdoors","created\_at":"2025-12-14T09:02:45.000000Z","updated\_at":"20\* Connection #0 to host 127.0.0.1 left intact 25-12-14T09:02:45.000000Z"},{"id":5,"name":"Smartphone","description":"Latest smartphone with 128GB storage and triple camera","price":"699.99","quantity":40,"category":"Electronics","created\_at":"2025-12-14T09:02:45.000000Z","updated\_at":"2025-12-14T09:02:45.000000Z"}],"count":5}cikal@devops-ci:\~$ cikal@devops-ci:\~$ APP\_DIR=/home/cikal/three-body-problem-main docker compose -p threebody-dev --env-file "$APP\_DIR/.env.dev.compose" -f "$APP\_DIR/docker-compose.dev.yml" logs --tail=80 laravel laravel-1 | \[Sun Dec 14 09:02:44.597380 2025] \[mpm\_prefork:notice] \[pid 1:tid 1] AH00163: Apache/2.4.65 (Debian) PHP/8.2.29 configured -- resuming normal operations laravel-1 | \[Sun Dec 14 09:02:44.599545 2025] \[core:notice] \[pid 1:tid 1] AH00094: Command line: 'apache2 -D FOREGROUND' laravel-1 | 172.19.0.6 - - \[14/Dec/2025:09:03:36 +0000] "GET /api/products HTTP/1.0" 200 1624 "-" "curl/8.5.0" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:09:04:06 +0000] "GET /api/products HTTP/1.0" 200 1624 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:09:04:14 +0000] "GET /api/products HTTP/1.0" 200 1624 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:09:04:16 +0000] "GET /api/products HTTP/1.0" 200 1624 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:09:04:29 +0000] "GET /api/products HTTP/1.0" 200 1624 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" cikal@devops-ci:\~$ mantap, sekarang sudah bisa lanjut ke ci nya? atau masih ada yang perlu di rapihkan lagi?Selesai berpikir selama 3m 4s

Iya, **routing Nginx + Go API + Laravel API kamu sudah beres** (sekarang 200 semua). Tapi sebelum “lanjut CI” sepenuhnya, ada **3 hal yang sebaiknya kamu rapihkan** supaya pipeline **nggak gagal lagi** dan deploy-nya stabil.

***

### 1) Beresin akar error `products already exists` (ini bikin job deploy bisa fail)

Kamu sudah lihat ini masih muncul saat `php artisan migrate`:

> `Table 'products' already exists`

Ini hampir pasti karena **Go API bikin tabel `products` duluan** (misalnya pakai GORM `AutoMigrate()`), lalu **Laravel migration** mencoba `Schema::create('products')` → bentrok.

✅ Solusi paling gampang & aman: **buat migration Laravel idempotent** (kalau tabel sudah ada, skip).

Edit file migration Laravel:\
`laravel/database/migrations/2025_07_25_075615_create_products_table.php`

Ubah jadi seperti ini:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        if (Schema::hasTable('products')) {
            return; // table sudah ada, jangan create lagi
        }

        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->text('description')->nullable();
            $table->decimal('price', 10, 2);
            $table->integer('quantity')->default(0);
            $table->string('category')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

Lalu build ulang image Laravel (karena migration masuk image), push, dan deploy lagi.

> Alternatif “paling ideal”: matikan auto-migrate di Go, dan biarkan **Laravel yang jadi single source of truth** untuk schema. Tapi itu butuh ubah code Go.

***

### 2) Rapihkan deploy directory: jangan taruh di `/home/cikal/...` kalau runner-nya user `gitlab-runner`

Shell runner saat pipeline jalan biasanya pakai user **gitlab-runner**. Walau folder `/home/cikal/threebody-deploy` kamu `chown` ke gitlab-runner, **parent folder `/home/cikal` sering 750**, jadi `gitlab-runner` nggak bisa “masuk” (traverse).

✅ Paling aman: pindahkan deploy dir ke yang memang “punya runner”, misalnya:

#### Opsi A (recommended): `/home/gitlab-runner/threebody-deploy`

```bash
sudo mkdir -p /home/gitlab-runner/threebody-deploy
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/threebody-deploy
```

Lalu di `.gitlab-ci.yml`:

```bash
DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
```

#### Opsi B: pakai `/srv/threebody-deploy`

```bash
sudo mkdir -p /srv/threebody-deploy
sudo chown -R gitlab-runner:gitlab-runner /srv/threebody-deploy
sudo chmod 755 /srv
```

***

### 3) Pastikan tag runner **bener** (biar build pakai docker-runner, deploy pakai shell-runner)

Kamu sudah ubah `deploy_dev` jadi:

```yaml
deploy_dev:
  tags: ["shell"]
```

Bagus. Tapi kamu sempat kena error saat `gitlab-runner register` karena **sekarang tag itu di-set di GitLab UI**, bukan lewat CLI (workflow baru GitLab Runner).

✅ Yang perlu kamu cek di GitLab UI:\
Project → **Settings → CI/CD → Runners**

* Runner docker: punya tag **docker**
* Runner shell: punya tag **shell**
* Runner shell status **online**

✅ Di VM juga cek servicenya jalan:

```bash
sudo systemctl enable --now gitlab-runner
sudo systemctl status gitlab-runner
sudo gitlab-runner verify
```

Kalau runner shell masih “offline”, biasanya service belum running / blocked.

***

### Bonus rapih (biar nggak keulang “Access denied MySQL”)

Kasus `Access denied for user 'threebody'` yang kemarin itu karena **volume MySQL lama masih kepakai tapi password env sudah berubah**. Jadi:

* Pastikan `DEV_MYSQL_ROOT_PASSWORD` dan `DEV_MYSQL_PASSWORD` **stabil** (jangan sering ganti).
* Kalau terpaksa ganti password, kamu memang harus **reset user MySQL** atau recreate volume (yang artinya data hilang).

***

## Kesimpulan

✅ **Bisa lanjut CI**, tapi sebelum itu **wajib rapihin minimal:**

1. Migration Laravel “skip kalau table sudah ada” (biar deploy job nggak fail)
2. Deploy dir pindah ke lokasi yang pasti bisa diakses `gitlab-runner`
3. Tag runner di GitLab UI: docker vs shell sudah benar & shell runner online

Kalau kamu mau, paste isi file migration `create_products_table` dan (kalau ada) bagian startup Go yang connect DB/migrate, nanti aku tunjukkan mana yang paling “bersih”: schema dikontrol Laravel saja atau tetap dua-duanya tapi aman.

berikan arahan yang lebih detail dan lengkap agar saya mudah memahaminya dan menjalankan tanpa ada kendala untuk merapihkan 3 hal tersebut contoh seperti dibawah ini Edit file migration Laravel: laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php itu saya tidak tahu dimana tempatnya beradaSelesai berpikir selama 36s

Siap. Aku bikin **panduan super detail** untuk merapihkan **3 hal** itu, lengkap dengan **lokasi file, perintah terminal, dan cara verifikasi**. Kamu tinggal ikuti langkahnya satu-per-satu.

Di bawah ini aku asumsikan repo kamu ada di:\
`/home/cikal/three-body-problem-main`

Kalau beda, nanti tinggal sesuaikan.

***

## 0) Pastikan kamu sedang di folder project yang benar

Di VM (devops-ci), jalankan:

```bash
cd /home/cikal/three-body-problem-main
pwd
ls -la
```

Harus kelihatan folder seperti: `laravel/`, `go/`, `frontend/`, `devops/`, `docker-compose.dev.yml`, `.gitlab-ci.yml`.

Kalau folder `laravel/` tidak ada, berarti kamu belum di root repo yang benar.

***

## 1) Rapihin Migration Laravel (biar tidak fail “products already exists”)

### 1A) Cari lokasi file migration “create products table”

Kamu bilang tidak tahu tempatnya. Cara paling aman: **cari pakai find**.

Dari root repo:

```bash
cd /home/cikal/three-body-problem-main

# lihat semua migration
ls -la laravel/database/migrations

# cari yang ada kata "products"
ls -la laravel/database/migrations/*products* 2>/dev/null || true

# kalau belum ketemu, cari pakai find
find laravel/database/migrations -maxdepth 1 -type f | grep -i products
```

Biasanya ketemu file seperti:\
`laravel/database/migrations/2025_07_25_075615_create_products_table.php`

Kalau nama tanggalnya beda, tidak masalah — yang penting ada “create\_products\_table”.

> Kalau kamu mau cek juga isi migration di container Laravel:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel ls -la /var/www/html/database/migrations | grep -i products
```

Tapi **yang diedit tetap file di repo**, bukan di container.

***

### 1B) Edit file migration-nya

Misal file kamu ketemu dan namanya ini:

`laravel/database/migrations/2025_07_25_075615_create_products_table.php`

Buka dengan nano:

```bash
nano laravel/database/migrations/2025_07_25_075615_create_products_table.php
```

Lalu ubah fungsi `up()` jadi “aman” (skip kalau tabel sudah ada). Contoh full yang aman:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        if (Schema::hasTable('products')) {
            return; // table sudah ada -> jangan create lagi
        }

        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->text('description')->nullable();
            $table->decimal('price', 10, 2);
            $table->integer('quantity')->default(0);
            $table->string('category')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

Simpan nano:

* `CTRL + O` → Enter
* `CTRL + X`

***

### 1C) Commit & push (biar masuk build image Laravel)

Karena pipeline kamu build image dari repo, perubahan ini harus di-commit:

```bash
git status
git add laravel/database/migrations/2025_07_25_075615_create_products_table.php
git commit -m "Make products migration idempotent (skip if table exists)"
git push
```

✅ Setelah ini, pipeline berikutnya harus **tidak gagal** lagi di step `php artisan migrate --force`.

***

### 1D) Verifikasi setelah deploy (cek migrate status)

Setelah deploy jalan, cek:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate:status | head
```

Kalau sebelumnya tabel `products` sudah ada dari Go, migration Laravel untuk products akan **skip create**, sehingga job deploy tidak fail.

***

## 2) Rapihin Deploy Directory (biar shell runner tidak “permission denied”)

Masalahnya biasanya:

* Runner jalan sebagai user `gitlab-runner`
* Folder `/home/cikal/...` kadang tidak bisa diakses karena permission parent folder (`/home/cikal`)

✅ Solusi paling aman: pakai deploy dir di home runner.

### 2A) Buat deploy dir yang aman

Jalankan ini:

```bash
sudo mkdir -p /home/gitlab-runner/threebody-deploy
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/threebody-deploy
sudo chmod 755 /home/gitlab-runner
```

Cek aksesnya:

```bash
sudo -u gitlab-runner ls -la /home/gitlab-runner/threebody-deploy
```

Harus bisa tanpa error.

***

### 2B) Update `.gitlab-ci.yml` bagian deploy

Buka file:

```bash
nano .gitlab-ci.yml
```

Cari ini di job `deploy_dev`:

```bash
DEPLOY_DIR="/home/cikal/threebody-deploy"
```

Ganti jadi:

```bash
DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
```

Simpan, lalu commit:

```bash
git add .gitlab-ci.yml
git commit -m "Use deploy dir under /home/gitlab-runner"
git push
```

✅ Setelah ini, shell runner tidak akan mentok permission.

***

## 3) Rapihin Runner Tags (build pakai docker-runner, deploy pakai shell-runner)

Tujuan kita:

* Job build/validate jalan di runner **docker** (tag: `docker`)
* Job deploy jalan di runner **shell** (tag: `shell`)

Kamu sudah set:

```yaml
default:
  tags: ["docker"]

deploy_dev:
  tags: ["shell"]
```

Itu sudah benar.

Sekarang pastikan runner “shell” kamu benar-benar siap.

***

### 3A) Pastikan service gitlab-runner hidup di VM

Jalankan:

```bash
sudo systemctl enable --now gitlab-runner
sudo systemctl status gitlab-runner --no-pager
```

Lalu cek runner terdeteksi:

```bash
sudo gitlab-runner verify
```

Kalau verify OK, berarti runner aktif.

***

### 3B) Pastikan runner shell punya tag `shell` di GitLab UI

Karena kamu tadi dapat pesan bahwa **tag tidak bisa diset dari CLI** untuk “runner authentication token”, maka tag harus diatur dari GitLab UI.

Masuk GitLab:\
Project → **Settings → CI/CD → Runners**

Cari runner shell kamu (yang baru). Pastikan:

* status **Online**
* tag ada **shell**
* “Run untagged” = OFF (boleh, karena kamu pakai tags)
* runner docker tetap ada tag **docker**

Kalau runner shell belum ada tag `shell`, set di UI.

***

### 3C) Cara memastikan job deploy benar-benar masuk shell runner

Nanti di pipeline job `deploy_dev`, lihat log bagian atas.\
Harus muncul runner yang tag-nya `shell`.

***

## Tambahan kecil (supaya lebih “kebal”)

Ini opsional, tapi bagus:

#### A) Jangan gagal deploy hanya karena migrate error kecil

Kalau kamu mau deploy tetap lanjut walau migrate error (misalnya “already exists”), kamu bisa ubah command deploy jadi:

```bash
docker compose ... exec -T laravel php artisan migrate --force || true
```

Tapi ini **opsional**. Yang paling bersih tetap: migration idempotent seperti langkah #1.

#### B) Bersihin “stack” lama agar tidak dobel

Kamu sudah pernah ada `threebody-deploy_*`. Bagusnya pastikan cuma satu project compose yang aktif:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep threebody || true
```

Idealnya hanya `threebody-dev-*`.

***

## Checklist cepat (biar kamu yakin aman lanjut CI)

✅ 1) Migration `products` sudah `Schema::hasTable()`\
✅ 2) DEPLOY\_DIR sudah `/home/gitlab-runner/threebody-deploy`\
✅ 3) Runner shell **online** + punya tag `shell` di UI\
✅ 4) Pipeline deploy tidak fail lagi karena migrate

***

Kalau kamu mau, kirim hasil output dari perintah ini (cukup paste teksnya):

```bash
cd /home/cikal/three-body-problem-main
find laravel/database/migrations -maxdepth 1 -type f | grep -i products
sed -n '1,120p' laravel/database/migrations/*products*.php
```

Nanti aku bantu pastikan migration kamu sudah benar **sesuai struktur file yang sebenarnya** (karena bisa beda nama/tanggal).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000008c8871faae878211cd1b3aa0\&ts=490475\&p=fs\&cid=1\&sig=3cac3bfe9b637efeba57c4cd0936d0025d82e4072c28ea69dd769c470dee7a2f\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000b46472078dee6989e54c7ef9\&ts=490475\&p=fs\&cid=1\&sig=419c13ecb6818a2101eeb9091782bd7ea4882a59fafa39399fc81817cdf99c46\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000009c6c71fab4b25aec426fc9e7\&ts=490475\&p=fs\&cid=1\&sig=05047d35ed321ab3a7aae58e4980492cc44220511f5d256fcb18b14a0cb0e0c1\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000bec87209b9a69a913114ae28\&ts=490475\&p=fs\&cid=1\&sig=f2572422ea59627f9c177a268967628f466252f8192763f6d09cc581251d8607\&v=0)cikal@devops-ci:\~$ cd /home/cikal/three-body-problem-main pwd ls -la /home/cikal/three-body-problem-main total 72 drwxrwxr-x 8 cikal cikal 4096 Dec 14 08:09 . drwxr-x--- 8 cikal cikal 4096 Dec 14 07:40 .. -rw-rw-r-- 1 cikal cikal 204 Dec 13 17:17 .env.dev.compose -rw-rw-r-- 1 cikal cikal 132 Dec 13 20:26 .env.dev.compose.example drwxrwxr-x 8 cikal cikal 4096 Dec 14 07:48 .git -rw-rw-r-- 1 cikal cikal 41 Dec 13 20:29 .gitignore -rw-rw-r-- 1 cikal cikal 6202 Dec 14 08:09 .gitlab-ci.yml -rw-rw-r-- 1 cikal cikal 8220 Dec 13 20:27 README.md drwxrwxr-x 3 cikal cikal 4096 Dec 13 20:26 certs -rw-rw-r-- 1 cikal cikal 809 Dec 9 04:22 compare\_apis.sh drwxrwxr-x 4 cikal cikal 4096 Dec 13 20:26 devops -rw-rw-r-- 1 cikal cikal 1897 Dec 13 20:29 docker-compose.dev.yml drwxrwxr-x 4 cikal cikal 4096 Dec 13 20:33 frontend drwxrwxr-x 2 cikal cikal 4096 Dec 13 20:33 go drwxrwxr-x 11 cikal cikal 4096 Dec 13 20:33 laravel cikal@devops-ci:\~/three-body-problem-main$ cd /home/cikal/three-body-problem-main # lihat semua migration ls -la laravel/database/migrations # cari yang ada kata "products" ls -la laravel/database/migrations/\*products\* 2>/dev/null || true # kalau belum ketemu, cari pakai find find laravel/database/migrations -maxdepth 1 -type f | grep -i products total 28 drwxrwxr-x 2 cikal cikal 4096 Dec 9 04:22 . drwxrwxr-x 5 cikal cikal 4096 Dec 9 04:22 .. -rw-rw-r-- 1 cikal cikal 798 Dec 9 04:22 2014\_10\_12\_000000\_create\_users\_table.php -rw-rw-r-- 1 cikal cikal 683 Dec 9 04:22 2014\_10\_12\_100000\_create\_password\_resets\_table.php -rw-rw-r-- 1 cikal cikal 820 Dec 9 04:22 2019\_08\_19\_000000\_create\_failed\_jobs\_table.php -rw-rw-r-- 1 cikal cikal 861 Dec 9 04:22 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table.php -rw-rw-r-- 1 cikal cikal 822 Dec 9 04:22 2025\_07\_25\_075615\_create\_products\_table.php -rw-rw-r-- 1 cikal cikal 822 Dec 9 04:22 laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel ls -la /var/www/html/database/migrations | grep -i products -rw-rw-r-- 1 root root 822 Dec 9 04:22 2025\_07\_25\_075615\_create\_products\_table.php cikal@devops-ci:\~/three-body-problem-main$ nano laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php cikal@devops-ci:\~/three-body-problem-main$ cat laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php \<?php use Illuminate\Database\Migrations\Migration; use Illuminate\Database\Schema\Blueprint; use Illuminate\Support\Facades\Schema; return new class extends Migration { public function up(): void { if (Schema::hasTable('products')) { return; // table sudah ada -> jangan create lagi } Schema::create('products', function (Blueprint $table) { $table->id(); $table->string('name'); $table->text('description')->nullable(); $table->decimal('price', 10, 2); $table->integer('quantity')->default(0); $table->string('category')->nullable(); $table->timestamps(); }); } public function down(): void { Schema::dropIfExists('products'); } }; cikal@devops-ci:\~/three-body-problem-main$ git status git add laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php git commit -m "Make products migration idempotent (skip if table exists)" git push On branch main Your branch is up to date with 'origin/main'. Changes not staged for commit: (use "git add \<file>..." to update what will be committed) (use "git restore \<file>..." to discard changes in working directory) modified: .gitlab-ci.yml modified: laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php no changes added to commit (use "git add" and/or "git commit -a") \[main b4124d8] Make products migration idempotent (skip if table exists) 1 file changed, 8 insertions(+), 15 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 11, done. Counting objects: 100% (11/11), done. Delta compression using up to 2 threads Compressing objects: 100% (6/6), done. Writing objects: 100% (6/6), 676 bytes | 676.00 KiB/s, done. Total 6 (delta 5), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 712b90b..b4124d8 main -> main cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate:status | head +------+-------------------------------------------------------+-------+ | Ran? | Migration | Batch | +------+-------------------------------------------------------+-------+ | Yes | 2014\_10\_12\_000000\_create\_users\_table | 1 | | Yes | 2014\_10\_12\_100000\_create\_password\_resets\_table | 1 | | Yes | 2019\_08\_19\_000000\_create\_failed\_jobs\_table | 1 | | Yes | 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table | 1 | | No | 2025\_07\_25\_075615\_create\_products\_table | | +------+-------------------------------------------------------+-------+ cikal@devops-ci:\~/three-body-problem-main$ sudo mkdir -p /home/gitlab-runner/threebody-deploy sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/threebody-deploy sudo chmod 755 /home/gitlab-runner \[sudo] password for cikal: cikal@devops-ci:\~/three-body-problem-main$ sudo -u gitlab-runner ls -la /home/gitlab-runner/threebody-deploy total 24 drwxrwxr-x 4 gitlab-runner gitlab-runner 4096 Dec 14 07:21 . drwxr-xr-x 5 gitlab-runner gitlab-runner 4096 Dec 14 07:21 .. -rw-rw-r-- 1 gitlab-runner gitlab-runner 207 Dec 14 07:21 .env.dev.compose drwxrwxr-x 3 gitlab-runner gitlab-runner 4096 Dec 14 07:21 certs drwxrwxr-x 4 gitlab-runner gitlab-runner 4096 Dec 14 06:12 devops -rw-rw-r-- 1 gitlab-runner gitlab-runner 1897 Dec 14 06:12 docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["docker"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' # test nginx config docker exec "$NG" nginx -t # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: tags: \["shell"] stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port) export COMPOSE\_PROJECT\_NAME="threebody-dev" # samakan TAG dengan build job (paling aman pakai FULL SHA) TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" #DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" #DEPLOY\_DIR="/home/cikal/threebody-deploy" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # hentikan stack yang sama dulu (volume DB tetap aman) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml git commit -m "Use deploy dir under /home/gitlab-runner" git push \[main 6d38fab] Use deploy dir under /home/gitlab-runner 1 file changed, 3 insertions(+), 1 deletion(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 347 bytes | 347.00 KiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git b4124d8..6d38fab main -> main cikal@devops-ci:\~/three-body-problem-main$ sudo systemctl enable --now gitlab-runner sudo systemctl status gitlab-runner --no-pager ● gitlab-runner.service - GitLab Runner Loaded: loaded (/etc/systemd/system/gitlab-runner.service; enabled; preset: enabled) Active: active (running) since Sun 2025-12-14 05:53:39 UTC; 3h 23min ago Main PID: 832 (gitlab-runner) Tasks: 9 (limit: 2265) Memory: 31.8M (peak: 107.1M swap: 1.5M swap peak: 1.7M) CPU: 44.232s CGroup: /system.slice/gitlab-runner.service └─832 /usr/bin/gitlab-runner run --config /etc/gitlab-runner/config.toml --working-directory /home/gitlab-runner -… Dec 14 09:15:45 devops-ci gitlab-runner\[832]: Updating job... bytesize=6074 checksu…5qLStHlf Dec 14 09:15:45 devops-ci gitlab-runner\[832]: Submitting job to coordinator...accepted, but not yet completed bytesize…erval=1s Dec 14 09:15:46 devops-ci gitlab-runner\[832]: Updating job... bytesize=6074 checksu…5qLStHlf Dec 14 09:15:47 devops-ci gitlab-runner\[832]: Submitting job to coordinator...ok bytesize=6074 checksu…erval=0s Dec 14 09:15:47 devops-ci gitlab-runner\[832]: Removed job from processing list builds=0 gitlab\_user\_id=29788… Dec 14 09:16:38 devops-ci gitlab-runner\[832]: Checking for jobs... received correlation\_id=9adc9a…5qLStHlf Dec 14 09:16:38 devops-ci gitlab-runner\[832]: Updating job... bytesize=0 checksum= …5qLStHlf Dec 14 09:16:39 devops-ci gitlab-runner\[832]: Submitting job to coordinator...ok bytesize=0 checksum= …erval=0s Dec 14 09:16:39 devops-ci gitlab-runner\[832]: Added job to processing list builds=1 gitlab\_user\_id=29788… Dec 14 09:16:39 devops-ci gitlab-runner\[832]: Using default image executor=docker gitla…5qLStHlf Hint: Some lines were ellipsized, use -l to show in full. cikal@devops-ci:\~/three-body-problem-main$ sudo gitlab-runner verify Runtime platform arch=amd64 os=linux pid=216143 revision=5908bc19 version=18.6.5 Running in system-mode. Verifying runner... is valid correlation\_id=9adc9abbf47d70ac-CGK runner=P5qLStHlf Verifying runner... is valid correlation\_id=9adc9abee751bd94-ATL runner=P5qLStHlf Verifying runner... is valid correlation\_id=9adc9ac1e4fa514a-ATL runner=ArpoA9OGu cikal@devops-ci:\~/three-body-problem-main$ docker ps --format "table \{{.Names\}}\t\{{.Ports\}}" | grep threebody || true threebody-dev-nginx-dev-1 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp threebody-dev-frontend-1 80/tcp threebody-dev-laravel-1 80/tcp threebody-dev-goapi-1 8080/tcp threebody-dev-mysql-1 3306/tcp, 33060/tcp cikal@devops-ci:\~/three-body-problem-main$ cd /home/cikal/three-body-problem-main find laravel/database/migrations -maxdepth 1 -type f | grep -i products sed -n '1,120p' laravel/database/migrations/\*products\*.php laravel/database/migrations/2025\_07\_25\_075615\_create\_products\_table.php \<?php use Illuminate\Database\Migrations\Migration; use Illuminate\Database\Schema\Blueprint; use Illuminate\Support\Facades\Schema; return new class extends Migration { public function up(): void { if (Schema::hasTable('products')) { return; // table sudah ada -> jangan create lagi } Schema::create('products', function (Blueprint $table) { $table->id(); $table->string('name'); $table->text('description')->nullable(); $table->decimal('price', 10, 2); $table->integer('quantity')->default(0); $table->string('category')->nullable(); $table->timestamps(); }); } public function down(): void { Schema::dropIfExists('products'); } }; cikal@devops-ci:\~/three-body-problem-main$ jadi ada error deploy\_dev nya seperti dibawah ini log nya Running with gitlab-runner 18.6.5 (5908bc19) on DevOps\_43\_runner P5qLStHlf, system ID: s\_7bd6fa72a2b0 Preparing the "docker" executor 00:08 Using default image Using Docker executor with image docker:latest ... Using default image Using effective pull policy of \[always] for container docker:latest Pulling docker image docker:latest ... Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... Preparing environment 00:01 Using effective pull policy of \[always] for container sha256:54e6bb7e83797064b15abaf06223c9083f9e7b807d7046c13920150573a66d97 Running on runner-p5qlsthlf-project-76926856-concurrent-0 via devops-ci... Getting source from Git repository 00:06 Gitaly correlation ID: 9adc9d57c653283c-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /builds/cikalfarid/three-body-problem/.git/ Created fresh repository. Checking out b4124d86 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:36 Using default image Using effective pull policy of \[always] for container docker:latest Using docker image sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf for docker:latest with digest docker@sha256:7370a6c49b7e708fb969b422dffe6cdd78a9f0ff5b3bfba0e0cddce736c49eaf ... $ set -euo pipefail $ docker version Client: Version: 29.1.3 API version: 1.52 Go version: go1.25.5 Git commit: f52814d Built: Fri Dec 12 14:48:47 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command .........+.......+++++++++++++++++++++++++++++++++++++++\*.+++++++++++++++++++++++++++++++++++++++\*.+............+..+.+...........+......+....+......+.........+...+...+.....+...................+.....+.........+....+...+...++++++ ......+..+.......+...+...+.....+++++++++++++++++++++++++++++++++++++++\*...+.......+........+...+.........+..........+......+++++++++++++++++++++++++++++++++++++++\*...+............+..+.+..+.......+........+.......+........+...+.+...+..+.+........+.+.....+...+......+.+...........+....+...+...+.....+.........+......+....+...+..+....+..+.......+.....+.+.........+..+.........+.+...........+....+........+.......+.........+.........+...+..+...+.........+......+....+...+...+..+.+.....+......+................+.....+...............+..........+.................+.+......+...........+....+........+.+...............+...............+.....+....+..+...+.......+...+......+..+...................+.....+..................+...+...+....+..+.+..+.+......+......+..............+...+..................+.......+............+...+..+.....................+............+......+......+.++++++ ----- Login Succeeded WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'. Configure a credential helper to remove this warning. See https://docs.docker.com/go/credential-store/ Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image \[MASKED]/\[MASKED]/frontend:b4124d86714c75163ced1847e9a82273c7ecf55c Pulling Image nginx:alpine Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/laravel:b4124d86714c75163ced1847e9a82273c7ecf55c Pulling Image \[MASKED]/\[MASKED]/goapi:b4124d86714c75163ced1847e9a82273c7ecf55c Pulling Image \[MASKED]/\[MASKED]/goapi:b4124d86714c75163ced1847e9a82273c7ecf55c Pulled Image \[MASKED]/\[MASKED]/frontend:b4124d86714c75163ced1847e9a82273c7ecf55c Pulled Image \[MASKED]/\[MASKED]/laravel:b4124d86714c75163ced1847e9a82273c7ecf55c Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started \[2025-12-14 09:19:26] local.ERROR: SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (using password: YES) (SQL: select \* from information\_schema.tables where table\_schema = \[MASKED] and table\_name = migrations and table\_type = 'BASE TABLE') {"exception":"\[object] (Illuminate\\\Database\\\QueryException(code: 1045): SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (using password: YES) (SQL: select \* from information\_schema.tables where table\_schema = \[MASKED] and table\_name = migrations and table\_type = 'BASE TABLE') at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:712) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('select \* from i...', Array, Object(Closure)) #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(359): Illuminate\\\Database\\\Connection->run('select \* from i...', Array, Object(Closure)) #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/MySqlBuilder.php(43): Illuminate\\\Database\\\Connection->select('select \* from i...', Array) #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/DatabaseMigrationRepository.php(169): Illuminate\\\Database\\\Schema\\\MySqlBuilder->hasTable('migrations') #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(673): Illuminate\\\Database\\\Migrations\\\DatabaseMigrationRepository->repositoryExists() #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(110): Illuminate\\\Database\\\Migrations\\\Migrator->repositoryExists() #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(78): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->prepareDatabase() #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #15 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #17 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #18 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #19 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #22 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #23 {main} \[previous exception] \[object] (PDOException(code: 1045): SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (using password: YES) at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php:70) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(70): PDO->\_\_construct('mysql:host=mysq...', '\[MASKED]', Object(SensitiveParameterValue), Array) #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(45): Illuminate\\\Database\\\Connectors\\\Connector->createPdoConnection('mysql:host=mysq...', '\[MASKED]', '\[MASKED]', Array) #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/MySqlConnector.php(24): Illuminate\\\Database\\\Connectors\\\Connector->createConnection('mysql:host=mysq...', Array, Array) #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/ConnectionFactory.php(184): Illuminate\\\Database\\\Connectors\\\MySqlConnector->connect(Array) #4 \[internal function]: Illuminate\\\Database\\\Connectors\\\ConnectionFactory->Illuminate\\\Database\\\Connectors\\\\{closure}() #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(1064): call\_user\_func(Object(Closure)) #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(1100): Illuminate\\\Database\\\Connection->getPdo() #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(442): Illuminate\\\Database\\\Connection->getReadPdo() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(368): Illuminate\\\Database\\\Connection->getPdoForSelect(true) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(705): Illuminate\\\Database\\\Connection->Illuminate\\\Database\\\\{closure}('select \* from i...', Array) #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('select \* from i...', Array, Object(Closure)) #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(359): Illuminate\\\Database\\\Connection->run('select \* from i...', Array, Object(Closure)) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/MySqlBuilder.php(43): Illuminate\\\Database\\\Connection->select('select \* from i...', Array) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/DatabaseMigrationRepository.php(169): Illuminate\\\Database\\\Schema\\\MySqlBuilder->hasTable('migrations') #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(673): Illuminate\\\Database\\\Migrations\\\DatabaseMigrationRepository->repositoryExists() #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(110): Illuminate\\\Database\\\Migrations\\\Migrator->repositoryExists() #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(78): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->prepareDatabase() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #22 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #24 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #25 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #26 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #27 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #31 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #32 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #33 {main} "} In Connection.php line 712: SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (usi ng password: YES) (SQL: select \* from information\_schema.tables where table \_schema = \[MASKED] and table\_name = migrations and table\_type = 'BASE TABL E') In Connector.php line 70: SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (usi ng password: YES) Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit code 1 dan error juga web nya seperti pada di gambar untuk detailnya, tolong bantu perbaikiSelesai berpikir selama 4m 29s

Oke — sekarang kamu sudah paham _akar masalahnya_ (dan ini penting banget buat CI/CD ke depan). Aku jawab dua hal sekaligus:

1. **Kenapa `deploy_dev` di CI error `SQLSTATE[HY000] [1045] Access denied`**
2. **Arahan detail “3 hal yang perlu dirapihkan” + cara menemukan file (misal migration Laravel) + cara fix yang rapi & aman**

***

### 1) Kenapa `deploy_dev` di CI error 1045 (Access denied)

Error ini hampir pasti terjadi karena **MySQL pakai volume yang sudah berisi data lama**, dan **password user/database di volume itu tidak otomatis berubah** walaupun kamu ganti environment variable.

> MySQL image official itu hanya membuat user/password pertama kali saat folder data masih kosong.\
> Kalau volume `threebody-dev_mysql_data` sudah ada, maka env `MYSQL_PASSWORD` baru **tidak akan mengubah password user lama** → Laravel login pakai password baru → ditolak → 1045.

Makanya waktu kamu manual “beres” tadi, solusi yang berhasil adalah **hapus volume** lalu `up` ulang.

Di log CI kamu terlihat `docker compose down --remove-orphans` (tanpa `-v`), artinya volume DB tetap dipakai → kalau password CI berbeda dengan password saat volume pertama dibuat → 1045 muncul.

***

### 2) “3 hal” yang wajib dirapihkan supaya CI/CD stabil

Aku buat sangat praktis: **apa yang dicek, perintahnya apa, dan hasil yang diharapkan apa.**

***

## A. Rapihkan Runner: Pastikan `deploy_dev` benar-benar jalan di _shell runner_

Di log kamu, job jalan pakai **docker executor** (`Using Docker executor with image docker:latest ...`).\
Kalau deploy job jalan di docker executor, itu sering bikin masalah deployment jangka panjang karena:

* folder kerja `/builds/...` itu sifatnya sementara (dibersihin setelah job),
* sementara service `nginx-dev` kamu pakai **bind mount** file config & cert (butuh file yang “nempel” di host).

✅ Target yang benar:

* Job build/validate → runner docker
* Job deploy → runner shell (host VM), supaya file deploy dir permanen (`/home/gitlab-runner/threebody-deploy`) dan bind mount aman.

#### Langkah cek & set di GitLab UI

1. Masuk **Project → Settings → CI/CD → Runners**
2. Klik runner **shell** kamu → edit:
   * set tag misalnya: `shell`, `vm2`, `deploy`
   * **matikan** “Run untagged jobs”
3. Klik runner **docker** kamu:
   * set tag misalnya: `docker`, `vm2`
   * **matikan** “Run untagged jobs”

#### Pastikan `.gitlab-ci.yml` tegas tag-nya

Biar gak ketukar, paling rapi kamu tulis tag per-job (jangan hanya mengandalkan `default`):

```yaml
validate:
  tags: ["docker"]

build_goapi:
  tags: ["docker"]

build_laravel:
  tags: ["docker"]

build_frontend:
  tags: ["docker"]

deploy_dev:
  tags: ["deploy"]   # hanya runner shell yang punya tag ini
```

> Dengan begini, deploy **tidak mungkin** diambil runner docker.

***

## B. Rapihkan MySQL: password + volume harus konsisten (ini sumber 1045 kamu)

Ada 2 cara rapi. Pilih salah satu sesuai kebutuhan kamu.

### Opsi 1 (paling aman untuk DEV): Password **jangan pernah berubah** setelah pertama deploy

* Set `DEV_MYSQL_ROOT_PASSWORD` dan `DEV_MYSQL_PASSWORD` di GitLab Variables
* Setelah berhasil sekali, **jangan diganti-ganti**, karena volume DB akan terus dipakai

✅ Ini membuat deploy berikutnya stabil.

### Opsi 2: Sediakan “tombol reset DB” (manual job) untuk kasus password berubah / DB rusak

Ini cara paling rapi untuk CI: buat job manual yang boleh hapus volume.

Tambahkan job ini ke `.gitlab-ci.yml`:

```yaml
reset_dev_db:
  stage: deploy
  tags: ["deploy"]
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - set -euo pipefail
    - export COMPOSE_PROJECT_NAME="threebody-dev"
    - DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
    - cd "$DEPLOY_DIR"
    - docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v
    - docker volume rm -f threebody-dev_mysql_data || true
    - docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d
    - docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

Cara pakai:

* Kalau deploy error 1045 → kamu klik **Run** job `reset_dev_db` sekali.
* Habis itu deploy normal jalan lagi.

> Ini solusi “resmi” yang aman: **default deploy gak menghapus data**, tapi kamu punya tombol reset kalau perlu.

***

## C. Rapihkan Path & file lokasi (biar kamu gak bingung “ini file ada di mana”)

Kamu tadi bingung contoh:\
`laravel/database/migrations/2025_07_25_075615_create_products_table.php` itu di mana?

✅ Patokannya: **itu path relatif dari root project repo**.

Kalau root repo kamu:\
`/home/cikal/three-body-problem-main`

Maka file migration itu ada di:\
`/home/cikal/three-body-problem-main/laravel/database/migrations/2025_07_25_075615_create_products_table.php`

#### Cara menemukan file (template yang bisa kamu pakai untuk file apa pun)

Dari root repo:

```bash
cd /home/cikal/three-body-problem-main

# lihat isi folder migrations
ls -ლა laravel/database/migrations

# cari file yang berhubungan "products"
find laravel/database/migrations -maxdepth 1 -type f | grep -i products
```

#### Cara edit file

Pakai nano:

```bash
nano laravel/database/migrations/2025_07_25_075615_create_products_table.php
```

Atau kalau pakai VSCode remote:

```bash
code laravel/database/migrations/2025_07_25_075615_create_products_table.php
```

***

### 3) Perbaikan spesifik untuk error deploy\_dev kamu sekarang

Karena error kamu di CI adalah 1045, maka langkah fix yang paling tepat:

1. **Pastikan deploy job jalan di runner shell** (bagian A di atas).\
   Ini wajib supaya deploy benar-benar “deploy ke VM”, bukan job sementara.
2. Setelah itu, jalankan deploy lagi.\
   Kalau masih 1045:
   * Jalankan job manual `reset_dev_db` sekali (bagian B opsi 2), atau manual di VM:

```bash
DEPLOY_DIR=/home/gitlab-runner/threebody-deploy
cd "$DEPLOY_DIR"

docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
docker volume rm -f threebody-dev_mysql_data || true
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

***

### Catatan keamanan penting

Di chat kamu sempat menampilkan token runner/credential. Anggap itu **sudah bocor**. Yang aman:

* **rotate** runner token (hapus runner lama / buat ulang),
* ganti password DB dev kalau perlu.

***

Kalau kamu mau, tempel isi **bagian deploy\_dev** paling terbaru dari `.gitlab-ci.yml` (hanya job deploy\_dev + tags runner yang ada di GitLab UI). Aku bisa bikinkan versi final yang:

* deploy pasti di shell runner,
* punya `reset_dev_db` manual,
* dan deploy normal tidak merusak data.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000005dc72079b884b5141efb5d4\&ts=490475\&p=fs\&cid=1\&sig=bdb037fe9b073f180563fdc845c9480ffaff86b0bb5dadaa77d067b6bf2beeae\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["docker"] # WAJIB: paksa jalan di runner VM2 before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' # test nginx config docker exec "$NG" nginx -t # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: tags: \["shell"] stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port) export COMPOSE\_PROJECT\_NAME="threebody-dev" # samakan TAG dengan build job (paling aman pakai FULL SHA) TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" #DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" #DEPLOY\_DIR="/home/cikal/threebody-deploy" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # hentikan stack yang sama dulu (volume DB tetap aman) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps reset\_dev\_db: stage: deploy tags: \["deploy"] when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - set -euo pipefail - export COMPOSE\_PROJECT\_NAME="threebody-dev" - DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" - cd "$DEPLOY\_DIR" - docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v - docker volume rm -f threebody-dev\_mysql\_data || true - docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d - docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ apa seperti ini sudah benar? bantu rapihkan sebelum di push dan saya punya 2 runner, saya mau hanya 1 runner saja agar rapih dan runner yang saya inginkan yang DevOps\_43\_runnerSelesai berpikir selama 4m 57s

Sudah **hampir benar**, tapi kalau target kamu adalah **hanya 1 runner saja (DevOps\_43\_runner)**, maka `.gitlab-ci.yml` kamu **harus dirapihkan di bagian TAG** (karena sekarang `default: tags: ["docker"]` masih akan “memaksa” job validate/build lari ke runner docker = DevOps\_44\_runner).

Di screenshot kamu:

* **DevOps\_43\_runner** punya tag: `vm2`, `shell`, `deploy`
* **DevOps\_44\_runner** punya tag: `vm2`, `docker`

Jadi cara paling rapi: **gunakan tag `deploy` untuk SEMUA job**, supaya **hanya DevOps\_43\_runner** yang bisa ambil job.

***

### 1) Rapihkan `.gitlab-ci.yml` (siap push)

#### Perubahan utama yang wajib

1. Ganti `default.tags` dari `["docker"]` ➜ `["deploy"]`
2. Samakan `deploy_dev` & `reset_dev_db` pakai tag yang sama (kalau `default` sudah `deploy`, job tidak perlu tag lagi).
3. `reset_dev_db` sebaiknya juga bikin `.env.dev.compose` (biar aman walau dijalankan sebelum deploy).

Berikut versi yang sudah rapi (copy-paste full):

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

default:
  tags: ["deploy"]   # <-- hanya DevOps_43_runner yang punya tag ini
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# --- VALIDATE ---
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

# --- BUILD + PUSH ---
build_goapi:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  stage: build
  retry: 2
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# --- DEPLOY DEV (main only) ---
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail
      export COMPOSE_PROJECT_NAME="threebody-dev"
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
      mkdir -p "$DEPLOY_DIR"

      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
      MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
      MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
      MYSQL_USER=threebody
      MYSQL_DATABASE=threebody
      HARBOR_URL=${HARBOR_URL}
      HARBOR_PROJECT=${HARBOR_PROJECT}
      CI_COMMIT_SHA=${TAG}
      APP_KEY=${DEV_APP_KEY}
      ENV

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps

reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail
      export COMPOSE_PROJECT_NAME="threebody-dev"
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
      mkdir -p "$DEPLOY_DIR"

      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
      MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
      MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
      MYSQL_USER=threebody
      MYSQL_DATABASE=threebody
      HARBOR_URL=${HARBOR_URL}
      HARBOR_PROJECT=${HARBOR_PROJECT}
      CI_COMMIT_SHA=${TAG}
      APP_KEY=${DEV_APP_KEY}
      ENV

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true
      docker volume rm -f threebody-dev_mysql_data || true

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

✅ Dengan file ini:

* Semua job hanya bisa jalan di runner yang punya tag `deploy` → **DevOps\_43\_runner**
* Runner `docker` tidak akan kepakai walaupun masih terdaftar

***

### 2) Kamu mau benar-benar “cuma 1 runner” (hapus DevOps\_44\_runner)

#### A) Hapus dari Project (paling gampang)

Di halaman **Settings → CI/CD → Runners**:

* Klik ikon **trash** pada **DevOps\_44\_runner** (yang tag `docker`)

#### B) Rapihkan di server VM (biar config.toml tinggal 1 runner)

Di VM:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

Cari blok runner yang **tags = \["vm2","docker"]** / atau namanya DevOps\_44\_runner, lalu **hapus 1 blok `[[runners]]` itu saja**.

Lalu restart:

```bash
sudo systemctl restart gitlab-runner
sudo gitlab-runner verify
```

***

### 3) PENTING: pastikan user `gitlab-runner` bisa akses docker (kalau tidak nanti error “permission denied”)

Jalankan ini sekali:

```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
sudo -u gitlab-runner docker ps
sudo -u gitlab-runner docker compose version
```

Kalau `docker ps` bisa jalan tanpa error → aman.

***

### 4) Baru setelah itu push

```bash
git add .gitlab-ci.yml
git commit -m "Unify pipeline tags to deploy (use only DevOps_43_runner)"
git push
```

***

Kalau setelah ini deploy masih kena **1045 Access denied**, itu biasanya karena **volume MySQL sudah terlanjur dibuat dengan password lama**. Solusinya tinggal klik job manual **`reset_dev_db`** sekali (itu memang dibuat khusus buat kasus itu).

cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["deploy"] # <-- hanya DevOps\_43\_runner yang punya tag ini before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail export COMPOSE\_PROJECT\_NAME="threebody-dev" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail export COMPOSE\_PROJECT\_NAME="threebody-dev" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ sudo nano /etc/gitlab-runner/config.toml \[sudo] password for cikal: cikal@devops-ci:\~/three-body-problem-main$ sudo cat /etc/gitlab-runner/config.toml concurrent = 1 check\_interval = 0 connection\_max\_age = "15m0s" shutdown\_timeout = 0 \[session\_server] session\_timeout = 1800 \[\[runners]] name = "DevOps\_43\_runner" url = "https://gitlab.com/" id = 50865738 token = "glrt-P5qLStHlfkb9wfPeQ1orxW86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1r0u5v0" token\_obtained\_at = 2025-12-09T04:19:01Z token\_expires\_at = 0001-01-01T00:00:00Z executor = "docker" \[runners.cache] MaxUploadedArchiveSize = 0 \[runners.cache.s3] \[runners.cache.gcs] \[runners.cache.azure] \[runners.docker] tls\_verify = false image = "docker:latest" privileged = true disable\_entrypoint\_overwrite = false oom\_kill\_disable = false disable\_cache = false volumes = \["/var/run/docker.sock:/var/run/docker.sock", "/cache"] shm\_size = 0 network\_mtu = 0 \[\[runners]] name = "vm2-shell-runner" url = "https://gitlab.com" id = 50865738 token = "glrt-P5qLStHlfkb9wfPeQ1orxW86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1r0u5v0" token\_obtained\_at = 2025-12-13T20:46:03Z token\_expires\_at = 0001-01-01T00:00:00Z executor = "shell" \[runners.cache] MaxUploadedArchiveSize = 0 \[runners.cache.s3] \[runners.cache.gcs] \[runners.cache.azure] \[\[runners]] name = "devops-ci" url = "https://gitlab.com" id = 50958772 token = "glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7" token\_obtained\_at = 2025-12-14T08:37:52Z token\_expires\_at = 0001-01-01T00:00:00Z executor = "shell" \[runners.cache] MaxUploadedArchiveSize = 0 \[runners.cache.s3] \[runners.cache.gcs] \[runners.cache.azure] cikal@devops-ci:\~/three-body-problem-main$ apakah sudah benar?Selesai berpikir selama 2m 50s

`.gitlab-ci.yml` kamu **sudah benar** (pakai `default: tags: ["deploy"]` itu tepat supaya semua job nempel ke runner yang punya tag `deploy`).

Yang **belum benar / belum rapi** justru bagian **runner di `/etc/gitlab-runner/config.toml`**:

* Kamu sekarang punya **3 blok `[[runners]]`**
* Ada **duplikasi ID + token** untuk runner `50865738` (itu tidak sehat dan bisa bikin runner “aneh”/tidak konsisten)
* Dari screenshot GitLab, runner yang kamu mau (**DevOps\_43\_runner**) itu **ID `50958772`** → di config kamu itu yang blok:
  * `name = "devops-ci"`
  * `id = 50958772`
  * `executor = "shell"`

Jadi targetnya: **sisakan 1 runner saja: ID 50958772 (shell)**, hapus semua runner lain (50865738).

***

### A. Rapihkan runner (paling aman & jelas)

#### 1) Backup config dulu

```bash
sudo cp /etc/gitlab-runner/config.toml /etc/gitlab-runner/config.toml.bak.$(date +%F-%H%M)
```

#### 2) Edit config.toml → HAPUS 2 blok runner yang ID = 50865738

Buka:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

**Hapus total 2 blok ini (yang id=50865738):**

* `name = "DevOps_43_runner"` (executor docker) ✅ ini sebenarnya bukan DevOps\_43\_runner versi GitLab UI kamu
* `name = "vm2-shell-runner"` (executor shell)

**Sisakan hanya blok ini (id=50958772):**

```toml
[[runners]]
  name = "devops-ci"
  url = "https://gitlab.com"
  id = 50958772
  token = "..."
  executor = "shell"
```

> Nama boleh tetap `devops-ci` (tidak masalah). Yang penting **runner ID 50958772** yang aktif dan punya tag `deploy` di GitLab.

#### 3) Restart & verify

```bash
sudo systemctl restart gitlab-runner
sudo gitlab-runner verify
sudo gitlab-runner list
```

***

### B. Pastikan di GitLab UI cuma 1 runner yang “Assigned”

Masuk **Project → Settings → CI/CD → Runners**:

* Runner **50865738 (docker)** → klik **trash / unassign / disable**
* Pastikan runner **50958772** tetap **Assigned** dan status **online**

***

### C. Penting: pastikan runner shell bisa akses docker

Karena job kamu pakai `docker build`, `docker compose`, dsb.

Jalankan:

```bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner

sudo -u gitlab-runner docker ps
sudo -u gitlab-runner docker compose version
```

Kalau dua perintah terakhir sukses → aman.

***

### Catatan keamanan (penting banget)

Di chat kamu terlihat **token runner** ikut kepampang. Itu sebaiknya **segera di-rotate** dari GitLab (kalau tidak, siapa pun yang lihat token bisa menyalahgunakan runner).

***

### Setelah ini, pipeline harusnya:

* Semua job pakai tag `deploy` → otomatis lari ke runner **DevOps\_43\_runner (ID 50958772)**
* Runner yang lain sudah tidak kepakai / tidak ada

Kalau setelah dirapihkan masih ketemu error `SQLSTATE[1045] Access denied`, jalankan job manual **`reset_dev_db`** sekali (itu memang untuk “reset volume mysql yang keburu kebentuk dengan password lama”).

cikal@devops-ci:\~/three-body-problem-main$ sudo cp /etc/gitlab-runner/config.toml /etc/gitlab-runner/config.toml.bak.$(date +%F-%H%M) cikal@devops-ci:\~/three-body-problem-main$ sudo nano /etc/gitlab-runner/config.toml cikal@devops-ci:\~/three-body-problem-main$ sudo cat /etc/gitlab-runner/config.toml concurrent = 1 check\_interval = 0 connection\_max\_age = "15m0s" shutdown\_timeout = 0 \[session\_server] session\_timeout = 1800 \[\[runners]] name = "devops-ci" url = "https://gitlab.com" id = 50958772 token = "glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7" token\_obtained\_at = 2025-12-14T08:37:52Z token\_expires\_at = 0001-01-01T00:00:00Z executor = "shell" \[runners.cache] MaxUploadedArchiveSize = 0 \[runners.cache.s3] \[runners.cache.gcs] \[runners.cache.azure] cikal@devops-ci:\~/three-body-problem-main$ sudo systemctl restart gitlab-runner sudo gitlab-runner verify sudo gitlab-runner list Runtime platform arch=amd64 os=linux pid=270430 revision=5908bc19 version=18.6.5 Running in system-mode. Verifying runner... is valid correlation\_id=9adce7d211cee77c-CGK runner=ArpoA9OGu Runtime platform arch=amd64 os=linux pid=270451 revision=5908bc19 version=18.6.5 Listing configured runners ConfigFile=/etc/gitlab-runner/config.toml devops-ci Executor=shell Token=glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7 URL=https://gitlab.com cikal@devops-ci:\~/three-body-problem-main$ sudo usermod -aG docker gitlab-runner sudo systemctl restart gitlab-runner sudo -u gitlab-runner docker ps sudo -u gitlab-runner docker compose version CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES d0fde82648b2 nginx:alpine "/docker-entrypoint.…" 46 minutes ago Up 45 minutes 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp threebody-dev-nginx-dev-1 aa49436c64dd 192.168.56.43:8081/threebody/frontend:6d38fabaa63fa48f4fe7ca1e915635324f2be6ab "/docker-entrypoint.…" 46 minutes ago Up 45 minutes 80/tcp threebody-dev-frontend-1 707c3022e3d8 192.168.56.43:8081/threebody/laravel:6d38fabaa63fa48f4fe7ca1e915635324f2be6ab "docker-php-entrypoi…" 46 minutes ago Up 45 minutes 80/tcp threebody-dev-laravel-1 872dda78d16f 192.168.56.43:8081/threebody/goapi:6d38fabaa63fa48f4fe7ca1e915635324f2be6ab "/app/goapi" 46 minutes ago Up 45 minutes 8080/tcp threebody-dev-goapi-1 e80066df6938 mysql:8.0 "docker-entrypoint.s…" 46 minutes ago Up 46 minutes (healthy) 3306/tcp, 33060/tcp threebody-dev-mysql-1 99fc5e71552e 289decab4142 "/docker-entrypoint.…" 4 hours ago Up 4 hours 80/tcp goapi 6729616da24c 289decab4142 "/docker-entrypoint.…" 4 hours ago Up 4 hours 80/tcp laravel bbc26a0be501 289decab4142 "/docker-entrypoint.…" 4 hours ago Up 4 hours 80/tcp frontend 82885c6c7e15 goharbor/nginx-photon:v2.10.0 "nginx -g 'daemon of…" 17 hours ago Up 4 hours (healthy) 0.0.0.0:8081->8080/tcp, \[::]:8081->8080/tcp nginx e501f5d63061 goharbor/harbor-jobservice:v2.10.0 "/harbor/entrypoint.…" 17 hours ago Up 4 hours (healthy) harbor-jobservice 9d071bb8a1df goharbor/harbor-core:v2.10.0 "/harbor/entrypoint.…" 17 hours ago Up 4 hours (healthy) harbor-core 43a07e2e2d41 goharbor/registry-photon:v2.10.0 "/home/harbor/entryp…" 17 hours ago Up 4 hours (healthy) registry d198b314fd34 goharbor/harbor-portal:v2.10.0 "nginx -g 'daemon of…" 17 hours ago Up 4 hours (healthy) harbor-portal b6be3be19741 goharbor/redis-photon:v2.10.0 "redis-server /etc/r…" 17 hours ago Up 4 hours (healthy) redis 0ead292610dd goharbor/harbor-registryctl:v2.10.0 "/home/harbor/start.…" 17 hours ago Up 4 hours (healthy) registryctl 7bcf866625c6 goharbor/harbor-db:v2.10.0 "/docker-entrypoint.…" 17 hours ago Up 4 hours (healthy) harbor-db 147c6797fc06 goharbor/harbor-log:v2.10.0 "/bin/sh -c /usr/loc…" 17 hours ago Up 4 hours (healthy) 127.0.0.1:1514->10514/tcp harbor-log Docker Compose version v5.0.0 cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 1m 30s

✅ Iya, **runner kamu sekarang sudah rapi dan benar**:

* `config.toml` tinggal **1 runner** (ID `50958772`, executor `shell`)
* `gitlab-runner verify` ✅ valid
* user `gitlab-runner` sudah bisa jalanin `docker` & `docker compose` ✅

Sekarang tinggal 2 hal penting biar **deploy\_dev** nggak error lagi dan sistem makin rapih.

***

### 1) Rapihkan runner di GitLab UI (biar bener-bener cuma 1)

Walaupun di server sudah 1, di GitLab UI runner lama masih bisa “nempel”.

1. Buka: **Project → Settings → CI/CD → Runners**
2. Di tab **Assigned project runners**
3. Untuk runner yang **bukan** `50958772 (DevOps_43_runner)`:
   * klik **Disable / Pause** atau **Unassign** (atau delete kalau project runner)
4. Sisakan hanya runner yang kamu mau.

⚠️ Penting juga: token runner kamu sempat kepost di chat. Demi aman, di GitLab **rotate** token runner itu (di halaman runner ada opsi rotate/regen token).

***

### 2) Fix error deploy\_dev: `SQLSTATE[1045] Access denied`

Error ini hampir selalu karena **password user MySQL yang ada di volume beda** dengan password yang dipakai Laravel di container.

Ada 2 penyebab paling umum:

#### Penyebab A — `.env.dev.compose` hasil deploy _tidak mengisi DB\_ untuk Laravel_\*

Solusi paling aman: saat deploy, isi juga variabel DB untuk Laravel (biar tidak ngambil default dari image / env lain).

Di `.gitlab-ci.yml`, pada bagian ini:

```bash
cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
MYSQL_ROOT_PASSWORD=...
MYSQL_PASSWORD=...
MYSQL_USER=threebody
MYSQL_DATABASE=threebody
...
ENV
```

✅ Ubah jadi begini (tambahkan DB\_\*):

```bash
cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=${DEV_MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
ENV
```

Setelah itu push, lalu jalankan pipeline lagi.

***

#### Penyebab B — Volume MySQL sudah pernah dibuat dengan password lama

Kalau sebelumnya volume `threebody-dev_mysql_data` pernah kebentuk pakai password A, lalu sekarang deploy pakai password B → MySQL **tidak otomatis ganti password**.

✅ Solusinya: jalankan job **manual** `reset_dev_db` sekali (yang sudah kamu buat) untuk buang volume dan bikin ulang database.

Atau manual via server:

```bash
DEPLOY_DIR="/home/gitlab-runner/threebody-deploy"
cd "$DEPLOY_DIR"
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
docker volume rm -f threebody-dev_mysql_data || true
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

***

### 3) Rapihkan container sisa (opsional tapi bagus)

Di `docker ps` kamu ada container:

* `goapi`, `laravel`, `frontend` (tanpa prefix threebody-dev)

Kalau itu memang sisa test dan tidak dipakai, hapus saja:

```bash
docker rm -f goapi laravel frontend 2>/dev/null || true
```

***

### 4) Checklist sebelum push `.gitlab-ci.yml`

Pastikan di GitLab **CI/CD Variables** ada ini (dan nilainya benar):

* `HARBOR_URL`
* `HARBOR_PROJECT`
* `HARBOR_USERNAME`
* `HARBOR_PASSWORD`
* `DEV_MYSQL_ROOT_PASSWORD`
* `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY`

Kalau `DEV_MYSQL_PASSWORD` sekarang **beda** dari yang pernah dipakai volume lama → wajib jalankan `reset_dev_db` sekali.

***

Kalau kamu mau, aku bisa bantu **rapihin `.gitlab-ci.yml` lebih bersih** (mengurangi duplikasi script deploy\_dev & reset\_dev\_db pakai YAML anchor), tapi yang paling penting untuk error 1045 adalah _tambah DB\_ di env file_\* + **reset volume sekali jika password berubah**.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000258472079e1ec926e6378662\&ts=490475\&p=fs\&cid=1\&sig=488085d6ff431cf195e65d0ce47241b1aeca5789950ee76d7b408ccd228716bc\&v=0)cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" default: tags: \["deploy"] # <-- hanya DevOps\_43\_runner yang punya tag ini before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # --- VALIDATE --- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # --- BUILD + PUSH --- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # --- DEPLOY DEV (main only) --- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail export COMPOSE\_PROJECT\_NAME="threebody-dev" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail export COMPOSE\_PROJECT\_NAME="threebody-dev" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" mkdir -p "$DEPLOY\_DIR" if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ ya bantu rapihkan saja agar mudah langsung kopas dan berikan arahan push nya lagi secara lengkapSelesai berpikir selama 1m 18s

Siap — ini versi **lebih rapi + gampang di-copy/paste** (mengurangi duplikasi, nambah **DB\_\*** biar error `1045 Access denied` hilang, dan tetap pakai **1 runner** tag `deploy`).

> **Catatan penting:** kamu _wajib_ tambahkan `DB_HOST/DB_USERNAME/DB_PASSWORD` di `.env.dev.compose` yang dibuat saat deploy. Tanpa ini, Laravel bisa pakai kredensial default/beda → muncul 1045.

***

### 1) Copy-paste `.gitlab-ci.yml` (final yang rapi)

Di folder repo kamu:

```bash
cd /home/cikal/three-body-problem-main
nano .gitlab-ci.yml
```

Hapus isi lama, lalu **paste full ini**:

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  # biar konsisten (nggak bikin stack baru yg bentrok port)
  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

default:
  tags: ["deploy"] # hanya runner DevOps_43_runner yg punya tag ini
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

# -------------------------
# BUILD + PUSH (Harbor)
# -------------------------
.build_login_and_tag: &build_login_and_tag |
  set -euo pipefail
  : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

build_goapi:
  stage: build
  script:
    - *build_login_and_tag
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  stage: build
  retry: 2
  script:
    - *build_login_and_tag
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - *build_login_and_tag
    - docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
    - docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# -------------------------
# DEPLOY COMMON (dipakai deploy & reset)
# -------------------------
.deploy_prepare: &deploy_prepare |
  set -euo pipefail
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

  mkdir -p "$DEPLOY_DIR"

  # sync file deploy
  if command -v rsync >/dev/null 2>&1; then
    rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
  else
    rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
    cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
  fi

  # cert (sekali saja)
  mkdir -p "$DEPLOY_DIR/certs/dev"
  if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
    docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
      apk add --no-cache openssl >/dev/null
      openssl req -newkey rsa:2048 -nodes \
        -keyout /out/dev.key \
        -x509 -days 365 \
        -out /out/dev.crt \
        -subj "/CN=dev.local"
    '
  fi

  # env untuk compose + LARAVEL DB (ini yang mencegah 1045)
  cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
  MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
  MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
  MYSQL_USER=threebody
  MYSQL_DATABASE=threebody

  DB_HOST=mysql
  DB_PORT=3306
  DB_DATABASE=threebody
  DB_USERNAME=threebody
  DB_PASSWORD=${DEV_MYSQL_PASSWORD}

  HARBOR_URL=${HARBOR_URL}
  HARBOR_PROJECT=${HARBOR_PROJECT}
  CI_COMMIT_SHA=${TAG}
  APP_KEY=${DEV_APP_KEY}
  ENV

  cd "$DEPLOY_DIR"
  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

# -------------------------
# DEPLOY DEV (main only)
# -------------------------
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps

# -------------------------
# RESET DEV DB (manual) - kalau password berubah / volume kacau
# -------------------------
reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true
      docker volume rm -f threebody-dev_mysql_data || true

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

***

### 2) Arahan push (lengkap, tinggal kopas)

Jalankan ini berurutan:

```bash
cd /home/cikal/three-body-problem-main

# cek perubahan
git diff
git status

# stage & commit
git add .gitlab-ci.yml
git commit -m "Refactor CI: deploy reuse + set Laravel DB env to prevent 1045"

# push
git push origin main
```

***

### 3) Setelah push, jalankan pipeline & langkah kalau masih 1045

1. Buka GitLab → **CI/CD → Pipelines**
2. Pastikan job `deploy_dev` jalan sampai selesai.
3. Kalau masih ada `SQLSTATE[1045]`:
   * Masuk **job manual** `reset_dev_db` → klik **Play** (sekali saja)
   * Setelah reset sukses, jalankan pipeline normal lagi.

***

### 4) Pastikan Variables di GitLab sudah ada

Di **Settings → CI/CD → Variables**, pastikan ada (dan nilainya benar):

* `HARBOR_URL`
* `HARBOR_PROJECT`
* `HARBOR_USERNAME`
* `HARBOR_PASSWORD`
* `DEV_MYSQL_ROOT_PASSWORD`
* `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY`

***

Kalau kamu mau, kirim **screenshot CI/CD Variables (tanpa terlihat value)** atau output `docker compose config` dari folder deploy (`/home/gitlab-runner/threebody-deploy`) — nanti aku cek apakah semua variable sudah “nyambung” ke service laravel & mysql dengan benar.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000adb07207ae785b37cc44edcb\&ts=490475\&p=fs\&cid=1\&sig=e2c68214d0526bed40c82e367e8b16cdd09dfd7cb5c2e0e2029337c800c4731a\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000008be071fa91b220d980b622dd\&ts=490475\&p=fs\&cid=1\&sig=3d71f38531e67627aec4b5f1319979b10be85338ae918029176d7ba5e1a45d8f\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000079f072079b9147a219c7c770\&ts=490475\&p=fs\&cid=1\&sig=337f0dd90e7c12c79b73aab5f01bee7c7adec4a9588baa0f460bd524d78d9f65\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" # biar konsisten (nggak bikin stack baru yg bentrok port) COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" default: tags: \["deploy"] # hanya runner DevOps\_43\_runner yg punya tag ini before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .build\_login\_and\_tag: \&build\_login\_and\_tag | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*build\_login\_and\_tag - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*build\_login\_and\_tag - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*build\_login\_and\_tag - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY COMMON (dipakai deploy & reset) # ------------------------- .deploy\_prepare: \&deploy\_prepare | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # env untuk compose + LARAVEL DB (ini yang mencegah 1045) cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # ------------------------- # DEPLOY DEV (main only) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps # ------------------------- # RESET DEV DB (manual) - kalau password berubah / volume kacau # ------------------------- reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ cd /home/cikal/three-body-problem-main # cek perubahan git diff git status # stage & commit git add .gitlab-ci.yml git commit -m "Refactor CI: deploy reuse + set Laravel DB env to prevent 1045" # push git push origin main diff --git a/.gitlab-ci.yml b/.gitlab-ci.yml index ed562ad..997d39f 100644 --- a/.gitlab-ci.yml +++ b/.gitlab-ci.yml @@ -7,23 +7,26 @@ variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" + # biar konsisten (nggak bikin stack baru yg bentrok port) + COMPOSE\_PROJECT\_NAME: "threebody-dev" + DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" + default: - tags: \["docker"] # WAJIB: paksa jalan di runner VM2 + tags: \["deploy"] # hanya runner DevOps\_43\_runner yg punya tag ini before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true -# --- VALIDATE --- +# ------------------------- +# VALIDATE +# ------------------------- validate: stage: validate script: - # biar tidak warning APP\_KEY kosong saat compose config - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - - | set -euo pipefail @@ -35,20 +38,16 @@ validate: docker network create "$NET" >/dev/null - # dummy upstream biar hostname frontend/laravel/goapi resolve docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null - # container untuk test nginx docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null - # siapkan folder, copy config TANPA bind-mount docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf - # generate cert di dalam container (tanpa mount) docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ @@ -56,108 +55,128 @@ validate: -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' - # test nginx config docker exec "$NG" nginx -t - # cleanup docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true -# --- BUILD + PUSH --- +# ------------------------- +# BUILD + PUSH (Harbor) +# ------------------------- +.build\_login\_and\_tag: \&build\_login\_and\_tag | + set -euo pipefail + : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" + echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin + TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" + build\_goapi: stage: build script: - - | - set -euo pipefail - : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" + - \*build\_login\_and\_tag + - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . + - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - - | - set -euo pipefail - : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" + - \*build\_login\_and\_tag + - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . + - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - - | - set -euo pipefail - : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" - -# --- DEPLOY DEV (main only) --- + - \*build\_login\_and\_tag + - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . + - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" + +# ------------------------- +# DEPLOY COMMON (dipakai deploy & reset) +# ------------------------- +.deploy\_prepare: \&deploy\_prepare | + set -euo pipefail + TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" + + mkdir -p "$DEPLOY\_DIR" + + # sync file deploy + if command -v rsync >/dev/null 2>&1; then + rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" + else + rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true + cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" + fi + + # cert (sekali saja) + mkdir -p "$DEPLOY\_DIR/certs/dev" + if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then + docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' + apk add --no-cache openssl >/dev/null + openssl req -newkey rsa:2048 -nodes \ + -keyout /out/dev.key \ + -x509 -days 365 \ + -out /out/dev.crt \ + -subj "/CN=dev.local" + ' + fi + + # env untuk compose + LARAVEL DB (ini yang mencegah 1045) + cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV + MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} + MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} + MYSQL\_USER=threebody + MYSQL\_DATABASE=threebody + + DB\_HOST=mysql + DB\_PORT=3306 + DB\_DATABASE=threebody + DB\_USERNAME=threebody + DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} + + HARBOR\_URL=${HARBOR\_URL} + HARBOR\_PROJECT=${HARBOR\_PROJECT} + CI\_COMMIT\_SHA=${TAG} + APP\_KEY=${DEV\_APP\_KEY} + ENV + + cd "$DEPLOY\_DIR" + echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin + +# ------------------------- +# DEPLOY DEV (main only) +# ------------------------- deploy\_dev: - tags: \["shell"] stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: + - \*deploy\_prepare - | - set -euo pipefail - - # PAKSA project name tetap (biar ga bikin stack baru yang bentrok port) - export COMPOSE\_PROJECT\_NAME="threebody-dev" - - # samakan TAG dengan build job (paling aman pakai FULL SHA) - TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" - - #DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" - #DEPLOY\_DIR="/home/cikal/threebody-deploy" - DEPLOY\_DIR="/home/gitlab-runner/threebody-deploy" - mkdir -p "$DEPLOY\_DIR" - - # sync file deploy - if command -v rsync >/dev/null 2>&1; then - rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" - else - rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true - cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" - fi - - mkdir -p "$DEPLOY\_DIR/certs/dev" - if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then - docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' - apk add --no-cache openssl >/dev/null - openssl req -newkey rsa:2048 -nodes \ - -keyout /out/dev.key \ - -x509 -days 365 \ - -out /out/dev.crt \ - -subj "/CN=dev.local" - ' - fi - - cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV - MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} - MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} - HARBOR\_URL=${HARBOR\_URL} - HARBOR\_PROJECT=${HARBOR\_PROJECT} - CI\_COMMIT\_SHA=${TAG} - APP\_KEY=${DEV\_APP\_KEY} - ENV - - cd "$DEPLOY\_DIR" - echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin - - # hentikan stack yang sama dulu (volume DB tetap aman) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true - docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps + +# ------------------------- +# RESET DEV DB (manual) - kalau password berubah / volume kacau +# ------------------------- +reset\_dev\_db: + stage: deploy + when: manual + rules: + - if: '$CI\_COMMIT\_BRANCH == "main"' + script: + - \*deploy\_prepare + - | + docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true + docker volume rm -f threebody-dev\_mysql\_data || true + + docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull + docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d + + docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force On branch main Your branch is up to date with 'origin/main'. Changes not staged for commit: (use "git add \<file>..." to update what will be committed) (use "git restore \<file>..." to discard changes in working directory) modified: .gitlab-ci.yml no changes added to commit (use "git add" and/or "git commit -a") \[main 8afeaf9] Refactor CI: deploy reuse + set Laravel DB env to prevent 1045 1 file changed, 102 insertions(+), 83 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.18 KiB | 202.00 KiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 6d38fab..8afeaf9 main -> main cikal@devops-ci:\~/three-body-problem-main$ masih ada error deploy\_dev nya Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on devops-ci... Getting source from Git repository 00:03 Gitaly correlation ID: 9adcfb8030276cf1-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out 8afeaf94 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:44 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command Login Succeeded $ docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true # collapsed multi-line command Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image \[MASKED]/\[MASKED]/frontend:8afeaf9452e27d5af79ecd10ecbd4121e7fee4ed Pulling Image mysql:8.0 Pulling Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/laravel:8afeaf9452e27d5af79ecd10ecbd4121e7fee4ed Pulling Image \[MASKED]/\[MASKED]/goapi:8afeaf9452e27d5af79ecd10ecbd4121e7fee4ed Pulling Image \[MASKED]/\[MASKED]/goapi:8afeaf9452e27d5af79ecd10ecbd4121e7fee4ed Pulled Image \[MASKED]/\[MASKED]/frontend:8afeaf9452e27d5af79ecd10ecbd4121e7fee4ed Pulled Image \[MASKED]/\[MASKED]/laravel:8afeaf9452e27d5af79ecd10ecbd4121e7fee4ed Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started \[2025-12-14 10:23:39] local.ERROR: SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (using password: YES) (SQL: select \* from information\_schema.tables where table\_schema = \[MASKED] and table\_name = migrations and table\_type = 'BASE TABLE') {"exception":"\[object] (Illuminate\\\Database\\\QueryException(code: 1045): SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (using password: YES) (SQL: select \* from information\_schema.tables where table\_schema = \[MASKED] and table\_name = migrations and table\_type = 'BASE TABLE') at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:712) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('select \* from i...', Array, Object(Closure)) #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(359): Illuminate\\\Database\\\Connection->run('select \* from i...', Array, Object(Closure)) #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/MySqlBuilder.php(43): Illuminate\\\Database\\\Connection->select('select \* from i...', Array) #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/DatabaseMigrationRepository.php(169): Illuminate\\\Database\\\Schema\\\MySqlBuilder->hasTable('migrations') #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(673): Illuminate\\\Database\\\Migrations\\\DatabaseMigrationRepository->repositoryExists() #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(110): Illuminate\\\Database\\\Migrations\\\Migrator->repositoryExists() #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(78): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->prepareDatabase() #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #15 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #17 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #18 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #19 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #22 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #23 {main} \[previous exception] \[object] (PDOException(code: 1045): SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (using password: YES) at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php:70) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(70): PDO->\_\_construct('mysql:host=mysq...', '\[MASKED]', Object(SensitiveParameterValue), Array) #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(45): Illuminate\\\Database\\\Connectors\\\Connector->createPdoConnection('mysql:host=mysq...', '\[MASKED]', '\[MASKED]', Array) #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/MySqlConnector.php(24): Illuminate\\\Database\\\Connectors\\\Connector->createConnection('mysql:host=mysq...', Array, Array) #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/ConnectionFactory.php(184): Illuminate\\\Database\\\Connectors\\\MySqlConnector->connect(Array) #4 \[internal function]: Illuminate\\\Database\\\Connectors\\\ConnectionFactory->Illuminate\\\Database\\\Connectors\\\\{closure}() #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(1064): call\_user\_func(Object(Closure)) #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(1100): Illuminate\\\Database\\\Connection->getPdo() #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(442): Illuminate\\\Database\\\Connection->getReadPdo() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(368): Illuminate\\\Database\\\Connection->getPdoForSelect(true) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(705): Illuminate\\\Database\\\Connection->Illuminate\\\Database\\\\{closure}('select \* from i...', Array) #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('select \* from i...', Array, Object(Closure)) #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(359): Illuminate\\\Database\\\Connection->run('select \* from i...', Array, Object(Closure)) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/MySqlBuilder.php(43): Illuminate\\\Database\\\Connection->select('select \* from i...', Array) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/DatabaseMigrationRepository.php(169): Illuminate\\\Database\\\Schema\\\MySqlBuilder->hasTable('migrations') #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(673): Illuminate\\\Database\\\Migrations\\\DatabaseMigrationRepository->repositoryExists() #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(110): Illuminate\\\Database\\\Migrations\\\Migrator->repositoryExists() #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(78): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->prepareDatabase() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #22 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #24 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #25 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #26 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #27 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #31 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #32 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #33 {main} "} In Connection.php line 712: SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (usi ng password: YES) (SQL: select \* from information\_schema.tables where table \_schema = \[MASKED] and table\_name = migrations and table\_type = 'BASE TABL E') In Connector.php line 70: SQLSTATE\[HY000] \[1045] Access denied for user '\[MASKED]'@'172.19.0.4' (usi ng password: YES) Cleaning up project directory and file based variables 00:01 ERROR: Job failed: exit status 1Selesai berpikir selama 14m 11s

Oke, aku rapihkan dan sekalian aku kasih **arahan yang benar-benar step-by-step** sampai error `SQLSTATE[HY000] [1045] Access denied` hilang.

***

### 1) Kenapa error 1045 masih muncul?

Error `1045 Access denied` itu **hampir selalu karena password user MySQL di volume lama tidak sama** dengan password yang kamu set sekarang di CI/CD variables.

* Docker MySQL **hanya membuat user `MYSQL_USER` + `MYSQL_PASSWORD` saat volume DB masih kosong (first init)**.
* Kalau volume `threebody-dev_mysql_data` sudah pernah dibuat sebelumnya, lalu kamu ubah `DEV_MYSQL_PASSWORD`, maka MySQL **tidak otomatis update password user** → akhirnya Laravel gagal login → 1045.

Jadi walaupun `.env.dev.compose` sudah kamu tulis benar, DB yang lama masih “ingat” password lama.

Solusi ada 2:

1. **Reset DB volume sekali (paling gampang untuk dev)**
2. Atau **paksa update user/password via root setiap deploy** (biar otomatis)

Aku buatkan CI yang melakukan **opsi #2** otomatis.

***

### 2) `.gitlab-ci.yml` versi rapih + auto-fix MySQL user (COPY PASTE)

Silakan **replace isi `.gitlab-ci.yml` kamu** dengan ini:

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  # konsisten, tidak bikin stack baru & bentrok port
  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

default:
  tags: ["deploy"] # pastikan hanya runner DevOps_43_runner yang punya tag "deploy"
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

# -------------------------
# BUILD + PUSH
# -------------------------
build_goapi:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/goapi:$TAG"

build_laravel:
  stage: build
  retry: 2
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/laravel:$TAG"

build_frontend:
  stage: build
  script:
    - |
      set -euo pipefail
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"
      docker build -t "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend .
      docker push "$HARBOR_URL/$HARBOR_PROJECT/frontend:$TAG"

# -------------------------
# DEPLOY DEV (auto-fix 1045)
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
      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      mkdir -p "$DEPLOY_DIR"

      # sync file deploy
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      # cert (sekali saja)
      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      # env untuk compose + laravel
      cat > "$DEPLOY_DIR/.env.dev.compose" <<ENV
MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=${DEV_MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
ENV

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      # ---- AUTO FIX 1045: paksa user/password mysql sesuai env (volume lama aman) ----
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc "
        set -e
        for i in \$(seq 1 60); do
          mysqladmin ping -uroot -p\"\$MYSQL_ROOT_PASSWORD\" --silent && break
          sleep 2
        done

        mysql -uroot -p\"\$MYSQL_ROOT_PASSWORD\" -e \"
          CREATE DATABASE IF NOT EXISTS \$MYSQL_DATABASE;
          CREATE USER IF NOT EXISTS '\$MYSQL_USER'@'%' IDENTIFIED BY '\$MYSQL_PASSWORD';
          ALTER USER '\$MYSQL_USER'@'%' IDENTIFIED BY '\$MYSQL_PASSWORD';
          GRANT ALL PRIVILEGES ON \$MYSQL_DATABASE.* TO '\$MYSQL_USER'@'%';
          FLUSH PRIVILEGES;
        \"
      "

      # clear cache biar env DB kepakai
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan config:clear
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan cache:clear

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps

# -------------------------
# RESET DEV DB (manual)
# -------------------------
reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - |
      set -euo pipefail
      cd "$DEPLOY_DIR"

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true
      docker volume rm -f threebody-dev_mysql_data || true

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan config:clear
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

**Kenapa ini lebih “beres”?**

* Deploy selalu pakai stack yang sama (`COMPOSE_PROJECT_NAME` fix).
* `.env.dev.compose` dibuat konsisten di deploy dir.
* Setelah MySQL up, CI **paksa create/alter user** sesuai password terbaru → **1045 hilang walaupun volume lama**.
* Laravel `config:clear` supaya tidak nyangkut config cache lama.

***

### 3) Cara commit + push (lengkap, tinggal copas)

Jalankan dari repo kamu:

```bash
cd /home/cikal/three-body-problem-main

git pull origin main

# edit file
nano .gitlab-ci.yml

# cek perubahan
git diff
git status

# commit
git add .gitlab-ci.yml
git commit -m "Fix deploy: auto-sync mysql user/password + clear laravel config"

# push
git push origin main
```

Setelah push, pipeline jalan lagi.

***

### 4) Kalau mau “instant fix” sekarang (tanpa nunggu CI)

Di server (VM) langsung:

```bash
cd /home/gitlab-runner/threebody-deploy

# paksa update user/password mysql
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
CREATE DATABASE IF NOT EXISTS $MYSQL_DATABASE;
CREATE USER IF NOT EXISTS '\''$MYSQL_USER'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_PASSWORD'\'';
ALTER USER '\''$MYSQL_USER'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_PASSWORD'\'';
GRANT ALL PRIVILEGES ON $MYSQL_DATABASE.* TO '\''$MYSQL_USER'\''@'\''%'\'';
FLUSH PRIVILEGES;
"
'

# clear cache laravel lalu migrate
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan config:clear
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

Lalu test:

* buka `https://dev.local/api/laravel/api/products`

***

### 5) Runner: biar hanya DevOps\_43\_runner (rapih)

Di GitLab UI kamu sekarang sudah tinggal 1 runner (bagus). Pastikan ini:

1. Runner itu punya tag: **deploy**
2. Runner lain **hapus / pause** (kalau masih muncul di “Other available project runners”)

Di server, untuk cek cuma 1 runner yang aktif:

```bash
sudo gitlab-runner list
sudo gitlab-runner verify
```

> Catatan penting: kamu sempat paste token runner di chat. Demi keamanan, sebaiknya **rotate token runner** (hapus runner lama di GitLab → register ulang). Ini mencegah token disalahgunakan.

***

Kalau setelah pakai `.gitlab-ci.yml` di atas masih 1045, biasanya berarti **docker-compose.dev.yml belum memasukkan env DB ke container laravel**. Kalau kamu kirim isi bagian `services: laravel:` dari `docker-compose.dev.yml`, aku bisa tunjukkan baris tepat yang harus ditambahkan (biasanya `env_file:` atau `environment:`).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000003c6872078bfc50ccda656065\&ts=490475\&p=fs\&cid=1\&sig=25eb1eec4da27b4aafe43b54096c03d485b5e8f6fc1300ca32589ffdeaae4c7d\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000003b0471fa9fc69d708dfe335e\&ts=490475\&p=fs\&cid=1\&sig=8138ab85c3be5237c94532f0fa4b241dd31046a7edc4c56a1e44fe44d9d2e1be\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000054647207bf113e2277856d60\&ts=490475\&p=fs\&cid=1\&sig=36a07e9117893ee397e0ed0056b7c09591b29b349fc1433626eed82374fbc699\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000225071faa5b7b44c01ead309\&ts=490475\&p=fs\&cid=1\&sig=9d4071b3c8795efb6ec52ecc04fb8805679cbe76dfe2754858cc08c742f8a668\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000000fc0720797ed84cbf5525fb5\&ts=490475\&p=fs\&cid=1\&sig=fd59a918842cb71a48bb8fc528f3e64e85c8b5bc694d623f119303b4c744b8d8\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" # konsisten, tidak bikin stack baru & bentrok port COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" default: tags: \["deploy"] # pastikan hanya runner DevOps\_43\_runner yang punya tag "deploy" before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH # ------------------------- build\_goapi: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (auto-fix 1045) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # env untuk compose + laravel cat > "$DEPLOY\_DIR/.env.dev.compose" <\<ENV MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_CONNECTION=mysql DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} ENV cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # ---- AUTO FIX 1045: paksa user/password mysql sesuai env (volume lama aman) ---- docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc " set -e for i in \\$(seq 1 60); do mysqladmin ping -uroot -p\\"\\$MYSQL\_ROOT\_PASSWORD\\" --silent && break sleep 2 done mysql -uroot -p\\"\\$MYSQL\_ROOT\_PASSWORD\\" -e \\" CREATE DATABASE IF NOT EXISTS \\$MYSQL\_DATABASE; CREATE USER IF NOT EXISTS '\\$MYSQL\_USER'@'%' IDENTIFIED BY '\\$MYSQL\_PASSWORD'; ALTER USER '\\$MYSQL\_USER'@'%' IDENTIFIED BY '\\$MYSQL\_PASSWORD'; GRANT ALL PRIVILEGES ON \\$MYSQL\_DATABASE.\* TO '\\$MYSQL\_USER'@'%'; FLUSH PRIVILEGES; \\" " # clear cache biar env DB kepakai docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan config:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan cache:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps # ------------------------- # RESET DEV DB (manual) # ------------------------- reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail cd "$DEPLOY\_DIR" docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan config:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml git commit -m "Fix deploy: auto-sync mysql user/password + clear laravel config" # push git push origin main \[main f3adfad] Fix deploy: auto-sync mysql user/password + clear laravel config 1 file changed, 100 insertions(+), 74 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.21 KiB | 617.00 KiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 8afeaf9..f3adfad main -> main cikal@devops-ci:\~/three-body-problem-main$ cd /home/gitlab-runner/threebody-deploy # paksa update user/password mysql docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " CREATE DATABASE IF NOT EXISTS $MYSQL\_DATABASE; CREATE USER IF NOT EXISTS '\\''$MYSQL\_USER'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; ALTER USER '\\''$MYSQL\_USER'\\''@'\\''%'\\'' IDENTIFIED BY '\\''$MYSQL\_PASSWORD'\\''; GRANT ALL PRIVILEGES ON $MYSQL\_DATABASE.\* TO '\\''$MYSQL\_USER'\\''@'\\''%'\\''; FLUSH PRIVILEGES; " ' # clear cache laravel lalu migrate docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan config:clear docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force mysql: \[Warning] Using a password on the command line interface can be insecure. ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) Configuration cache cleared! \[2025-12-14 10:44:51] local.ERROR: SQLSTATE\[HY000] \[1045] Access denied for user 'threebody'@'172.19.0.4' (using password: YES) (SQL: select \* from information\_schema.tables where table\_schema = threebody and table\_name = migrations and table\_type = 'BASE TABLE') {"exception":"\[object] (Illuminate\\\Database\\\QueryException(code: 1045): SQLSTATE\[HY000] \[1045] Access denied for user 'threebody'@'172.19.0.4' (using password: YES) (SQL: select \* from information\_schema.tables where table\_schema = threebody and table\_name = migrations and table\_type = 'BASE TABLE') at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php:712) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('select \* from i...', Array, Object(Closure)) #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(359): Illuminate\\\Database\\\Connection->run('select \* from i...', Array, Object(Closure)) #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/MySqlBuilder.php(43): Illuminate\\\Database\\\Connection->select('select \* from i...', Array) #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/DatabaseMigrationRepository.php(169): Illuminate\\\Database\\\Schema\\\MySqlBuilder->hasTable('migrations') #4 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(673): Illuminate\\\Database\\\Migrations\\\DatabaseMigrationRepository->repositoryExists() #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(110): Illuminate\\\Database\\\Migrations\\\Migrator->repositoryExists() #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(78): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->prepareDatabase() #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #15 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #17 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #18 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #19 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #22 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #23 {main} \[previous exception] \[object] (PDOException(code: 1045): SQLSTATE\[HY000] \[1045] Access denied for user 'threebody'@'172.19.0.4' (using password: YES) at /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php:70) \[stacktrace] #0 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(70): PDO->\_\_construct('mysql:host=mysq...', 'threebody', Object(SensitiveParameterValue), Array) #1 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(45): Illuminate\\\Database\\\Connectors\\\Connector->createPdoConnection('mysql:host=mysq...', 'threebody', 'Harbor12345', Array) #2 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/MySqlConnector.php(24): Illuminate\\\Database\\\Connectors\\\Connector->createConnection('mysql:host=mysq...', Array, Array) #3 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connectors/ConnectionFactory.php(184): Illuminate\\\Database\\\Connectors\\\MySqlConnector->connect(Array) #4 \[internal function]: Illuminate\\\Database\\\Connectors\\\ConnectionFactory->Illuminate\\\Database\\\Connectors\\\\{closure}() #5 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(1064): call\_user\_func(Object(Closure)) #6 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(1100): Illuminate\\\Database\\\Connection->getPdo() #7 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(442): Illuminate\\\Database\\\Connection->getReadPdo() #8 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(368): Illuminate\\\Database\\\Connection->getPdoForSelect(true) #9 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(705): Illuminate\\\Database\\\Connection->Illuminate\\\Database\\\\{closure}('select \* from i...', Array) #10 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(672): Illuminate\\\Database\\\Connection->runQueryCallback('select \* from i...', Array, Object(Closure)) #11 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Connection.php(359): Illuminate\\\Database\\\Connection->run('select \* from i...', Array, Object(Closure)) #12 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Schema/MySqlBuilder.php(43): Illuminate\\\Database\\\Connection->select('select \* from i...', Array) #13 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/DatabaseMigrationRepository.php(169): Illuminate\\\Database\\\Schema\\\MySqlBuilder->hasTable('migrations') #14 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(673): Illuminate\\\Database\\\Migrations\\\DatabaseMigrationRepository->repositoryExists() #15 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(110): Illuminate\\\Database\\\Migrations\\\Migrator->repositoryExists() #16 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(78): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->prepareDatabase() #17 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Migrations/Migrator.php(606): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->Illuminate\\\Database\\\Console\\\Migrations\\\\{closure}() #18 /var/www/html/vendor/laravel/framework/src/Illuminate/Database/Console/Migrations/MigrateCommand.php(77): Illuminate\\\Database\\\Migrations\\\Migrator->usingConnection(NULL, Object(Closure)) #19 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(36): Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand->handle() #20 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Util.php(40): Illuminate\\\Container\\\BoundMethod::Illuminate\\\Container\\\\{closure}() #21 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(93): Illuminate\\\Container\\\Util::unwrapIfClosure(Object(Closure)) #22 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/BoundMethod.php(35): Illuminate\\\Container\\\BoundMethod::callBoundMethod(Object(Illuminate\\\Foundation\\\Application), Array, Object(Closure)) #23 /var/www/html/vendor/laravel/framework/src/Illuminate/Container/Container.php(653): Illuminate\\\Container\\\BoundMethod::call(Object(Illuminate\\\Foundation\\\Application), Array, Array, NULL) #24 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(136): Illuminate\\\Container\\\Container->call(Array) #25 /var/www/html/vendor/symfony/console/Command/Command.php(298): Illuminate\\\Console\\\Command->execute(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #26 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Command.php(120): Symfony\\\Component\\\Console\\\Command\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Illuminate\\\Console\\\OutputStyle)) #27 /var/www/html/vendor/symfony/console/Application.php(1040): Illuminate\\\Console\\\Command->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #28 /var/www/html/vendor/symfony/console/Application.php(301): Symfony\\\Component\\\Console\\\Application->doRunCommand(Object(Illuminate\\\Database\\\Console\\\Migrations\\\MigrateCommand), Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #29 /var/www/html/vendor/symfony/console/Application.php(171): Symfony\\\Component\\\Console\\\Application->doRun(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #30 /var/www/html/vendor/laravel/framework/src/Illuminate/Console/Application.php(94): Symfony\\\Component\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #31 /var/www/html/vendor/laravel/framework/src/Illuminate/Foundation/Console/Kernel.php(129): Illuminate\\\Console\\\Application->run(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #32 /var/www/html/artisan(35): Illuminate\\\Foundation\\\Console\\\Kernel->handle(Object(Symfony\\\Component\\\Console\\\Input\\\ArgvInput), Object(Symfony\\\Component\\\Console\\\Output\\\ConsoleOutput)) #33 {main} "} In Connection.php line 712: SQLSTATE\[HY000] \[1045] Access denied for user 'threebody'@'172.19.0.4' (usi ng password: YES) (SQL: select \* from information\_schema.tables where table \_schema = threebody and table\_name = migrations and table\_type = 'BASE TABL E') In Connector.php line 70: SQLSTATE\[HY000] \[1045] Access denied for user 'threebody'@'172.19.0.4' (usi ng password: YES) cikal@devops-ci:/home/gitlab-runner/threebody-deploy$ sudo gitlab-runner list sudo gitlab-runner verify \[sudo] password for cikal: Runtime platform arch=amd64 os=linux pid=303066 revision=5908bc19 version=18.6.5 Listing configured runners ConfigFile=/etc/gitlab-runner/config.toml devops-ci Executor=shell Token=glrt-ArpoA9OGuN\_vrA\_25AdWUm86MQpwOjE5c3Q1NAp0OjMKdTpocWd0ahg.01.1j1mzr2o7 URL=https://gitlab.com Runtime platform arch=amd64 os=linux pid=303098 revision=5908bc19 version=18.6.5 Running in system-mode. Verifying runner... is valid correlation\_id=9add1c02811c4083-ATL runner=ArpoA9OGu cikal@devops-ci:/home/gitlab-runner/threebody-deploy$Selesai berpikir selama 6m 32s

Masalah `1045 Access denied` kamu itu **bukan karena CI-nya kurang “clear cache”**, tapi karena **MySQL Docker official image cuma set `MYSQL_ROOT_PASSWORD/MYSQL_USER/MYSQL_PASSWORD` saat container pertama kali init volume**. Kalau **volume `threebody-dev_mysql_data` sudah pernah kebentuk** dengan password lama, lalu kamu ganti `DEV_MYSQL_ROOT_PASSWORD` / `DEV_MYSQL_PASSWORD`, maka:

* Laravel pakai password **baru** → MySQL user di dalam volume masih password **lama** → `1045`
* Kamu coba “auto-fix” via root (`mysql -uroot -p...`) → root password di volume **nggak sama** dengan env sekarang → root juga `1045`

Jadi langkah paling cepat untuk beresin kondisi sekarang adalah **reset DB volume sekali** (dev biasanya aman). Setelah itu deploy akan stabil selama password tidak kamu ubah-ubah.

***

### 1) Fix sekarang (paling aman & cepat)

Pilih salah satu:

#### Opsi A — dari GitLab (recommended)

Jalankan job **manual** `reset_dev_db` (di stage deploy). Itu akan drop volume dan init ulang sesuai env terbaru.

#### Opsi B — manual di server (kalau mau langsung)

```bash
cd /home/gitlab-runner/threebody-deploy

docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v
docker volume rm -f threebody-dev_mysql_data || true

docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml pull
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d

docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

> Kalau kamu **tidak ingin data hilang**, satu-satunya cara “tanpa reset” adalah: **kembalikan `DEV_MYSQL_ROOT_PASSWORD` & `DEV_MYSQL_PASSWORD` persis sama seperti saat volume pertama dibuat** (atau lakukan prosedur reset root password MySQL dengan `--skip-grant-tables`, tapi itu jauh lebih ribet).

***

### 2) .gitlab-ci.yml versi rapi + fail-fast kalau volume/password mismatch

Ini versi yang rapi, reusable, dan **kalau root login gagal dia stop dengan pesan jelas** (bukan maksa ALTER USER yang akhirnya gagal terus).

Silakan **copy–paste full**:

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  # konsisten (nggak bikin stack baru yg bentrok port)
  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

default:
  tags: ["deploy"] # pastikan hanya runner DevOps_43_runner yg punya tag ini
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

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
# DEPLOY PREP (reuse)
# -------------------------
.deploy_prepare: &deploy_prepare |
  set -euo pipefail
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

  mkdir -p "$DEPLOY_DIR"

  # sync file deploy
  if command -v rsync >/dev/null 2>&1; then
    rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
  else
    rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
    cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
  fi

  # cert (sekali saja)
  mkdir -p "$DEPLOY_DIR/certs/dev"
  if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
    docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
      apk add --no-cache openssl >/dev/null
      openssl req -newkey rsa:2048 -nodes \
        -keyout /out/dev.key \
        -x509 -days 365 \
        -out /out/dev.crt \
        -subj "/CN=dev.local"
    '
  fi

  # env untuk compose + laravel
  cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=${DEV_MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
EOF

  cd "$DEPLOY_DIR"

  # export vars dari env file (buat dipakai di script)
  set -a
  source .env.dev.compose
  set +a

  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

# -------------------------
# DEPLOY DEV (main only)
# -------------------------
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      # FAIL-FAST: kalau root password mismatch (volume lama), kasih instruksi jelas
      if ! docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql \
          mysqladmin ping -uroot -p"$MYSQL_ROOT_PASSWORD" --silent; then
        echo "ERROR: Tidak bisa login root MySQL."
        echo "Biasanya ini karena volume mysql sudah pernah dibuat dengan root password berbeda."
        echo "Solusi: jalankan job manual reset_dev_db (akan recreate volume) ATAU kembalikan DEV_MYSQL_ROOT_PASSWORD ke password lama saat volume pertama dibuat."
        exit 1
      fi

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps

# -------------------------
# RESET DEV DB (manual)
# -------------------------
reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true
      docker volume rm -f threebody-dev_mysql_data || true

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

***

### 3) Cara push (lengkap, tinggal copas)

Di repo kamu:

```bash
cd /home/cikal/three-body-problem-main

nano .gitlab-ci.yml
# paste file di atas, save

git diff
git status

git add .gitlab-ci.yml
git commit -m "CI: deploy fail-fast for mysql volume mismatch + reusable deploy prep"
git push origin main
```

***

### 4) Kenapa error kamu “nggak bisa auto-fix” sekarang?

Karena log kamu menunjukkan ini:

* `ERROR 1045 root@localhost` saat eksekusi di container mysql\
  → root password di volume **beda** dengan yang sekarang kamu set di env/CI.

Jadi **jalankan `reset_dev_db` sekali** (atau recreate volume manual) supaya root/user/password sinkron. Setelah itu, deploy otomatis harusnya mulus dan Laravel API nggak 500 lagi.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000d73c7209ac5300909d7ff39f\&ts=490475\&p=fs\&cid=1\&sig=fbb1acb6d2cbdfaba8bbdfa19b4b6b6cf00f8126f5a8b6d21637cca092d9eff6\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000002e8872068fbbf259c4b97d05\&ts=490475\&p=fs\&cid=1\&sig=f0f84f4c726972939add029d5ff23fc49ea78f8ab41e94049c188b685d6bea58\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000001638720993c04afe7f68af91\&ts=490475\&p=fs\&cid=1\&sig=d73153673f4e71900c91195cd2b3621342f6e05d849ecf05768150da16158c80\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000098e472068e90bd4aa4041a10\&ts=490475\&p=fs\&cid=1\&sig=9998e8b391ae27326a6d3ebbf18f9022988b22ca6dc510c8cb07407048f48c94\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000827c72098fe4f703b697da02\&ts=490475\&p=fs\&cid=1\&sig=8cefa54c640249cecf84d630e8c79682ae1772803a3ef802e3bdab9f22cd7f84\&v=0)cikal@devops-ci:/home/gitlab-runner/threebody-deploy$ cd /home/gitlab-runner/threebody-deploy docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml down -v docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force \[+] down 6/7 ✔ Container threebody-dev-nginx-dev-1 Removed 1.5s ✔ Container threebody-dev-frontend-1 Removed 1.1s ✔ Container threebody-dev-goapi-1 Removed 1.3s ✔ Container threebody-dev-laravel-1 Removed 2.5s ✔ Container threebody-dev-mysql-1 Removed 2.4s ⠹ Network threebody-dev\_devnet Removing 0.3s ✔ Volume threebody-dev\_mysql\_data Removed 0.1s threebody-dev\_mysql\_data \[+] up 7/7/5 ✔ Network threebody-dev\_devnet Created 0.4s ✔ Volume threebody-dev\_mysql\_data Created 0.0s ✔ Container threebody-dev-mysql-1 Healthy 69.8s ✔ Container threebody-dev-goapi-1 Created 0.5s ✔ Container threebody-dev-laravel-1 Created 0.7s ✔ Container threebody-dev-frontend-1 Created 0.4s ✔ Container threebody-dev-nginx-dev-1 Created 0.2s Cached events cleared! Compiled views cleared! Application cache cleared! Route cache cleared! Configuration cache cleared! Compiled services and packages files removed! Caches cleared successfully! Migration table created successfully. Migrating: 2014\_10\_12\_000000\_create\_users\_table Migrated: 2014\_10\_12\_000000\_create\_users\_table (653.12ms) Migrating: 2014\_10\_12\_100000\_create\_password\_resets\_table Migrated: 2014\_10\_12\_100000\_create\_password\_resets\_table (542.59ms) Migrating: 2019\_08\_19\_000000\_create\_failed\_jobs\_table Migrated: 2019\_08\_19\_000000\_create\_failed\_jobs\_table (383.20ms) Migrating: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table Migrated: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table (757.20ms) Migrating: 2025\_07\_25\_075615\_create\_products\_table Migrated: 2025\_07\_25\_075615\_create\_products\_table (42.96ms) cikal@devops-ci:/home/gitlab-runner/threebody-deploy$ ll total 24 drwxrwxr-x 4 gitlab-runner gitlab-runner 4096 Dec 14 10:22 ./ drwxr-xr-x 5 gitlab-runner gitlab-runner 4096 Dec 14 07:21 ../ -rw-rw-r-- 1 gitlab-runner gitlab-runner 384 Dec 14 10:22 .env.dev.compose drwxrwxr-x 3 gitlab-runner gitlab-runner 4096 Dec 14 07:21 certs/ drwxrwxr-x 4 gitlab-runner gitlab-runner 4096 Dec 14 09:23 devops/ -rw-rw-r-- 1 gitlab-runner gitlab-runner 1897 Dec 14 09:23 docker-compose.dev.yml cikal@devops-ci:/home/gitlab-runner/threebody-deploy$ cd /home/cikal/three-body-problem-main nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml git commit -m "CI: deploy fail-fast for mysql volume mismatch + reusable deploy prep" git push origin main \[main e381771] CI: deploy fail-fast for mysql volume mismatch + reusable deploy prep 1 file changed, 80 insertions(+), 88 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.08 KiB | 1.08 MiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git f3adfad..e381771 main -> main cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" # konsisten (nggak bikin stack baru yg bentrok port) COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" default: tags: \["deploy"] # pastikan hanya runner DevOps\_43\_runner yg punya tag ini before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY PREP (reuse) # ------------------------- .deploy\_prepare: \&deploy\_prepare | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # env untuk compose + laravel cat > "$DEPLOY\_DIR/.env.dev.compose" <\<EOF MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_CONNECTION=mysql DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} EOF cd "$DEPLOY\_DIR" # export vars dari env file (buat dipakai di script) set -a source .env.dev.compose set +a echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # ------------------------- # DEPLOY DEV (main only) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # FAIL-FAST: kalau root password mismatch (volume lama), kasih instruksi jelas if ! docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql \ mysqladmin ping -uroot -p"$MYSQL\_ROOT\_PASSWORD" --silent; then echo "ERROR: Tidak bisa login root MySQL." echo "Biasanya ini karena volume mysql sudah pernah dibuat dengan root password berbeda." echo "Solusi: jalankan job manual reset\_dev\_db (akan recreate volume) ATAU kembalikan DEV\_MYSQL\_ROOT\_PASSWORD ke password lama saat volume pertama dibuat." exit 1 fi docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps # ------------------------- # RESET DEV DB (manual) # ------------------------- reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ mantap tapi error, bantu perbaiki yaSelesai berpikir selama 3m 20s

Error **`yaml invalid: could not find expected ':' while scanning a simple key`** itu hampir pasti karena **isi heredoc (`MYSQL_ROOT_PASSWORD=...` dst) tidak ter-indent di dalam blok YAML `|`**, jadi YAML mengira baris `MYSQL_ROOT_PASSWORD=...` adalah _key YAML_ (padahal bukan), lalu error.

Di file kamu yang terakhir, bagian ini penyebabnya:

```yml
cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
MYSQL_ROOT_PASSWORD=...
...
EOF
```

Baris `MYSQL_ROOT_PASSWORD=...` harus ikut **masuk ke literal block** (ter-indent), biar tetap dianggap “teks script”.

Di bawah ini versi `.gitlab-ci.yml` yang sudah:

* **Fix YAML (indent heredoc benar)** ✅
* **Wait MySQL siap** ✅
* **Fail-fast jika root password mismatch (volume lama)** ✅
* **Auto-sync user/password DB app (kalau yang mismatch cuma user pass, bukan root)** ✅
* **Clear cache Laravel sebelum migrate** ✅

Silakan **copy-paste full**:

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  # konsisten (nggak bikin stack baru yg bentrok port)
  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

default:
  tags: ["deploy"] # pastikan hanya runner yg punya tag "deploy"
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

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
# DEPLOY PREP (reuse)
# -------------------------
.deploy_prepare: &deploy_prepare |
  set -euo pipefail
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

  mkdir -p "$DEPLOY_DIR"

  # sync file deploy
  if command -v rsync >/dev/null 2>&1; then
    rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
  else
    rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
    cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
  fi

  # cert (sekali saja)
  mkdir -p "$DEPLOY_DIR/certs/dev"
  if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
    docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
      apk add --no-cache openssl >/dev/null
      openssl req -newkey rsa:2048 -nodes \
        -keyout /out/dev.key \
        -x509 -days 365 \
        -out /out/dev.crt \
        -subj "/CN=dev.local"
    '
  fi

  # IMPORTANT: heredoc harus ter-indent supaya YAML valid
  cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
  MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
  MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
  MYSQL_USER=threebody
  MYSQL_DATABASE=threebody

  DB_CONNECTION=mysql
  DB_HOST=mysql
  DB_PORT=3306
  DB_DATABASE=threebody
  DB_USERNAME=threebody
  DB_PASSWORD=${DEV_MYSQL_PASSWORD}

  HARBOR_URL=${HARBOR_URL}
  HARBOR_PROJECT=${HARBOR_PROJECT}
  CI_COMMIT_SHA=${TAG}
  APP_KEY=${DEV_APP_KEY}
  EOF

  cd "$DEPLOY_DIR"

  # load vars buat dipakai di script (POSIX safe)
  set -a
  . ./.env.dev.compose
  set +a

  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

# -------------------------
# DEPLOY DEV (main only)
# -------------------------
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      set -euo pipefail

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans

      # tunggu mysql siap + cek root login (fail-fast kalau root mismatch)
      if ! docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
        set -e
        for i in $(seq 1 60); do
          mysqladmin ping -uroot -p"$MYSQL_ROOT_PASSWORD" --silent && exit 0
          sleep 2
        done
        exit 1
      '; then
        echo "ERROR: Tidak bisa login root MySQL."
        echo "Biasanya karena volume mysql sudah pernah dibuat dengan root password berbeda."
        echo "Solusi:"
        echo " - Jalankan job manual: reset_dev_db (recreate volume), atau"
        echo " - Kembalikan DEV_MYSQL_ROOT_PASSWORD ke password lama saat volume pertama dibuat."
        exit 1
      fi

      # auto-sync user/password aplikasi (kalau yang mismatch cuma user pass)
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc '
        set -e
        mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
      '

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps

# -------------------------
# RESET DEV DB (manual)
# -------------------------
reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      set -euo pipefail

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true
      docker volume rm -f threebody-dev_mysql_data || true

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d

      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear
      docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force
```

#### Cara push ulang

Jalankan dari repo kamu:

```bash
cd /home/cikal/three-body-problem-main

nano .gitlab-ci.yml   # paste isi di atas

git add .gitlab-ci.yml
git commit -m "CI: fix YAML heredoc + mysql wait/sync user"
git push origin main
```

Kalau setelah ini pipeline jalan tapi `deploy_dev` fail-fast soal root mismatch, itu expected — tinggal klik job **manual `reset_dev_db`** sekali untuk recreate volume (seperti yang kamu sudah lakukan manual di server).

> Catatan keamanan: kamu sempat mem-paste token runner di chat sebelumnya. Setelah beres, sebaiknya **rotate runner token** (dan pastikan semua secret di GitLab Variables sudah Masked/Protected).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000de247206af4546a29fc76b2a\&ts=490475\&p=fs\&cid=1\&sig=97029d09c4d755ff2e591f668c7a7f7fb03a6785c75ffba62e4d70d329081218\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000d0e872099501a0de7c0a2962\&ts=490475\&p=fs\&cid=1\&sig=63b327c181cc1d761a485d3f96d0409732057f10074ec05a022abb65b42e3f7f\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000839472069a31a151aed140f8\&ts=490475\&p=fs\&cid=1\&sig=10b5119a2e4074271119ee60469d6103c2506198eec0b662a5f827ad3e39c1db\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000bbe07209995c8cd4bd8e2d38\&ts=490475\&p=fs\&cid=1\&sig=df870b80505450bf82acbe90b99865c55ae9e0447f295656d6bbf61958060ee1\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000683c72068aa41868fb1b4e20\&ts=490475\&p=fs\&cid=1\&sig=df9dab2ad3a06eee74e9d32c81f40df1bf4e40d358b2697ec933231e33ce37b1\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" # konsisten (nggak bikin stack baru yg bentrok port) COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" default: tags: \["deploy"] # pastikan hanya runner yg punya tag "deploy" before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY PREP (reuse) # ------------------------- .deploy\_prepare: \&deploy\_prepare | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # IMPORTANT: heredoc harus ter-indent supaya YAML valid cat > "$DEPLOY\_DIR/.env.dev.compose" <\<EOF MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_CONNECTION=mysql DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} EOF cd "$DEPLOY\_DIR" # load vars buat dipakai di script (POSIX safe) set -a . ./.env.dev.compose set +a echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # ------------------------- # DEPLOY DEV (main only) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | set -euo pipefail docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # tunggu mysql siap + cek root login (fail-fast kalau root mismatch) if ! docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' set -e for i in $(seq 1 60); do mysqladmin ping -uroot -p"$MYSQL\_ROOT\_PASSWORD" --silent && exit 0 sleep 2 done exit 1 '; then echo "ERROR: Tidak bisa login root MySQL." echo "Biasanya karena volume mysql sudah pernah dibuat dengan root password berbeda." echo "Solusi:" echo " - Jalankan job manual: reset\_dev\_db (recreate volume), atau" echo " - Kembalikan DEV\_MYSQL\_ROOT\_PASSWORD ke password lama saat volume pertama dibuat." exit 1 fi # auto-sync user/password aplikasi (kalau yang mismatch cuma user pass) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' set -e mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps # ------------------------- # RESET DEV DB (manual) # ------------------------- reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | set -euo pipefail docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml git commit -m "CI: fix YAML heredoc + mysql wait/sync user" git push origin main \[main aac2285] CI: fix YAML heredoc + mysql wait/sync user 1 file changed, 50 insertions(+), 26 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.02 KiB | 1.02 MiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git e381771..aac2285 main -> main cikal@devops-ci:\~/three-body-problem-main$ mantapSelesai berpikir selama 1m 40s

Sip, ini sudah **bener**—kelihatan dari pipeline **Passed** dan endpoint **Laravel/Go** sama-sama balikin data.

Ringkasnya, error yang tadi kamu alami itu 2 sumber utama:

1. **YAML invalid**\
   Penyebab paling umum: _heredoc_ (`<<EOF ... EOF`) di dalam `script: |`/anchor **nggak “ikut” indent YAML**, jadi YAML nganggep baris `MYSQL_ROOT_PASSWORD=...` itu key baru → muncullah error “could not find expected ':'”.\
   Fix kamu sudah tepat: semua baris heredoc **tetap berada di dalam block scalar** (indent konsisten), termasuk penutup `EOF`.
2. **MySQL 1045 (Access denied)**\
   Ada 2 kasus:
   * **Root password mismatch karena volume lama** → _nggak bisa di-auto-fix_ (benar kamu bikin **fail-fast** + instruksi `reset_dev_db`).
   * **User app (`threebody`) password mismatch** → bisa di-fix dengan `CREATE/ALTER USER + GRANT` (kamu sudah tambahin auto-sync setelah root login lolos).

Kalau mau makin “bulletproof”, saran kecil:

* Pastikan baris `EOF` penutup heredoc **sejajar** dengan `cat > ... <<EOF` di dalam block (jangan ada indent ekstra “khusus” di heredoc saja), supaya file `.env.dev.compose` tidak kebawa spasi di awal (dotenv/compose kadang sensitif).
* Tambahin `php artisan config:clear` sudah ke-cover oleh `optimize:clear`, jadi aman.

Kalau nanti kamu kirim potongan log job `deploy_dev` terakhir (yang hijau itu), aku bisa bantu cek apakah masih ada warning yang perlu diberesin (mis. mysqladmin ping, migrate idempotency, dsb.).

dibawah ini log job deploy\_dev nya Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on devops-ci... Getting source from Git repository 00:02 Gitaly correlation ID: 9add5c7f8678fe1c-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out aac2285a as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:20 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command Login Succeeded $ set -euo pipefail # collapsed multi-line command Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image \[MASKED]/\[MASKED]/goapi:aac2285a4e561276f479742ceb9d691edb2acc75 Pulling Image nginx:alpine Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/frontend:aac2285a4e561276f479742ceb9d691edb2acc75 Pulling Image \[MASKED]/\[MASKED]/laravel:aac2285a4e561276f479742ceb9d691edb2acc75 Pulling Image \[MASKED]/\[MASKED]/goapi:aac2285a4e561276f479742ceb9d691edb2acc75 Pulled Image \[MASKED]/\[MASKED]/frontend:aac2285a4e561276f479742ceb9d691edb2acc75 Pulled Image \[MASKED]/\[MASKED]/laravel:aac2285a4e561276f479742ceb9d691edb2acc75 Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started mysqladmin: \[Warning] Using a password on the command line interface can be insecure. mysqld is alive mysql: \[Warning] Using a password on the command line interface can be insecure. Cached events cleared! Compiled views cleared! Application cache cleared! Route cache cleared! Configuration cache cleared! Compiled services and packages files removed! Caches cleared successfully! Nothing to migrate. NAME IMAGE COMMAND SERVICE CREATED STATUS PORTS \[MASKED]-dev-frontend-1 \[MASKED]/\[MASKED]/frontend:aac2285a4e561276f479742ceb9d691edb2acc75 "/docker-entrypoint.…" frontend 10 seconds ago Up 2 seconds 80/tcp \[MASKED]-dev-goapi-1 \[MASKED]/\[MASKED]/goapi:aac2285a4e561276f479742ceb9d691edb2acc75 "/app/goapi" goapi 10 seconds ago Up 3 seconds 8080/tcp \[MASKED]-dev-laravel-1 \[MASKED]/\[MASKED]/laravel:aac2285a4e561276f479742ceb9d691edb2acc75 "docker-php-entrypoi…" laravel 10 seconds ago Up 3 seconds 80/tcp \[MASKED]-dev-mysql-1 mysql:8.0 "docker-entrypoint.s…" mysql 11 seconds ago Up 9 seconds (healthy) 3306/tcp, 33060/tcp \[MASKED]-dev-nginx-dev-1 nginx:alpine "/docker-entrypoint.…" nginx-dev 10 seconds ago Up 2 seconds 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp Cleaning up project directory and file based variables 00:00 Job succeededSelesai berpikir selama 3m 2s

Log `deploy_dev` kamu **udah clean dan sukses** — yang paling penting:

* `docker compose ... pull` jalan, image terbaru kepull.
* Semua container kebentuk & jalan.
* **MySQL healthy** (ada `Waiting` lalu `Healthy`).
* `mysqladmin ping` sukses (`mysqld is alive`).
* `php artisan optimize:clear` sukses.
* `migrate` sukses (`Nothing to migrate.`).
* `docker compose ps` nunjukkin service up + port mapping nginx `80/443` ok.

Yang “error”/catatan di log cuma **warning bawaan MySQL client**:

#### 1) Warning: “Using a password on the command line interface can be insecure.”

Ini muncul karena kamu pakai `-p"$MYSQL_ROOT_PASSWORD"`.

Kalau mau **hilangin warning** (lebih rapi di CI), pakai env `MYSQL_PWD`:

```yaml
# ping
MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent

# sync user/pass
MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "SQL..."
```

Di `.gitlab-ci.yml` tinggal ubah bagian `exec -T mysql ...` jadi:

```bash
MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent
MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "..."
```

#### 2) Minor improvement (opsional tapi bagus): heredoc `.env.dev.compose`

Di file kamu, isi heredoc sempat kamu “indent”. Meskipun sekarang jalan, lebih aman bikin `.env` **tanpa leading spaces** supaya parser env-file/compose/dotenv nggak beda perilaku:

```bash
cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=${DEV_MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
EOF
```

#### 3) Opsional: kurangi downtime deploy

Sekarang kamu selalu `down` → semua container stop dulu baru up lagi. Kalau mau lebih “rolling-ish”:

*   drop `down` dan pakai:

    ```bash
    docker compose ... pull
    docker compose ... up -d --remove-orphans
    ```

Ini biasanya lebih cepat + downtime lebih kecil (tergantung perubahan service).

***

Kalau kamu mau, paste juga **bagian script deploy\_dev terbaru** yang kamu pakai sekarang (yang ada mysql wait/sync itu), nanti aku bikinin patch final `.gitlab-ci.yml` yang:

* warning mysql hilang,
* heredoc aman,
* dan volume name di `reset_dev_db` dibuat dinamis (nggak hardcode `threebody-dev_mysql_data`).

coba lebih lengkap dan detail serta command nya yang lengkap cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" # konsisten (nggak bikin stack baru yg bentrok port) COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" default: tags: \["deploy"] # pastikan hanya runner yg punya tag "deploy" before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY PREP (reuse) # ------------------------- .deploy\_prepare: \&deploy\_prepare | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # IMPORTANT: heredoc harus ter-indent supaya YAML valid cat > "$DEPLOY\_DIR/.env.dev.compose" <\<EOF MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_CONNECTION=mysql DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} EOF cd "$DEPLOY\_DIR" # load vars buat dipakai di script (POSIX safe) set -a . ./.env.dev.compose set +a echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # ------------------------- # DEPLOY DEV (main only) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | set -euo pipefail docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # tunggu mysql siap + cek root login (fail-fast kalau root mismatch) if ! docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' set -e for i in $(seq 1 60); do mysqladmin ping -uroot -p"$MYSQL\_ROOT\_PASSWORD" --silent && exit 0 sleep 2 done exit 1 '; then echo "ERROR: Tidak bisa login root MySQL." echo "Biasanya karena volume mysql sudah pernah dibuat dengan root password berbeda." echo "Solusi:" echo " - Jalankan job manual: reset\_dev\_db (recreate volume), atau" echo " - Kembalikan DEV\_MYSQL\_ROOT\_PASSWORD ke password lama saat volume pertama dibuat." exit 1 fi # auto-sync user/password aplikasi (kalau yang mismatch cuma user pass) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' set -e mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps # ------------------------- # RESET DEV DB (manual) # ------------------------- reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | set -euo pipefail docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$Selesai berpikir selama 2m 44s

Siap — aku bikinin versi yang **lebih rapih, “fail-fast”, tanpa warning `mysqladmin/mysql`**, **heredoc bener (tanpa leading spasi di file `.env`)**, dan **reset volume nggak hardcode**.

Di file kamu sekarang ada 2 “bom waktu” utama:

1. **Heredoc kamu TERISI leading 2 spasi** (karena kamu tulis `MYSQL_ROOT_PASSWORD=...`).\
   Itu bikin `.env.dev.compose` isinya jadi `"␠␠MYSQL_ROOT_PASSWORD=..."` → **nggak valid** buat `--env-file` dan juga **bakal error** kalau di-`source`.
2. Warning `mysqladmin/mysql: Using a password on the command line…` → bisa dihilangin pakai **`MYSQL_PWD`**.

***

### 1) .gitlab-ci.yml versi final (lengkap)

Silakan **replace** `.gitlab-ci.yml` kamu dengan ini:

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  # konsisten (nggak bikin stack baru yg bentrok port)
  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

default:
  tags: ["deploy"] # pastikan hanya runner yg punya tag "deploy"
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

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
# DEPLOY PREP (reuse)
# -------------------------
.deploy_prepare: &deploy_prepare |
  set -euo pipefail
  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

  # fail-fast: pastikan secrets ada
  : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
  : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

  mkdir -p "$DEPLOY_DIR"

  # sync file deploy
  if command -v rsync >/dev/null 2>&1; then
    rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
  else
    rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
    cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
  fi

  # cert (sekali saja)
  mkdir -p "$DEPLOY_DIR/certs/dev"
  if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
    docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
      apk add --no-cache openssl >/dev/null
      openssl req -newkey rsa:2048 -nodes \
        -keyout /out/dev.key \
        -x509 -days 365 \
        -out /out/dev.crt \
        -subj "/CN=dev.local"
    '
  fi

  # env untuk compose + laravel
  # IMPORTANT: JANGAN kasih leading spasi, biar valid utk --env-file & "source"
  cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=${DEV_MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
EOF

  cd "$DEPLOY_DIR"

  # load vars buat dipakai di script
  set -a
  . ./.env.dev.compose
  set +a

  # helper biar gak ngetik panjang2
  dc() { docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; }

  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

# -------------------------
# DEPLOY DEV (main only)
# -------------------------
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      set -euo pipefail

      # pull & start
      dc down --remove-orphans || true
      dc pull
      dc up -d --remove-orphans

      # wait mysql up + root login (fail-fast kalau root mismatch)
      if ! dc exec -T mysql sh -lc '
        set -e
        for i in $(seq 1 60); do
          MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent && exit 0
          sleep 2
        done
        exit 1
      '; then
        echo "ERROR: Tidak bisa login root MySQL."
        echo "Biasanya karena volume mysql sudah pernah dibuat dengan root password berbeda."
        echo "Solusi:"
        echo " - Jalankan job manual: reset_dev_db (recreate volume), atau"
        echo " - Kembalikan DEV_MYSQL_ROOT_PASSWORD ke password lama saat volume pertama dibuat."
        exit 1
      fi

      # auto-sync user/password aplikasi (hilangkan warning dengan MYSQL_PWD)
      dc exec -T mysql sh -lc '
        set -e
        SQL="
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
        MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "$SQL"
      '

      # laravel clear + migrate
      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force
      dc ps

# -------------------------
# RESET DEV DB (manual)
# -------------------------
reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      set -euo pipefail

      # volume name dari compose: <project>_<volume>
      MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"

      dc down -v --remove-orphans || true

      # extra safety: kalau volume masih nyangkut
      docker volume rm -f "$MYSQL_VOL" || true

      dc pull
      dc up -d --remove-orphans

      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force
      dc ps
```

***

### 2) Kenapa ini “lebih bener”

#### A. Heredoc `.env.dev.compose` sekarang aman

* Tidak ada leading spasi → valid untuk:
  * `docker compose --env-file .env.dev.compose`
  * `. ./.env.dev.compose` (source)

Kalau kamu mau cek di server deploy, pakai:

```bash
cat -A /home/gitlab-runner/threebody-deploy/.env.dev.compose | head
```

Kalau ada spasi di depan, bakal kelihatan `^I` / `··` dll.

#### B. Warning MySQL hilang

Kita ganti:

* dari: `mysqladmin ... -p"$MYSQL_ROOT_PASSWORD"`
* jadi: `MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ...`

Sama untuk `mysql -uroot`.

#### C. `reset_dev_db` nggak hardcode volume

Sekarang:

```bash
MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"
docker volume rm -f "$MYSQL_VOL"
```

Jadi kalau nanti project name kamu ganti, job reset tetap aman.

***

### 3) Command manual (lengkap) untuk troubleshooting di server

Anggap kamu ada di:

```bash
cd /home/gitlab-runner/threebody-deploy
export COMPOSE_PROJECT_NAME="threebody-dev"
```

Bikin helper:

```bash
dc() { docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; }
```

#### Cek semua service

```bash
dc ps
dc logs -n 200 mysql
dc logs -n 200 laravel
```

#### Cek root mysql bisa login (tanpa warning)

```bash
dc exec -T mysql sh -lc 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent && echo OK'
```

#### Sync user `threebody` (tanpa warning)

```bash
dc exec -T mysql sh -lc '
SQL="
CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
FLUSH PRIVILEGES;
"
MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "$SQL"
'
```

#### Laravel clear + migrate

```bash
dc exec -T laravel php artisan optimize:clear
dc exec -T laravel php artisan migrate --force
```

#### Reset total DB (hapus volume)

```bash
MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"
dc down -v --remove-orphans || true
docker volume rm -f "$MYSQL_VOL" || true
dc up -d --remove-orphans
dc exec -T laravel php artisan migrate --force
```

***

Kalau kamu mau, paste juga **`docker-compose.dev.yml` bagian service mysql + volumes** (cukup potongan yang relevan). Nanti aku bisa bikin versi reset volume yang _lebih robust lagi_ (misal: auto-detect volume name dari `dc config` bukan asumsi `mysql_data`).

cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" # konsisten (nggak bikin stack baru yg bentrok port) COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" default: tags: \["deploy"] # pastikan hanya runner yg punya tag "deploy" before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY PREP (reuse) # ------------------------- .deploy\_prepare: \&deploy\_prepare | set -euo pipefail TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # IMPORTANT: heredoc harus ter-indent supaya YAML valid cat > "$DEPLOY\_DIR/.env.dev.compose" <\<EOF MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_CONNECTION=mysql DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} EOF cd "$DEPLOY\_DIR" # load vars buat dipakai di script (POSIX safe) set -a . ./.env.dev.compose set +a echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin # ------------------------- # DEPLOY DEV (main only) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | set -euo pipefail docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down --remove-orphans || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d --remove-orphans # tunggu mysql siap + cek root login (fail-fast kalau root mismatch) if ! docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' set -e for i in $(seq 1 60); do mysqladmin ping -uroot -p"$MYSQL\_ROOT\_PASSWORD" --silent && exit 0 sleep 2 done exit 1 '; then echo "ERROR: Tidak bisa login root MySQL." echo "Biasanya karena volume mysql sudah pernah dibuat dengan root password berbeda." echo "Solusi:" echo " - Jalankan job manual: reset\_dev\_db (recreate volume), atau" echo " - Kembalikan DEV\_MYSQL\_ROOT\_PASSWORD ke password lama saat volume pertama dibuat." exit 1 fi # auto-sync user/password aplikasi (kalau yang mismatch cuma user pass) docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql sh -lc ' set -e mysql -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml ps # ------------------------- # RESET DEV DB (manual) # ------------------------- reset\_dev\_db: stage: deploy when: manual rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - \*deploy\_prepare - | set -euo pipefail docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml down -v || true docker volume rm -f threebody-dev\_mysql\_data || true docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml pull docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml up -d docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan optimize:clear docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate --force cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_DATABASE: threebody MYSQL\_USER: threebody MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: test: \["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p\$${MYSQL\_ROOT\_PASSWORD} --silent"] interval: 5s timeout: 3s retries: 20 restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: threebody DB\_USERNAME: threebody DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: threebody DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: threebody networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ ini .gitlab-ci.yml dan docker-compose.dev.yml, bantu buatkan agar saya tinggal kopas sajaSelesai berpikir selama 1m 39s

Berikut versi **siap kopas** untuk **`.gitlab-ci.yml`** dan **`docker-compose.dev.yml`** (fix heredoc tanpa spasi, mysql wait + sync user, hilangin warning password pakai `MYSQL_PWD`, dan reset volume nggak hardcode).

***

### 1) `.gitlab-ci.yml` (kopas full replace)

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  # konsisten (nggak bikin stack baru yg bentrok port)
  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

default:
  tags: ["deploy"] # pastikan hanya runner yg punya tag "deploy"
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

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
# DEPLOY PREP (reuse)
# -------------------------
.deploy_prepare: &deploy_prepare |
  set -euo pipefail

  # fail-fast: secrets wajib ada
  : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
  : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

  TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

  mkdir -p "$DEPLOY_DIR"

  # sync file deploy
  if command -v rsync >/dev/null 2>&1; then
    rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
  else
    rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
    cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
  fi

  # cert (sekali saja)
  mkdir -p "$DEPLOY_DIR/certs/dev"
  if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
    docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
      apk add --no-cache openssl >/dev/null
      openssl req -newkey rsa:2048 -nodes \
        -keyout /out/dev.key \
        -x509 -days 365 \
        -out /out/dev.crt \
        -subj "/CN=dev.local"
    '
  fi

  # export vars yg juga dipakai di script job
  export MYSQL_ROOT_PASSWORD="$DEV_MYSQL_ROOT_PASSWORD"
  export MYSQL_PASSWORD="$DEV_MYSQL_PASSWORD"
  export MYSQL_USER="threebody"
  export MYSQL_DATABASE="threebody"

  # IMPORTANT: isi heredoc JANGAN ada leading spasi
  cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${MYSQL_PASSWORD}
MYSQL_USER=${MYSQL_USER}
MYSQL_DATABASE=${MYSQL_DATABASE}

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=${MYSQL_DATABASE}
DB_USERNAME=${MYSQL_USER}
DB_PASSWORD=${MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
EOF

  cd "$DEPLOY_DIR"

  # helper command docker compose
  dc() { docker compose -p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; }

  echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

# -------------------------
# DEPLOY DEV (main only)
# -------------------------
deploy_dev:
  stage: deploy
  needs: ["build_goapi", "build_laravel", "build_frontend"]
  resource_group: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      set -euo pipefail

      dc down --remove-orphans || true
      dc pull
      dc up -d --remove-orphans

      # tunggu mysql siap + cek root login (fail-fast kalau root mismatch)
      if ! dc exec -T mysql sh -lc '
        set -e
        for i in $(seq 1 60); do
          MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent && exit 0
          sleep 2
        done
        exit 1
      '; then
        echo "ERROR: Tidak bisa login root MySQL."
        echo "Biasanya karena volume mysql sudah pernah dibuat dengan root password berbeda."
        echo "Solusi:"
        echo " - Jalankan job manual: reset_dev_db (recreate volume), atau"
        echo " - Kembalikan DEV_MYSQL_ROOT_PASSWORD ke password lama saat volume pertama dibuat."
        exit 1
      fi

      # auto-sync user/password aplikasi (kalau mismatch cuma user pass)
      dc exec -T mysql sh -lc '
        set -e
        SQL="
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
        MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "$SQL"
      '

      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force
      dc ps

# -------------------------
# RESET DEV DB (manual)
# -------------------------
reset_dev_db:
  stage: deploy
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - *deploy_prepare
    - |
      set -euo pipefail

      MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"

      dc down -v --remove-orphans || true
      docker volume rm -f "$MYSQL_VOL" || true

      dc pull
      dc up -d --remove-orphans

      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force
      dc ps
```

***

### 2) `docker-compose.dev.yml` (kopas full replace)

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      # no warning password: pakai MYSQL_PWD
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 -uroot --silent"]
      interval: 5s
      timeout: 3s
      retries: 20
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

Kalau sudah kamu replace dua file itu, tinggal commit + push.\
Kalau mau aku bantu cek juga, paste isi `.env.dev.compose.example` (yang di repo) supaya validasi job `validate` makin “ketat” dan konsisten sama env deploy.

saya mau tanpa reset dev db secara manual dan berikan langkah yang lengkap sampai push nyaSelesai berpikir selama 2m 55s

Siap. Ini versi **tanpa job `reset_dev_db` manual** — kalau root MySQL **nggak bisa login karena volume lama (root password mismatch)**, pipeline **otomatis wipe volume mysql\_data + recreate**, lalu lanjut deploy (wait mysql → sync user/password → optimize clear → migrate).

Di bawah ini aku kasih:

1. **`.gitlab-ci.yml` final (kopas replace)**
2. **`docker-compose.dev.yml` final (kopas replace)**
3. **Langkah lengkap sampai commit & push (command full)**

***

### 1) `.gitlab-ci.yml` (kopas full replace)

> Catatan penting: bagian heredoc `cat > ... <<EOF` **tidak boleh ada spasi** di isi maupun penutup `EOF`.

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # auto wipe volume kalau root mysql mismatch (dev only)
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

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
# DEPLOY DEV (main only) - auto reset volume jika root mismatch
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

      # secrets wajib ada
      : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      mkdir -p "$DEPLOY_DIR"

      # sync file deploy
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      # cert (sekali saja)
      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      # env compose (NO leading spaces!)
      cat > "$DEPLOY_DIR/.env.dev.compose" <<EOF
MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}
MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=threebody
DB_USERNAME=threebody
DB_PASSWORD=${DEV_MYSQL_PASSWORD}

HARBOR_URL=${HARBOR_URL}
HARBOR_PROJECT=${HARBOR_PROJECT}
CI_COMMIT_SHA=${TAG}
APP_KEY=${DEV_APP_KEY}
EOF

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      COMPOSE_FLAGS=(-p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml)

      wait_mysql_root() {
        docker compose "${COMPOSE_FLAGS[@]}" exec -T mysql sh -lc '
          set -e
          for i in $(seq 1 60); do
            MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent && exit 0
            sleep 2
          done
          exit 1
        '
      }

      docker compose "${COMPOSE_FLAGS[@]}" down --remove-orphans || true
      docker compose "${COMPOSE_FLAGS[@]}" pull
      docker compose "${COMPOSE_FLAGS[@]}" up -d --remove-orphans

      # wait mysql + root login
      if ! wait_mysql_root; then
        echo "WARNING: Root MySQL tidak bisa login (kemungkinan volume lama beda root password)."

        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          echo "AUTO_RESET_DB_ON_ROOT_MISMATCH=1 -> Wipe volume mysql_data & recreate (DEV ONLY)."

          MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"

          docker compose "${COMPOSE_FLAGS[@]}" down -v --remove-orphans || true
          docker volume rm -f "$MYSQL_VOL" || true

          docker compose "${COMPOSE_FLAGS[@]}" up -d --remove-orphans

          # wait lagi
          wait_mysql_root || {
            echo "ERROR: Setelah auto reset, root MySQL masih gagal login."
            exit 1
          }
        else
          echo "ERROR: AUTO_RESET_DB_ON_ROOT_MISMATCH=0 -> stop."
          exit 1
        fi
      fi

      # sync user/password aplikasi (buat kasus user pass mismatch)
      docker compose "${COMPOSE_FLAGS[@]}" exec -T mysql sh -lc '
        set -e
        SQL="
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
        MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "$SQL"
      '

      docker compose "${COMPOSE_FLAGS[@]}" exec -T laravel php artisan optimize:clear
      docker compose "${COMPOSE_FLAGS[@]}" exec -T laravel php artisan migrate --force
      docker compose "${COMPOSE_FLAGS[@]}" ps
```

***

### 2) `docker-compose.dev.yml` (kopas full replace)

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 -uroot --silent"]
      interval: 5s
      timeout: 3s
      retries: 20
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

### 3) Langkah lengkap sampai push (command lengkap)

> Jalankan dari repo kamu (`~/three-body-problem-main`).

```bash
cd ~/three-body-problem-main
git checkout main
git pull origin main
```

#### A) Replace `.gitlab-ci.yml`

```bash
cat > .gitlab-ci.yml <<'YAML'
# (PASTE ISI .gitlab-ci.yml DARI ATAS DI SINI)
YAML
```

#### B) Replace `docker-compose.dev.yml`

```bash
cat > docker-compose.dev.yml <<'YAML'
# (PASTE ISI docker-compose.dev.yml DARI ATAS DI SINI)
YAML
```

#### C) Cek cepat file & YAML (opsional tapi recommended)

```bash
git diff --stat
docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"
```

#### D) Commit & Push

```bash
git add .gitlab-ci.yml docker-compose.dev.yml
git commit -m "CI: auto reset mysql volume on root mismatch + deploy self-heal"
git push origin main
```

***

#### Penting (biar pipeline nggak gagal di `:${VAR:?}`)

Pastikan di GitLab **CI/CD Variables** sudah ada:

* `HARBOR_URL`, `HARBOR_PROJECT`, `HARBOR_USERNAME`, `HARBOR_PASSWORD`
* `DEV_MYSQL_ROOT_PASSWORD`, `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY`

***

Kalau kamu mau, paste juga isi file `.env.dev.compose.example` di repo kamu — nanti aku rapihin biar validate stage benar-benar mirror sama env deploy (biar makin konsisten).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000ed087206891a7055a8a807ef\&ts=490476\&p=fs\&cid=1\&sig=113d90d586bdd009e9162bb8933bfa3328c97b1e9efabf99c87e82c2b6987b7a\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ nano docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml docker-compose.dev.yml git commit -m "CI: auto reset mysql volume on root mismatch + deploy self-heal" git push origin main \[main 64cbbfb] CI: auto reset mysql volume on root mismatch + deploy self-heal 2 files changed, 110 insertions(+), 116 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 7, done. Counting objects: 100% (7/7), done. Delta compression using up to 2 threads Compressing objects: 100% (4/4), done. Writing objects: 100% (4/4), 1.53 KiB | 781.00 KiB/s, done. Total 4 (delta 3), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git aac2285..64cbbfb main -> main cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" # auto wipe volume kalau root mysql mismatch (dev only) AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH: "1" default: tags: \["deploy"] before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (main only) - auto reset volume jika root mismatch # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # secrets wajib ada : "${DEV\_MYSQL\_ROOT\_PASSWORD:?}" "${DEV\_MYSQL\_PASSWORD:?}" "${DEV\_APP\_KEY:?}" : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # env compose (NO leading spaces!) cat > "$DEPLOY\_DIR/.env.dev.compose" <\<EOF MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD} MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD} MYSQL\_USER=threebody MYSQL\_DATABASE=threebody DB\_CONNECTION=mysql DB\_HOST=mysql DB\_PORT=3306 DB\_DATABASE=threebody DB\_USERNAME=threebody DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD} HARBOR\_URL=${HARBOR\_URL} HARBOR\_PROJECT=${HARBOR\_PROJECT} CI\_COMMIT\_SHA=${TAG} APP\_KEY=${DEV\_APP\_KEY} EOF cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin COMPOSE\_FLAGS=(-p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml) wait\_mysql\_root() { docker compose "${COMPOSE\_FLAGS\[@]}" exec -T mysql sh -lc ' set -e for i in $(seq 1 60); do MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -uroot --silent && exit 0 sleep 2 done exit 1 ' } docker compose "${COMPOSE\_FLAGS\[@]}" down --remove-orphans || true docker compose "${COMPOSE\_FLAGS\[@]}" pull docker compose "${COMPOSE\_FLAGS\[@]}" up -d --remove-orphans # wait mysql + root login if ! wait\_mysql\_root; then echo "WARNING: Root MySQL tidak bisa login (kemungkinan volume lama beda root password)." if \[ "${AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH:-1}" = "1" ]; then echo "AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=1 -> Wipe volume mysql\_data & recreate (DEV ONLY)." MYSQL\_VOL="${COMPOSE\_PROJECT\_NAME}\_mysql\_data" docker compose "${COMPOSE\_FLAGS\[@]}" down -v --remove-orphans || true docker volume rm -f "$MYSQL\_VOL" || true docker compose "${COMPOSE\_FLAGS\[@]}" up -d --remove-orphans # wait lagi wait\_mysql\_root || { echo "ERROR: Setelah auto reset, root MySQL masih gagal login." exit 1 } else echo "ERROR: AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=0 -> stop." exit 1 fi fi # sync user/password aplikasi (buat kasus user pass mismatch) docker compose "${COMPOSE\_FLAGS\[@]}" exec -T mysql sh -lc ' set -e SQL=" CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysql -uroot -e "$SQL" ' docker compose "${COMPOSE\_FLAGS\[@]}" exec -T laravel php artisan optimize:clear docker compose "${COMPOSE\_FLAGS\[@]}" exec -T laravel php artisan migrate --force docker compose "${COMPOSE\_FLAGS\[@]}" ps cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_DATABASE: ${MYSQL\_DATABASE} MYSQL\_USER: ${MYSQL\_USER} MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: test: \["CMD-SHELL", "MYSQL\_PWD=\$$MYSQL\_ROOT\_PASSWORD mysqladmin ping -h 127.0.0.1 -uroot --silent"] interval: 5s timeout: 3s retries: 20 restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: ${MYSQL\_DATABASE} DB\_USERNAME: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: ${MYSQL\_DATABASE} networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ nano .env.dev.compose.example cikal@devops-ci:\~/three-body-problem-main$ cat .env.dev.compose.example MYSQL\_ROOT\_PASSWORD=CHANGE\_ME MYSQL\_PASSWORD=CHANGE\_ME HARBOR\_URL=192.168.56.43:8081 HARBOR\_PROJECT=threebody CI\_COMMIT\_SHA=local cikal@devops-ci:\~/three-body-problem-main$ apakah .env.dev.compose.example dipakai? dan bantu perbaiki ada error tidak mau jalan dan berikan secara detail sampai push nyaSelesai berpikir selama 6m 0s

Ya, **`.env.dev.compose.example` dipakai** — tapi **hanya di job `validate`** untuk ngecek `docker compose ... config`. Deploy **nggak pakai** file itu, karena deploy bikin file baru: `"$DEPLOY_DIR/.env.dev.compose"`.

#### Kenapa pipeline kamu “yaml invalid” lagi?

Error `could not find expected ':' while scanning a simple key` biasanya muncul karena ada **baris di kolom 1** yang “keluar” dari block YAML. Di file kamu, ini paling sering terjadi gara-gara **heredoc** (`<<EOF ... EOF`) yang _nggak ke-indent_ sehingga `EOF` atau isi heredoc kebaca sebagai key YAML.

Supaya **nggak kejadian lagi**, aku ubah pembuatan `.env.dev.compose` **tanpa heredoc** (pakai `{ echo ... } > file`). Ini paling aman.

***

### 1) FIX `.gitlab-ci.yml` (kopas full replace)

> Ini versi yang **auto reset mysql volume** kalau root mismatch **tanpa manual job**, dan **tanpa heredoc** biar YAML aman.

```yaml
stages:
  - validate
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # DEV only: kalau root mismatch, auto wipe volume mysql_data
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

# -------------------------
# VALIDATE
# -------------------------
validate:
  stage: validate
  script:
    - export APP_KEY="base64:dummy_dummy_dummy_dummy_dummy_dummy="
    - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
    - |
      set -euo pipefail

      NET="ci-nginx-${CI_PIPELINE_ID}"
      FRONT="ci-frontend-${CI_PIPELINE_ID}"
      LAR="ci-laravel-${CI_PIPELINE_ID}"
      GO="ci-goapi-${CI_PIPELINE_ID}"
      NG="ci-nginx-test-${CI_PIPELINE_ID}"

      docker network create "$NET" >/dev/null

      docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null
      docker run -d --rm --name "$LAR"   --network "$NET" --network-alias laravel  nginx:alpine >/dev/null
      docker run -d --rm --name "$GO"    --network "$NET" --network-alias goapi    nginx:alpine >/dev/null

      docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null

      docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d'
      docker cp "$CI_PROJECT_DIR/devops/nginx/dev.conf"      "$NG":/etc/nginx/conf.d/dev.conf
      docker cp "$CI_PROJECT_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf

      docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \
        openssl req -newkey rsa:2048 -nodes \
          -keyout /etc/nginx/certs/dev.key \
          -x509 -days 365 \
          -out /etc/nginx/certs/dev.crt \
          -subj "/CN=dev.local"'

      docker exec "$NG" nginx -t

      docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true
      docker network rm "$NET" >/dev/null 2>&1 || true

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
# DEPLOY DEV (main only) - auto reset volume jika root mismatch
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

      # sync file deploy
      if command -v rsync >/dev/null 2>&1; then
        rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY_DIR/"
      else
        rm -rf "$DEPLOY_DIR/devops" "$DEPLOY_DIR/docker-compose.dev.yml" || true
        cp -a docker-compose.dev.yml devops "$DEPLOY_DIR/"
      fi

      # cert (sekali saja)
      mkdir -p "$DEPLOY_DIR/certs/dev"
      if [ ! -f "$DEPLOY_DIR/certs/dev/dev.crt" ] || [ ! -f "$DEPLOY_DIR/certs/dev/dev.key" ]; then
        docker run --rm -v "$DEPLOY_DIR/certs/dev:/out" alpine:3.19 sh -lc '
          apk add --no-cache openssl >/dev/null
          openssl req -newkey rsa:2048 -nodes \
            -keyout /out/dev.key \
            -x509 -days 365 \
            -out /out/dev.crt \
            -subj "/CN=dev.local"
        '
      fi

      # tulis env file TANPA heredoc (biar YAML aman)
      umask 077
      {
        echo "MYSQL_ROOT_PASSWORD=${DEV_MYSQL_ROOT_PASSWORD}"
        echo "MYSQL_PASSWORD=${DEV_MYSQL_PASSWORD}"
        echo "MYSQL_USER=threebody"
        echo "MYSQL_DATABASE=threebody"
        echo ""
        echo "DB_CONNECTION=mysql"
        echo "DB_HOST=mysql"
        echo "DB_PORT=3306"
        echo "DB_DATABASE=threebody"
        echo "DB_USERNAME=threebody"
        echo "DB_PASSWORD=${DEV_MYSQL_PASSWORD}"
        echo ""
        echo "HARBOR_URL=${HARBOR_URL}"
        echo "HARBOR_PROJECT=${HARBOR_PROJECT}"
        echo "CI_COMMIT_SHA=${TAG}"
        echo "APP_KEY=${DEV_APP_KEY}"
      } > "$DEPLOY_DIR/.env.dev.compose"

      cd "$DEPLOY_DIR"
      echo "$HARBOR_PASSWORD" | docker login "$HARBOR_URL" -u "$HARBOR_USERNAME" --password-stdin

      COMPOSE_FLAGS=(-p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml)

      wait_mysql_root() {
        docker compose "${COMPOSE_FLAGS[@]}" exec -T mysql sh -lc '
          set -e
          for i in $(seq 1 60); do
            MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent && exit 0
            sleep 2
          done
          exit 1
        '
      }

      docker compose "${COMPOSE_FLAGS[@]}" down --remove-orphans || true
      docker compose "${COMPOSE_FLAGS[@]}" pull
      docker compose "${COMPOSE_FLAGS[@]}" up -d --remove-orphans

      if ! wait_mysql_root; then
        echo "WARNING: Root MySQL tidak bisa login (kemungkinan volume lama beda root password)."

        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          echo "AUTO RESET: wipe mysql_data & recreate (DEV ONLY)"
          MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"

          docker compose "${COMPOSE_FLAGS[@]}" down -v --remove-orphans || true
          docker volume rm -f "$MYSQL_VOL" || true
          docker compose "${COMPOSE_FLAGS[@]}" up -d --remove-orphans

          wait_mysql_root || { echo "ERROR: setelah auto reset root masih gagal login"; exit 1; }
        else
          echo "ERROR: AUTO_RESET_DB_ON_ROOT_MISMATCH=0 -> stop"
          exit 1
        fi
      fi

      # sync user/password aplikasi
      docker compose "${COMPOSE_FLAGS[@]}" exec -T mysql sh -lc '
        set -e
        SQL="
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
        MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "$SQL"
      '

      docker compose "${COMPOSE_FLAGS[@]}" exec -T laravel php artisan optimize:clear
      docker compose "${COMPOSE_FLAGS[@]}" exec -T laravel php artisan migrate --force
      docker compose "${COMPOSE_FLAGS[@]}" ps
```

***

### 2) FIX `.env.dev.compose.example` (dipakai validate)

Kalau tetap mau `validate` jalan rapi, isi example-nya **lengkap** sesuai variable di compose:

```bash
cat > .env.dev.compose.example <<'EOF'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

APP_KEY=base64:dummy_dummy_dummy_dummy_dummy_dummy=

HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody
CI_COMMIT_SHA=local
EOF
```

***

### 3) Langkah lengkap sampai push (command lengkap)

> Kamu sekarang sudah punya commit yang bikin pipeline nggak bisa jalan. Jadi tinggal commit perbaikan di atas.

```bash
cd ~/three-body-problem-main
git checkout main
git pull origin main

# 1) replace .gitlab-ci.yml (paste full dari atas)
nano .gitlab-ci.yml

# 2) update .env.dev.compose.example (pakai yang aku kasih)
nano .env.dev.compose.example

# 3) cek cepat compose render (ini harus sukses)
docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"

# 4) commit + push
git add .gitlab-ci.yml .env.dev.compose.example
git commit -m "CI: fix yaml + safe envfile generation + auto reset mysql volume"
git push origin main
```

***

Kalau setelah ini pipeline masih “yaml invalid”, paste **baris sekitar line yang disebut error** (misal line 130–150) dari `.gitlab-ci.yml` — tapi dengan versi tanpa heredoc ini, 99% kasus itu langsung beres.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000cad87206be06557d89d47b59\&ts=490476\&p=fs\&cid=1\&sig=85eee91003b10b1eb815a73d18e611e1e1c0197d66a5ae55e525ce041ca7ddc6\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000618c720682c5416b2fb73a91\&ts=490476\&p=fs\&cid=1\&sig=112ebdfe735cd5fb3e614e1e1dcb7b4a3a9e4649194f38034d6ff4f7ff5fee68\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000aff47206bdd83008d1e7f059\&ts=490476\&p=fs\&cid=1\&sig=f0a3eb6954d42aac63e63844b1a5107195e9fa9b5ff4ee52941804bf15238ca4\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000002b6c720697826f15b549f3ac\&ts=490476\&p=fs\&cid=1\&sig=c597c8437cb080b7aad7179f32a2030ce771ff7abb8f1231a48a62ca9012369d\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000cbe0720694ee7082379c0be2\&ts=490476\&p=fs\&cid=1\&sig=1236c70343e32c3b2aab5790aa9c2164dcafc68b1cb21326fe979b854560028b\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat > .env.dev.compose.example <<'EOF' MYSQL\_ROOT\_PASSWORD=CHANGE\_ME MYSQL\_PASSWORD=CHANGE\_ME MYSQL\_USER=threebody MYSQL\_DATABASE=threebody APP\_KEY=base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy= HARBOR\_URL=192.168.56.43:8081 HARBOR\_PROJECT=threebody CI\_COMMIT\_SHA=local EOF cikal@devops-ci:\~/three-body-problem-main$ docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null echo "compose config OK" compose config OK cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml .env.dev.compose.example git commit -m "CI: fix yaml + safe envfile generation + auto reset mysql volume" git push origin main \[main ad07ed7] CI: fix yaml + safe envfile generation + auto reset mysql volume 2 files changed, 29 insertions(+), 33 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 7, done. Counting objects: 100% (7/7), done. Delta compression using up to 2 threads Compressing objects: 100% (4/4), done. Writing objects: 100% (4/4), 922 bytes | 922.00 KiB/s, done. Total 4 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 64cbbfb..ad07ed7 main -> main cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - validate - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" # DEV only: kalau root mismatch, auto wipe volume mysql\_data AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH: "1" default: tags: \["deploy"] before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # VALIDATE # ------------------------- validate: stage: validate script: - export APP\_KEY="base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy=" - docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null - | set -euo pipefail NET="ci-nginx-${CI\_PIPELINE\_ID}" FRONT="ci-frontend-${CI\_PIPELINE\_ID}" LAR="ci-laravel-${CI\_PIPELINE\_ID}" GO="ci-goapi-${CI\_PIPELINE\_ID}" NG="ci-nginx-test-${CI\_PIPELINE\_ID}" docker network create "$NET" >/dev/null docker run -d --rm --name "$FRONT" --network "$NET" --network-alias frontend nginx:alpine >/dev/null docker run -d --rm --name "$LAR" --network "$NET" --network-alias laravel nginx:alpine >/dev/null docker run -d --rm --name "$GO" --network "$NET" --network-alias goapi nginx:alpine >/dev/null docker run -d --rm --name "$NG" --network "$NET" nginx:alpine sleep 300 >/dev/null docker exec "$NG" sh -lc 'mkdir -p /etc/nginx/certs /etc/nginx/conf.d' docker cp "$CI\_PROJECT\_DIR/devops/nginx/dev.conf" "$NG":/etc/nginx/conf.d/dev.conf docker cp "$CI\_PROJECT\_DIR/devops/nginx/frontend.conf" "$NG":/etc/nginx/conf.d/default.conf docker exec "$NG" sh -lc 'apk add --no-cache openssl >/dev/null && \ openssl req -newkey rsa:2048 -nodes \ -keyout /etc/nginx/certs/dev.key \ -x509 -days 365 \ -out /etc/nginx/certs/dev.crt \ -subj "/CN=dev.local"' docker exec "$NG" nginx -t docker rm -f "$NG" "$FRONT" "$LAR" "$GO" >/dev/null 2>&1 || true docker network rm "$NET" >/dev/null 2>&1 || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (main only) - auto reset volume jika root mismatch # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail : "${DEV\_MYSQL\_ROOT\_PASSWORD:?}" "${DEV\_MYSQL\_PASSWORD:?}" "${DEV\_APP\_KEY:?}" : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -f "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # tulis env file TANPA heredoc (biar YAML aman) umask 077 { echo "MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD}" echo "MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD}" echo "MYSQL\_USER=threebody" echo "MYSQL\_DATABASE=threebody" echo "" echo "DB\_CONNECTION=mysql" echo "DB\_HOST=mysql" echo "DB\_PORT=3306" echo "DB\_DATABASE=threebody" echo "DB\_USERNAME=threebody" echo "DB\_PASSWORD=${DEV\_MYSQL\_PASSWORD}" echo "" echo "HARBOR\_URL=${HARBOR\_URL}" echo "HARBOR\_PROJECT=${HARBOR\_PROJECT}" echo "CI\_COMMIT\_SHA=${TAG}" echo "APP\_KEY=${DEV\_APP\_KEY}" } > "$DEPLOY\_DIR/.env.dev.compose" cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin COMPOSE\_FLAGS=(-p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml) wait\_mysql\_root() { docker compose "${COMPOSE\_FLAGS\[@]}" exec -T mysql sh -lc ' set -e for i in $(seq 1 60); do MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -uroot --silent && exit 0 sleep 2 done exit 1 ' } docker compose "${COMPOSE\_FLAGS\[@]}" down --remove-orphans || true docker compose "${COMPOSE\_FLAGS\[@]}" pull docker compose "${COMPOSE\_FLAGS\[@]}" up -d --remove-orphans if ! wait\_mysql\_root; then echo "WARNING: Root MySQL tidak bisa login (kemungkinan volume lama beda root password)." if \[ "${AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH:-1}" = "1" ]; then echo "AUTO RESET: wipe mysql\_data & recreate (DEV ONLY)" MYSQL\_VOL="${COMPOSE\_PROJECT\_NAME}\_mysql\_data" docker compose "${COMPOSE\_FLAGS\[@]}" down -v --remove-orphans || true docker volume rm -f "$MYSQL\_VOL" || true docker compose "${COMPOSE\_FLAGS\[@]}" up -d --remove-orphans wait\_mysql\_root || { echo "ERROR: setelah auto reset root masih gagal login"; exit 1; } else echo "ERROR: AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=0 -> stop" exit 1 fi fi # sync user/password aplikasi docker compose "${COMPOSE\_FLAGS\[@]}" exec -T mysql sh -lc ' set -e SQL=" CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysql -uroot -e "$SQL" ' docker compose "${COMPOSE\_FLAGS\[@]}" exec -T laravel php artisan optimize:clear docker compose "${COMPOSE\_FLAGS\[@]}" exec -T laravel php artisan migrate --force docker compose "${COMPOSE\_FLAGS\[@]}" ps cikal@devops-ci:\~/three-body-problem-main$ log job deploy\_dev error dibawah ini Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on devops-ci... Getting source from Git repository 00:02 Gitaly correlation ID: 9adda5cc0219a136-CGK Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out ad07ed78 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:22 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command Login Succeeded Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image nginx:alpine Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/goapi:ad07ed7870177c3016ab8c104c2ca4ec2b3aa3ff Pulling Image \[MASKED]/\[MASKED]/frontend:ad07ed7870177c3016ab8c104c2ca4ec2b3aa3ff Pulling Image \[MASKED]/\[MASKED]/laravel:ad07ed7870177c3016ab8c104c2ca4ec2b3aa3ff Pulling Image \[MASKED]/\[MASKED]/goapi:ad07ed7870177c3016ab8c104c2ca4ec2b3aa3ff Pulled Image \[MASKED]/\[MASKED]/frontend:ad07ed7870177c3016ab8c104c2ca4ec2b3aa3ff Pulled Image \[MASKED]/\[MASKED]/laravel:ad07ed7870177c3016ab8c104c2ca4ec2b3aa3ff Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) Cleaning up project directory and file based variables 00:00 ERROR: Job failed: exit status 1 saya mau tanpa stage validate dan bantu perbaiki error ini secara detail dan lengkap sampai ke push nyaSelesai berpikir selama 7m 32s

Siap — ini versi **tinggal kopas** yang:

* **Tanpa stage `validate`**
* **Auto “self-heal”** kalau root MySQL mismatch (otomatis wipe **volume mysql\_data** lalu recreate)
* **Auto sync** user/password `threebody` biar nggak kena `Access denied for user 'threebody'...`
* Fix penyebab error kamu: `ERROR 1045 root@localhost` yang **tidak sempat masuk ke blok auto-reset** karena check MySQL di dalam container “exit terlalu cepat” (loop-nya kepotong). Di bawah ini loop-nya dibuat aman pakai `if ...; then` sehingga auto-reset kepanggil.

> ⚠️ Catatan: auto-reset ini akan **menghapus data MySQL** kalau root password mismatch (DEV only). Kalau kamu mau _jangan pernah wipe data_, set `AUTO_RESET_DB_ON_ROOT_MISMATCH: "0"`.

***

### 1) Copas `docker-compose.dev.yml`

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 -uroot --silent"]
      interval: 5s
      timeout: 3s
      retries: 20
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

### 2) Copas `.gitlab-ci.yml` (tanpa validate, auto-reset jalan)

```yaml
stages:
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # DEV only: kalau root mismatch, auto wipe volume mysql_data
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

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
# DEPLOY DEV (main only) - self heal + auto reset volume mysql if needed
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

      # CI variables wajib ada
      : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      mkdir -p "$DEPLOY_DIR"

      # sync file deploy ke DEPLOY_DIR
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

      # generate env file (aman, tanpa heredoc biar YAML ga rewel)
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

      COMPOSE_FLAGS=(-p "$COMPOSE_PROJECT_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml)

      # sanity check compose (tidak butuh .env.dev.compose.example)
      docker compose "${COMPOSE_FLAGS[@]}" config >/dev/null

      mysql_root_ok() {
        docker compose "${COMPOSE_FLAGS[@]}" exec -T mysql sh -lc '
          set -u
          for i in $(seq 1 60); do
            if MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -uroot --silent >/dev/null 2>&1; then
              exit 0
            fi
            sleep 2
          done
          exit 1
        '
      }

      deploy_up() {
        docker compose "${COMPOSE_FLAGS[@]}" down --remove-orphans || true
        docker compose "${COMPOSE_FLAGS[@]}" pull
        docker compose "${COMPOSE_FLAGS[@]}" up -d --remove-orphans
      }

      deploy_up

      # cek root login; kalau gagal -> auto wipe volume mysql_data (DEV only)
      if ! mysql_root_ok; then
        echo "WARNING: Root MySQL gagal login. Kemungkinan volume lama dibuat dengan root password berbeda."

        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          echo "AUTO_RESET_DB_ON_ROOT_MISMATCH=1 -> wipe volume mysql_data & recreate (DEV ONLY)."

          MYSQL_VOL="${COMPOSE_PROJECT_NAME}_mysql_data"

          docker compose "${COMPOSE_FLAGS[@]}" down -v --remove-orphans || true
          docker volume rm -f "$MYSQL_VOL" || true

          docker compose "${COMPOSE_FLAGS[@]}" up -d --remove-orphans

          mysql_root_ok || {
            echo "ERROR: Setelah auto reset, root MySQL masih gagal login."
            docker compose "${COMPOSE_FLAGS[@]}" logs --no-color mysql | tail -n 200 || true
            exit 1
          }
        else
          echo "ERROR: AUTO_RESET_DB_ON_ROOT_MISMATCH=0 -> stop."
          exit 1
        fi
      fi

      # sync user/password aplikasi (idempotent)
      docker compose "${COMPOSE_FLAGS[@]}" exec -T mysql sh -lc '
        set -eu
        SQL="
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'' ;
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'' ;
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
        MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot -e "$SQL"
      '

      # optional: restart biar cepat “pick up” koneksi DB yg udah bener
      docker compose "${COMPOSE_FLAGS[@]}" restart laravel goapi || true

      docker compose "${COMPOSE_FLAGS[@]}" exec -T laravel php artisan optimize:clear
      docker compose "${COMPOSE_FLAGS[@]}" exec -T laravel php artisan migrate --force
      docker compose "${COMPOSE_FLAGS[@]}" ps
```

***

### 3) `.env.dev.compose.example` dipakai nggak?

* **Kalau kamu hapus stage validate:** file `.env.dev.compose.example` **nggak dipakai oleh CI**.
* Tapi **berguna untuk local** (mis. `docker compose --env-file .env.dev.compose.example config`) atau dokumentasi variabel.

Kalau mau tetap ada, pakai ini:

```bash
cat > .env.dev.compose.example <<'EOF'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

APP_KEY=base64:dummy_dummy_dummy_dummy_dummy_dummy=

HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody
CI_COMMIT_SHA=local
EOF
```

***

### 4) Langkah lengkap sampai push (copy-paste)

Di repo kamu:

```bash
cd /home/cikal/three-body-problem-main

# 1) update docker-compose.dev.yml
nano docker-compose.dev.yml
# (paste isi docker-compose.dev.yml di atas)

# 2) update .gitlab-ci.yml
nano .gitlab-ci.yml
# (paste isi .gitlab-ci.yml di atas)

# 3) (opsional) update .env.dev.compose.example untuk local check
nano .env.dev.compose.example
# (paste contoh di atas)

# 4) test config lokal (opsional tapi mantap)
docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"

# 5) commit & push
git add .gitlab-ci.yml docker-compose.dev.yml .env.dev.compose.example
git commit -m "CI: deploy self-heal + auto reset mysql volume on root mismatch (no validate)"
git push origin main
```

***

### 5) Checklist variable di GitLab CI/CD (wajib ada)

Pastikan di **Settings → CI/CD → Variables** ada:

* `HARBOR_URL`
* `HARBOR_PROJECT`
* `HARBOR_USERNAME`
* `HARBOR_PASSWORD`
* `DEV_MYSQL_ROOT_PASSWORD`
* `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY`

Kalau kamu mau, kirim juga **log deploy\_dev setelah update ini** (yang bagian sesudah `docker compose up -d`) — kalau masih ada error, biasanya tinggal kelihatan dari `docker compose logs mysql/laravel`.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000053dc72068e167d249c7771af\&ts=490476\&p=fs\&cid=1\&sig=358b229cd0823e3b7804e23623c1f72791cc582198037cf40b4ef82ef7c23b85\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000216c7206adee3294b9b629c8\&ts=490476\&p=fs\&cid=1\&sig=8a488e064df0ca259a75638b1864b4a8151184cb874045c9699dd70c2dfb0538\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000000d8c7206bc990c9d59a07141\&ts=490476\&p=fs\&cid=1\&sig=248ea58355245dbece6ba0073b884a66d847286bab83dcc46422f796c74cf3ff\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000037b47206b5751fa8ca5174af\&ts=490476\&p=fs\&cid=1\&sig=caf73877e7601eb7dafbba94b0879fa3985ee7efb21d05c9a8086ad4d6ceb0a8\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000a274720695d36aafe2b9dad0\&ts=490476\&p=fs\&cid=1\&sig=498c29014cb2b076b3445ffd38836b0112722c831cb2eade816bb837adfc7928\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_DATABASE: ${MYSQL\_DATABASE} MYSQL\_USER: ${MYSQL\_USER} MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: test: \["CMD-SHELL", "MYSQL\_PWD=\$$MYSQL\_ROOT\_PASSWORD mysqladmin ping -h 127.0.0.1 -uroot --silent"] interval: 5s timeout: 3s retries: 20 restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: ${MYSQL\_DATABASE} DB\_USERNAME: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: ${MYSQL\_DATABASE} networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" # DEV only: kalau root mismatch, auto wipe volume mysql\_data AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH: "1" default: tags: \["deploy"] before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (main only) - self heal + auto reset volume mysql if needed # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # CI variables wajib ada : "${DEV\_MYSQL\_ROOT\_PASSWORD:?}" "${DEV\_MYSQL\_PASSWORD:?}" "${DEV\_APP\_KEY:?}" : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy ke DEPLOY\_DIR if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' set -e apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # generate env file (aman, tanpa heredoc biar YAML ga rewel) umask 077 { echo "MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD}" echo "MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD}" echo "MYSQL\_USER=threebody" echo "MYSQL\_DATABASE=threebody" echo "APP\_KEY=${DEV\_APP\_KEY}" echo "HARBOR\_URL=${HARBOR\_URL}" echo "HARBOR\_PROJECT=${HARBOR\_PROJECT}" echo "CI\_COMMIT\_SHA=${TAG}" } > "$DEPLOY\_DIR/.env.dev.compose" cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin COMPOSE\_FLAGS=(-p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml) # sanity check compose (tidak butuh .env.dev.compose.example) docker compose "${COMPOSE\_FLAGS\[@]}" config >/dev/null mysql\_root\_ok() { docker compose "${COMPOSE\_FLAGS\[@]}" exec -T mysql sh -lc ' set -u for i in $(seq 1 60); do if MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -uroot --silent >/dev/null 2>&1; then exit 0 fi sleep 2 done exit 1 ' } deploy\_up() { docker compose "${COMPOSE\_FLAGS\[@]}" down --remove-orphans || true docker compose "${COMPOSE\_FLAGS\[@]}" pull docker compose "${COMPOSE\_FLAGS\[@]}" up -d --remove-orphans } deploy\_up # cek root login; kalau gagal -> auto wipe volume mysql\_data (DEV only) if ! mysql\_root\_ok; then echo "WARNING: Root MySQL gagal login. Kemungkinan volume lama dibuat dengan root password berbeda." if \[ "${AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH:-1}" = "1" ]; then echo "AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=1 -> wipe volume mysql\_data & recreate (DEV ONLY)." MYSQL\_VOL="${COMPOSE\_PROJECT\_NAME}\_mysql\_data" docker compose "${COMPOSE\_FLAGS\[@]}" down -v --remove-orphans || true docker volume rm -f "$MYSQL\_VOL" || true docker compose "${COMPOSE\_FLAGS\[@]}" up -d --remove-orphans mysql\_root\_ok || { echo "ERROR: Setelah auto reset, root MySQL masih gagal login." docker compose "${COMPOSE\_FLAGS\[@]}" logs --no-color mysql | tail -n 200 || true exit 1 } else echo "ERROR: AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=0 -> stop." exit 1 fi fi # sync user/password aplikasi (idempotent) docker compose "${COMPOSE\_FLAGS\[@]}" exec -T mysql sh -lc ' set -eu SQL=" CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\'' ; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\'' ; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysql -uroot -e "$SQL" ' # optional: restart biar cepat “pick up” koneksi DB yg udah bener docker compose "${COMPOSE\_FLAGS\[@]}" restart laravel goapi || true docker compose "${COMPOSE\_FLAGS\[@]}" exec -T laravel php artisan optimize:clear docker compose "${COMPOSE\_FLAGS\[@]}" exec -T laravel php artisan migrate --force docker compose "${COMPOSE\_FLAGS\[@]}" ps cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml docker-compose.dev.yml .env.dev.compose.example git commit -m "CI: deploy self-heal + auto reset mysql volume on root mismatch (no validate)" git push origin main \[main 378e10d] CI: deploy self-heal + auto reset mysql volume on root mismatch (no validate) 1 file changed, 41 insertions(+), 70 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 5, done. Counting objects: 100% (5/5), done. Delta compression using up to 2 threads Compressing objects: 100% (3/3), done. Writing objects: 100% (3/3), 1.04 KiB | 1.04 MiB/s, done. Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git ad07ed7..378e10d main -> main cikal@devops-ci:\~/three-body-problem-main$ error log job deploy\_dev dibawah ini Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:01 Running on devops-ci... Getting source from Git repository 00:03 Gitaly correlation ID: 9addc0e1a6518a9a-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out 378e10db as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 01:06 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command Login Succeeded Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image \[MASKED]/\[MASKED]/goapi:378e10db6548206cf09475dc54ca0bbfb9a00f2e Pulling Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/frontend:378e10db6548206cf09475dc54ca0bbfb9a00f2e Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/laravel:378e10db6548206cf09475dc54ca0bbfb9a00f2e Pulling Image \[MASKED]/\[MASKED]/goapi:378e10db6548206cf09475dc54ca0bbfb9a00f2e Pulled Image \[MASKED]/\[MASKED]/frontend:378e10db6548206cf09475dc54ca0bbfb9a00f2e Pulled Image mysql:8.0 Pulled Image \[MASKED]/\[MASKED]/laravel:378e10db6548206cf09475dc54ca0bbfb9a00f2e Pulled Image nginx:alpine Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) Cleaning up project directory and file based variables 00:00 ERROR: Job failed: exit status 1Selesai berpikir selama 8m 59s

Siap. Dari log kamu, masalah utamanya ini:

* **Healthcheck MySQL kamu pakai TCP ke `127.0.0.1`** (jadi MySQL “Healthy”).
* Tapi **script deploy kamu nge-run `mysql -uroot ...` tanpa `-h 127.0.0.1`**, jadi dia konek via **socket (`localhost`)** → hasilnya **`root@localhost` kena 1045**.\
  Makanya job langsung gagal sebelum sempat “self-heal”.

Solusinya: **paksa semua cek root + eksekusi SQL pakai TCP** (`-h 127.0.0.1 --protocol=tcp`). Sekalian kita **hapus stage validate** dan tetap **auto-reset volume** kalau root beneran mismatch (DEV only).

***

### 1) Copy–paste `docker-compose.dev.yml` (revisi kecil: healthcheck tambah `--protocol=tcp`)

> **Replace seluruh file** dengan ini:

```yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - devnet
    healthcheck:
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent"]
      interval: 5s
      timeout: 3s
      retries: 20
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

### 2) Copy–paste `.gitlab-ci.yml` (tanpa validate + MySQL self-heal + auto wipe volume)

> **Replace seluruh file** dengan ini:

```yml
stages:
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  COMPOSE_DOCKER_CLI_BUILD: "1"

  COMPOSE_PROJECT_NAME: "threebody-dev"
  DEPLOY_DIR: "/home/gitlab-runner/threebody-deploy"

  # DEV only: kalau root mismatch, auto wipe volume mysql_data
  AUTO_RESET_DB_ON_ROOT_MISMATCH: "1"

default:
  tags: ["deploy"]
  before_script:
    - set -euo pipefail
    - docker version
    - docker compose version
    - docker info | sed -n '/Insecure Registries/,+10p' || true

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
# DEPLOY DEV (main only)
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

      # CI variables wajib ada
      : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      mkdir -p "$DEPLOY_DIR"

      # sync file deploy ke DEPLOY_DIR
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

      # generate env file (aman, tanpa heredoc)
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

      # sanity check compose
      dc config >/dev/null

      deploy_up() {
        dc down --remove-orphans || true
        dc pull
        dc up -d --remove-orphans
      }

      mysql_root_tcp_ok() {
        # paksa TCP supaya gak kena root@localhost/socket issue
        dc exec -T mysql sh -lc '
          i=0
          while [ "$i" -lt 60 ]; do
            if MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent >/dev/null 2>&1; then
              exit 0
            fi
            i=$((i+1))
            sleep 2
          done
          exit 1
        '
      }

      deploy_up

      # tunggu mysql siap via TCP
      if ! mysql_root_tcp_ok; then
        echo "WARNING: Root MySQL gagal login via TCP."
        echo "Kemungkinan: volume lama beda root password / mysql belum sehat / config bermasalah."

        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          echo "AUTO_RESET_DB_ON_ROOT_MISMATCH=1 -> wipe volume mysql_data & recreate (DEV ONLY, DATA HILANG)."
          dc down -v --remove-orphans || true
          dc up -d --remove-orphans

          mysql_root_tcp_ok || {
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

      # sync user/password aplikasi (idempotent) - paksa TCP juga
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

      # restart biar koneksi DB kebaca ulang
      dc restart laravel goapi || true

      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force
      dc ps
```

***

### 3) `.env.dev.compose.example` dipakai nggak?

**Di pipeline: tidak dipakai.**\
Pipeline kamu **generate** `$DEPLOY_DIR/.env.dev.compose` dari **GitLab CI Variables**.

Tapi **tetap berguna** untuk:

* test local (`docker compose --env-file .env.dev.compose.example ... config`)
* dokumentasi variable apa saja yang dibutuhkan.

Kalau mau keep, ini versi yang konsisten:

```bash
cat > .env.dev.compose.example <<'EOF'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

APP_KEY=base64:dummy_dummy_dummy_dummy_dummy_dummy=

HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody
CI_COMMIT_SHA=local
EOF
```

***

### 4) Langkah lengkap (edit → test → commit → push)

Jalankan di repo kamu:

```bash
cd ~/three-body-problem-main

# 1) edit/replace file
nano docker-compose.dev.yml
nano .gitlab-ci.yml

# (opsional) update example file
nano .env.dev.compose.example

# 2) cek compose config lokal (opsional tapi disarankan)
docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"

# 3) commit & push
git add .gitlab-ci.yml docker-compose.dev.yml .env.dev.compose.example
git commit -m "CI: fix mysql root TCP check + self-heal + auto reset volume (no validate)"
git push origin main
```

***

### 5) Pastikan GitLab CI Variables ada

Minimal ini harus ada di **Settings → CI/CD → Variables**:

* `HARBOR_URL`
* `HARBOR_PROJECT`
* `HARBOR_USERNAME`
* `HARBOR_PASSWORD`
* `DEV_MYSQL_ROOT_PASSWORD`
* `DEV_MYSQL_PASSWORD`
* `DEV_APP_KEY`
* (opsional) `AUTO_RESET_DB_ON_ROOT_MISMATCH` = `1`

***

Kalau setelah ini Laravel masih 500, ambil error paling “jujur” dengan:

```bash
docker compose -p threebody-dev --env-file /home/gitlab-runner/threebody-deploy/.env.dev.compose -f /home/gitlab-runner/threebody-deploy/docker-compose.dev.yml logs --no-color laravel | tail -n 200
```

dan aku bantu rapihin sampai clean.

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000286472068a4ab4d8fec43b63\&ts=490476\&p=fs\&cid=1\&sig=24cfc756459307d4f26dbf830444799b2d435b86e974618c3fe1e23a9c670cf5\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000070e87206a66bdc0b46c6ebf3\&ts=490476\&p=fs\&cid=1\&sig=d89bf9d9d4a926885045d581a23d6b21ae8eb96c41a782a9a6026e55047d32b6\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000026247206b4919ed57d3889d6\&ts=490476\&p=fs\&cid=1\&sig=cb5c6c32ba3f5e4e8724d01d99868860be018ceab4216935648f9749429f3703\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000e6507206a69ab2e8bc7a3161\&ts=490476\&p=fs\&cid=1\&sig=6223a2d87f25dd72e2185eb204813a5e5f549fb776d506d3ddce28e7230a74c0\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000004bf072068530a5f9d7a6f1f7\&ts=490476\&p=fs\&cid=1\&sig=2fec119287e1b9adb11175b66f4524a555ffaff1ae99aec3b5194cdaaf8ccf2f\&v=0)error log job deploy\_dev dibawah ini Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:01 Running on devops-ci... Getting source from Git repository 00:03 Gitaly correlation ID: 9adddb3e31644ac9-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out 30f1ca4c as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:59 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ docker info | sed -n '/Insecure Registries/,+10p' || true Insecure Registries: \[MASKED] ::1/128 127.0.0.0/8 Live Restore Enabled: false Firewall Backend: iptables $ set -euo pipefail # collapsed multi-line command Login Succeeded Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image \[MASKED]/\[MASKED]/goapi:30f1ca4cf3f377ab19423d5d9ab464aee8f98637 Pulling Image nginx:alpine Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/frontend:30f1ca4cf3f377ab19423d5d9ab464aee8f98637 Pulling Image \[MASKED]/\[MASKED]/laravel:30f1ca4cf3f377ab19423d5d9ab464aee8f98637 Pulling Image \[MASKED]/\[MASKED]/goapi:30f1ca4cf3f377ab19423d5d9ab464aee8f98637 Pulled Image \[MASKED]/\[MASKED]/frontend:30f1ca4cf3f377ab19423d5d9ab464aee8f98637 Pulled Image \[MASKED]/\[MASKED]/laravel:30f1ca4cf3f377ab19423d5d9ab464aee8f98637 Pulled Image mysql:8.0 Pulled Image nginx:alpine Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started ERROR 1045 (28000): Access denied for user 'root'@'127.0.0.1' (using password: YES) Cleaning up project directory and file based variables 00:00 ERROR: Job failed: exit status 1 cikal@devops-ci:\~/three-body-problem-main$ nano docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_DATABASE: ${MYSQL\_DATABASE} MYSQL\_USER: ${MYSQL\_USER} MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: test: \["CMD-SHELL", "MYSQL\_PWD=\$$MYSQL\_ROOT\_PASSWORD mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent"] interval: 5s timeout: 3s retries: 20 restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: ${MYSQL\_DATABASE} DB\_USERNAME: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: ${MYSQL\_DATABASE} networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" # DEV only: kalau root mismatch, auto wipe volume mysql\_data AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH: "1" default: tags: \["deploy"] before\_script: - set -euo pipefail - docker version - docker compose version - docker info | sed -n '/Insecure Registries/,+10p' || true # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (main only) # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # CI variables wajib ada : "${DEV\_MYSQL\_ROOT\_PASSWORD:?}" "${DEV\_MYSQL\_PASSWORD:?}" "${DEV\_APP\_KEY:?}" : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy ke DEPLOY\_DIR if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' set -e apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # generate env file (aman, tanpa heredoc) umask 077 { echo "MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD}" echo "MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD}" echo "MYSQL\_USER=threebody" echo "MYSQL\_DATABASE=threebody" echo "APP\_KEY=${DEV\_APP\_KEY}" echo "HARBOR\_URL=${HARBOR\_URL}" echo "HARBOR\_PROJECT=${HARBOR\_PROJECT}" echo "CI\_COMMIT\_SHA=${TAG}" } > "$DEPLOY\_DIR/.env.dev.compose" cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin dc() { docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; } # sanity check compose dc config >/dev/null deploy\_up() { dc down --remove-orphans || true dc pull dc up -d --remove-orphans } mysql\_root\_tcp\_ok() { # paksa TCP supaya gak kena root@localhost/socket issue dc exec -T mysql sh -lc ' i=0 while \[ "$i" -lt 60 ]; do if MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent >/dev/null 2>&1; then exit 0 fi i=$((i+1)) sleep 2 done exit 1 ' } deploy\_up # tunggu mysql siap via TCP if ! mysql\_root\_tcp\_ok; then echo "WARNING: Root MySQL gagal login via TCP." echo "Kemungkinan: volume lama beda root password / mysql belum sehat / config bermasalah." if \[ "${AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH:-1}" = "1" ]; then echo "AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=1 -> wipe volume mysql\_data & recreate (DEV ONLY, DATA HILANG)." dc down -v --remove-orphans || true dc up -d --remove-orphans mysql\_root\_tcp\_ok || { echo "ERROR: Setelah auto reset, root MySQL masih gagal login." dc logs --no-color mysql | tail -n 200 || true exit 1 } else echo "ERROR: AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=0 -> stop." dc logs --no-color mysql | tail -n 200 || true exit 1 fi fi # sync user/password aplikasi (idempotent) - paksa TCP juga dc exec -T mysql sh -lc ' set -eu SQL=" CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\'' ; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\'' ; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "$SQL" ' # restart biar koneksi DB kebaca ulang dc restart laravel goapi || true dc exec -T laravel php artisan optimize:clear dc exec -T laravel php artisan migrate --force dc ps cikal@devops-ci:\~/three-body-problem-main$ cat > .env.dev.compose.example <<'EOF' MYSQL\_ROOT\_PASSWORD=CHANGE\_ME MYSQL\_PASSWORD=CHANGE\_ME MYSQL\_USER=threebody MYSQL\_DATABASE=threebody APP\_KEY=base64:dummy\_dummy\_dummy\_dummy\_dummy\_dummy= HARBOR\_URL=192.168.56.43:8081 HARBOR\_PROJECT=threebody CI\_COMMIT\_SHA=local EOF cikal@devops-ci:\~/three-body-problem-main$ docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null echo "compose config OK" compose config OK cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml docker-compose.dev.yml .env.dev.compose.example git commit -m "CI: fix mysql root TCP check + self-heal + auto reset volume (no validate)" git push origin main \[main 30f1ca4] CI: fix mysql root TCP check + self-heal + auto reset volume (no validate) 2 files changed, 37 insertions(+), 39 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 7, done. Counting objects: 100% (7/7), done. Delta compression using up to 2 threads Compressing objects: 100% (4/4), done. Writing objects: 100% (4/4), 945 bytes | 315.00 KiB/s, done. Total 4 (delta 3), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 378e10d..30f1ca4 main -> main cikal@devops-ci:\~/three-body-problem-main$ docker compose -p threebody-dev --env-file /home/gitlab-runner/threebody-deploy/.env.dev.compose -f /home/gitlab-runner/threebody-deploy/docker-compose.dev.yml logs --no-color laravel | tail -n 200 laravel-1 | \[Sun Dec 14 12:56:31.118612 2025] \[mpm\_prefork:notice] \[pid 1:tid 1] AH00163: Apache/2.4.65 (Debian) PHP/8.2.29 configured -- resuming normal operations laravel-1 | \[Sun Dec 14 12:56:31.118866 2025] \[core:notice] \[pid 1:tid 1] AH00094: Command line: 'apache2 -D FOREGROUND' laravel-1 | 172.19.0.6 - - \[14/Dec/2025:12:57:39 +0000] "GET /api/products HTTP/1.0" 500 528 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:12:57:42 +0000] "GET /api/products HTTP/1.0" 500 528 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:12:58:06 +0000] "GET /api/products HTTP/1.0" 500 528 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" cikal@devops-ci:\~/three-body-problem-main$ tolong bantu perbaiki dan berikan arahan yang lengkap dan detail sampai ke push nyaSelesai berpikir selama 8m 48s

Berikut versi yang **siap kopas** + **memperbaiki error `ERROR 1045 root@127.0.0.1/localhost`** + **tanpa stage validate** + **tanpa reset DB manual** (auto wipe volume saat root mismatch).

Masalahmu sekarang ada 2:

1. **Job deploy gagal** karena kamu mencoba login root MySQL pakai **TCP ke 127.0.0.1**, sementara di MySQL official image root umumnya “aman” dipakai via **socket (localhost)**. Jadi root bisa “Access denied” walau container healthy.
2. **Laravel 500 / Access denied user `threebody`** karena user/password di volume lama **nggak ikut berubah** saat env berubah. Harus dipaksa `ALTER USER + GRANT` (butuh root login yang bener).

***

### 1) docker-compose.dev.yml (KOPAS PENUH)

> Perubahan penting:
>
> * Healthcheck pakai **socket** (tanpa `-h 127.0.0.1`).
> * Tambah `MYSQL_ROOT_HOST: "%"` supaya root bisa juga dipakai via TCP bila perlu (dev only).

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
      # pakai socket -> lebih stabil untuk root (dev)
      test: ["CMD-SHELL", "MYSQL_PWD=$$MYSQL_ROOT_PASSWORD mysqladmin ping -uroot --silent"]
      interval: 5s
      timeout: 3s
      retries: 30
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

### 2) .gitlab-ci.yml (KOPAS PENUH, tanpa validate)

> Perubahan penting:
>
> * Root check **pakai socket** (`mysql --protocol=socket ...`), bukan TCP.
> * Kalau root mismatch → **auto wipe volume mysql\_data** pakai label compose (jadi gak pusing nama volume).
> * Setelah root OK → **ALTER USER + GRANT** agar `threebody` pasti bener (ini yang bikin Laravel 500 hilang).

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

      # CI variables wajib ada
      : "${DEV_MYSQL_ROOT_PASSWORD:?}" "${DEV_MYSQL_PASSWORD:?}" "${DEV_APP_KEY:?}"
      : "${HARBOR_URL:?}" "${HARBOR_PROJECT:?}" "${HARBOR_USERNAME:?}" "${HARBOR_PASSWORD:?}"

      TAG="${CI_COMMIT_TAG:-$CI_COMMIT_SHA}"

      mkdir -p "$DEPLOY_DIR"

      # sync file deploy ke DEPLOY_DIR
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

      # generate env file (aman)
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

      # sanity check compose
      dc config >/dev/null

      deploy_up() {
        dc down --remove-orphans || true
        dc pull
        dc up -d --remove-orphans
      }

      wait_mysql_root_socket() {
        # cek root via socket (paling aman)
        dc exec -T mysql sh -lc '
          i=0
          while [ "$i" -lt 60 ]; do
            if mysql --protocol=socket -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1" >/dev/null 2>&1; then
              exit 0
            fi
            i=$((i+1))
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

        dc up -d --remove-orphans
      }

      deploy_up

      if ! wait_mysql_root_socket; then
        echo "WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password / init bermasalah)."

        if [ "${AUTO_RESET_DB_ON_ROOT_MISMATCH:-1}" = "1" ]; then
          wipe_mysql_volume

          wait_mysql_root_socket || {
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

      # pastikan user/db app selalu benar (ini yang memperbaiki Laravel 500 Access denied)
      dc exec -T mysql sh -lc '
        set -e
        mysql --protocol=socket -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
          CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
          CREATE USER IF NOT EXISTS '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          ALTER USER '\''${MYSQL_USER}'\''@'\''%'\'' IDENTIFIED BY '\''${MYSQL_PASSWORD}'\'';
          GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '\''${MYSQL_USER}'\''@'\''%'\'' ;
          FLUSH PRIVILEGES;
        "
      '

      # restart agar service cepat pick-up koneksi DB yang sudah benar
      dc restart laravel goapi || true

      dc exec -T laravel php artisan optimize:clear
      dc exec -T laravel php artisan migrate --force

      dc ps
```

***

### 3) Apakah `.env.dev.compose.example` dipakai?

**Di pipeline kamu saat ini: tidak dipakai.**\
Karena deploy job **generate** file `.env.dev.compose` di server (`$DEPLOY_DIR/.env.dev.compose`) dan compose dijalankan pakai `--env-file .env.dev.compose`.

Tapi `.env.dev.compose.example` tetap berguna untuk:

* test lokal: `docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config`
* dokumentasi variabel yang dibutuhkan.

Kalau mau tetap ada, ini contoh yang benar:

```bash
cat > .env.dev.compose.example <<'EOF'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_USER=threebody
MYSQL_DATABASE=threebody

APP_KEY=base64:dummy_dummy_dummy_dummy_dummy_dummy=

HARBOR_URL=192.168.56.43:8081
HARBOR_PROJECT=threebody
CI_COMMIT_SHA=local
EOF
```

***

### 4) Langkah lengkap sampai push

#### A. Set CI/CD Variables di GitLab

Masuk **Project → Settings → CI/CD → Variables**, tambahkan:

**Wajib (dipakai build & deploy)**

* `HARBOR_URL` = `192.168.56.43:8081`
* `HARBOR_PROJECT` = `threebody`
* `HARBOR_USERNAME` = (user harbor)
* `HARBOR_PASSWORD` = (pass harbor)

**Wajib (dipakai deploy untuk env compose)**

* `DEV_MYSQL_ROOT_PASSWORD` = password root dev
* `DEV_MYSQL_PASSWORD` = password user `threebody`
* `DEV_APP_KEY` = Laravel APP\_KEY yang valid (contoh `base64:...`)

> Saran: set **Masked** untuk password, dan kalau repo kamu pakai protected main, set **Protected** juga.

***

#### B. Update file di repo

1. Edit file:

```bash
nano docker-compose.dev.yml
nano .gitlab-ci.yml
```

2. (Opsional) buat example env:

```bash
nano .env.dev.compose.example
```

3. Test config compose lokal (opsional tapi bagus):

```bash
docker compose --env-file .env.dev.compose.example -f docker-compose.dev.yml config >/dev/null
echo "compose config OK"
```

4. Commit & push:

```bash
git add .gitlab-ci.yml docker-compose.dev.yml .env.dev.compose.example
git commit -m "CI: deploy self-heal mysql + auto wipe volume + fix laravel DB auth"
git push origin main
```

***

### 5) Setelah pipeline jalan: cek cepat (di server runner)

Kalau masih ada error, ini 3 command paling berguna:

```bash
cd /home/gitlab-runner/threebody-deploy

docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml ps
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color mysql | tail -n 200
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color laravel | tail -n 200
```

Dan untuk memastikan user `threebody` sudah benar:

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql \
  sh -lc 'mysql --protocol=socket -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT user,host FROM mysql.user; SHOW GRANTS FOR '\''threebody'\''@'\''%'\'';"'
```

***

Kalau kamu sudah kopas file di atas dan push, **harusnya:**

* job `deploy_dev` tidak gagal lagi karena root check pakai socket,
* Laravel 500 karena `threebody` access denied akan hilang karena `ALTER USER + GRANT` dipaksa setiap deploy.

Kalau setelah ini Laravel masih 500, kirimkan output:

* `dc logs --no-color laravel | tail -n 200`
* dan `dc exec -T laravel php -v && php -m | grep -i mysql`

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000a64472308f3c2f796226b912\&ts=490477\&p=fs\&cid=1\&sig=b040740fce18ee80adb2d6fe68ff515942254e63bbe834aa55c1e03b7c0a260c\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000009e4471f49a3459b0c31a00f0\&ts=490477\&p=fs\&cid=1\&sig=68be9fa1c5fefa3374ea3730b917bf7df388509fc86175b91a341c33d5f2b713\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000003efc72078adda0a6ca70b851\&ts=490477\&p=fs\&cid=1\&sig=c0b7f6bf39b2de744d35a9837254010f7647e8d181b534ce680bc9a959250a12\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000001a8c71f885d9579093ea9fec\&ts=490477\&p=fs\&cid=1\&sig=682749e5fc2af637d0c0a4819270ac399ff5aa9c7a4258ebd915e6daad025566\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_0000000089d4720ca5cb852761a5e91c\&ts=490477\&p=fs\&cid=1\&sig=757ca23b210d970149e2fee8af941bbd9a3d3442fa3ea7fca3871087534a52ad\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_ROOT\_HOST: "%" MYSQL\_DATABASE: ${MYSQL\_DATABASE} MYSQL\_USER: ${MYSQL\_USER} MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: # pakai socket -> lebih stabil untuk root (dev) test: \["CMD-SHELL", "MYSQL\_PWD=\$$MYSQL\_ROOT\_PASSWORD mysqladmin ping -uroot --silent"] interval: 5s timeout: 3s retries: 30 restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: ${MYSQL\_DATABASE} DB\_USERNAME: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: ${MYSQL\_DATABASE} networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" # DEV only: kalau root mismatch, auto wipe volume mysql\_data (DATA HILANG) AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH: "1" default: tags: \["deploy"] before\_script: - set -euo pipefail - docker version - docker compose version # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (main only) - self heal + auto reset volume jika root mismatch # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail # CI variables wajib ada : "${DEV\_MYSQL\_ROOT\_PASSWORD:?}" "${DEV\_MYSQL\_PASSWORD:?}" "${DEV\_APP\_KEY:?}" : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync file deploy ke DEPLOY\_DIR if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' set -e apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # generate env file (aman) umask 077 { echo "MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD}" echo "MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD}" echo "MYSQL\_USER=threebody" echo "MYSQL\_DATABASE=threebody" echo "APP\_KEY=${DEV\_APP\_KEY}" echo "HARBOR\_URL=${HARBOR\_URL}" echo "HARBOR\_PROJECT=${HARBOR\_PROJECT}" echo "CI\_COMMIT\_SHA=${TAG}" } > "$DEPLOY\_DIR/.env.dev.compose" cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin dc() { docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; } # sanity check compose dc config >/dev/null deploy\_up() { dc down --remove-orphans || true dc pull dc up -d --remove-orphans } wait\_mysql\_root\_socket() { # cek root via socket (paling aman) dc exec -T mysql sh -lc ' i=0 while \[ "$i" -lt 60 ]; do if mysql --protocol=socket -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e "SELECT 1" >/dev/null 2>&1; then exit 0 fi i=$((i+1)) sleep 2 done exit 1 ' } wipe\_mysql\_volume() { echo "AUTO RESET: wipe mysql\_data volume (DEV ONLY, DATA HILANG)" dc down -v --remove-orphans || true # extra safety: hapus volume mysql\_data berdasarkan label compose for v in $(docker volume ls -q \ --filter label=com.docker.compose.project="$COMPOSE\_PROJECT\_NAME" \ --filter label=com.docker.compose.volume=mysql\_data); do docker volume rm -f "$v" || true done dc up -d --remove-orphans } deploy\_up if ! wait\_mysql\_root\_socket; then echo "WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password / init bermasalah)." if \[ "${AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH:-1}" = "1" ]; then wipe\_mysql\_volume wait\_mysql\_root\_socket || { echo "ERROR: Setelah auto reset, root MySQL masih gagal login." dc logs --no-color mysql | tail -n 200 || true exit 1 } else echo "ERROR: AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=0 -> stop." dc logs --no-color mysql | tail -n 200 || true exit 1 fi fi # pastikan user/db app selalu benar (ini yang memperbaiki Laravel 500 Access denied) dc exec -T mysql sh -lc ' set -e mysql --protocol=socket -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e " CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " ' # restart agar service cepat pick-up koneksi DB yang sudah benar dc restart laravel goapi || true dc exec -T laravel php artisan optimize:clear dc exec -T laravel php artisan migrate --force dc ps cikal@devops-ci:\~/three-body-problem-main$ git add .gitlab-ci.yml docker-compose.dev.yml .env.dev.compose.example git commit -m "CI: deploy self-heal mysql + auto wipe volume + fix laravel DB auth" git push origin main \[main 9c95aa0] CI: deploy self-heal mysql + auto wipe volume + fix laravel DB auth 2 files changed, 36 insertions(+), 24 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 7, done. Counting objects: 100% (7/7), done. Delta compression using up to 2 threads Compressing objects: 100% (4/4), done. Writing objects: 100% (4/4), 1.08 KiB | 1.08 MiB/s, done. Total 4 (delta 3), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 30f1ca4..9c95aa0 main -> main cikal@devops-ci:\~/three-body-problem-main$ cd /home/gitlab-runner/threebody-deploy docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml ps docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color mysql | tail -n 200 docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color laravel | tail -n 200 NAME IMAGE COMMAND SERVICE CREATED STATUS PORTS threebody-dev-frontend-1 192.168.56.43:8081/threebody/frontend:9c95aa06cd8f015d22091d01eac83ecfc749fd47 "/docker-entrypoint.…" frontend 2 minutes ago Up About a minute 80/tcp threebody-dev-goapi-1 192.168.56.43:8081/threebody/goapi:9c95aa06cd8f015d22091d01eac83ecfc749fd47 "/app/goapi" goapi 2 minutes ago Up About a minute 8080/tcp threebody-dev-laravel-1 192.168.56.43:8081/threebody/laravel:9c95aa06cd8f015d22091d01eac83ecfc749fd47 "docker-php-entrypoi…" laravel 2 minutes ago Up About a minute 80/tcp threebody-dev-mysql-1 mysql:8.0 "docker-entrypoint.s…" mysql 2 minutes ago Up 2 minutes (healthy) 3306/tcp, 33060/tcp threebody-dev-nginx-dev-1 nginx:alpine "/docker-entrypoint.…" nginx-dev 2 minutes ago Up About a minute 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp mysql-1 | 2025-12-14 13:17:41+00:00 \[Note] \[Entrypoint]: Entrypoint script for MySQL Server 8.0.44-1.el9 started. mysql-1 | 2025-12-14 13:17:43+00:00 \[Note] \[Entrypoint]: Switching to dedicated user 'mysql' mysql-1 | 2025-12-14 13:17:43+00:00 \[Note] \[Entrypoint]: Entrypoint script for MySQL Server 8.0.44-1.el9 started. mysql-1 | 2025-12-14 13:17:44+00:00 \[Note] \[Entrypoint]: Initializing database files mysql-1 | 2025-12-14T13:17:44.130174Z 0 \[Warning] \[MY-011068] \[Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host\_cache\_size=0 instead. mysql-1 | 2025-12-14T13:17:44.130335Z 0 \[System] \[MY-013169] \[Server] /usr/sbin/mysqld (mysqld 8.0.44) initializing of server in progress as process 78 mysql-1 | 2025-12-14T13:17:44.177955Z 1 \[System] \[MY-013576] \[InnoDB] InnoDB initialization has started. mysql-1 | 2025-12-14T13:17:49.232959Z 1 \[System] \[MY-013577] \[InnoDB] InnoDB initialization has ended. mysql-1 | 2025-12-14T13:18:01.705353Z 6 \[Warning] \[MY-010453] \[Server] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option. mysql-1 | 2025-12-14 13:18:29+00:00 \[Note] \[Entrypoint]: Database files initialized mysql-1 | 2025-12-14 13:18:29+00:00 \[Note] \[Entrypoint]: Starting temporary server mysql-1 | 2025-12-14T13:18:30.031823Z 0 \[Warning] \[MY-011068] \[Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host\_cache\_size=0 instead. mysql-1 | 2025-12-14T13:18:30.037743Z 0 \[System] \[MY-010116] \[Server] /usr/sbin/mysqld (mysqld 8.0.44) starting as process 163 mysql-1 | 2025-12-14T13:18:30.138121Z 1 \[System] \[MY-013576] \[InnoDB] InnoDB initialization has started. mysql-1 | 2025-12-14T13:18:37.515626Z 1 \[System] \[MY-013577] \[InnoDB] InnoDB initialization has ended. mysql-1 | 2025-12-14T13:18:39.628393Z 0 \[Warning] \[MY-010068] \[Server] CA certificate ca.pem is self signed. mysql-1 | 2025-12-14T13:18:39.629717Z 0 \[System] \[MY-013602] \[Server] Channel mysql\_main configured to support TLS. Encrypted connections are now supported for this channel. mysql-1 | 2025-12-14T13:18:39.653632Z 0 \[Warning] \[MY-011810] \[Server] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory. mysql-1 | 2025-12-14T13:18:39.737766Z 0 \[System] \[MY-011323] \[Server] X Plugin ready for connections. Socket: /var/run/mysqld/mysqlx.sock mysql-1 | 2025-12-14T13:18:39.738795Z 0 \[System] \[MY-010931] \[Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.44' socket: '/var/run/mysqld/mysqld.sock' port: 0 MySQL Community Server - GPL. mysql-1 | 2025-12-14 13:18:39+00:00 \[Note] \[Entrypoint]: Temporary server started. mysql-1 | '/var/lib/mysql/mysql.sock' -> '/var/run/mysqld/mysqld.sock' mysql-1 | Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it. mysql-1 | Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it. mysql-1 | Warning: Unable to load '/usr/share/zoneinfo/leapseconds' as time zone. Skipping it. mysql-1 | Warning: Unable to load '/usr/share/zoneinfo/tzdata.zi' as time zone. Skipping it. mysql-1 | Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it. mysql-1 | Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it. mysql-1 | 2025-12-14 13:19:29+00:00 \[Note] \[Entrypoint]: Creating database threebody mysql-1 | 2025-12-14 13:19:29+00:00 \[Note] \[Entrypoint]: Creating user threebody mysql-1 | 2025-12-14 13:19:29+00:00 \[Note] \[Entrypoint]: Giving user threebody access to schema threebody mysql-1 | mysql-1 | 2025-12-14 13:19:30+00:00 \[Note] \[Entrypoint]: Stopping temporary server mysql-1 | 2025-12-14T13:19:30.185470Z 33 \[System] \[MY-013172] \[Server] Received SHUTDOWN from user root. Shutting down mysqld (Version: 8.0.44). mysql-1 | 2025-12-14T13:19:35.272818Z 0 \[System] \[MY-010910] \[Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.44) MySQL Community Server - GPL. mysql-1 | 2025-12-14 13:19:36+00:00 \[Note] \[Entrypoint]: Temporary server stopped mysql-1 | mysql-1 | 2025-12-14 13:19:36+00:00 \[Note] \[Entrypoint]: MySQL init process done. Ready for start up. mysql-1 | mysql-1 | 2025-12-14T13:19:36.937546Z 0 \[Warning] \[MY-011068] \[Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host\_cache\_size=0 instead. mysql-1 | 2025-12-14T13:19:36.953548Z 0 \[System] \[MY-010116] \[Server] /usr/sbin/mysqld (mysqld 8.0.44) starting as process 1 mysql-1 | 2025-12-14T13:19:36.983586Z 1 \[System] \[MY-013576] \[InnoDB] InnoDB initialization has started. mysql-1 | 2025-12-14T13:19:40.811507Z 1 \[System] \[MY-013577] \[InnoDB] InnoDB initialization has ended. mysql-1 | 2025-12-14T13:19:43.102143Z 0 \[Warning] \[MY-010068] \[Server] CA certificate ca.pem is self signed. mysql-1 | 2025-12-14T13:19:43.102732Z 0 \[System] \[MY-013602] \[Server] Channel mysql\_main configured to support TLS. Encrypted connections are now supported for this channel. mysql-1 | 2025-12-14T13:19:43.114417Z 0 \[Warning] \[MY-011810] \[Server] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory. mysql-1 | 2025-12-14T13:19:43.131450Z 0 \[System] \[MY-010931] \[Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.44' socket: '/var/run/mysqld/mysqld.sock' port: 3306 MySQL Community Server - GPL. mysql-1 | 2025-12-14T13:19:43.132813Z 0 \[System] \[MY-011323] \[Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock laravel-1 | \[Sun Dec 14 13:18:48.965980 2025] \[mpm\_prefork:notice] \[pid 1:tid 1] AH00163: Apache/2.4.65 (Debian) PHP/8.2.29 configured -- resuming normal operations laravel-1 | \[Sun Dec 14 13:18:48.966389 2025] \[core:notice] \[pid 1:tid 1] AH00094: Command line: 'apache2 -D FOREGROUND' laravel-1 | 172.19.0.6 - - \[14/Dec/2025:13:20:11 +0000] "GET /api/products HTTP/1.0" 500 528 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" laravel-1 | 172.19.0.6 - - \[14/Dec/2025:13:20:19 +0000] "GET /api/products HTTP/1.0" 500 528 "https://dev.local/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36" cikal@devops-ci:/home/gitlab-runner/threebody-deploy$ docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql \ sh -lc 'mysql --protocol=socket -uroot -p"$MYSQL\_ROOT\_PASSWORD" -e "SELECT user,host FROM mysql.user; SHOW GRANTS FOR '\\''threebody'\\''@'\\''%'\\'';"' mysql: \[Warning] Using a password on the command line interface can be insecure. user host root % threebody % mysql.infoschema localhost mysql.session localhost mysql.sys localhost root localhost Grants for threebody@% GRANT USAGE ON \*.\* TO `threebody`@`%` GRANT ALL PRIVILEGES ON `threebody`.\* TO `threebody`@`%` cikal@devops-ci:/home/gitlab-runner/threebody-deploy$ error log job deploy\_dev dibawah ini Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on devops-ci... Getting source from Git repository 00:03 Gitaly correlation ID: 9addf675015c7a21-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out 9c95aa06 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 05:17 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ set -euo pipefail # collapsed multi-line command Login Succeeded Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/laravel:9c95aa06cd8f015d22091d01eac83ecfc749fd47 Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/goapi:9c95aa06cd8f015d22091d01eac83ecfc749fd47 Pulling Image \[MASKED]/\[MASKED]/frontend:9c95aa06cd8f015d22091d01eac83ecfc749fd47 Pulling Image \[MASKED]/\[MASKED]/goapi:9c95aa06cd8f015d22091d01eac83ecfc749fd47 Pulled Image \[MASKED]/\[MASKED]/frontend:9c95aa06cd8f015d22091d01eac83ecfc749fd47 Pulled Image \[MASKED]/\[MASKED]/laravel:9c95aa06cd8f015d22091d01eac83ecfc749fd47 Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password / init bermasalah). AUTO RESET: wipe mysql\_data volume (DEV ONLY, DATA HILANG) Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Volume \[MASKED]-dev\_mysql\_data Removing Network \[MASKED]-dev\_devnet Removing Volume \[MASKED]-dev\_mysql\_data Removed Network \[MASKED]-dev\_devnet Removed Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Volume \[MASKED]-dev\_mysql\_data Creating Volume \[MASKED]-dev\_mysql\_data Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started mysql: \[Warning] Using a password on the command line interface can be insecure. ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2) Cleaning up project directory and file based variables 00:02 ERROR: Job failed: exit status 1 tolong bantu perbaiki dan berikan langkah nya secara detail dan lengkap sampai ke push nyaSelesai berpikir selama 9m 22s

Masalah utamanya ada di **timing MySQL saat init**.

* Image `mysql:8` itu **start “temporary server” dulu** (sering port **0**, kadang cuma socket), lalu bikin DB/user, lalu **stop**, lalu start mysqld “final” di **3306**.
* Di config kamu, healthcheck pakai **socket**, jadi container bisa jadi **Healthy terlalu cepat** (pas temporary server masih hidup). Setelah itu temporary server stop → **socket hilang sebentar** → step kamu yang jalanin `mysql ... --protocol=socket` kena **ERROR 2002 (mysqld.sock not found)**.

Solusinya: **ubah healthcheck MySQL jadi TCP port 3306** (biar “Healthy” cuma kalau mysqld final udah bener-bener up), lalu di CI deploy **start mysql dulu**, tunggu siap, baru start service lain.

Di bawah ini langkah lengkap sampai push.

***

### 1) Ubah `docker-compose.dev.yml` (MySQL healthcheck = TCP 3306)

Edit bagian `mysql:` jadi seperti ini (yang lain boleh tetap):

```yml
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
```

> Penting: tetap pakai `MYSQL_ROOT_HOST: "%"`, karena nanti root TCP dari 127.0.0.1 akan cocok ke `root@%`.

***

### 2) Ubah `.gitlab-ci.yml` (deploy: start mysql dulu, tunggu, baru naikkan yang lain)

Di job `deploy_dev`, ganti script-nya jadi versi yang lebih “deterministik” ini:

```yml
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

Kunci perubahan:

* **Healthcheck MySQL = TCP** → tidak “healthy” saat temporary server.
* Deploy **mysql dulu** → set DB/user → baru `up` service lain.
* SQL pakai `MYSQL_PWD=... mysql ...` (lebih rapi dari `-p...`).

***

### 3) Commit & push

Dari repo kamu:

```bash
git checkout main
git pull origin main

# edit file:
nano docker-compose.dev.yml
nano .gitlab-ci.yml

git add docker-compose.dev.yml .gitlab-ci.yml
git commit -m "CI: make mysql healthcheck TCP + deploy start mysql first to avoid socket race"
git push origin main
```

***

### 4) Kalau setelah deploy Laravel masih error “products table doesn’t exist”

Itu bukan masalah auth lagi—itu murni **migrasi belum bikin tabel `products`**.

Cek cepat:

```bash
cd /home/gitlab-runner/threebody-deploy
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T laravel php artisan migrate:status
```

Kalau memang tidak ada migration untuk `products`, bikin di project Laravel (di repo source) lalu commit & push:

```bash
# di source project (bukan di deploy dir):
docker compose exec laravel php artisan make:migration create_products_table --create=products
# isi schema-nya, commit, push
```

Kalau kamu mau, paste isi folder `database/migrations` atau minimal list filenya, nanti aku bikinin migration `products` yang cocok dengan field yang dipakai Go API (name, description, price, quantity, category, timestamps).

![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000a03471fa943b745cf6a5c192\&ts=490477\&p=fs\&cid=1\&sig=6dc0f8c3ace46d03124e3782af9f02cece3452f299a0bfec508e8150b4789688\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000900871fda3ca06dcfe48bffd\&ts=490477\&p=fs\&cid=1\&sig=727c5ccb8589c7ee10fa7a27939363ddd3d9a9377aa83dc9eb6d2b7176bc5e97\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_000000007098720c8b1b9a43f23f8ab0\&ts=490477\&p=fs\&cid=1\&sig=ff8cfbbb14d32d676399347034234b53cf593ebd62af29aca411aad03e684781\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000999871f48a3d29637e11c6ee\&ts=490477\&p=fs\&cid=1\&sig=bed1641d8f0c288435d29acfef256fa6614a549ffa43e9d83401585e6bc07890\&v=0)![Gambar yang diunggah](https://chatgpt.com/backend-api/estuary/content?id=file_00000000ce8c71f99394ebbca3eb1d90\&ts=490477\&p=fs\&cid=1\&sig=7f4ec983812d8b68e9a7d07dc4d12f37477505323e6e69172da028b485c8823c\&v=0)cikal@devops-ci:\~/three-body-problem-main$ nano docker-compose.dev.yml cikal@devops-ci:\~/three-body-problem-main$ cat docker-compose.dev.yml services: mysql: image: mysql:8.0 environment: MYSQL\_ROOT\_PASSWORD: ${MYSQL\_ROOT\_PASSWORD} MYSQL\_ROOT\_HOST: "%" MYSQL\_DATABASE: ${MYSQL\_DATABASE} MYSQL\_USER: ${MYSQL\_USER} MYSQL\_PASSWORD: ${MYSQL\_PASSWORD} volumes: - mysql\_data:/var/lib/mysql networks: - devnet healthcheck: # TCP: hanya sehat kalau mysqld FINAL sudah listen di 3306 (temporary server port 0 tidak akan lolos) test: \["CMD-SHELL", "MYSQL\_PWD=\$$MYSQL\_ROOT\_PASSWORD mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent"] interval: 5s timeout: 5s retries: 40 start\_period: 20s restart: unless-stopped laravel: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/laravel:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: APP\_ENV: local APP\_DEBUG: "true" APP\_KEY: ${APP\_KEY} APP\_URL: https://dev.local/api/laravel LOG\_CHANNEL: stderr DB\_CONNECTION: mysql DB\_HOST: mysql DB\_PORT: 3306 DB\_DATABASE: ${MYSQL\_DATABASE} DB\_USERNAME: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} networks: - devnet restart: unless-stopped goapi: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/goapi:${CI\_COMMIT\_SHA} depends\_on: mysql: condition: service\_healthy environment: DB\_HOST: mysql DB\_PORT: 3306 DB\_USER: ${MYSQL\_USER} DB\_PASSWORD: ${MYSQL\_PASSWORD} DB\_NAME: ${MYSQL\_DATABASE} networks: - devnet restart: unless-stopped frontend: image: ${HARBOR\_URL}/${HARBOR\_PROJECT}/frontend:${CI\_COMMIT\_SHA} depends\_on: - laravel - goapi networks: - devnet restart: unless-stopped nginx-dev: image: nginx:alpine depends\_on: - frontend - laravel - goapi volumes: - ./devops/nginx/dev.conf:/etc/nginx/conf.d/dev.conf:ro - ./certs/dev:/etc/nginx/certs:ro ports: - "80:80" - "443:443" networks: - devnet restart: unless-stopped volumes: mysql\_data: networks: devnet: cikal@devops-ci:\~/three-body-problem-main$ nano .gitlab-ci.yml cikal@devops-ci:\~/three-body-problem-main$ cat .gitlab-ci.yml stages: - build - deploy variables: DOCKER\_BUILDKIT: "1" COMPOSE\_DOCKER\_CLI\_BUILD: "1" COMPOSE\_PROJECT\_NAME: "threebody-dev" DEPLOY\_DIR: "/home/gitlab-runner/threebody-deploy" # DEV only: kalau root mismatch, auto wipe volume mysql\_data (DATA HILANG) AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH: "1" default: tags: \["deploy"] before\_script: - set -euo pipefail - docker version - docker compose version # ------------------------- # BUILD + PUSH (Harbor) # ------------------------- .harbor\_login: \&harbor\_login | set -euo pipefail : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" build\_goapi: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" -f devops/docker/Dockerfile.goapi . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/goapi:$TAG" build\_laravel: stage: build retry: 2 script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" -f devops/docker/Dockerfile.laravel . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/laravel:$TAG" build\_frontend: stage: build script: - \*harbor\_login - docker build -t "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" -f devops/docker/Dockerfile.frontend . - docker push "$HARBOR\_URL/$HARBOR\_PROJECT/frontend:$TAG" # ------------------------- # DEPLOY DEV (main only) - self heal + auto reset volume jika root mismatch # ------------------------- deploy\_dev: stage: deploy needs: \["build\_goapi", "build\_laravel", "build\_frontend"] resource\_group: dev rules: - if: '$CI\_COMMIT\_BRANCH == "main"' script: - | set -euo pipefail : "${DEV\_MYSQL\_ROOT\_PASSWORD:?}" "${DEV\_MYSQL\_PASSWORD:?}" "${DEV\_APP\_KEY:?}" : "${HARBOR\_URL:?}" "${HARBOR\_PROJECT:?}" "${HARBOR\_USERNAME:?}" "${HARBOR\_PASSWORD:?}" TAG="${CI\_COMMIT\_TAG:-$CI\_COMMIT\_SHA}" mkdir -p "$DEPLOY\_DIR" # sync deploy files if command -v rsync >/dev/null 2>&1; then rsync -a --delete docker-compose.dev.yml devops/ "$DEPLOY\_DIR/" else rm -rf "$DEPLOY\_DIR/devops" "$DEPLOY\_DIR/docker-compose.dev.yml" || true cp -a docker-compose.dev.yml devops "$DEPLOY\_DIR/" fi # cert (sekali saja) mkdir -p "$DEPLOY\_DIR/certs/dev" if \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.crt" ] || \[ ! -s "$DEPLOY\_DIR/certs/dev/dev.key" ]; then docker run --rm -v "$DEPLOY\_DIR/certs/dev:/out" alpine:3.19 sh -lc ' set -e apk add --no-cache openssl >/dev/null openssl req -newkey rsa:2048 -nodes \ -keyout /out/dev.key \ -x509 -days 365 \ -out /out/dev.crt \ -subj "/CN=dev.local" ' fi # env compose umask 077 { echo "MYSQL\_ROOT\_PASSWORD=${DEV\_MYSQL\_ROOT\_PASSWORD}" echo "MYSQL\_PASSWORD=${DEV\_MYSQL\_PASSWORD}" echo "MYSQL\_USER=threebody" echo "MYSQL\_DATABASE=threebody" echo "APP\_KEY=${DEV\_APP\_KEY}" echo "HARBOR\_URL=${HARBOR\_URL}" echo "HARBOR\_PROJECT=${HARBOR\_PROJECT}" echo "CI\_COMMIT\_SHA=${TAG}" } > "$DEPLOY\_DIR/.env.dev.compose" cd "$DEPLOY\_DIR" echo "$HARBOR\_PASSWORD" | docker login "$HARBOR\_URL" -u "$HARBOR\_USERNAME" --password-stdin dc() { docker compose -p "$COMPOSE\_PROJECT\_NAME" --env-file .env.dev.compose -f docker-compose.dev.yml "$@"; } dc config >/dev/null wait\_mysql\_root\_tcp() { dc exec -T mysql sh -lc ' for i in $(seq 1 90); do if MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysqladmin ping -h 127.0.0.1 --protocol=tcp -uroot --silent >/dev/null 2>&1; then exit 0 fi sleep 2 done exit 1 ' } wipe\_mysql\_volume() { echo "AUTO RESET: wipe mysql\_data volume (DEV ONLY, DATA HILANG)" dc down -v --remove-orphans || true # extra safety: hapus volume mysql\_data berdasarkan label compose for v in $(docker volume ls -q \ --filter label=com.docker.compose.project="$COMPOSE\_PROJECT\_NAME" \ --filter label=com.docker.compose.volume=mysql\_data); do docker volume rm -f "$v" || true done } ensure\_db\_user() { dc exec -T mysql sh -lc ' set -eu SQL=" CREATE DATABASE IF NOT EXISTS \\`${MYSQL_DATABASE}\`; CREATE USER IF NOT EXISTS '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; ALTER USER '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' IDENTIFIED BY '\\''${MYSQL\_PASSWORD}'\\''; GRANT ALL PRIVILEGES ON \\`${MYSQL_DATABASE}\`.\* TO '\\''${MYSQL\_USER}'\\''@'\\''%'\\'' ; FLUSH PRIVILEGES; " MYSQL\_PWD="$MYSQL\_ROOT\_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "$SQL" ' } # bersih + pull dc down --remove-orphans || true dc pull # 1) naikkan mysql dulu saja dc up -d --remove-orphans mysql # 2) tunggu mysql FINAL siap (TCP 3306 + root auth) if ! wait\_mysql\_root\_tcp; then echo "WARNING: Root MySQL gagal login (kemungkinan volume lama beda root password)." if \[ "${AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH:-1}" = "1" ]; then wipe\_mysql\_volume dc up -d --remove-orphans mysql wait\_mysql\_root\_tcp || { echo "ERROR: Setelah auto reset, root MySQL masih gagal login." dc logs --no-color mysql | tail -n 200 || true exit 1 } else echo "ERROR: AUTO\_RESET\_DB\_ON\_ROOT\_MISMATCH=0 -> stop." dc logs --no-color mysql | tail -n 200 || true exit 1 fi fi # 3) pastikan db/user app selalu benar (ini yang bikin Laravel tidak Access denied) ensure\_db\_user # 4) baru naikkan service lainnya dc up -d --remove-orphans # 5) restart service app biar pasti pick up koneksi DB dc restart laravel goapi || true # 6) migrate dc exec -T laravel php artisan optimize:clear dc exec -T laravel php artisan migrate --force dc ps cikal@devops-ci:\~/three-body-problem-main$ git add docker-compose.dev.yml .gitlab-ci.yml git commit -m "CI: make mysql healthcheck TCP + deploy start mysql first to avoid socket race" git push origin main \[main e7a218b] CI: make mysql healthcheck TCP + deploy start mysql first to avoid socket race 2 files changed, 41 insertions(+), 41 deletions(-) Username for 'https://gitlab.com': cikalfarid Password for 'https://cikalfarid@gitlab.com': Enumerating objects: 7, done. Counting objects: 100% (7/7), done. Delta compression using up to 2 threads Compressing objects: 100% (4/4), done. Writing objects: 100% (4/4), 1.11 KiB | 125.00 KiB/s, done. Total 4 (delta 3), reused 0 (delta 0), pack-reused 0 To https://gitlab.com/cikalfarid/three-body-problem.git 9c95aa0..e7a218b main -> main cikal@devops-ci:\~/three-body-problem-main$ berikut log job sukses deploy\_dev dibawah ini Running with gitlab-runner 18.6.5 (5908bc19) on devops-ci ArpoA9OGu, system ID: s\_7bd6fa72a2b0 Preparing the "shell" executor 00:00 Using Shell (bash) executor... Preparing environment 00:00 Running on devops-ci... Getting source from Git repository 00:02 Gitaly correlation ID: 9ade2e22067fea87-ATL Fetching changes with git depth set to 20... Reinitialized existing Git repository in /home/gitlab-runner/builds/ArpoA9OGu/0/cikalfarid/three-body-problem/.git/ Checking out e7a218b9 as detached HEAD (ref is main)... Skipping Git submodules setup Executing "step\_script" stage of the job script 00:33 $ set -euo pipefail $ docker version Client: Docker Engine - Community Version: 29.1.2 API version: 1.52 Go version: go1.25.5 Git commit: 890dcca Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Context: default Server: Docker Engine - Community Engine: Version: 29.1.2 API version: 1.52 (minimum version 1.44) Go version: go1.25.5 Git commit: de45c2a Built: Tue Dec 2 21:55:19 2025 OS/Arch: linux/amd64 Experimental: false containerd: Version: v2.2.0 GitCommit: 1c4457e00facac03ce1d75f7b6777a7a851e5c41 runc: Version: 1.3.4 GitCommit: v1.3.4-0-gd6d73eb8 docker-init: Version: 0.19.0 GitCommit: de40ad0 $ docker compose version Docker Compose version v5.0.0 $ set -euo pipefail # collapsed multi-line command Login Succeeded Container \[MASKED]-dev-nginx-dev-1 Stopping Container \[MASKED]-dev-nginx-dev-1 Stopped Container \[MASKED]-dev-nginx-dev-1 Removing Container \[MASKED]-dev-nginx-dev-1 Removed Container \[MASKED]-dev-frontend-1 Stopping Container \[MASKED]-dev-frontend-1 Stopped Container \[MASKED]-dev-frontend-1 Removing Container \[MASKED]-dev-frontend-1 Removed Container \[MASKED]-dev-goapi-1 Stopping Container \[MASKED]-dev-laravel-1 Stopping Container \[MASKED]-dev-goapi-1 Stopped Container \[MASKED]-dev-goapi-1 Removing Container \[MASKED]-dev-goapi-1 Removed Container \[MASKED]-dev-laravel-1 Stopped Container \[MASKED]-dev-laravel-1 Removing Container \[MASKED]-dev-laravel-1 Removed Container \[MASKED]-dev-mysql-1 Stopping Container \[MASKED]-dev-mysql-1 Stopped Container \[MASKED]-dev-mysql-1 Removing Container \[MASKED]-dev-mysql-1 Removed Network \[MASKED]-dev\_devnet Removing Network \[MASKED]-dev\_devnet Removed Image \[MASKED]/\[MASKED]/laravel:e7a218b98ba903923a939de0444e0f2c8e0a1818 Pulling Image nginx:alpine Pulling Image \[MASKED]/\[MASKED]/goapi:e7a218b98ba903923a939de0444e0f2c8e0a1818 Pulling Image \[MASKED]/\[MASKED]/frontend:e7a218b98ba903923a939de0444e0f2c8e0a1818 Pulling Image mysql:8.0 Pulling Image \[MASKED]/\[MASKED]/goapi:e7a218b98ba903923a939de0444e0f2c8e0a1818 Pulled Image \[MASKED]/\[MASKED]/frontend:e7a218b98ba903923a939de0444e0f2c8e0a1818 Pulled Image \[MASKED]/\[MASKED]/laravel:e7a218b98ba903923a939de0444e0f2c8e0a1818 Pulled Image nginx:alpine Pulled Image mysql:8.0 Pulled Network \[MASKED]-dev\_devnet Creating Network \[MASKED]-dev\_devnet Created Container \[MASKED]-dev-mysql-1 Creating Container \[MASKED]-dev-mysql-1 Created Container \[MASKED]-dev-mysql-1 Starting Container \[MASKED]-dev-mysql-1 Started Container \[MASKED]-dev-mysql-1 Running Container \[MASKED]-dev-laravel-1 Creating Container \[MASKED]-dev-goapi-1 Creating Container \[MASKED]-dev-goapi-1 Created Container \[MASKED]-dev-laravel-1 Created Container \[MASKED]-dev-frontend-1 Creating Container \[MASKED]-dev-frontend-1 Created Container \[MASKED]-dev-nginx-dev-1 Creating Container \[MASKED]-dev-nginx-dev-1 Created Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Waiting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-goapi-1 Starting Container \[MASKED]-dev-mysql-1 Healthy Container \[MASKED]-dev-laravel-1 Starting Container \[MASKED]-dev-laravel-1 Started Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-frontend-1 Starting Container \[MASKED]-dev-frontend-1 Started Container \[MASKED]-dev-nginx-dev-1 Starting Container \[MASKED]-dev-nginx-dev-1 Started Container \[MASKED]-dev-goapi-1 Restarting Container \[MASKED]-dev-laravel-1 Restarting Container \[MASKED]-dev-goapi-1 Started Container \[MASKED]-dev-laravel-1 Started Cached events cleared! Compiled views cleared! Application cache cleared! Route cache cleared! Configuration cache cleared! Compiled services and packages files removed! Caches cleared successfully! Migration table created successfully. Migrating: 2014\_10\_12\_000000\_create\_users\_table Migrated: 2014\_10\_12\_000000\_create\_users\_table (147.20ms) Migrating: 2014\_10\_12\_100000\_create\_password\_resets\_table Migrated: 2014\_10\_12\_100000\_create\_password\_resets\_table (142.92ms) Migrating: 2019\_08\_19\_000000\_create\_failed\_jobs\_table Migrated: 2019\_08\_19\_000000\_create\_failed\_jobs\_table (154.23ms) Migrating: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table Migrated: 2019\_12\_14\_000001\_create\_personal\_access\_tokens\_table (410.00ms) Migrating: 2025\_07\_25\_075615\_create\_products\_table Migrated: 2025\_07\_25\_075615\_create\_products\_table (15.98ms) NAME IMAGE COMMAND SERVICE CREATED STATUS PORTS \[MASKED]-dev-frontend-1 \[MASKED]/\[MASKED]/frontend:e7a218b98ba903923a939de0444e0f2c8e0a1818 "/docker-entrypoint.…" frontend 8 seconds ago Up 6 seconds 80/tcp \[MASKED]-dev-goapi-1 \[MASKED]/\[MASKED]/goapi:e7a218b98ba903923a939de0444e0f2c8e0a1818 "/app/goapi" goapi 9 seconds ago Up 4 seconds 8080/tcp \[MASKED]-dev-laravel-1 \[MASKED]/\[MASKED]/laravel:e7a218b98ba903923a939de0444e0f2c8e0a1818 "docker-php-entrypoi…" laravel 9 seconds ago Up 3 seconds 80/tcp \[MASKED]-dev-mysql-1 mysql:8.0 "docker-entrypoint.s…" mysql 20 seconds ago Up 18 seconds (healthy) 3306/tcp, 33060/tcp \[MASKED]-dev-nginx-dev-1 nginx:alpine "/docker-entrypoint.…" nginx-dev 8 seconds ago Up 5 seconds 0.0.0.0:80->80/tcp, \[::]:80->80/tcp, 0.0.0.0:443->443/tcp, \[::]:443->443/tcp Cleaning up project directory and file based variables 00:00 Job succeeded mantap kerenSelesai berpikir selama 33s

Mantap! 🎉 Yang bikin akhirnya **deploy\_dev tembus** itu bukan “sekadar ganti password”, tapi kamu berhasil ngerapihin **urutan start** + **cara nunggu MySQL siap** supaya gak kena _race condition_.

Berikut rangkuman **kenapa tadi gagal** dan **kenapa fix terakhir sukses** (biar besok kalau kejadian lagi kamu langsung tau titiknya).

### Akar masalahnya

#### 1) Root MySQL “mismatch” gara-gara volume lama

Kalau `mysql_data` sudah pernah terisi, environment `MYSQL_ROOT_PASSWORD` **tidak akan mengubah** root password lagi (MySQL entrypoint hanya inisialisasi saat data dir kosong).\
Efeknya: pipeline login root gagal → `ERROR 1045`.

✅ Solusi DEV yang kamu pakai sudah benar:

* Deteksi root gagal login
* Kalau `AUTO_RESET_DB_ON_ROOT_MISMATCH=1` → **wipe volume** dan init ulang

***

#### 2) “Socket race” setelah wipe (ini yang bikin `ERROR 2002`)

Setelah volume dihapus, MySQL startup itu ada fase:

* **temporary server** (kadang port bukan 3306 / socket belum siap)
* lalu stop → start lagi jadi **final server**

Waktu kamu pakai cek via socket / service lain keburu jalan, CI kadang eksekusi `mysql ...` pas socket belum ada:

> `Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)`

✅ Fix paling tepat yang kamu lakukan:

* **Healthcheck pakai TCP** ke `127.0.0.1:3306` (bukan socket)\
  Jadi container baru dianggap healthy kalau **mysqld final sudah listen 3306**
* Di deploy, **naikkan mysql dulu** → tunggu siap → baru service lain

***

#### 3) Laravel 500 awalnya karena DB belum siap / user / table belum ada

Setelah MySQL bener, Laravel masih bisa 500 kalau:

* user/password belum match, atau
* migration belum jalan (mis. `products` table belum ada)

✅ Makanya step ini penting:

* `ensure_db_user` (idempotent)
* `php artisan migrate --force`

***

### Alur deploy yang sekarang sudah “benar & stabil” (versi singkat)

Urutan final yang bikin job kamu sukses:

1. `dc down` + `dc pull`
2. `dc up -d mysql`
3. `wait_mysql_root_tcp` (retry sampai mysqld final siap)
4. kalau gagal → `wipe_mysql_volume` → `dc up -d mysql` → tunggu lagi
5. `ensure_db_user` (create db/user/grants)
6. `dc up -d` (service lain)
7. `dc restart laravel goapi`
8. `php artisan optimize:clear` + `php artisan migrate --force`

Itu persis yang kebaca dari log sukses kamu: mysql healthy → restart app → migrate jalan → selesai ✅

***

### Checklist cepat kalau suatu saat kejadian lagi

Kalau nanti Laravel tiba-tiba 500 lagi, ini 3 perintah paling cepat:

```bash
cd /home/gitlab-runner/threebody-deploy
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml ps
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color mysql | tail -n 200
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml logs --no-color laravel | tail -n 200
```

Cek DB langsung (validasi user/grant):

```bash
docker compose -p threebody-dev --env-file .env.dev.compose -f docker-compose.dev.yml exec -T mysql \
  sh -lc 'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -h 127.0.0.1 --protocol=tcp -uroot -e "SHOW DATABASES; SELECT user,host FROM mysql.user;"'
```

***

Kalau kamu mau, aku bisa bantu “merapikan” versi finalnya lagi supaya lebih _clean_ (mis. bikin fungsi deploy jadi lebih modular + output log lebih enak dibaca), tapi yang sekarang **sudah solid** dan terbukti lolos pipeline.
