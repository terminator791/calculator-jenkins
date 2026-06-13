# Panduan Praktik BAB 4 - Continuous Integration Pipeline

Panduan langkah-demi-langkah untuk mempraktikkan seluruh isi BAB 4 dari awal sampai akhir.
Setup yang dipakai: **Jenkins controller via Docker Compose + Docker agent per-stage**.


> Semua perintah ditulis siap copy-paste. Jalankan dari direktori proyek `/home/iqbal/Project/java/tubes-jenkins` kecuali disebutkan lain.

## Daftar Isi
1. [Persiapan Prasyarat](#1-persiapan-prasyarat)
2. [Menjalankan Jenkins via Docker Compose](#2-menjalankan-jenkins-via-docker-compose)
3. [Konfigurasi Awal Jenkins & Plugin](#3-konfigurasi-awal-jenkins--plugin)
4. [Memahami Docker Agent Per-Stage](#4-memahami-docker-agent-per-stage)

5. [Tahap 0 - Hello World Pipeline](#5-tahap-0---hello-world-pipeline)
6. [Tahap 1 - Commit Pipeline Dasar (sample1)](#6-tahap-1---commit-pipeline-dasar-sample1)
7. [Tahap 2 - Jenkinsfile from SCM](#7-tahap-2---jenkinsfile-from-scm)
8. [Tahap 3 - Code Coverage (sample2)](#8-tahap-3---code-coverage-sample2)
9. [Tahap 4 - Static Code Analysis (sample3)](#9-tahap-4---static-code-analysis-sample3)
10. [Tahap 5 - Triggers & Notifications](#10-tahap-5---triggers--notifications)
11. [Tahap 6 - Multi-branch & Team Strategies](#11-tahap-6---multi-branch--team-strategies)
12. [Tahap 7 - Exercises Python](#12-tahap-7---exercises-python)
13. [Troubleshooting](#13-troubleshooting)
14. [Checklist Lengkap](#14-checklist-lengkap)

---

## 1. Persiapan Prasyarat

Cek tool yang sudah ada di mesin:

```bash
docker --version       # wajib (Jenkins + agent jalan di Docker)
git --version          # wajib
python --version       # untuk exercise Python
java -version          # opsional di host; build Java jalan di dalam container gradle
```

Yang perlu disiapkan:

1. **Docker** sudah aktif (`docker ps` tidak error). Pastikan user-mu bisa menjalankan `docker` tanpa sudo, atau gunakan `sudo`.
2. **Akun GitHub** + buat repository kosong bernama `calculator` (centang "Initialize with README"). Catat URL HTTPS-nya, mis. `https://github.com/<username>/calculator.git`.
3. **Git identity** (kalau belum):
   ```bash
   git config --global user.name "Nama Kamu"
   git config --global user.email "email@kamu.com"
   ```

> Catatan: Dengan pendekatan Docker agent per-stage, **kamu tidak wajib memasang JDK/Gradle di host** karena kompilasi & test Java berjalan di image `gradle:7.6-jdk11`. Java di host hanya berguna untuk uji coba lokal manual.

<!-- SEC_1 -->

---

## 2. Menjalankan Jenkins via Docker Compose

Kita menjalankan **Jenkins controller via Docker Compose** supaya simpel (tanpa perintah `docker run` panjang). Build per-stage tetap memakai **Docker agent** di Jenkinsfile. Agar Jenkins bisa memerintah Docker daemon host, container-nya mount Docker socket (`/var/run/docker.sock`) dan punya Docker CLI di dalamnya.

> ⚠️ Catatan keamanan: Mount `docker.sock` memberi Jenkins kontrol penuh atas Docker host (setara akses root). Ini wajar untuk belajar di mesin lokal, tapi **jangan** dilakukan di server produksi tanpa pengamanan tambahan.

### 2.1 File yang dipakai

Dua file ini sudah tersedia di root proyek:

**`jenkins-docker/Dockerfile`** — custom image Jenkins berisi Docker CLI:

```dockerfile
FROM jenkins/jenkins:lts
USER root
# Install Docker CLI (client saja; daemon tetap pakai host via socket)
RUN apt-get update && apt-get install -y docker.io && rm -rf /var/lib/apt/lists/*
# Tambahkan user jenkins ke group docker agar bisa akses socket
RUN groupadd -f docker && usermod -aG docker jenkins
USER jenkins
```

**`docker-compose.yml`** — definisi controller Jenkins:

```yaml
services:
  jenkins:
    build: ./jenkins-docker
    image: jenkins-docker:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"     # Web UI
      - "50000:50000"   # agent (JNLP)
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    # GID group docker host agar user jenkins boleh akses docker.sock.
    group_add:
      - "${DOCKER_GID:-999}"

volumes:
  jenkins_home:
```

### 2.2 Jalankan Jenkins dengan Compose

Cari dulu GID group `docker` host supaya user `jenkins` punya izin ke socket, lalu build & jalankan:

```bash
cd /home/iqbal/Project/java/tubes-jenkins
export DOCKER_GID=$(stat -c '%g' /var/run/docker.sock)
docker compose up -d --build
```

- `--build` memastikan image custom (berisi Docker CLI) ter-build pertama kali.
- `DOCKER_GID` otomatis mengisi `group_add` di compose. Kalau tidak di-set, default ke `999`.

### 2.3 Verifikasi Jenkins bisa pakai Docker

```bash
docker compose exec jenkins docker ps   # harus menampilkan daftar container, bukan permission denied
```

Buka `http://localhost:8080` di browser.

<!-- SEC_2 -->

---

## 3. Konfigurasi Awal Jenkins & Plugin

### 3.1 Unlock Jenkins

Ambil initial admin password:

```bash
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```


Tempel di halaman web → pilih **Install suggested plugins** → buat user admin pertama → simpan URL Jenkins.

### 3.2 Install plugin tambahan

Buka **Manage Jenkins → Plugins → Available plugins**, cari & install:

- **Docker Pipeline** — wajib, untuk `agent { docker { ... } }`.
- **HTML Publisher** — untuk publish report JaCoCo & Checkstyle.
- **GitHub** — untuk external trigger/webhook (opsional tapi disarankan).
- **Slack Notification** — opsional, untuk notifikasi chat.

Restart Jenkins bila diminta.

### 3.3 Verifikasi Docker Pipeline aktif

Setelah plugin Docker Pipeline terpasang dan kontainer Jenkins bisa akses `docker.sock` (Section 2.3), pipeline dengan `agent { docker { ... } }` sudah bisa dijalankan.

<!-- SEC_3 -->

---

## 4. Memahami Docker Agent Per-Stage

Dengan pola ini, tiap stage berjalan di dalam container image-nya sendiri, lalu container dihapus setelah stage selesai. Keuntungannya: lingkungan build bersih & reprodusibel, tanpa memasang Gradle/Python di host.

Pola dasar:

```groovy
pipeline {
    agent none                       // tidak ada agent global
    stages {
        stage("Contoh") {
            agent { docker { image 'gradle:7.6-jdk11' } }   // agent khusus stage
            steps {
                sh "gradle --version"
            }
        }
    }
}
```

Catatan penting:
- `agent none` di level pipeline berarti tiap stage **wajib** mendefinisikan agent-nya sendiri.
- Karena kita pakai image `gradle`, perintahnya bisa langsung `gradle <task>` (tidak harus `./gradlew`). Keduanya valid; `./gradlew` memastikan versi Gradle sama dengan wrapper proyek.
- Untuk mempercepat build (cache dependency Gradle antar-run), tambahkan argumen volume:
  ```groovy
  agent { docker { image 'gradle:7.6-jdk11'; args '-v $HOME/.gradle:/home/gradle/.gradle' } }
  ```
- Image yang dipakai di panduan ini:
  - Java/Gradle: `gradle:7.6-jdk11`
  - Python: `python:3`

<!-- SEC_4 -->

---

## 5. Tahap 0 - Hello World Pipeline

Tujuan: memahami struktur stage & step dan visualisasinya.

1. Di Jenkins, klik **New Item**.
2. Nama: `hello-world`, pilih **Pipeline**, klik OK.
3. Scroll ke bagian **Pipeline**, di kolom Script tempel:

```groovy
pipeline {
    agent any
    stages {
        stage('First Stage') {
            steps {
                echo 'Step 1. Hello World'
            }
        }
        stage('Second Stage') {
            steps {
                echo 'Step 2. Second time Hello'
                echo 'Step 3. Third time Hello'
            }
        }
    }
}
```

4. Save → **Build Now**. Kamu akan melihat 2 kotak stage. Klik build di "Stage View" / "Console Output" untuk melihat detail step.

> Catatan: di Tahap 0 ini `agent any` sudah cukup (tidak butuh Docker) karena hanya `echo`.

<!-- SEC_5 -->

---

## 6. Tahap 1 - Commit Pipeline Dasar (sample1)

Tujuan: membangun commit pipeline 3-stage (Checkout → Compile → Unit test) dengan Docker agent.

### 6.1 Push kode sample1 ke repo GitHub kamu

```bash
cd /home/iqbal/Project/java/tubes-jenkins/sample1

# Inisialisasi git untuk folder sample1 dan push ke repo calculator kamu
git init
git add .
git commit -m "Add sample1 commit pipeline project"
git branch -M main
git remote add origin https://github.com/<username>/calculator.git
git push -u origin main
```

> Ganti `<username>` dengan username GitHub-mu. Kalau repo sudah berisi README dari langkah persiapan, jalankan `git pull origin main --allow-unrelated-histories` dulu sebelum push, atau mulai dari repo kosong.

### 6.2 (Opsional) Uji lokal manual

Kalau ada Docker, kamu bisa uji tanpa memasang Gradle di host:

```bash
cd /home/iqbal/Project/java/tubes-jenkins/sample1
docker run --rm -v "$PWD":/app -w /app gradle:7.6-jdk11 gradle test
```

### 6.3 Buat pipeline `calculator` dengan Docker agent

1. **New Item** → nama `calculator` → **Pipeline** → OK.
2. Di bagian **Pipeline → Script**, tempel pipeline berikut (versi Docker agent per-stage, lengkap dengan Checkout):

```groovy
pipeline {
    agent none
    stages {
        stage("Checkout") {
            agent any
            steps {
                git url: 'https://github.com/<username>/calculator.git', branch: 'main'
            }
        }
        stage("Compile") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps {
                sh "gradle compileJava"
            }
        }
        stage("Unit test") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps {
                sh "gradle test"
            }
        }
    }
}
```

3. Save → **Build Now**. Harus muncul 3 stage hijau.

> Penting: dengan `agent none`, setiap stage punya agent sendiri. Stage Checkout cukup `agent any`; stage build pakai container `gradle`. Karena tiap stage container terpisah, kode dari Checkout otomatis tersedia karena Jenkins men-share workspace antar-stage pada node yang sama.

<!-- SEC_6 -->

---

## 7. Tahap 2 - Jenkinsfile from SCM

Tujuan: memindahkan definisi pipeline ke `Jenkinsfile` di repo (tanpa stage Checkout, karena Jenkins otomatis checkout).

### 7.1 Siapkan Jenkinsfile versi Docker agent

`sample1/Jenkinsfile` bawaan memakai `agent any`. Untuk konsisten dengan setup kita, ganti isinya menjadi versi Docker agent. Buat/timpa file `sample1/Jenkinsfile`:

```groovy
pipeline {
    agent none
    stages {
        stage("Compile") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps {
                sh "gradle compileJava"
            }
        }
        stage("Unit test") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps {
                sh "gradle test"
            }
        }
    }
}
```

Commit & push:

```bash
cd /home/iqbal/Project/java/tubes-jenkins/sample1
git add Jenkinsfile
git commit -m "Use Docker agent in Jenkinsfile"
git push
```

### 7.2 Ubah job agar baca Jenkinsfile dari SCM

1. Buka job `calculator` → **Configure**.
2. Di bagian **Pipeline**, ubah **Definition** menjadi **Pipeline script from SCM**.
3. **SCM**: Git.
4. **Repository URL**: `https://github.com/<username>/calculator.git`.
5. **Branch Specifier**: `*/main`.
6. **Script Path**: `Jenkinsfile` (default).
7. Save → **Build Now**.

Sekarang build selalu mengikuti versi `Jenkinsfile` terbaru di repo.

<!-- SEC_7 -->

---

## 8. Tahap 3 - Code Coverage (sample2)

Tujuan: menambah stage Code coverage memakai JaCoCo (minimum coverage 20%).

`sample2/` sudah berisi plugin `jacoco` + `jacocoTestCoverageVerification` di `build.gradle` dan stage Code coverage di Jenkinsfile.

### 8.1 Pindahkan isi sample2 ke repo

Cara paling sederhana untuk lanjut praktik: salin isi `sample2` ke working copy repo `calculator` (yang tadi dipush dari sample1), lalu commit. Atau, jika kamu mengelola tiap sample sebagai repo terpisah, push `sample2` ke repo-nya sendiri.

Contoh menyalin konten sample2 ke folder sample1 (repo aktif):

```bash
cd /home/iqbal/Project/java/tubes-jenkins
cp -r sample2/build.gradle sample1/build.gradle
cp -r sample2/Jenkinsfile sample1/Jenkinsfile
# (file src sama; cukup pastikan build.gradle & Jenkinsfile ter-update)
```

### 8.2 Jenkinsfile versi Docker agent (Compile → Unit test → Code coverage)

Timpa `Jenkinsfile`:

```groovy
pipeline {
    agent none
    stages {
        stage("Compile") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps { sh "gradle compileJava" }
        }
        stage("Unit test") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps { sh "gradle test" }
        }
        stage("Code coverage") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps {
                sh "gradle jacocoTestReport"
                sh "gradle jacocoTestCoverageVerification"
            }
        }
    }
}
```

### 8.3 (Opsional) Uji lokal

```bash
docker run --rm -v "$PWD/sample2":/app -w /app gradle:7.6-jdk11 \
  gradle test jacocoTestCoverageVerification
docker run --rm -v "$PWD/sample2":/app -w /app gradle:7.6-jdk11 \
  gradle test jacocoTestReport     # report: build/reports/jacoco/test/html/index.html
```

### 8.4 (Opsional) Publish report JaCoCo di Jenkins

Tambahkan `publishHTML` di stage Code coverage (butuh plugin HTML Publisher):

```groovy
stage("Code coverage") {
    agent { docker { image 'gradle:7.6-jdk11' } }
    steps {
        sh "gradle jacocoTestReport"
        publishHTML(target: [
            reportDir: 'build/reports/jacoco/test/html',
            reportFiles: 'index.html',
            reportName: "JaCoCo Report"
        ])
        sh "gradle jacocoTestCoverageVerification"
    }
}
```

Commit, push, lalu **Build Now**. Harus muncul 3 stage hijau + link "JaCoCo Report" di menu kiri.

<!-- SEC_8 -->

---

## 9. Tahap 4 - Static Code Analysis (sample3)

Tujuan: menambah stage Static code analysis memakai Checkstyle.

`sample3/` sudah berisi plugin `checkstyle`, file aturan `config/checkstyle/checkstyle.xml` (aturan `ConstantName`), dan Jenkinsfile 4-stage.

### 9.1 Pindahkan isi sample3 ke repo

```bash
cd /home/iqbal/Project/java/tubes-jenkins
cp sample3/build.gradle sample1/build.gradle
cp sample3/Jenkinsfile sample1/Jenkinsfile
mkdir -p sample1/config/checkstyle
cp sample3/config/checkstyle/checkstyle.xml sample1/config/checkstyle/checkstyle.xml
```

### 9.2 Jenkinsfile versi Docker agent (4 stage)

```groovy
pipeline {
    agent none
    stages {
        stage("Compile") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps { sh "gradle compileJava" }
        }
        stage("Unit test") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps { sh "gradle test" }
        }
        stage("Code coverage") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps {
                sh "gradle jacocoTestReport"
                sh "gradle jacocoTestCoverageVerification"
            }
        }
        stage("Static code analysis") {
            agent { docker { image 'gradle:7.6-jdk11' } }
            steps { sh "gradle checkstyleMain" }
        }
    }
}
```

### 9.3 Uji aturan Checkstyle (sengaja gagal)

Untuk membuktikan build gagal saat melanggar aturan, tambahkan konstanta dengan nama salah di `CalculatorApplication.java`:

```java
private static final String constant = "constant";   // huruf kecil -> langgar ConstantName
```

Build → stage **Static code analysis** harus gagal. Hapus lagi konstanta itu untuk membuat build hijau kembali.

### 9.4 (Opsional) Publish report Checkstyle

```groovy
publishHTML(target: [
    reportDir: 'build/reports/checkstyle/',
    reportFiles: 'main.html',
    reportName: "Checkstyle Report"
])
```

### 9.5 (Opsional) SonarQube sebagai alternatif

Jalankan SonarQube server via Docker:

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
```

Buka `http://localhost:9000` (login awal admin/admin), buat token, install **SonarQube Scanner plugin** di Jenkins, lalu tambahkan stage `sonar` yang memanggil scanner. SonarQube menggantikan kombinasi code coverage + static analysis dengan dashboard terpusat.

<!-- SEC_9 -->

---

## 10. Tahap 5 - Triggers & Notifications

Tujuan: pipeline berjalan otomatis dan tim mendapat notifikasi.

### 10.1 Polling SCM

Tambahkan blok `triggers` setelah `agent none` di Jenkinsfile:

```groovy
pipeline {
    agent none
    triggers {
        pollSCM('* * * * *')
    }
    stages {
        // ... stage seperti sebelumnya
    }
}
```

Commit & push. Jalankan build manual sekali untuk mengaktifkan trigger. Setelah itu, setiap push baru akan memicu build dalam ~1 menit. Uji dengan mengubah file kecil lalu `git push`.

> `* * * * *` = format cron, artinya cek repo setiap menit.

### 10.2 (Opsional) External trigger via GitHub webhook

Karena Jenkins lokal tidak bisa diakses GitHub publik, gunakan tunnel seperti **ngrok**:

```bash
ngrok http 8080
```

Lalu di GitHub repo → **Settings → Webhooks → Add webhook**:
- Payload URL: `https://<ngrok-id>.ngrok.io/github-webhook/`
- Content type: `application/json`
- Event: Just the push event.

Di Jenkins job, aktifkan **GitHub hook trigger for GITScm polling**.

### 10.3 (Opsional) Notifikasi email

1. **Manage Jenkins → Configure System** → atur **E-mail Notification** (SMTP server, dll).
2. Tambahkan blok `post` di Jenkinsfile:

```groovy
post {
    always {
        mail to: 'team@company.com',
        subject: "Completed Pipeline: ${currentBuild.fullDisplayName}",
        body: "Your build completed, please check: ${env.BUILD_URL}"
    }
}
```

### 10.4 (Opsional) Notifikasi Slack

Install **Slack Notification plugin**, konfigurasi workspace & token, lalu:

```groovy
post {
    failure {
        slackSend channel: '#dragons-team',
        color: 'danger',
        message: "The pipeline ${currentBuild.fullDisplayName} failed."
    }
}
```

<!-- SEC_10 -->

---

## 11. Tahap 6 - Multi-branch & Team Strategies

Tujuan: memastikan kode di branch sehat sebelum merge ke main, dengan Multibranch Pipeline.

### 11.1 Buat Multibranch Pipeline

1. **New Item** → nama `calculator-branches` → **Multibranch Pipeline** → OK.
2. Di **Branch Sources** → **Add source** → **Git**.
3. **Project Repository**: `https://github.com/<username>/calculator.git`.
4. Di **Scan Multibranch Pipeline Triggers**, centang **Periodically if not otherwise run**, set interval **1 minute**.
5. Save.

Jenkins akan memindai repo dan otomatis membuat pipeline untuk tiap branch yang punya `Jenkinsfile`.

### 11.2 Uji dengan branch baru

```bash
cd /home/iqbal/Project/java/tubes-jenkins/sample1
git checkout -b feature
git push origin feature
```

Dalam ~1 menit, pipeline untuk branch `feature` muncul otomatis dan berjalan. Periksa hijau/tidak sebelum merge ke `main`. Pendekatan ini menjaga `main` selalu sehat.

### 11.3 (Opsional) Feature toggle

Daripada branch panjang, gunakan flag di kode supaya semua dev tetap di trunk:

```java
if (featureToggle) {
    // kode fitur baru
}
```

- Coding di main dengan `featureToggle = true`.
- Release dari main dengan `featureToggle = false`.
- Setelah fitur stabil, hapus blok `if` dan toggle-nya.

<!-- SEC_11 -->

---

## 12. Tahap 7 - Exercises Python

Tujuan: membangun CI untuk proyek Python (bahasa terinterpretasi → tanpa stage Compile).

### 12.1 Exercise 1 - jalankan & uji lokal

```bash
cd /home/iqbal/Project/java/tubes-jenkins/exercise1
python calculator.py 2 2        # -> 4
python test_calculator.py       # -> Ran 1 test ... OK
```

Atau via Docker agent image Python:

```bash
docker run --rm -v "$PWD":/app -w /app python:3 python test_calculator.py
```

### 12.2 Exercise 2 - pipeline Python dengan Docker agent

Buat repo GitHub kedua (mis. `calculator-python`) berisi `calculator.py`, `test_calculator.py`, dan `Jenkinsfile`. Versi Docker agent:

```groovy
pipeline {
    agent none
    triggers {
        pollSCM('* * * * *')
    }
    stages {
        stage("Unit test") {
            agent { docker { image 'python:3' } }
            steps {
                sh "python test_calculator.py"
            }
        }
    }
}
```

Push, lalu buat job Pipeline baru (script from SCM) yang menunjuk ke repo Python tersebut. Build → satu stage **Unit test** hijau. Perhatikan: tidak ada stage Compile karena Python tidak dikompilasi.

<!-- SEC_12 -->

---

## 13. Troubleshooting

| Masalah | Penyebab & Solusi |
|---|---|
| `permission denied` saat akses `/var/run/docker.sock` | GID group docker salah. Set ulang: `export DOCKER_GID=$(stat -c '%g' /var/run/docker.sock)` lalu `docker compose up -d --force-recreate`. |
| Stage Docker gagal: `docker: not found` | Docker CLI belum terpasang di image Jenkins. Rebuild: `docker compose up -d --build`. |
| `gradle: command not found` di stage | Image salah. Pastikan `agent { docker { image 'gradle:7.6-jdk11' } }`. |
| Build Gradle lambat tiap run | Tambahkan cache: `args '-v $HOME/.gradle:/home/gradle/.gradle'` pada docker agent. |
| `git push` ditolak (repo sudah ada README) | `git pull origin main --allow-unrelated-histories` lalu push lagi, atau buat repo kosong tanpa README. |
| Report HTML tidak tampil | Install **HTML Publisher** plugin; bila masih kosong, sesuaikan Content Security Policy Jenkins. |
| Build Static code analysis tidak gagal padahal harusnya | Pastikan `config/checkstyle/checkstyle.xml` ikut di-commit ke repo. |
| pollSCM tidak jalan | Jalankan build manual sekali dulu untuk mengaktifkan trigger. |
| GitHub webhook tidak sampai | Jenkins lokal tidak publik; pakai ngrok dan URL berakhiran `/github-webhook/`. |

### Perintah manajemen Jenkins via Compose

```bash
docker compose logs -f jenkins   # lihat log Jenkins
docker compose restart jenkins   # restart
docker compose stop              # stop semua service
docker compose up -d             # start lagi (data tetap, karena volume jenkins_home)
docker compose down              # hapus container (volume jenkins_home tetap aman)
```

<!-- SEC_13 -->


---

## 14. Checklist Lengkap

### Persiapan & Jenkins
- [ ] `docker`, `git`, `python` tersedia
- [ ] Akun GitHub + repo `calculator` dibuat
- [ ] `export DOCKER_GID=$(stat -c '%g' /var/run/docker.sock)`
- [ ] `docker compose up -d --build` (Jenkins controller jalan)
- [ ] `docker compose exec jenkins docker ps` berhasil
- [ ] Unlock Jenkins + install suggested plugins
- [ ] Install plugin: Docker Pipeline, HTML Publisher, GitHub (Slack opsional)

### Tahap 0-2

- [ ] Tahap 0: pipeline Hello World 2 stage hijau
- [ ] Tahap 1: push sample1 ke repo
- [ ] Tahap 1: pipeline `calculator` (Checkout + Compile + Unit test) 3 stage hijau
- [ ] Tahap 2: Jenkinsfile (Docker agent) di repo
- [ ] Tahap 2: job pakai "Pipeline script from SCM"

### Tahap 3-4
- [ ] Tahap 3: build.gradle + Jenkinsfile sample2 (JaCoCo) terpasang
- [ ] Tahap 3: stage Code coverage hijau
- [ ] Tahap 3 (opsional): JaCoCo report ter-publish
- [ ] Tahap 4: build.gradle + checkstyle.xml + Jenkinsfile sample3 terpasang
- [ ] Tahap 4: stage Static code analysis hijau
- [ ] Tahap 4: uji langgar aturan → build gagal → perbaiki → hijau

### Tahap 5-7
- [ ] Tahap 5: `pollSCM` aktif, push memicu build otomatis
- [ ] Tahap 5 (opsional): webhook / email / Slack
- [ ] Tahap 6: Multibranch Pipeline `calculator-branches`
- [ ] Tahap 6: branch `feature` memicu pipeline otomatis
- [ ] Tahap 7: exercise1 jalan lokal (multiply)
- [ ] Tahap 7: pipeline Python (1 stage Unit test) hijau

### Verifikasi akhir
- [ ] Runtime commit pipeline < 5 menit
- [ ] Perintah di Jenkins konsisten dengan perintah lokal
<!-- SEC_14 -->
