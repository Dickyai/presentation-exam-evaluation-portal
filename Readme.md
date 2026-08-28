# Presentation Exam Evaluation Portal
## Sistem Evaluasi Ujian Presentasi Berbasis Web dengan Google Apps Script dan Google Sheets

## 1. PENDAHULUAN DAN GROUNDING MASALAH

### 1.1 Latar Belakang

Presentation Exam Evaluation Portal dikembangkan untuk mengatasi permasalahan operasional dalam proses penilaian Ujian Presentasi yang sebelumnya dilakukan secara langsung melalui spreadsheet.

Pendekatan berbasis spreadsheet pada dasarnya tetap dapat menjalankan proses penilaian. Permasalahan muncul ketika proses tersebut dilakukan secara REALTIME, melibatkan banyak praktikan, beberapa asisten penilai, dan aktivitas lain yang harus dilakukan secara simultan oleh asisten, seperti mengamati presentasi, mendengarkan sesi tanya jawab, memberikan skor, serta melakukan komunikasi internal.

Pada sistem spreadsheet konvensional, nilai dari beberapa asisten ditempatkan berdekatan tanpa isolasi antarpengguna yang kuat. Kondisi tersebut menciptakan beberapa risiko operasional:

1. Asisten dapat memasukkan nilai pada baris yang seharusnya menjadi tanggung jawab asisten lain.
2. Nilai yang telah diisi dapat tertimpa secara tidak sengaja.
3. Kesalahan semakin sulit dikenali ketika nilai yang tertimpa berada dalam rentang yang secara visual masih terlihat wajar.
4. Perhatian asisten terpecah antara aktivitas observasi presentasi dan navigasi spreadsheet.
5. Komunikasi antar-asisten melalui sel spreadsheet tidak memiliki identitas pengirim dan bersifat destruktif karena pesan sebelumnya harus dihapus ketika pembicaraan berganti.
6. Pengelolaan timer presentasi tidak memiliki satu sumber waktu terpusat yang dapat dipantau oleh seluruh asisten.

Dengan demikian, persoalan utama bukan sekadar "bagaimana menyimpan nilai", tetapi bagaimana merancang lingkungan evaluasi yang mengurangi peluang salah input, menjaga integritas data, dan tetap responsif pada kondisi penggunaan secara simultan.

### 1.2 Tujuan Sistem

Sistem dirancang dengan sasaran utama:

1. Mengisolasi daftar kelompok berdasarkan tanggung jawab asisten.
2. Memastikan setiap nilai terkait secara eksplisit dengan mahasiswa, kelompok, asisten, dan rubrik.
3. Mengurangi kemungkinan penimpaan nilai akibat kesalahan navigasi pengguna.
4. Menyediakan kalkulasi nilai secara realtime sekaligus melakukan validasi ulang pada sisi server.
5. Menyediakan timer presentasi dengan satu sumber waktu akhir di server.
6. Memfasilitasi komunikasi internal secara ringan tanpa menjadikan Google Sheets sebagai media utama untuk chat.
7. Mempertahankan kompatibilitas terhadap struktur database Google Sheets yang telah digunakan sebelumnya.

### 1.3 Prinsip Perancangan

Arsitektur sistem dibangun berdasarkan beberapa prinsip:

1. SERVER REMAINS AUTHORITATIVE  
   Nilai akhir, status submission, otorisasi, dan data yang digunakan untuk rekap divalidasi kembali pada backend.

2. DATA ISOLATION BY CONTEXT  
   Asisten bekerja pada kelompok yang memang menjadi tanggung jawabnya sehingga antarmuka tidak mengharuskan pengguna memilih baris global secara manual.

3. DEFENSIVE DATA PROCESSING  
   Input dari spreadsheet diperlakukan sebagai data yang dapat mengandung inkonsistensi struktural maupun format.

4. BACKWARD COMPATIBILITY  
   Penambahan fungsi baru harus sebisa mungkin tidak merusak data produksi lama.

5. MINIMIZE WRITE FREQUENCY  
   Operasi terhadap Google Sheets dikurangi pada aktivitas yang tidak membutuhkan persistensi setiap detik.

