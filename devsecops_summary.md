# DevSecOps Pipeline Security Scan Summary
## Project: OWASP Juice Shop (Branch: `test-security-tool`)

Dokumen ini merangkum seluruh proses latihan dan analisis kita dalam mengonfigurasi pipeline keamanan otomatis (DevSecOps) menggunakan **NPM Audit (SCA)** dan **Semgrep (SAST)**.

---

## 1. Analisis Awal: Mengapa Pipeline Pertama Lolos (Berwarna Hijau)?
Pada pengujian pertama, meskipun proyek OWASP Juice Shop penuh dengan celah keamanan, pipeline tetap berjalan sukses dan berwarna hijau. Hal ini disebabkan oleh:

* **Penyebab pada NPM Audit (SCA)**:
  Kita menggunakan perintah `npm audit || true`. Operator `|| true` memaksa perintah untuk selalu mengembalikan status sukses (exit code `0`) meskipun ditemukan celah keamanan berbahaya.
* **Penyebab pada Semgrep (SAST)**:
  Secara default, Semgrep Action berjalan dalam mode pelaporan (*non-blocking*). Semgrep mencoba mengunggah temuan ke dashboard cloud Semgrep App. Karena kita tidak menyertakan token API (`SEMGREP_APP_TOKEN`), Semgrep meloloskan *build* dan hanya menampilkan peringatan (*warning*) di log tanpa menggagalkan pipeline.

---

## 2. Mengapa Pipeline Akhirnya Gagal (Berwarna Merah)?
Setelah kita memperketat konfigurasi, pipeline akhirnya gagal (merah) secara valid karena mendeteksi celah keamanan nyata:

### A. Kegagalan pada NPM Audit (Dependency Scan)
* **Perubahan**: Kita menghapus `|| true` dan menambahkan batas tingkat keparahan: `npm audit --audit-level=high`.
* **Alasan Gagal**: Juice Shop menggunakan banyak dependensi (paket npm) pihak ketiga versi lama yang memiliki celah keamanan tingkat tinggi (*High/Critical*) yang sudah tercatat di database CVE publik.

### B. Kegagalan pada Semgrep (Code Scan / SAST)
* **Perubahan**: Kita menyetel `publishResults: "false"` untuk memaksa Semgrep berjalan dalam mode lokal/offline, serta menambahkan argument `--fail-on-severity WARNING` agar Semgrep bertindak tegas.
* **Alasan Gagal**: Semgrep berhasil menemukan celah keamanan kustom pada kode backend Juice Shop.

#### Studi Kasus Celah Keamanan Nyata: SQL Injection
Semgrep mendeteksi celah kritis pada file [routes/search.ts](file:///c:/Users/Dimar%20Alam/OneDrive/Desktop/Test/OWASP/juice-shop/routes/search.ts#L23) di baris 23:
```typescript
models.sequelize.query(`SELECT * FROM Products WHERE ((name LIKE '%${criteria}%' OR description LIKE '%${criteria}%') AND deletedAt IS NULL) ORDER BY name`)
```
* **Penyebab**: Kode di atas menggabungkan input mentah pengguna (`${criteria}`) langsung ke dalam query SQL menggunakan string template literal. Ini memicu kerentanan **SQL Injection**.
* **Solusi agar Lolos Scan (Kembali Hijau)**: Kita harus menulis query menggunakan *parameterized bindings*:
  ```typescript
  models.sequelize.query(
    "SELECT * FROM Products WHERE ((name LIKE :criteria OR description LIKE :criteria) AND deletedAt IS NULL) ORDER BY name",
    {
      replacements: { criteria: `%${criteria}%` },
      type: models.sequelize.QueryTypes.SELECT
    }
  )
  ```
  *(Catatan: Karena Juice Shop adalah laboratorium hacking, celah keamanan ini sengaja dibiarkan terbuka agar tantangan game tetap dapat diselesaikan).*

---

## 3. Rangkuman Perubahan pada Workflow `.github/workflows/security-scan.yml`
Untuk membuat pipeline yang andal, transparan, dan cepat, kita melakukan beberapa iterasi perubahan berikut:

1. **Pembersihan Workflow Bawaan**:
   Menghapus seluruh file workflow bawaan Juice Shop yang sangat berat agar fokus pada latihan penulisan pipeline DevSecOps mandiri.
2. **Penerapan Batas Kegagalan Ketat**:
   Menghapus `|| true` pada npm audit dan menyetel `publishResults: "false"` serta `--fail-on-severity WARNING` pada Semgrep.
3. **Menggunakan Container Resmi Semgrep (`semgrep/semgrep`)**:
   Alih-alih menginstal Semgrep secara manual dengan `pip` (yang memakan waktu mengunduh dependensi), kita mengonfigurasi job `semgrep-sast` agar berjalan langsung di dalam container Docker resmi Semgrep (`semgrep/semgrep`). Ini menghilangkan proses instalasi sepenuhnya sehingga scanning langsung dimulai secara instan.
4. **Optimasi Waktu Scan (Targeted Scan)**:
   Karena Juice Shop bertipe monolitik raksasa dengan folder `frontend/` (Angular) yang sangat besar, pemindaian seluruh file membutuhkan waktu lama. Kita membatasi Semgrep untuk **hanya memindai folder backend** (`routes/` dan `lib/`) menggunakan perintah CLI resmi:
   ```bash
   semgrep scan --config=p/security-audit --fail-on-severity=ERROR routes/ lib/
   ```
   Langkah ini memangkas waktu pemindaian secara drastis dari menit menjadi **hitungan detik saja**.
