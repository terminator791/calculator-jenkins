# BAB 4 - Continuous Integration Pipeline

Rangkuman lengkap dan panduan praktik dari buku *Continuous Delivery with Docker and Jenkins (3rd Edition)* karya Rafał Leszko.

> Tujuan bab ini: membangun proses **Continuous Integration (CI)** lengkap dari nol menggunakan Jenkins Pipeline.

## Daftar Isi
1. [Technical Requirements](#1-technical-requirements)
2. [Introducing Pipelines](#2-introducing-pipelines)
3. [The Commit Pipeline](#3-the-commit-pipeline)
4. [Jenkinsfile](#4-jenkinsfile)
5. [Code-Quality Stages](#5-code-quality-stages)
6. [Triggers and Notifications](#6-triggers-and-notifications)
7. [Team Development Strategies](#7-team-development-strategies)
8. [Non-Technical Requirements](#8-non-technical-requirements)
9. [Ringkasan / Key Takeaways](#9-ringkasan--key-takeaways)
10. [Pemetaan File Proyek (Hasil Download)](#10-pemetaan-file-proyek-hasil-download)
11. [Exercises (Solusi)](#11-exercises-solusi)
12. [Checklist Praktik](#12-checklist-praktik)


---

## 1. Technical Requirements

Perangkat lunak yang dibutuhkan untuk menyelesaikan bab ini:

- **Jenkins** (sudah terpasang dari Bab 3)
- **Java JDK 8+**
- **Git** (terpasang pada node tempat build dijalankan)

Contoh kode resmi tersedia di:
`https://github.com/PacktPublishing/Continuous-Delivery-With-Docker-and-Jenkins-3rd-Edition/tree/main/Chapter04`

Proyek contoh yang dibangun di bab ini adalah aplikasi **calculator** sederhana menggunakan teknologi: **Git, Java, Gradle, dan Spring Boot**. Prinsipnya tetap berlaku untuk teknologi lain.

---

## 2. Introducing Pipelines

**Pipeline** adalah rangkaian operasi otomatis yang biasanya merepresentasikan bagian dari proses software delivery dan quality assurance. Anggap saja sebagai rantai script dengan keuntungan:

- **Operation grouping**: Operasi dikelompokkan ke dalam *stages* (disebut juga *gates* / *quality gates*). Aturannya: jika satu stage gagal, stage berikutnya tidak dijalankan.
- **Visibility**: Semua aspek proses divisualisasikan, membantu analisis kegagalan dengan cepat dan mendorong kolaborasi tim.
- **Feedback**: Anggota tim langsung tahu ketika ada masalah sehingga bisa bereaksi cepat.

### Struktur Pipeline

Pipeline Jenkins terdiri dari dua elemen utama:

- **Step**: Operasi tunggal yang memberi tahu Jenkins apa yang harus dilakukan (misalnya checkout kode dari repo, eksekusi script).
- **Stage**: Pengelompokan logis dari beberapa step yang konseptual berbeda (misalnya build, test, deploy). Digunakan untuk memvisualisasikan progres pipeline.

> Catatan: Secara teknis bisa membuat *parallel steps*, namun perlakukan sebagai pengecualian yang hanya untuk optimasi.

### Multi-stage Hello World

Contoh pipeline dengan dua stage dan tiga step:

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

Klik **Build Now** untuk melihat representasi visualnya. Jika ada step yang gagal, proses berhenti dan step berikutnya tidak dijalankan. Inti dari pipeline adalah mencegah eksekusi step berikutnya dan memvisualisasikan titik kegagalan.

### Sintaks Pipeline (Declarative Syntax)

Buku ini menggunakan **declarative syntax** yang direkomendasikan untuk semua proyek baru. Alternatif lain: Groovy-based DSL dan (sebelum Jenkins 2) XML.

Contoh pipeline kompleks yang mencakup hampir semua instruksi Jenkins:

```groovy
pipeline {
    agent any
    triggers { cron('* * * * *') }
    options { timeout(time: 5) }
    parameters {
        booleanParam(name: 'DEBUG_BUILD', defaultValue: true,
        description: 'Is it the debug build?')
    }
    stages {
        stage('Example') {
            environment { NAME = 'Rafal' }
            when { expression { return params.DEBUG_BUILD } }
            steps {
                echo "Hello from $NAME"
                script {
                    def browsers = ['chrome', 'firefox']
                    for (int i = 0; i < browsers.size(); ++i) {
                        echo "Testing the ${browsers[i]} browser."
                    }
                }
            }
        }
    }
    post { always { echo 'I will always say Hello again!' } }
}
```

Apa yang dilakukan pipeline di atas:
1. Menggunakan agent apa pun yang tersedia.
2. Dijalankan otomatis setiap menit (`cron`).
3. Berhenti jika eksekusi lebih dari 5 menit (`timeout`).
4. Meminta input parameter boolean sebelum mulai.
5. Set `NAME = Rafal` sebagai environment variable.
6. Hanya jika input `true`: cetak "Hello from Rafal", "Testing the chrome browser", "Testing the firefox browser".
7. Selalu cetak "I will always say Hello again!" terlepas ada error atau tidak.

#### Sections (mendefinisikan struktur pipeline)
- **stages**: Mendefinisikan rangkaian satu atau lebih `stage`.
- **steps**: Mendefinisikan rangkaian satu atau lebih instruksi step.
- **post**: Step yang dijalankan di akhir build, ditandai kondisi (`always`, `success`, `failure`, dll). Biasanya untuk notifikasi.
- **agent**: Menentukan di mana eksekusi terjadi (`label` untuk mencocokkan agent, atau `docker` untuk container).

#### Directives (konfigurasi pipeline atau bagiannya)
- **triggers**: Cara otomatis memicu pipeline (`cron` untuk jadwal waktu, `pollSCM` untuk cek perubahan repo).
- **options**: Opsi spesifik pipeline, misalnya `timeout`, `retry`.
- **environment**: Set key-value sebagai environment variable.
- **parameters**: Daftar input parameter dari user.
- **stage**: Pengelompokan logis dari step.
- **when**: Menentukan apakah stage dijalankan berdasarkan kondisi.
- **tools**: Mendefinisikan tools untuk diinstall dan dimasukkan ke PATH.
- **input**: Meminta input parameter.
- **parallel**: Menjalankan stage secara paralel.
- **matrix**: Kombinasi parameter untuk menjalankan stage secara paralel.

#### Steps (operasi paling fundamental)
- **sh**: Mengeksekusi shell command (hampir semua operasi bisa didefinisikan dengan `sh`).
- **custom**: Operasi bawaan Jenkins (misalnya `echo`), banyak yang sekadar pembungkus `sh`. Plugin bisa menambah operasi sendiri.
- **script**: Mengeksekusi blok kode Groovy untuk skenario non-trivial yang butuh flow control.

> Spesifikasi sintaks lengkap: `https://jenkins.io/doc/book/pipeline/syntax/`
> Spesifikasi step lengkap: `https://jenkins.io/doc/pipeline/steps/`
<!-- SECTION_2 -->

---

## 3. The Commit Pipeline

Proses CI paling dasar disebut **commit pipeline**. Dimulai dari commit (atau push di Git) ke repository utama dan menghasilkan laporan sukses/gagal build. Karena berjalan setelah setiap perubahan kode:

- Build harus selesai **tidak lebih dari 5 menit**.
- Harus hemat sumber daya.
- Memberikan feedback paling penting: apakah kode dalam keadaan sehat.

**Cara kerja commit phase**: developer check-in kode ke repo → CI server mendeteksi perubahan → build dimulai.

Commit pipeline paling fundamental terdiri dari **3 stage**:
1. **Checkout**: download source code dari repository.
2. **Compile**: kompilasi source code.
3. **Unit test**: menjalankan suite unit test.

### 3.1 Checkout

Operasi pertama di pipeline mana pun. Membutuhkan repository terlebih dahulu.

**Membuat GitHub repository:**
1. Buka `https://github.com/`.
2. Buat akun jika belum ada.
3. Klik **New** di samping **Repositories**.
4. Beri nama: `calculator`.
5. Centang **Initialize this repository with a README**.
6. Klik **Create repository**.

Alamat repo contoh: `https://github.com/leszko/calculator.git`

**Membuat Checkout stage** (buat pipeline baru bernama `calculator`):

```groovy
pipeline {
    agent any
    stages {
        stage("Checkout") {
            steps {
                git url: 'https://github.com/leszko/calculator.git', branch: 'main'
            }
        }
    }
}
```

> Git toolkit harus terpasang pada node tempat build dijalankan.

Klik **Build Now** untuk memverifikasi.

### 3.2 Compile

Langkah-langkah:
1. Buat proyek dengan source code.
2. Push ke repository.
3. Tambahkan Compile stage ke pipeline.

**Membuat proyek Java Spring Boot (via Gradle):**
1. Buka `http://start.spring.io/`.
2. Pilih **Gradle Project** (boleh Maven jika lebih suka).
3. Isi **Group** dan **Artifact** (misalnya `com.leszko` dan `calculator`).
4. Tambahkan **Web** ke Dependencies.
5. Klik **Generate** → file `calculator.zip` terunduh.

**Push kode ke GitHub:**

```bash
# Clone repo
git clone https://github.com/leszko/calculator.git

# Ekstrak proyek dari start.spring.io ke direktori hasil clone
# Struktur direktori calculator:
#   build.gradle .git .gitignore gradle gradlew gradlew.bat HELP.md README.md settings.gradle src

# Kompilasi lokal (butuh Java JDK terpasang)
./gradlew compileJava        # Maven: ./mvnw compile

# Commit dan push
git add .
git commit -m "Add Spring Boot skeleton"
git push -u origin main
```

**Membuat Compile stage:**

```groovy
stage("Compile") {
    steps {
        sh "./gradlew compileJava"
    }
}
```

> Perintah lokal dan di pipeline sama persis — pertanda bagus bahwa proses development lokal konsisten dengan environment CI.

### 3.3 Unit Tests

Langkah-langkah:
1. Tambahkan source code logika calculator.
2. Tulis unit test.
3. Tambahkan stage Jenkins untuk eksekusi unit test.

**Business logic** — `src/main/java/com/leszko/calculator/Calculator.java`:

```java
package com.leszko.calculator;
import org.springframework.stereotype.Service;

@Service
public class Calculator {
    public int sum(int a, int b) {
        return a + b;
    }
}
```

**Web service controller** — `src/main/java/com/leszko/calculator/CalculatorController.java`:

```java
package com.leszko.calculator;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
class CalculatorController {
    @Autowired
    private Calculator calculator;

    @RequestMapping("/sum")
    String sum(@RequestParam("a") Integer a,
               @RequestParam("b") Integer b) {
        return String.valueOf(calculator.sum(a, b));
    }
}
```

Jalankan aplikasi dan uji manual:

```bash
./gradlew bootRun
# Buka browser: http://localhost:8080/sum?a=1&b=2  → menampilkan 3
```

**Unit test** — `src/test/java/com/leszko/calculator/CalculatorTest.java`:

```java
package com.leszko.calculator;
import org.junit.Test;
import static org.junit.Assert.assertEquals;

public class CalculatorTest {
    private Calculator calculator = new Calculator();

    @Test
    public void testSum() {
        assertEquals(5, calculator.sum(2, 3));
    }
}
```

Tambahkan dependency JUnit di `build.gradle`:

```groovy
dependencies {
    ...
    testImplementation 'junit:junit:4.13'
}
```

Jalankan test lokal lalu commit & push:

```bash
./gradlew test
git add .
git commit -m "Add sum logic, controller and unit test"
git push
```

**Membuat Unit test stage:**

```groovy
stage("Unit test") {
    steps {
        sh "./gradlew test"     // Maven: ./mvnw test
    }
}
```

Setelah build, akan terlihat **3 kotak (stage)** yang berarti commit pipeline lengkap.
<!-- SECTION_3 -->

---

## 4. Jenkinsfile

Daripada menaruh kode pipeline langsung di Jenkins, kita bisa menaruh definisi pipeline dalam file bernama **Jenkinsfile** dan commit ke repository bersama source code. Pendekatan ini lebih konsisten karena bentuk pipeline terkait erat dengan proyek itu sendiri.

**Keuntungan menyimpan pipeline di Jenkinsfile:**
- Jika Jenkins gagal, definisi pipeline tidak hilang (tersimpan di repo).
- Riwayat perubahan pipeline tersimpan.
- Perubahan pipeline melewati proses development standar (misalnya code review).
- Akses ke perubahan pipeline dibatasi sama seperti akses ke source code.

> Pipeline sebaiknya dibuat oleh orang yang menulis kode (developer), dan disimpan bersama kode di repository. Misalnya, bahasa terinterpretasi (Python) tidak butuh stage Compile; tools juga berbeda tergantung environment (Gradle/Maven untuk Java, PyBuilder untuk Python).

**Membuat Jenkinsfile** di root direktori proyek. Isinya hampir sama dengan commit pipeline, kecuali **stage Checkout dihilangkan** (Jenkins sudah otomatis checkout kode termasuk Jenkinsfile):

```groovy
pipeline {
    agent any
    stages {
        stage("Compile") {
            steps {
                sh "./gradlew compileJava"
            }
        }
        stage("Unit test") {
            steps {
                sh "./gradlew test"
            }
        }
    }
}
```

Commit dan push:

```bash
git add Jenkinsfile
git commit -m "Add Jenkinsfile"
git push
```

**Menjalankan pipeline dari Jenkinsfile** — buka konfigurasi pipeline, di bagian Pipeline:
1. Ubah **Definition** dari *Pipeline script* menjadi **Pipeline script from SCM**.
2. Pilih **Git** di SCM.
3. Isi Repository URL: `https://github.com/leszko/calculator.git`.
4. Gunakan `*/main` sebagai Branch Specifier.

Setelah disimpan, build akan selalu berjalan dari versi Jenkinsfile terkini di repository. Ini adalah commit pipeline lengkap pertama — minimum viable product yang sering kali sudah cukup sebagai proses CI.
<!-- SECTION_4 -->

---

## 5. Code-Quality Stages

Tiga stage klasik CI (checkout, compile, unit test) dapat diperluas dengan stage tambahan. Yang paling populer: **code coverage** dan **static code analysis**.

### 5.1 Code Coverage

Masalah: pipeline lulus semua build, tapi belum tentu kode benar-benar diuji (mungkin tidak ada yang menulis unit test). Solusinya tambahkan **code coverage tool** yang menjalankan semua test dan memverifikasi bagian kode mana yang dieksekusi, lalu membuat report bagian yang belum diuji. Build bisa dibuat gagal jika terlalu banyak kode tidak diuji.

Tool coverage populer untuk Java: **JaCoCo, OpenClover, Cobertura**.

Langkah menggunakan JaCoCo:

**1. Tambahkan JaCoCo ke `build.gradle`:**

```groovy
plugins {
    ...
    id 'jacoco'
}
```

Buat Gradle gagal jika coverage rendah (minimum 20%):

```groovy
jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.2
            }
        }
    }
}
```

Jalankan verifikasi dan generate report:

```bash
./gradlew test jacocoTestCoverageVerification
./gradlew test jacocoTestReport
# Report HTML: build/reports/jacoco/test/html/index.html
```

**2. Tambahkan Code coverage stage:**

```groovy
stage("Code coverage") {
    steps {
        sh "./gradlew jacocoTestReport"
        sh "./gradlew jacocoTestCoverageVerification"
    }
}
```

**3. (Opsional) Publish JaCoCo report di Jenkins** menggunakan plugin **HTML Publisher**:

```groovy
stage("Code coverage") {
    steps {
        sh "./gradlew jacocoTestReport"
        publishHTML (target: [
            reportDir: 'build/reports/jacoco/test/html',
            reportFiles: 'index.html',
            reportName: "JaCoCo Report"
        ])
        sh "./gradlew jacocoTestCoverageVerification"
    }
}
```

> Butuh plugin **HTML Publisher**. Jika report tidak tampil benar, mungkin perlu konfigurasi Jenkins Security (Content Security Policy).
> Untuk coverage lebih ketat, lihat konsep **mutation testing** dengan framework **PIT** (`http://pitest.org/`).

### 5.2 Static Code Analysis

**Static code analysis** adalah proses otomatis memeriksa kode tanpa benar-benar menjalankannya — mengecek sejumlah aturan pada source code. Contoh aturan: semua public class harus punya komentar Javadoc, panjang baris maksimal 120 karakter, jika class punya `equals()` harus punya `hashCode()` juga.

Tool populer untuk Java: **Checkstyle, FindBugs, PMD**.

Contoh menggunakan **Checkstyle** dalam 3 langkah:

**1. Tambahkan konfigurasi Checkstyle** — file `config/checkstyle/checkstyle.xml`:

```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Puppy Crawl//DTD Check Configuration 1.2//EN"
    "http://www.puppycrawl.com/dtds/configuration_1_2.dtd">
<module name="Checker">
    <module name="TreeWalker">
        <module name="ConstantName" />
    </module>
</module>
```

Konfigurasi ini hanya berisi satu aturan: cek apakah semua konstanta Java mengikuti naming convention (huruf kapital semua).

Tambahkan plugin checkstyle ke `build.gradle`:

```groovy
plugins {
    ...
    id 'checkstyle'
}
```

Jalankan:

```bash
./gradlew checkstyleMain
```

Untuk menguji kegagalan, tambahkan konstanta dengan nama salah, misalnya di `CalculatorApplication.java`:

```java
@SpringBootApplication
public class CalculatorApplication {
    private static final String constant = "constant";   // melanggar -> build gagal
    public static void main(String[] args) {
        SpringApplication.run(CalculatorApplication.class, args);
    }
}
```

**2. Tambahkan Static code analysis stage:**

```groovy
stage("Static code analysis") {
    steps {
        sh "./gradlew checkstyleMain"
    }
}
```

**3. (Opsional) Publish Checkstyle report:**

```groovy
publishHTML (target: [
    reportDir: 'build/reports/checkstyle/',
    reportFiles: 'main.html',
    reportName: "Checkstyle Report"
])
```

### 5.3 SonarQube

**SonarQube** adalah tool manajemen kualitas source code paling tersebar luas. Mendukung banyak bahasa pemrograman dan bisa menjadi alternatif untuk step code coverage dan static code analysis. SonarQube adalah server terpisah yang mengagregasi berbagai framework analisis (Checkstyle, FindBugs, JaCoCo), punya dashboard sendiri, dan terintegrasi baik dengan Jenkins.

Alih-alih menambah step kualitas kode ke pipeline, kita install SonarQube, tambahkan plugin di sana, lalu tambahkan stage `sonar` ke pipeline. Keuntungannya: web interface yang ramah pengguna untuk mengatur aturan dan menampilkan kerentanan kode.

> Info lebih lanjut: `https://www.sonarqube.org/`
<!-- SECTION_5 -->

---

## 6. Triggers and Notifications

Selama ini build dijalankan manual dengan tombol **Build Now**. Tidak praktis. Dengan trigger, pipeline mulai otomatis; dengan notifikasi, tim diberi tahu statusnya.

### 6.1 Triggers

Aksi otomatis untuk memulai build disebut **pipeline trigger**. Ada 3 tipe:

#### a) External
Jenkins memulai build setelah dipanggil oleh notifier (pipeline lain, sistem SCM seperti GitHub, atau remote script). GitHub memicu Jenkins setelah push ke repo.

Setup:
1. Install plugin **GitHub** di Jenkins.
2. Generate secret key untuk Jenkins.
3. Set GitHub webhook dengan alamat Jenkins dan key.

Cara generik lain: trigger via REST call ke endpoint `<jenkins_url>/job/<job_name>/build?token=<token>`. Demi keamanan, butuh set `token` di Jenkins.

> Jenkins harus dapat diakses dari server SCM. Jika menggunakan GitHub publik untuk memicu, Jenkins juga harus publik.

#### b) Polling SCM
Jenkins secara periodik memanggil GitHub dan mengecek apakah ada push, lalu memulai build. Berguna ketika:
- Jenkins berada di belakang firewall (GitHub tidak punya akses).
- Commit sering dan build lama, sehingga build per commit menyebabkan overload.

Konfigurasi (tambahkan setelah `agent`):

```groovy
triggers {
    pollSCM('* * * * *')
}
```

Setelah dijalankan manual pertama kali, trigger otomatis aktif. Argumen `* * * * *` adalah format string gaya **cron** yang menentukan seberapa sering Jenkins mengecek perubahan source.

> Format cron: `https://en.wikipedia.org/wiki/Cron`

#### c) Scheduled Builds
Jenkins menjalankan build secara periodik, terlepas ada commit atau tidak. Tidak butuh komunikasi dengan sistem lain. Implementasinya sama dengan polling SCM, tapi menggunakan keyword `cron` bukan `pollSCM`. Jarang dipakai untuk commit pipeline, tapi cocok untuk **nightly builds** (misalnya integration testing kompleks di malam hari).

```groovy
triggers {
    cron('* * * * *')
}
```

### 6.2 Notifications

Jenkins punya banyak cara mengumumkan status build (bisa ditambah lewat plugin).

#### Email
Cara klasik. Kelebihan: semua orang punya mailbox. Kekurangan: email Jenkins sering ter-filter dan tidak dibaca.

Konfigurasi:
1. Konfigurasi server **SMTP**.
2. Set detailnya di Jenkins (**Manage Jenkins | Configure System**).
3. Gunakan instruksi `mail` di pipeline.

```groovy
post {
    always {
        mail to: 'team@company.com',
        subject: "Completed Pipeline: ${currentBuild.fullDisplayName}",
        body: "Your build completed, please check: ${env.BUILD_URL}"
    }
}
```

Notifikasi biasanya dipanggil di section `post`, dieksekusi setelah semua step. Opsi kondisi `post`:
- **always**: selalu dieksekusi terlepas status.
- **changed**: hanya jika pipeline berubah status.
- **fixed**: hanya jika berubah dari failed ke success.
- **regression**: hanya jika berubah dari success ke failed, unstable, atau aborted.
- **aborted**: hanya jika pipeline dibatalkan manual.
- **failure**: hanya jika status failed.
- **success**: hanya jika status success.
- **unstable**: hanya jika status unstable (biasanya karena test gagal atau pelanggaran kode).
- **unsuccessful**: hanya jika status selain success.

#### Group Chats (misalnya Slack)
Prosedur:
1. Cari & install plugin chat tool (misalnya **Slack Notification plugin**).
2. Konfigurasi plugin (server URL, channel, authorization token, dll).
3. Tambahkan instruksi pengiriman ke pipeline.

```groovy
post {
    failure {
        slackSend channel: '#dragons-team',
        color: 'danger',
        message: "The pipeline ${currentBuild.fullDisplayName} failed."
    }
}
```

#### Team Spaces (Build Radiators)
Memasang layar besar (**build radiator**) di ruang tim yang menampilkan status pipeline terkini. Dianggap salah satu strategi notifikasi paling efektif — memastikan semua orang sadar build gagal dan memupuk semangat tim. Ada variasi kreatif: speaker yang berbunyi, mainan yang berkedip, atau **Pipeline State UFO** (`https://github.com/Dynatrace/ufo`). Komunitas Jenkins juga membuat RSS feeds, SMS, aplikasi mobile, dan desktop notifier.
<!-- SECTION_6 -->

---

## 7. Team Development Strategies

CI dipicu setelah commit, tapi ke branch yang mana? Cara menggunakan CI bergantung pada **development workflow** tim.

### 7.1 Development Workflows

Tiga tipe utama:

#### a) Trunk-based Workflow
Strategi paling sederhana. Ada satu repository pusat dengan satu entri (disebut **trunk** atau **master**). Setiap anggota tim clone repo pusat, dan perubahan di-commit langsung ke repo pusat.

#### b) Branching Workflow
Kode disimpan di banyak branch. Saat mulai fitur baru, developer membuat branch khusus dari trunk dan commit semua perubahan terkait fitur di sana. Memudahkan banyak developer bekerja tanpa merusak main code base, sehingga master tetap sehat. Setelah fitur selesai: developer rebase branch dari master, buat **pull request**, lalu kode di-review, dicek otomatis, dan di-merge ke main code base.

#### c) Forking Workflow
Populer di komunitas open source. Setiap developer punya repository sisi server sendiri. **Forking** = membuat repository baru dari repository lain. Developer push ke repo masing-masing, lalu buat pull request ke repo lain untuk integrasi. Kelebihan: integrasi tidak harus via repo pusat; membantu kepemilikan karena bisa menerima pull request tanpa memberi akses tulis.

> Detail semua workflow: `https://www.atlassian.com/git/tutorials/comparing-workflows`

### 7.2 Adopting Continuous Integration

Setiap workflow berimplikasi pada pendekatan CI:

- **Trunk-based**: pipeline sering gagal karena semua commit ke main code base. Aturan lama CI: *Jika build rusak, tim development berhenti dan langsung memperbaikinya.*
- **Branching**: menyelesaikan masalah trunk rusak, tapi memunculkan masalah lain — jika semua develop di branch sendiri, di mana integrasinya? Fitur bisa berminggu/berbulan tidak terintegrasi ke main code; tidak benar-benar *continuous integration*. Plus ada kebutuhan merge & resolve konflik terus-menerus.
- **Forking**: setiap pemilik repo mengelola proses CI sendiri; berbagi masalah yang sama dengan branching.

**Solusi terbaik**: gunakan teknik branching workflow dengan filosofi trunk-based — buat branch sangat kecil dan integrasikan sering ke master. Butuh fitur kecil atau **feature toggles**.

### 7.3 Feature Toggles

Teknik alternatif untuk memelihara banyak branch, agar fitur bisa diuji sebelum selesai/siap rilis. Fitur dinonaktifkan untuk user tapi diaktifkan untuk developer saat testing. Feature toggle pada dasarnya variabel dalam conditional statement (flag + if statement).

Alur:
1. Fitur baru perlu diimplementasi.
2. Buat flag/config property `feature_toggle` (bukan feature branch).
3. Semua kode terkait fitur dimasukkan ke dalam `if`:

```java
if (feature_toggle) {
    // do something
}
```

4. Selama pengembangan:
   - Coding di master dengan `feature_toggle = true`.
   - Release dari master dengan `feature_toggle = false`.
5. Setelah fitur selesai: hapus semua `if` dan `feature_toggle` dari config.

Manfaat: semua development di trunk, memfasilitasi CI nyata dan mengurangi masalah merge.

### 7.4 Jenkins Multi-branch

Jika menggunakan branch (long-feature atau short-lived), berguna mengetahui kode sehat sebelum merge ke master. Caranya:

1. Buka halaman utama Jenkins.
2. Klik **New Item**.
3. Masukkan nama `calculator-branches`, pilih **Multibranch Pipeline**, klik OK.
4. Di section **Branch Sources**, klik **Add source**, pilih **Git**.
5. Masukkan alamat repo di field **Project Repository**.
6. Centang **Periodically if not otherwise run** dan set interval 1 menit.
7. Klik **Save**.

Setiap menit, Jenkins mengecek apakah ada branch yang ditambah/dihapus dan membuat/menghapus pipeline khusus yang didefinisikan oleh Jenkinsfile.

Uji dengan membuat branch baru:

```bash
git checkout -b feature
git push origin feature
```

Setelah sesaat, pipeline branch baru otomatis dibuat dan dijalankan. Sebelum merge `feature` ke master, kita bisa cek apakah hijau (sehat). Pendekatan ini menjaga master selalu sehat. Alternatif serupa: pipeline per pull request, hasilnya sama.

---

## 8. Non-Technical Requirements

CI bukan hanya soal teknologi — teknologi malah datang nomor dua. James Shore, dalam artikel *Continuous Integration on a Dollar a Day*, menjelaskan cara setup CI tanpa software tambahan, hanya dengan **rubber chicken dan bel**. Idenya: tim bekerja di satu ruangan dengan satu komputer kosong. Saat akan check-in kode, ambil ayam karet, check-in kode, pindah ke komputer kosong, checkout kode segar, jalankan semua test di sana; jika lulus, kembalikan ayam karet dan bunyikan bel agar semua tahu ada penambahan ke repo.

Pesan utamanya: tanpa keterlibatan setiap anggota tim, tool terbaik pun tak akan membantu.

Prasyarat CI menurut Jez Humble:
- **Check in regularly**: minimum sekali sehari (continuously lebih sering dari yang dibayangkan).
- **Create comprehensive unit tests**: bukan sekadar coverage tinggi; mungkin saja punya 100% coverage tanpa assertion.
- **Keep the process quick**: CI harus cepat, idealnya di bawah 5 menit. 10 menit sudah banyak.
- **Monitor the builds**: bisa tanggung jawab bersama, atau gunakan peran **build master** yang dirotasi mingguan.

---

## 9. Ringkasan / Key Takeaways

- Pipeline menyediakan mekanisme umum untuk mengorganisasi proses otomasi; use case paling umum adalah CI dan continuous delivery.
- Jenkins menerima beberapa cara mendefinisikan pipeline, tapi yang direkomendasikan adalah **declarative syntax**.
- **Commit pipeline** adalah proses CI paling dasar dan harus dijalankan setiap commit ke repo.
- Definisi pipeline sebaiknya disimpan di repo sebagai file **Jenkinsfile**.
- Commit pipeline bisa diperluas dengan **code quality stages** (code coverage, static analysis, SonarQube).
- Apa pun build tool proyek, perintah Jenkins harus selalu konsisten dengan perintah development lokal.
- Jenkins menawarkan banyak **triggers** (external, polling SCM, scheduled) dan **notifications** (email, group chat, build radiator).
- Strategi development tim (trunk-based, branching, forking) memengaruhi cara CI diadopsi; **feature toggles** dan **Jenkins multi-branch** membantu menjaga master tetap sehat.

---

## 10. Pemetaan File Proyek (Hasil Download)

Berikut pemetaan file Chapter 4 yang sudah di-download ke materi di atas, beserta catatan penting yang berbeda/lebih detail dari contoh buku.

### Struktur direktori

```
.
├── README.md                # Petunjuk semua sample & exercise
├── sample1/                 # Commit pipeline dasar (Compile + Unit test)
├── sample2/                 # sample1 + Code coverage (JaCoCo)
├── sample3/                 # sample2 + Static code analysis (Checkstyle)
├── exercise1/               # Proyek Python Calculator + unit test
└── exercise2/               # Jenkinsfile CI untuk proyek Python
```

### Sample 1 — Basic Commit Pipeline (`sample1/`)
Proyek Spring Boot yang menjumlahkan dua angka.

```bash
cd sample1
./gradlew bootRun                       # jalankan service
curl "localhost:8080/sum?a=1&b=2"       # -> 3
./gradlew test                          # jalankan unit test
```

`sample1/Jenkinsfile` berisi 2 stage: **Compile** dan **Unit test** (tanpa Checkout, sesuai pola Jenkinsfile in SCM).

### Sample 2 — Code Coverage (`sample2/`)
Memperluas sample1 dengan JaCoCo.

```bash
cd sample2
./gradlew test jacocoTestCoverageVerification   # verifikasi minimum coverage
./gradlew test jacocoTestReport                 # generate report -> build/reports/jacoco/test
```

`sample2/Jenkinsfile` menambahkan stage **Code coverage** setelah Unit test:

```groovy
stage("Code coverage") {
    steps {
        sh "./gradlew jacocoTestReport"
        sh "./gradlew jacocoTestCoverageVerification"
    }
}
```

### Sample 3 — Static Code Analysis (`sample3/`)
Memperluas sample2 dengan Checkstyle.

```bash
cd sample3
./gradlew checkstyleMain
```

`sample3/Jenkinsfile` berisi 4 stage berurutan: **Compile → Unit test → Code coverage → Static code analysis**. Aturan Checkstyle ada di `sample3/config/checkstyle/checkstyle.xml` (hanya aturan `ConstantName`).

### Catatan penting: isi `build.gradle` nyata
`build.gradle` pada sample lebih lengkap dari potongan di buku. Versi aktual (sample2/sample3):

```groovy
plugins {
    id 'org.springframework.boot' version '2.6.1'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
    id 'jacoco'          // hanya sample2 & sample3
    id 'checkstyle'      // hanya sample3
}

group = 'com.leszko'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'        // proyek pakai Java 11

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'junit:junit:4.13'
}

test {
    useJUnitPlatform()
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.2
            }
        }
    }
}
```

Hal-hal yang perlu diperhatikan vs contoh buku:
- Proyek aktual memakai **Spring Boot 2.6.1** dan **Java 11** (`sourceCompatibility = '11'`), bukan JDK 8.
- Ada dependency tambahan `spring-boot-starter-test` dan blok `test { useJUnitPlatform() }`.
- Setiap sample memakai `gradlew` wrapper (jalankan `./gradlew`, tidak perlu Gradle terpasang global).

---

## 11. Exercises (Solusi)

### Exercise 1 — Proyek Python Calculator + Unit Test (`exercise1/`)

Solusi memakai operasi **multiply** (bukan sum), berbeda dari contoh Java di bab.

`exercise1/calculator.py`:

```python
import sys

def multiply(a, b):
    return a * b

if __name__ == '__main__':
    print(multiply(int(sys.argv[1]), int(sys.argv[2])))
```

`exercise1/test_calculator.py`:

```python
import unittest
from calculator import multiply

class TestSomething(unittest.TestCase):
    def test_multiply(self):
        self.assertEqual(6, multiply(2, 3))

if __name__ == '__main__':
    unittest.main()
```

Cara menjalankan:

```bash
cd exercise1
python calculator.py 2 2        # -> 4
python test_calculator.py       # -> Ran 1 test ... OK
```

### Exercise 2 — CI Pipeline berbasis Python (`exercise2/`)

`exercise2/Jenkinsfile` — pipeline Python tidak punya stage Compile (bahasa terinterpretasi), dan memakai trigger `pollSCM`:

```groovy
pipeline {
    agent any
    triggers {
        pollSCM('* * * * *')
    }
    stages {
        stage("Unit test") {
            steps {
                sh "python test_calculator.py"
            }
        }
    }
}
```

Ini menegaskan poin di Section 4: bentuk pipeline mengikuti teknologi proyek — Python hanya butuh stage Unit test.

---

## 12. Checklist Praktik

Gunakan checklist ini saat mempraktikkan semua materi Bab 4 secara berurutan.


### Persiapan
- [ ] Pastikan Jenkins berjalan (dari Bab 3).
- [ ] Pastikan Java JDK 8+ terpasang (`java -version`).
- [ ] Pastikan Git terpasang (`git --version`).

### Commit Pipeline Dasar
- [ ] Buat repository GitHub bernama `calculator`.
- [ ] Buat proyek Spring Boot (Gradle + dependency Web) dari `start.spring.io`.
- [ ] Clone repo, ekstrak proyek, push skeleton ke GitHub.
- [ ] Buat pipeline Jenkins `calculator` dengan stage **Checkout**.
- [ ] Tambahkan stage **Compile** (`./gradlew compileJava`).
- [ ] Tambahkan `Calculator.java` + `CalculatorController.java`.
- [ ] Uji manual: `./gradlew bootRun` → `http://localhost:8080/sum?a=1&b=2`.
- [ ] Tulis `CalculatorTest.java` + tambah dependency JUnit.
- [ ] Tambahkan stage **Unit test** (`./gradlew test`).
- [ ] Verifikasi 3 stage hijau di Jenkins.

### Jenkinsfile
- [ ] Buat `Jenkinsfile` di root proyek (Compile + Unit test, tanpa Checkout).
- [ ] Push `Jenkinsfile` ke repo.
- [ ] Ubah konfigurasi pipeline ke **Pipeline script from SCM** (Git, `*/main`).
- [ ] Verifikasi build berjalan dari Jenkinsfile.

### Code-Quality Stages
- [ ] Tambahkan plugin `jacoco` + `jacocoTestCoverageVerification` di `build.gradle`.
- [ ] Tambahkan stage **Code coverage**.
- [ ] (Opsional) Install plugin HTML Publisher + publish JaCoCo report.
- [ ] Buat `config/checkstyle/checkstyle.xml` + tambah plugin `checkstyle`.
- [ ] Tambahkan stage **Static code analysis**.
- [ ] (Opsional) Publish Checkstyle report.
- [ ] (Opsional) Coba SonarQube sebagai alternatif.

### Triggers & Notifications
- [ ] Tambahkan trigger `pollSCM('* * * * *')` dan uji dengan push.
- [ ] (Opsional) Konfigurasi external trigger via GitHub webhook.
- [ ] (Opsional) Konfigurasi notifikasi email/Slack di section `post`.

### Team Development Strategies
- [ ] Pahami trunk-based vs branching vs forking workflow.
- [ ] Coba buat Multibranch Pipeline `calculator-branches`.
- [ ] Buat branch `feature`, push, dan verifikasi pipeline branch otomatis berjalan.
- [ ] (Opsional) Eksperimen dengan feature toggle.

### Penutup
- [ ] Pastikan total runtime pipeline < 5 menit.
- [ ] Pastikan perintah CI konsisten dengan perintah lokal.