---

## 2. LIVE PORTAL DAN KREDENSIAL PENGUJIAN

### 2.1 Live Demo

Portal dapat diakses melalui:

https://script.google.com/macros/s/AKfycbzsju2sULnwRAV1N7iVn4OZkgJGErCSk1xW6e4Y0qHbb0BFkj36X7r3IotzT7Jp7NnOBQ/exec

Sistem demo ditujukan terutama untuk simulasi alur kerja ASISTEN dan OPERATOR, bukan sebagai portal praktikan.

### 2.2 Role-Based Access

```
                     PRESENTATION EXAM PORTAL
                              |
                     +--------+--------+
                     |                 |
                  ADMIN             OPERATOR
               (Assistant)        (Time Keeper)
                     |                 |
          +----------+---------+       |
          |                    |       |
       Login               Evaluation  |
          |                    |       |
       USERS Sheet          Scores      |
          |                    |       |
          +----------+---------+       |
                     |                 |
                     +--------+--------+
                              |
                        Timer Service
                              |
                    Browser-side Countdown
```

Role ASISTEN/ADMIN digunakan untuk aktivitas penilaian.

Role OPERATOR digunakan untuk pengendalian timer presentasi. Operator memiliki otoritas untuk memulai dan mengatur timer yang kemudian digunakan bersama oleh perangkat asisten lain.

Tidak terdapat akses praktikan pada konfigurasi saat ini karena sistem difokuskan untuk mempercepat dan mengamankan aktivitas internal asisten.

### 2.3 Kredensial Pengujian

Gunakan akun berikut hanya untuk kebutuhan demonstrasi:


Role     : Admin 1

Token
```
BUD-4821KLM
```

Role     : Admin 2

Token
```
BUK-PCN-999
```

Token tersebut merupakan kredensial DEMO. Untuk repositori publik, token sebaiknya dianggap sebagai kredensial sementara dan tidak digunakan untuk lingkungan produksi.

### 2.4 Alur Login

```text
INPUT TOKEN
    |
    v
apiLogin()
    |
    v
USERS SHEET
    |
    +---- Tidak ditemukan ----> LOGIN DITOLAK
    |
    +---- Ditemukan ----------> Role + Identity
                                  |
                                  v
                              DASHBOARD
```

Proses login memeriksa token dan menggunakan identitas pengguna untuk menentukan kelompok yang dapat ditampilkan.

---

## 3. ARSITEKTUR SISTEM DAN ALUR DATA

### 3.1 Gambaran Arsitektur

Sistem menggunakan pendekatan single-page application dengan Google Apps Script sebagai backend dan Google Sheets sebagai persistent data store.

```text
+---------------------------------------------------------+
|                     CLIENT LAYER                        |
|                                                         |
|                    HTML / CSS / JS                     |
|                    Single Page App                     |
|                                                         |
|       Login | Dashboard | Evaluation | Timer | Chat   |
+-----------------------------+---------------------------+
                              |
                              | google.script.run
                              v
+---------------------------------------------------------+
|                   APPLICATION LAYER                     |
|                                                         |
| apiLogin()                                              |
| apiGetDashboard()                                       |
| apiGetEvaluation()                                      |
| apiSaveScores()                                         |
| apiGetTimer()                                           |
| apiStartTimer()                                         |
| apiSaveChat()                                           |
| apiFinalizeGroup()                                      |
|                                                         |
| Validation | Authorization | Business Logic           |
+-----------------------------+---------------------------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
       Google Sheets    PropertiesService   LockService
              |
              v
+---------------------------------------------------------+
|                     DATA LAYER                          |
|                                                         |
| USERS                                                   |
| STUDENTS                                                |
| GROUPS                                                  |
| RUBRICS                                                 |
| SCORES                                                  |
| TIMER                                                   |
| Configuration / Supporting Sheets                       |
+---------------------------------------------------------+
```

### 3.2 Alur Penilaian

```text
ASISTEN LOGIN
     |
     v
VALIDASI USERS
     |
     v
LOAD DASHBOARD
     |
     v
DAFTAR KELOMPOK ASISTEN
     |
     v
PILIH KELOMPOK
     |
     +---------------------------+
     |                           |
     v                           v
DATA MAHASISWA              DATA RUBRIK
     |                           |
     +-------------+-------------+
                   |
                   v
             INPUT NILAI
                   |
                   v
             FRONTEND CALC
                   |
                   v
             PREVIEW NILAI
                   |
                   v
              AUTOSAVE
                   |
                   v
             apiSaveScores()
                   |
                   v
             LOCK SERVICE
                   |
                   v
               UPSERT
                   |
                   v
              SCORES SHEET
```

### 3.3 Alur Timer

Timer tidak menggunakan browser tertentu sebagai sumber kebenaran.

```text
OPERATOR
   |
   v
START / UPDATE TIMER
   |
   v
SERVER MENYIMPAN END TIMESTAMP
   |
   v
+-------------------------------+
|  ASISTEN A     ASISTEN B      |
|      |             |          |
|      v             v          |
| LOCAL COUNTDOWN  LOCAL TIMER  |
|      |             |          |
|      +------10s----+          |
|             |                 |
|             v                 |
|        SERVER SYNC            |
+-------------------------------+
```

Visualisasi realtime dilakukan secara lokal di browser menggunakan `setInterval`, sedangkan konsistensi waktu ditentukan oleh timestamp akhir yang disimpan pada server.

### 3.4 Fungsi API

Komunikasi antara frontend dan backend menggunakan pola fungsi `apiXxx`.

```text
Frontend
    |
    +--> apiLogin()
    +--> apiGetDashboard()
    +--> apiGetEvaluation()
    +--> apiSaveScores()
    +--> apiGetTimer()
    +--> apiStartTimer()
    +--> apiFinalizeGroup()
    |
    v
Google Apps Script Backend
```

---

## 4. FORMULASI PEMODELAN NILAI / LOGIKA BISNIS

### 4.1 Model Nilai Rubrik

Setiap rubrik memiliki:

1. Skor maksimum.
2. Bobot dalam persen.
3. Skor aktual yang diberikan asisten.

Kontribusi suatu rubrik dihitung dengan:

```text
Kontribusi Rubrik =
(Skor Aktual / Skor Maksimum) x Bobot
```

Contoh:

```text
Skor Aktual   = 4
Skor Maksimum = 5
Bobot         = 20%

Kontribusi =
(4 / 5) x 20
= 16
```

### 4.2 Total Nilai Asisten

Apabila terdapat `n` rubrik:

```text
Total Nilai Asisten =
Σ (Skor Aktual_i / Skor Maksimum_i) x Bobot_i
```

Apabila seluruh bobot berjumlah 100%, maka:

```text
0 <= Total Nilai <= 100
```

### 4.3 Nilai Akhir Mahasiswa

Satu mahasiswa dapat dinilai oleh beberapa asisten.

```text
Nilai Akhir =
(Total Nilai Asisten_1 +
 Total Nilai Asisten_2 +
 ...
 Total Nilai Asisten_m) / m
```

Nilai akhir menggunakan arithmetic mean terhadap seluruh total nilai asisten yang telah memberikan penilaian valid.

### 4.4 Konsistensi Frontend dan Backend

Perhitungan dilakukan di dua sisi:

```text
              INPUT
                |
        +-------+-------+
        |               |
        v               v
     FRONTEND        BACKEND
        |               |
   Live Preview      Final Recalc
        |               |
        +-------+-------+
                |
                v
          Final Evaluation
```

Frontend menyediakan umpan balik instan, sedangkan backend menghitung ulang untuk memastikan hasil akhir tidak hanya bergantung pada state browser.

### 4.5 Toleransi dan Validasi Input

Data numerik dinormalisasi sebelum digunakan dalam kalkulasi:

```text
" 4,5 "
   |
   v
trim()
   |
   v
replace(",", ".")
   |
   v
Number(...)
   |
   v
isNaN()
```

Identifier dibersihkan menggunakan fungsi seperti:

```text
sanitizeId_()
sanitizeStr_()
```

Proses ini mengurangi kegagalan pencocokan akibat spasi berlebih dan perbedaan kapitalisasi.

### 4.6 Ambang Batas Sistem

Validasi utama meliputi:

1. Skor maksimum harus lebih besar dari nol.
2. Skor aktual harus dapat dikonversi menjadi angka valid.
3. Bobot harus dapat diproses sebagai nilai numerik.
4. Mahasiswa, asisten, rubrik, dan kelompok harus memiliki identifier valid.

Status kelompok mengikuti model:

```text
DRAFT
  |
  v
EDITABLE
  |
  v
SUBMITTED
  |
  v
LOCKED
```

Setelah status menjadi `SUBMITTED`, perubahan nilai tidak diperbolehkan melalui alur normal sistem.

---

## 5. ANALISIS PERMASALAHAN TEKNIKAL DAN SOLUSI TERAPAN

### 5.1 Struktur Header Spreadsheet Tidak Konsisten

Google Sheets dikelola secara manual sehingga struktur tabel tidak selalu stabil.

Contoh:

```text
Baris 1 : DATA NILAI PRAKTIKUM
Baris 2 : kosong
Baris 3 : Kelompok | Nama | Nilai
Baris 4 : Data
```

Selain itu, nama kolom dapat berbeda:

```text
Kelompok
ID Kelompok
Grup
```

Solusi menggunakan `resolveSchema_()` untuk memindai beberapa baris awal dan menentukan kandidat header yang paling sesuai.

`SCHEMA_ALIASES` memetakan berbagai istilah ke field internal:

```text
"Kelompok"
"ID Kelompok"
"Grup"
      |
      v
group_id
```

Data kemudian dibaca mulai tepat setelah baris header yang terdeteksi.

### 5.2 Multi-Header dan Baris Data

Setelah header ditemukan, iterasi data harus dimulai pada:

```text
dataStartRow = detectedHeaderRow + 1
```

Tanpa aturan ini, judul atau baris metadata dapat salah diperlakukan sebagai record.

### 5.3 Type Coercion dan Dirty Numeric Data

Spreadsheet dapat mengandung:

```text
"4"
"4.5"
"4,5"
" 4.5 "
4
```

Data dinormalisasi sebelum operasi aritmatika dilakukan.

```text
RAW VALUE
   |
   v
NORMALIZE STRING
   |
   v
STANDARDIZE DECIMAL
   |
   v
Number()
   |
   v
isNaN()
   |
   +---- invalid ---> reject / fallback
   |
   +---- valid -----> calculation
```

Pendekatan ini penting untuk mencegah kegagalan kalkulasi akibat coercion implisit JavaScript.

### 5.4 Data Deduplication dan Upsert

Identitas record nilai secara konseptual dibentuk oleh:

```text
Mahasiswa
+
Asisten
+
Rubrik
```

Mekanisme penyimpanan:

```text
Jika kombinasi telah ada
        |
        v
      UPDATE

Jika kombinasi belum ada
        |
        v
       INSERT
```

Dengan demikian, autosave tidak menghasilkan duplikasi record untuk kombinasi evaluasi yang sama.

### 5.5 Race Condition pada Concurrent Write

Google Sheets bukan database transaksional dengan isolation level penuh.

Sistem menggunakan:

```javascript
LockService.getScriptLock()
```

untuk melindungi critical section:

```text
REQUEST A ----> GET LOCK ----> WRITE ----> RELEASE
                      |
REQUEST B ------------+-------> WAIT
                                  |
                                  v
                            GET LOCK
                                  |
                                  v
                                WRITE
                                  |
                                  v
                              RELEASE
```

Lock dilepas pada blok `finally` agar proses tidak macet apabila terjadi error.

### 5.6 Sinkronisasi Timer tanpa WebSocket

Apps Script tidak menyediakan WebSocket persistent untuk pola realtime konvensional.

Polling setiap detik akan meningkatkan beban request.

Solusi yang digunakan:

```text
SERVER
  |
  | endTimestamp
  v
CLIENT
  |
  +--> local countdown every 1 second
  |
  +--> server synchronization every 10 seconds
```

Pendekatan ini menghasilkan persepsi realtime tanpa request server setiap detik.

### 5.7 Autosave dan Reduksi Beban Spreadsheet

Penyimpanan setiap ketikan tidak efisien.

Sistem menggunakan autosave sekitar setiap 15 detik:

```text
USER INPUT
   |
   +--> LOCAL STATE
   |
   +--> TIMER 15s
          |
          v
      BATCH SAVE
          |
          v
      apiSaveScores()
```

Trade-off pendekatan ini adalah perubahan terakhir dapat hilang apabila browser berhenti sebelum proses autosave berikutnya.

### 5.8 Chat dan PropertiesService

Chat bersifat lebih sementara dibandingkan nilai evaluasi.

Karena itu, pesan disimpan menggunakan `PropertiesService` dengan pembatasan jumlah pesan dan masa berlaku.

Tujuannya adalah mengurangi operasi baca-tulis terhadap Google Sheets untuk data yang tidak membutuhkan persistensi permanen.

### 5.9 Evolusi Skema Tanpa Merusak Data Lama

Timer dan role management ditambahkan ketika data lama sudah tersedia.

`SetupDatabaseForTimer()` menerapkan additive migration:

```text
CHECK COLUMN EXISTS
        |
        +---- YES ----> NO ACTION
        |
        +---- NO -----> APPEND COLUMN
```

Kolom baru seperti `status` dan `role_in_group` ditambahkan tanpa memindahkan atau menghapus kolom lama.

### 5.10 Auto-Healing Header

Apabila sheet `SCORES` kosong atau header mengalami kerusakan, `apiSaveScores()` dapat memvalidasi struktur dasar dan menulis ulang header standar apabila diperlukan.

```text
SCORES SHEET
     |
     v
VALIDATE STRUCTURE
     |
     +---- VALID ----> CONTINUE
     |
     +---- INVALID --> REBUILD STANDARD HEADER
                           |
                           v
                         CONTINUE
```

Mekanisme ini ditujukan untuk kerusakan struktural yang dapat dikenali, bukan sebagai pemulihan universal terhadap seluruh jenis kehilangan data.

---

## 6. BATASAN ARSITEKTUR DAN PELUANG REVISI LANJUTAN

### 6.1 Google Sheets Tetap Menjadi Bottleneck

Google Sheets bukan database transaksional yang dirancang untuk workload concurrent application.

Konsekuensinya:

1. Skalabilitas write operation tetap terbatas.
2. Lock hanya bekerja pada critical section yang dikendalikan aplikasi.
3. Query masih bergantung pada pembacaan range spreadsheet.
4. Struktur data dapat dipengaruhi oleh perubahan manual administrator.

Pendekatan saat ini pragmatis untuk skala penggunaan terbatas hingga menengah, tetapi tidak ideal untuk concurrency tinggi.

### 6.2 Tidak Ada True Realtime Transport

Timer menggunakan polling berkala.

Sistem mencapai:

```text
Near-Realtime Client Experience
+
Periodic Server Reconciliation
```

bukan realtime network push.

Untuk kebutuhan lebih ketat, backend dapat dikembangkan menggunakan layanan yang mendukung event-driven communication.

### 6.3 Risiko Kehilangan Client-Side State

Autosave mengurangi write frequency, tetapi menciptakan window kehilangan perubahan.

Pengembangan berikutnya dapat menggunakan:

```text
LocalStorage / IndexedDB
        +
Periodic Server Autosave
        +
Retry Queue
```

### 6.4 Validasi Hak Akses Harus Server-Side

Keamanan tidak boleh bergantung pada elemen UI yang disembunyikan.

Model yang lebih aman:

```text
CLIENT REQUEST
      |
      v
AUTHENTICATION
      |
      v
AUTHORIZATION
      |
      v
RESOURCE OWNERSHIP CHECK
      |
      v
BUSINESS OPERATION
```

Backend harus memastikan asisten memang memiliki hak terhadap kelompok yang diminta.

### 6.5 Implikasi Kredensial Demo

Token demo yang terdapat dalam dokumentasi harus dianggap sebagai non-secret.

Untuk produksi:

1. Jangan menyimpan kredensial produksi di README.
2. Jangan melakukan hard-code secret di source code.
3. Pisahkan environment development, staging, dan production.
4. Gunakan konfigurasi atau secret management yang sesuai.

### 6.6 Revisi Arsitektur Jangka Menengah

Apabila skala meningkat, arsitektur dapat dikembangkan menjadi:

```text
                 +------------------+
                 |   Web Frontend   |
                 +--------+---------+
                          |
                          v
                 +------------------+
                 |   API Backend    |
                 +--------+---------+
                          |
             +------------+------------+
             |            |            |
             v            v            v
         SQL DB      Cache Layer   Realtime Service
             |                         |
             v                         v
        Persistent Data            Timer / Chat
```

Google Sheets dapat diposisikan sebagai reporting layer atau media administrasi dan import-export.

### 6.7 Potensi Optimasi

Pengembangan lanjutan dapat difokuskan pada:

1. Schema validation yang lebih formal.
2. Repository layer untuk memisahkan business logic dari operasi spreadsheet.
3. Typed data model.
4. Retry mechanism untuk transient failure.
5. Structured error response dari API.
6. Client-side offline queue.
7. Audit log untuk setiap perubahan nilai.
8. Database transaksional apabila skala pengguna meningkat.
9. Event-driven synchronization untuk kebutuhan realtime yang lebih ketat.
10. Automated testing untuk parser schema, sanitization, upsert, dan score calculation.

---

## 7. LISENSI DAN ISOLASI KODE

### 7.1 Lisensi

Status lisensi proyek mengikuti keputusan pemilik repositori.

Apabila belum memilih lisensi open-source, kode dapat dipublikasikan untuk kepentingan akademik, dokumentasi, dan demonstrasi tanpa memberikan hak distribusi komersial secara implisit.

Apabila proyek akan dikembangkan sebagai open-source software, lisensi formal seperti MIT, Apache-2.0, atau GPL perlu dipilih secara eksplisit sesuai kebutuhan distribusi.

### 7.2 Isolasi Kode dan Data

Repositori publik sebaiknya tidak mengandung:

1. Production tokens.
2. Private API credentials.
3. Spreadsheet ID untuk dataset rahasia.
4. Data pribadi mahasiswa.
5. Private deployment configuration.
6. Institutional credentials.

Arsitektur isolasi yang disarankan:

```text
PUBLIC REPOSITORY
      |
      +--> Source Code
      +--> Documentation
      +--> Sample Data
      |
      X
      |
      +--> Production Credentials
      +--> Private Dataset
      +--> Institutional Secrets
```

Kode aplikasi dapat dipublikasikan sebagai artefak teknis, sedangkan data dan kredensial lingkungan produksi tetap dipisahkan.

---

## PENUTUP

Presentation Exam Evaluation Portal merupakan implementasi aplikasi evaluasi yang memanfaatkan Google Apps Script dan Google Sheets sebagai fondasi aplikasi operasional.

Nilai utama sistem bukan sekadar menggantikan spreadsheet, melainkan mengubah model interaksi:

```text
SPREADSHEET-CENTERED EVALUATION
            |
            v
USER-CENTERED EVALUATION
            |
            v
CONTEXT ISOLATION
            |
            v
SERVER VALIDATION
            |
            v
CONTROLLED PERSISTENCE
```

Arsitektur yang digunakan merupakan kompromi antara kebutuhan operasional, keterbatasan platform, dan keberadaan data yang telah digunakan sebelumnya.

Sistem ini belum ditujukan sebagai pengganti database transaksional maupun platform realtime penuh. Namun, penerapan schema resolution, input normalization, locking, upsert, autosave batching, server-authoritative timer, dan additive migration menunjukkan pendekatan defensif terhadap permasalahan nyata yang muncul ketika spreadsheet berkembang dari media pencatatan menjadi fondasi aplikasi operasional.

Repository ini dapat dipahami sebagai studi implementasi mengenai bagaimana ekosistem Google Workspace dapat direkayasa secara lebih terstruktur untuk mendukung proses evaluasi realtime dengan concurrency dan struktur data yang tidak sepenuhnya terkontrol.
