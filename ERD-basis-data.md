# ERD Basis Data · Satgas Soedirman

Rancangan basis data untuk aplikasi operasional Satgas Soedirman (PostgreSQL di Supabase).

**Sumber rancangan**

1. Bagan struktur organisasi (acuan garis komando bila bertentangan dengan diagram).
2. Diagram aktivitas T01–T04 dan T06–T09.
3. Proyek `yafidsa`: bentuk data di `src/lib/types.ts`, paket izin di `src/lib/permissions.ts`, mesin status di `src/lib/status.ts`, matriks di `src/lib/penugasan.ts`, dan aturan server contoh di `src/mocks/server/`.
4. Rancangan UI halaman Masuk dan Lupa password.

**Isi:** 41 tabel dalam 6 kelompok (43 setelah penyesuaian implementasi; lihat bagian Penyesuaian saat implementasi). Setiap kelompok punya satu diagram; tabel dari kelompok lain tampil sebagai kotak tanpa kolom.

![Ikhtisar kelompok tabel](erd/0-ikhtisar.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:0-ikhtisar -->
```mermaid
flowchart LR
  A["<b>1 · Data master</b><br/>wilayah<br/>unit<br/>unit_wilayah<br/>personel<br/>kategori_label<br/>masa_penugasan<br/>pengaturan<br/>impor_master<br/>impor_master_galat"]
  B["<b>2 · Peran & hak akses</b><br/>peran<br/>izin<br/>menu<br/>peran_izin<br/>peran_menu<br/>penetapan_peran<br/>penyesuaian_izin<br/>penyesuaian_menu<br/>matriks_penugasan<br/>permintaan_reset_password<br/>log_audit"]
  C["<b>3 · Presensi & kinerja</b><br/>presensi<br/>kinerja_unit"]
  D["<b>4 · Penugasan & laporan</b><br/>tugas<br/>langkah_tugas<br/>penerima_tugas<br/>bacaan_tugas<br/>tembusan_tugas<br/>laporan_tugas<br/>laporan_langkah"]
  E["<b>5 · Aspirasi & koordinasi</b><br/>tiket<br/>permohonan_tugas<br/>catatan_tiket<br/>utas<br/>pesan_utas<br/>baca_utas"]
  F["<b>6 · Commander intent & pendukung</b><br/>commander_intent<br/>ci_sasaran<br/>ci_penerima<br/>notifikasi<br/>lampiran<br/>kunci_idempoten"]
  A -->|"personel dan unit memegang peran"| B
  A -->|"Serdik dan Tim"| C
  A -->|"unit pemberi dan penerima"| D
  A -->|"pelapor, kategori, wilayah"| E
  A -->|"unit sasaran, penerima"| F
  B -->|"matriks penugasan, izin, cakupan"| D
  E -->|"tiket ditindaklanjuti sebagai tugas"| D
  C -->|"bukti"| F
  D -->|"foto dan lampiran"| F
  E -->|"foto dan lampiran"| F
```

</details>

## Konvensi

- **Nama** tabel dan kolom berbahasa Indonesia, huruf kecil dengan garis bawah, mengikuti istilah di `yafidsa`.
- **Kunci utama** bertipe `uuid`, kecuali tabel rujukan yang memakai kode bermakna (`peran.id`, `izin.kode`, `menu.kunci`, `wilayah.kode`).
- **Waktu** disimpan sebagai `timestamptz` (UTC). Tanggal operasional (`presensi.tanggal`) adalah tanggal WIB.
- **Status** disimpan sebagai teks dengan `CHECK`; daftar nilainya mengikuti mesin status `yafidsa` (lihat bagian Status).
- **"Terlambat" bukan status**, melainkan hasil hitung dari tenggat, sama seperti di `yafidsa`.
- **Berkas** (foto, lampiran, PDF) disimpan di Supabase Storage; basis data hanya menyimpan jalurnya.
- **Akun** masuk dikelola Supabase Auth (`auth.users`); `personel.akun_id` menghubungkannya.

## 1 · Data master (T08)

Dikelola Admin lewat input manual atau impor CSV/Excel. Dipakai semua modul.

![Data master](erd/1-data-master.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:1-data-master -->
```mermaid
erDiagram
  wilayah ||--o{ wilayah : "induk"
  unit ||--o{ unit : "induk"
  unit ||--o{ unit_wilayah : "mencakup"
  wilayah ||--o{ unit_wilayah : "dicakup"
  unit ||--o{ personel : "penempatan"
  auth_users ||--o| personel : "akun"
  peran ||--o{ unit : "peran pemegang"
  personel ||--o{ impor_master : "mengimpor"
  impor_master ||--o{ impor_master_galat : "galat per baris"
  personel ||--o{ pengaturan : "mengubah"

  wilayah {
    text kode PK "kode BPS 2, 4, 7, atau 10 digit"
    text nama
    text tingkat "PROVINSI, KAB_KOTA, KECAMATAN, DESA_KEL"
    text induk_kode FK "kosong untuk provinsi"
    boolean aktif
  }
  unit {
    uuid id PK
    text kode UK "mis. KORWIL-3, T-3-11"
    text jenis "pusat, wakil, asisten, katim, korwil, sektor, tim, serdik, pengarah, pelaksana, pokja, eselon1, dewan_ahli"
    text nama
    uuid induk_id FK "pohon komando"
    text peran_pemegang FK "peran yang memegang unit"
    smallint korwil_no "1 sampai 4, diwarisi ke bawah"
    numeric lat "markas, atau titik geofence Tim"
    numeric lon
    integer radius_geofence_m "hanya Tim"
    boolean aktif
  }
  unit_wilayah {
    uuid unit_id PK, FK
    text kode_wilayah PK, FK
  }
  personel {
    uuid id PK
    uuid akun_id FK, UK "auth.users, kosong sampai akun dibuat"
    text username UK
    text nama
    text gender "L, P"
    text unsur "TNI, POLRI, AKADEMISI, SERDIK, LAINNYA"
    text pangkat_gelar "mis. Letkol, Profesor"
    text nomor_identitas UK
    text jabatan
    text telepon
    uuid unit_id FK "unit penempatan"
    uuid penetapan_aktif_id FK "peran aktif yang terakhir dipilih"
    text status "AKTIF, NONAKTIF"
    timestamptz dibuat_pada
  }
  kategori_label {
    uuid id PK
    text kode UK
    text nama "mis. Harga bahan pokok"
    text warna
    boolean berlaku_tiket
    boolean berlaku_laporan
    boolean berlaku_tugas
    boolean aktif
  }
  masa_penugasan {
    uuid id PK
    text nama "mis. Fase 1"
    date mulai
    date selesai
    boolean aktif
  }
  pengaturan {
    text kunci PK "mis. presensi.jam_buka"
    jsonb nilai
    text keterangan
    uuid diubah_oleh FK
    timestamptz diubah_pada
  }
  impor_master {
    uuid id PK
    text jenis_data "wilayah, korwil, sektor, tim, serdik, label"
    text nama_berkas
    integer jumlah_baris
    integer berhasil
    integer gagal
    text status "DIPROSES, SELESAI, GAGAL"
    uuid oleh FK
    timestamptz waktu
  }
  impor_master_galat {
    uuid id PK
    uuid impor_id FK
    integer baris
    text kolom
    text pesan
  }
  auth_users {
    uuid id PK "dikelola Supabase Auth"
  }
  peran {
    text id PK
  }
```

</details>

**Aturan penting**

- **`unit` adalah satu pohon untuk seluruh struktur.** Korwil dan Wakil Komandan sama-sama berinduk ke unit `pusat` (Komandan Satgas), sesuai bagan organisasi. Di `yafidsa` Korwil masih berinduk ke Wakil Komandan; ini perlu diperbaiki di data, bukan di skema.
- **Jenis `katim` disediakan tetapi belum dipakai.** Tiga Katim di bagan (Data & IT, Anev, Konten & Layanan) belum dimasukkan. Bila nanti dibutuhkan, cukup menambah satu baris `peran`, tiga baris `unit` di bawah Wakil Komandan, paket izinnya di `peran_izin`, dan barisnya di `matriks_penugasan`; tidak ada perubahan struktur tabel.
- **Setiap Serdik punya satu unit berjenis `serdik`** di bawah Tim, sama seperti di `yafidsa`. Dengan begitu tugas, tembusan, dan cakupan data selalu merujuk unit, dan berpindah otomatis ke pemegang baru bila personel diganti.
- **Pemegang unit tidak disimpan di `unit`.** Pemegang adalah personel dengan `penetapan_peran` aktif pada unit itu (lihat kelompok 2). Anggota Tim adalah pemegang unit `serdik` di bawahnya.
- **`unit_wilayah`** memetakan cakupan ke kode BPS: Korwil ke provinsi (Korwil II mencakup dua provinsi), Sektor ke kab/kota, Tim ke kecamatan atau desa. Wilayah anak harus berada di dalam wilayah induk.
- **Geofence Tim** = `lat`, `lon`, `radius_geofence_m`. `yafidsa` memakai radius 1.500 m dan akurasi maksimal 50 m.
- **`pengaturan`** menyimpan nilai operasional yang di `yafidsa` masih berupa konstanta: jendela check-in 05.00–09.00 WIB, tenggat verifikasi 2 jam, SLA telaah tiket 24 jam (4 jam bila mendesak), tenggat tanggapan utas 24 jam, tenggat terima tugas 2 jam.
- **`kategori_label`** menggantikan daftar kategori tetap di `yafidsa` (8 kategori aspirasi) dengan data yang bisa dikelola Admin.
- **Validasi impor** (T08): kode BPS sah dan berjenjang, rantai induk utuh, kode label dan identitas personel unik. Galat dicatat per baris di `impor_master_galat`.

## 2 · Peran & hak akses · administrasi sistem (T09, T08)

Hanya Komandan Satgas definitif yang mengubah tabel di kelompok ini. Admin tidak punya akses ke sini.

![Peran dan hak akses](erd/2-peran-hak-akses.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:2-peran-hak-akses -->
```mermaid
erDiagram
  personel ||--o{ penetapan_peran : "memegang"
  peran ||--o{ penetapan_peran : "ditetapkan sebagai"
  unit ||--o{ penetapan_peran : "unit cakupan"
  peran ||--o{ peran_izin : "paket default"
  izin ||--o{ peran_izin : "termasuk"
  peran ||--o{ peran_menu : "menu default"
  menu ||--o{ peran_menu : "termasuk"
  izin ||--o{ menu : "syarat tampil"
  penetapan_peran ||--o{ penyesuaian_izin : "disesuaikan"
  izin ||--o{ penyesuaian_izin : "izin"
  penetapan_peran ||--o{ penyesuaian_menu : "disesuaikan"
  menu ||--o{ penyesuaian_menu : "menu"
  peran ||--o{ matriks_penugasan : "pemberi"
  peran ||--o{ matriks_penugasan : "penerima"
  personel ||--o{ permintaan_reset_password : "memproses"
  personel ||--o{ log_audit : "pelaku"

  peran {
    text id PK "komandan, plh, wakil, asisten, pengarah, pelaksana, korwil, sektor, ketua_tim, serdik, pokja, eselon1, dewan_ahli, admin"
    text label "mis. Komandan Satgas"
    text cakupan_default "semua, korwil, sektor, tim, diri"
    boolean sementara "benar untuk PLH"
    smallint maks_hari "PLH 30"
    smallint maks_pemegang "komandan 1, asisten 5"
    text unsur_wajib "mis. ketua_tim: TNI, POLRI, AKADEMISI"
    text kelompok_nav "serdik, ketua_tim, pimpinan"
    boolean aktif
  }
  izin {
    text kode PK "modul.aksi, mis. presensi.verifikasi"
    text modul "beranda, presensi, ci, tugas, aspirasi, koordinasi, kinerja, analitik, organisasi, master, akses, audit"
    text aksi "lihat, absen, verifikasi, koreksi, tambah, langsung, ubah, nilai, buat, telaah, kirim, kelola, ekspor"
    text keterangan
  }
  menu {
    text kunci PK "mis. presensi, penugasan, akses"
    text label
    text jalur "mis. /presensi"
    text grup "Utama, Operasional, Analitik, Organisasi, Administrasi"
    smallint urutan
    text izin_kode FK "izin yang wajib dimiliki"
  }
  peran_izin {
    text peran_id PK, FK
    text izin_kode PK, FK
    text cakupan "semua, korwil, sektor, tim, diri"
    uuid diubah_oleh FK
    timestamptz diubah_pada
  }
  peran_menu {
    text peran_id PK, FK
    text menu_kunci PK, FK
  }
  penetapan_peran {
    uuid id PK
    uuid personel_id FK
    text peran_id FK
    uuid unit_id FK "unit yang dipegang, dasar cakupan data"
    text cakupan "semua, korwil, sektor, tim, diri"
    timestamptz berlaku_mulai
    timestamptz berlaku_sampai "wajib untuk PLH"
    text status "DIJADWALKAN, AKTIF, BERAKHIR, DICABUT"
    text alasan
    uuid ditetapkan_oleh FK
    timestamptz ditetapkan_pada
    uuid dicabut_oleh FK
    timestamptz dicabut_pada
    text alasan_cabut
  }
  penyesuaian_izin {
    uuid id PK
    uuid penetapan_id FK
    text izin_kode FK
    text efek "IZINKAN, TOLAK"
    text cakupan "menimpa cakupan default"
    uuid oleh FK
    timestamptz pada
  }
  penyesuaian_menu {
    uuid penetapan_id PK, FK
    text menu_kunci PK, FK
    boolean tampil
  }
  matriks_penugasan {
    text peran_pemberi PK, FK
    text peran_penerima PK, FK
    text jalur "BIASA, LANGSUNG"
    boolean wajib_alasan "benar untuk jalur langsung"
    boolean aktif
  }
  permintaan_reset_password {
    uuid id PK
    text username "tidak dirujuk ke personel"
    text status "MENUNGGU, SELESAI, DITOLAK"
    timestamptz dibuat_pada
    uuid diproses_oleh FK
    timestamptz diproses_pada
    text catatan
  }
  log_audit {
    bigint id PK
    timestamptz waktu
    uuid pelaku_id FK "kosong = Sistem"
    text pelaku_nama "salinan saat kejadian"
    text peran_aktif
    text modul
    text aksi "mis. tugas.ack, akses.ditolak, peran.ganti"
    text objek_jenis
    text objek_id
    text nilai_lama
    text nilai_baru
    text alasan
    text perangkat
    inet ip
  }
  personel {
    uuid id PK
  }
  unit {
    uuid id PK
  }
```

</details>

**Cara hak akses dihitung**

1. Pengguna memilih satu `penetapan_peran` berstatus AKTIF sebagai peran aktif (`personel.penetapan_aktif_id`). Pengguna berperan ganda, misalnya Wakil Komandan sekaligus PLH, punya beberapa baris.
2. Izin dasar diambil dari `peran_izin` untuk peran itu: kode izin beserta cakupannya.
3. `penyesuaian_izin` pada penetapan itu menambah (IZINKAN), mencabut (TOLAK), atau mempersempit cakupan izin. `penyesuaian_menu` menampilkan atau menyembunyikan menu.
4. Cakupan data diukur dari `penetapan_peran.unit_id`: `semua` = seluruh Satgas; `korwil`, `sektor`, `tim` = unit itu beserta semua turunannya di pohon `unit`; `diri` = unit itu saja.
5. Menu tampil bila ada di `peran_menu` (setelah penyesuaian) **dan** izin pada `menu.izin_kode` dimiliki.

Aturan ini ditegakkan di server (kebijakan RLS dan fungsi basis data), bukan hanya di tampilan.

**Aturan penting**

- **Hanya pemegang `komandan` aktif** yang boleh menulis ke `penetapan_peran`, `peran_izin`, `peran_menu`, `penyesuaian_*`, dan `matriks_penugasan`. PLH tidak boleh. Percobaan oleh peran lain ditolak dan dicatat sebagai `akses.ditolak`.
- **Paling banyak satu `komandan` AKTIF.** Penggantian Komandan definitif dilakukan di luar aplikasi (T09).
- **Batas dari `peran`:** Asisten paling banyak 5 pemegang aktif; PLH wajib punya `berlaku_sampai` paling lama 30 hari; unsur personel harus sesuai `unsur_wajib`.
- **Kedaluwarsa otomatis:** proses terjadwal mengubah penetapan yang lewat `berlaku_sampai` menjadi BERAKHIR, memberi tahu Komandan, dan mencatat log. Peran lain milik orang itu tetap berlaku.
- **Pencabutan atau penonaktifan akun** mengisi `dicabut_*`, mengubah status menjadi DICABUT, dan mengakhiri sesi Supabase Auth pengguna itu.
- **Personel baru dari Admin** belum punya `penetapan_peran`, jadi belum bisa membuka apa pun sampai Komandan menetapkan perannya (T08).
- **`matriks_penugasan`** menggantikan matriks yang ditulis tetap di `yafidsa/src/lib/penugasan.ts`, sehingga garis komando bisa disesuaikan tanpa mengubah kode. Isi awalnya ada di tabel di bawah.
- **`log_audit` hanya bisa ditambah.** Tidak ada izin ubah atau hapus untuk siapa pun, dan hanya izin `audit.lihat` (Komandan definitif) yang bisa membacanya. Setiap perubahan status di semua modul menulis tepat satu baris.
- **`permintaan_reset_password`** diisi dari halaman Lupa password tanpa masuk. Username sengaja tidak dirujuk ke `personel` supaya halaman itu tidak bisa dipakai menebak akun.

**Paket izin default** (isi awal `peran_izin`, dari `yafidsa/src/lib/permissions.ts`)

| Peran | Cakupan | Izin |
|---|---|---|
| Komandan Satgas | semua | Semua izin kecuali `master.kelola`, `presensi.absen`, `presensi.verifikasi`, `aspirasi.buat` |
| PLH Komandan | semua | Sama dengan Komandan, tanpa `akses.kelola`, `audit.lihat`, `audit.ekspor` |
| Wakil Komandan | semua | `beranda.lihat`, `presensi.lihat`, `ci.lihat`, `tugas.lihat/tambah/nilai`, `aspirasi.lihat`, `kinerja.lihat`, `analitik.lihat`, `organisasi.lihat` |
| Asisten | diri | `beranda.lihat`, `ci.lihat`, `tugas.lihat`, `aspirasi.lihat/buat`, `kinerja.lihat`; `organisasi.lihat` (semua) |
| Ketua Pengarah | semua | `beranda.lihat`, `aspirasi.lihat`, `kinerja.lihat`, `analitik.lihat`, `organisasi.lihat` |
| Ketua Pelaksana | semua | `beranda.lihat`, `tugas.lihat/tambah/nilai`, `aspirasi.lihat`, `koordinasi.lihat`, `organisasi.lihat` |
| Korwil | korwil | `beranda.lihat`, `presensi.lihat`, `ci.lihat`, `tugas.lihat/tambah/nilai`, `aspirasi.lihat`, `kinerja.lihat`, `analitik.lihat`, `organisasi.lihat` |
| Komandan Sektor | sektor | `beranda.lihat`, `presensi.lihat`, `ci.lihat`, `tugas.lihat/tambah/nilai`, `aspirasi.lihat`, `organisasi.lihat` |
| Ketua Tim | tim | `beranda.lihat`, `presensi.lihat/absen/verifikasi`, `ci.lihat`, `tugas.lihat/tambah/nilai`, `aspirasi.lihat/buat`, `organisasi.lihat` |
| Serdik | diri | `beranda.lihat`, `presensi.lihat/absen`, `ci.lihat`, `tugas.lihat`, `aspirasi.lihat/buat`; `organisasi.lihat` (tim) |
| Pokja, Eselon I, Dewan Ahli | diri | `beranda.lihat`, `tugas.lihat`, `aspirasi.lihat`, `koordinasi.lihat/kirim` |
| Admin | semua | `organisasi.lihat`, `master.kelola` |

**Isi awal `matriks_penugasan`** (diputuskan 15 November 2026)

| Pemberi | Penerima | Jalur | Alasan wajib |
|---|---|---|---|
| Komandan Satgas | Wakil Komandan, Korwil | BIASA | Tidak |
| Komandan Satgas | Asisten, Komandan Sektor, Ketua Tim, Serdik | LANGSUNG | Ya |
| Wakil Komandan | Korwil, Asisten | BIASA | Tidak |
| Korwil | Komandan Sektor, Ketua Tim | BIASA | Tidak |
| Komandan Sektor | Ketua Tim | BIASA | Tidak |
| Ketua Tim | Serdik | BIASA | Tidak |
| Ketua Pelaksana | Pokja, Eselon I, Dewan Ahli | BIASA | Tidak |

- **PLH Komandan** memakai baris yang sama dengan Komandan Satgas, seperti di `yafidsa`.
- **Jalur BIASA** berarti penerima adalah bawahan langsung pemberi. Komandan → Korwil termasuk di sini karena Korwil berinduk ke Komandan di bagan.
- **Jalur LANGSUNG** berarti Komandan melewati satu jenjang atau lebih. Mengikuti `yafidsa`, jalur ini mewajibkan alasan dan menembuskan tugas ke setiap jenjang yang dilewati. Kewajiban alasan adalah kolom `wajib_alasan`, jadi bisa dimatikan tanpa mengubah kode.
- **Korwil punya dua pemberi tugas:** Komandan dan Wakil Komandan. Wakil Komandan → Korwil sah karena cakupan Wakil Komandan adalah `semua`, walaupun Korwil bukan turunannya di pohon `unit`.
- **Serdik** tidak punya baris sebagai pemberi: pelaksana akhir.

## 3 · Presensi & kinerja (T01, T07)

![Presensi dan kinerja](erd/3-presensi-kinerja.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:3-presensi-kinerja -->
```mermaid
erDiagram
  personel ||--o{ presensi : "absen"
  unit ||--o{ presensi : "tim saat itu"
  personel ||--o{ presensi : "memverifikasi"
  unit ||--o{ kinerja_unit : "dinilai"

  presensi {
    uuid id PK
    uuid personel_id FK "Serdik"
    uuid tim_id FK "unit Tim pada tanggal itu"
    date tanggal "tanggal WIB, unik per personel"
    text status "MENUNGGU_VERIFIKASI, HADIR, IZIN, SAKIT, TIDAK_HADIR"
    timestamptz waktu "diterima server"
    timestamptz waktu_perangkat "saat foto diambil"
    numeric lat
    numeric lon
    integer akurasi_m
    integer jarak_m "jarak ke titik Tim"
    text foto_path "selfie"
    jsonb alasan_gagal "sebab gagal validasi"
    text pengajuan_jenis "IZIN, SAKIT"
    text pengajuan_alasan
    uuid verifikasi_oleh FK "Ketua Tim"
    timestamptz verifikasi_pada
    text verifikasi_alasan "wajib bila tidak sah"
    uuid koreksi_oleh FK "Komandan"
    timestamptz koreksi_pada
    text koreksi_alasan
    text status_sebelum_koreksi
  }
  kinerja_unit {
    uuid id PK
    uuid unit_id FK "Korwil atau Asisten"
    text periode "HARIAN, MINGGUAN"
    date tanggal "unik per unit dan periode"
    numeric ack_tepat_waktu
    numeric tugas_tepat_waktu
    numeric laporan_tanpa_revisi
    numeric kehadiran
    integer temuan_ditindaklanjuti
    timestamptz dihitung_pada
  }
  personel {
    uuid id PK
  }
  unit {
    uuid id PK
  }
```

</details>

**Aturan penting**

- **Satu baris per Serdik per tanggal** (`personel_id` + `tanggal` unik). Belum ada baris berarti "Belum absen"; status itu tidak disimpan.
- **Lolos validasi langsung HADIR.** Gagal validasi (di luar geofence, akurasi buruk, di luar jendela jam, jam perangkat selisih lebih dari 10 menit) atau pengajuan izin/sakit menjadi MENUNGGU_VERIFIKASI, dengan sebabnya di `alasan_gagal`.
- **Batas jam check-in:** proses terjadwal membuat baris TIDAK_HADIR untuk Serdik yang belum punya baris hari itu.
- **Koreksi** hanya dengan izin `presensi.koreksi` (Komandan) dan alasan wajib. Riwayat lengkap ada di `log_audit`.
- **Rekap berjenjang** Tim → Sektor → Korwil → Satgas dihitung dari `presensi` lewat pohon `unit` (view), tidak disimpan sebagai tabel.
- **`kinerja_unit`** adalah hasil hitung berkala (T07). Modul ini belum dibangun di `yafidsa`; kolomnya diturunkan dari diagram T07.

## 4 · Penugasan & laporan (T02, T03, T07)

![Penugasan dan laporan](erd/4-penugasan-laporan.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:4-penugasan-laporan -->
```mermaid
erDiagram
  unit ||--o{ tugas : "pemberi"
  personel ||--o{ tugas : "membuat"
  penerima_tugas ||--o{ tugas : "induk sub-tugas"
  tiket ||--o{ tugas : "tindak lanjut"
  kategori_label ||--o{ tugas : "label"
  tugas ||--o{ langkah_tugas : "langkah"
  tugas ||--|{ penerima_tugas : "ditujukan ke"
  unit ||--o{ penerima_tugas : "penerima"
  tugas ||--o{ tembusan_tugas : "tembusan"
  unit ||--o{ tembusan_tugas : "ditembuskan ke"
  penerima_tugas ||--o{ bacaan_tugas : "dibaca anggota"
  personel ||--o{ bacaan_tugas : "membaca"
  penerima_tugas ||--o{ laporan_tugas : "laporan per versi"
  laporan_tugas ||--o{ laporan_langkah : "langkah selesai"
  langkah_tugas ||--o{ laporan_langkah : "dicentang"
  wilayah ||--o{ laporan_tugas : "lokasi"

  tugas {
    uuid id PK
    text nomor UK "TGS-2026-11-0001"
    text judul
    text isi
    uuid pemberi_unit_id FK
    text pemberi_peran FK
    uuid dibuat_oleh FK
    boolean oleh_plh
    uuid induk_penerima_id FK "terisi bila sub-tugas"
    boolean langsung "jalur langsung Komandan"
    text alasan_langsung "wajib bila langsung"
    timestamptz mulai
    timestamptz sampai
    timestamptz tenggat_ack
    timestamptz tenggat_laporan "tidak melewati tenggat induk"
    text prioritas "BIASA, PENTING"
    text lokasi
    text bukti_diminta
    uuid tiket_id FK "tindak lanjut tiket aspirasi"
    uuid label_id FK
    timestamptz dibuat_pada
    uuid dibatalkan_oleh FK
    timestamptz dibatalkan_pada
    text alasan_batal
  }
  langkah_tugas {
    uuid id PK
    uuid tugas_id FK
    smallint urutan
    text isi
  }
  penerima_tugas {
    uuid id PK
    uuid tugas_id FK
    uuid unit_id FK "unik per tugas"
    text status "BARU, DITERIMA, DIKERJAKAN, LAPORAN_TERKIRIM, REVISI, SELESAI, DIBATALKAN"
    timestamptz ack_pada
    uuid ack_oleh FK
    boolean atas_nama_tim "Ketua Tim menerima untuk Tim"
    text dikerjakan "sendiri, turunkan"
    boolean siap_rekap "semua sub-tugas selesai"
  }
  tembusan_tugas {
    uuid tugas_id PK, FK
    uuid unit_id PK, FK
    text jenis "OTOMATIS, TAMBAHAN"
  }
  bacaan_tugas {
    uuid penerima_tugas_id PK, FK
    uuid personel_id PK, FK
    timestamptz dibaca_pada
  }
  laporan_tugas {
    uuid id PK
    uuid penerima_tugas_id FK
    smallint versi "unik per penerima"
    text nomor UK "LAP-2026-11-0001, terisi saat dikirim"
    text status "DRAF, TERKIRIM, REVISI, DITERIMA"
    text hasil
    integer jumlah_warga "warga ditemui"
    numeric lat
    numeric lon
    text kode_wilayah FK
    boolean rekap "laporan rekap tugas induk"
    text pdf_path
    uuid disusun_oleh FK
    timestamptz dikirim_pada
    uuid dinilai_oleh FK
    timestamptz dinilai_pada
    boolean memadai
    text catatan_penilaian "wajib bila tidak memadai"
  }
  laporan_langkah {
    uuid laporan_id PK, FK
    uuid langkah_id PK, FK
  }
  unit {
    uuid id PK
  }
  personel {
    uuid id PK
  }
  tiket {
    uuid id PK
  }
  kategori_label {
    uuid id PK
  }
  wilayah {
    text kode PK
  }
```

</details>

**Aturan penting**

- **Satu tugas bisa punya banyak penerima**, dan status dilacak per penerima di `penerima_tugas`, sama seperti di `yafidsa`.
- **Pemberi dan penerima adalah unit**, bukan orang. Siapa pun yang memegang unit itu melihat dan menindaklanjuti tugasnya.
- **Sub-tugas** menunjuk baris `penerima_tugas` induknya lewat `induk_penerima_id`. Rantai penugasan dan progres naik ditelusuri dari sini.
- **Validasi saat dibuat** (T02): pasangan peran pemberi–penerima ada di `matriks_penugasan`, penerima berada di dalam cakupan pemberi, `tenggat_laporan` tidak melewati tenggat induk maupun akhir `masa_penugasan`, dan `alasan_langsung` terisi bila jalur langsung.
- **Tembusan otomatis** disimpan di `tembusan_tugas` dan hanya bisa membaca. Penerimanya dihitung dari pohon `unit`: semua induk di atas unit pemberi, ditambah induk unit penerima yang dilewati pada jalur langsung. Contoh: Wakil Komandan → Korwil ditembuskan ke Komandan; Komandan → Ketua Tim ditembuskan ke Korwil dan Komandan Sektor di atas Tim itu.
- **`bacaan_tugas`** mencatat anggota Tim yang sudah membaca tugas untuk Tim.
- **Laporan dikunci per versi.** Mengirim memberi `nomor` dan PDF; penolakan (catatan wajib) mengubah laporan menjadi REVISI dan pelaksana membuat versi berikutnya. Versi lama tidak diubah.
- **Laporan diterima** mengubah penerima menjadi SELESAI. Bila semua sub-tugas dari satu induk selesai, `siap_rekap` pada penerima induk menyala dan pelaksana induk menyusun laporan `rekap`.
- **Tidak ada data pribadi warga.** Seperti di `yafidsa`, laporan hanya menyimpan jumlah warga yang ditemui.

## 5 · Aspirasi & koordinasi (T04, T06, T07)

![Aspirasi dan koordinasi](erd/5-aspirasi-koordinasi.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:5-aspirasi-koordinasi -->
```mermaid
erDiagram
  personel ||--o{ tiket : "melapor"
  unit ||--o{ tiket : "unit asal"
  unit ||--o{ tiket : "pelaksana"
  kategori_label ||--o{ tiket : "kategori"
  wilayah ||--o{ tiket : "lokasi"
  tiket ||--o{ catatan_tiket : "linimasa"
  personel ||--o{ catatan_tiket : "menulis"
  tiket ||--o{ permohonan_tugas : "diajukan"
  unit ||--o{ permohonan_tugas : "unit tujuan"
  tugas ||--o| permohonan_tugas : "hasil persetujuan"
  tiket ||--o{ utas : "dibahas di"
  tugas ||--o{ utas : "dibahas di"
  unit ||--o{ utas : "pokja"
  personel ||--o{ utas : "membuka"
  utas ||--|{ pesan_utas : "pesan"
  personel ||--o{ pesan_utas : "mengirim"
  utas ||--o{ baca_utas : "dibaca"
  personel ||--o{ baca_utas : "membaca"

  tiket {
    uuid id PK
    text nomor UK "ASP-2026-0001"
    text judul
    text uraian
    uuid kategori_id FK
    integer jumlah_warga
    text nama_warga "opsional, disamarkan"
    text urgensi "BIASA, MENDESAK"
    numeric lat
    numeric lon
    text kode_wilayah FK "desa atau kelurahan"
    uuid unit_asal_id FK "Tim atau unit Asisten pelapor"
    uuid pelapor_id FK
    text pelapor_peran FK
    text status "BARU, DITERIMA, DIKERJAKAN, MENUNGGU_REVIEW, SELESAI, DITUTUP"
    integer sla_jam
    timestamptz tenggat_sla "untuk status yang sedang berjalan"
    text gerbang_wakil "MENUNGGU, DITERUSKAN, INTERNAL, atau kosong"
    boolean minta_rekomendasi
    uuid pelaksana_unit_id FK "Pokja, Eselon I, atau Dewan Ahli"
    text ringkasan_selesai
    timestamptz dibuat_pada
    timestamptz diperbarui_pada
  }
  catatan_tiket {
    uuid id PK
    uuid tiket_id FK
    text jenis "CATATAN, PERMINTAAN_REKOMENDASI, REKOMENDASI, HASIL, PENGEMBALIAN, PENUTUPAN"
    text isi
    uuid oleh FK
    uuid oleh_unit_id FK
    timestamptz waktu
  }
  permohonan_tugas {
    uuid id PK
    uuid tiket_id FK
    uuid unit_tujuan_id FK
    text dasar
    timestamptz tenggat_usulan
    text status "MENUNGGU, DISETUJUI, DITOLAK"
    uuid diajukan_oleh FK "Komandan"
    timestamptz diajukan_pada
    uuid diputuskan_oleh FK "Ketua Pelaksana"
    timestamptz diputuskan_pada
    text instruksi
    text alasan_tolak
    uuid tugas_id FK "tugas yang dibuat saat disetujui"
  }
  utas {
    uuid id PK
    text nomor UK "UTS-2026-11-0001"
    text judul
    text topik "KEBIJAKAN, ASPIRASI, TUGAS"
    uuid pokja_unit_id FK
    uuid dibuat_oleh FK
    text status "MENUNGGU_TANGGAPAN, DITANGGAPI, SELESAI"
    text menunggu_pihak "KOMANDAN, POKJA, atau kosong"
    timestamptz menunggu_sejak
    timestamptz tenggat
    uuid tiket_id FK
    uuid tugas_id FK
    text ringkasan "wajib saat ditutup"
    uuid ditutup_oleh FK
    timestamptz ditutup_pada
    timestamptz dibuat_pada
  }
  pesan_utas {
    uuid id PK
    uuid utas_id FK
    uuid pengirim_id FK
    text peran FK
    text isi
    timestamptz waktu
  }
  baca_utas {
    uuid utas_id PK, FK
    uuid personel_id PK, FK
    timestamptz dibaca_sampai
  }
  personel {
    uuid id PK
  }
  unit {
    uuid id PK
  }
  tugas {
    uuid id PK
  }
  kategori_label {
    uuid id PK
  }
  wilayah {
    text kode PK
  }
```

</details>

**Aturan penting**

- **Tiket dibuat Serdik, Ketua Tim, atau Asisten.** Temuan Asisten masuk dengan `gerbang_wakil = MENUNGGU`; Wakil Komandan memutuskan DITERUSKAN (menjadi tiket untuk Komandan) atau INTERNAL (T07).
- **Hanya Komandan (atau PLH)** yang menelaah, memilih jalur tindak lanjut, dan menyelesaikan tiket.
- **Dua jalur tindak lanjut** (T04): koordinasi langsung dengan Pokja lewat `utas`, atau `permohonan_tugas` ke Ketua Pelaksana untuk Eselon I, Dewan Ahli, atau K/L. Permohonan yang ditolak tetap tersimpan, dan Komandan bisa mengajukan yang baru.
- **Persetujuan permohonan** membuat `tugas` untuk unit tujuan, mengisi `pelaksana_unit_id`, dan mengubah tiket menjadi DIKERJAKAN.
- **`catatan_tiket`** menyatukan rekomendasi Dewan Ahli, hasil tindak lanjut, catatan pengembalian, dan alasan penutupan yang di `yafidsa` tersebar di beberapa daftar.
- **SLA dipantau di setiap status** lewat `tenggat_sla`; lewat tenggat memicu pengingat ke penanggung jawab serta Komandan dan Wakil Komandan.
- **Utas** hanya antara Komandan dan satu Pokja. Ketua Pelaksana membaca semua utas lewat izin `koordinasi.lihat`, jadi tembusannya tidak perlu tabel sendiri.
- **Menutup utas** mewajibkan `ringkasan`; bila utas terkait tiket, ringkasan itu ditambahkan sebagai `catatan_tiket` berjenis HASIL.
- **`nama_warga`** adalah satu-satunya data pribadi warga di skema ini; tampil hanya untuk peran yang berhak.

## 6 · Commander intent & pendukung

![Commander intent dan pendukung](erd/6-commander-intent-pendukung.png)

<details>
<summary>Sumber diagram (Mermaid)</summary>

<!-- erd:6-commander-intent-pendukung -->
```mermaid
erDiagram
  personel ||--o{ commander_intent : "mengirim"
  commander_intent ||--|{ ci_sasaran : "sasaran"
  unit ||--o{ ci_sasaran : "disasar"
  commander_intent ||--o{ ci_penerima : "penerima"
  personel ||--o{ ci_penerima : "menerima"
  personel ||--o{ notifikasi : "menerima"
  personel ||--o{ lampiran : "mengunggah"
  tugas ||--o{ lampiran : "lampiran"
  laporan_tugas ||--o{ lampiran : "foto"
  tiket ||--o{ lampiran : "foto"
  pesan_utas ||--o{ lampiran : "lampiran"
  presensi ||--o{ lampiran : "bukti izin atau sakit"
  personel ||--o{ kunci_idempoten : "mengirim"

  commander_intent {
    uuid id PK
    text nomor UK "CI-2026-11-0001"
    text jenis "PERINGATAN_DINI, COMMANDER_INTENT, PEMBERITAHUAN"
    text tingkat "WASPADA, SIAGA, hanya peringatan dini"
    text judul "maksimal 80 karakter"
    text isi "maksimal 500 karakter"
    text tindakan
    text status "DRAF, AKTIF, KEDALUWARSA, DITARIK"
    timestamptz berlaku_sampai
    uuid pengirim_id FK
    boolean oleh_plh
    timestamptz dibuat_pada
    timestamptz dikirim_pada
    text alasan_tarik
  }
  ci_sasaran {
    uuid ci_id PK, FK
    uuid unit_id PK, FK
  }
  ci_penerima {
    uuid ci_id PK, FK
    uuid personel_id PK, FK
    text status "TERKIRIM, DITERIMA_PERANGKAT, DIBACA, DIKONFIRMASI, GAGAL"
    timestamptz waktu
  }
  notifikasi {
    uuid id PK
    uuid personel_id FK
    text jenis "tugas, laporan, presensi, aspirasi, koordinasi, ci, sistem"
    text judul
    text isi
    text tautan
    text prioritas "tinggi, atau kosong"
    timestamptz waktu
    timestamptz dibaca_pada
  }
  lampiran {
    uuid id PK
    text path "berkas di Storage"
    text nama_berkas
    text tipe_mime
    integer ukuran_byte
    uuid tugas_id FK
    uuid laporan_id FK
    uuid tiket_id FK
    uuid pesan_utas_id FK
    uuid presensi_id FK "tepat satu dari lima yang terisi"
    uuid diunggah_oleh FK
    timestamptz diunggah_pada
  }
  kunci_idempoten {
    text kunci PK "dari antrean offline"
    uuid personel_id FK
    text jalur
    jsonb hasil
    timestamptz dibuat_pada
  }
  personel {
    uuid id PK
  }
  unit {
    uuid id PK
  }
  tugas {
    uuid id PK
  }
  laporan_tugas {
    uuid id PK
  }
  tiket {
    uuid id PK
  }
  pesan_utas {
    uuid id PK
  }
  presensi {
    uuid id PK
  }
```

</details>

**Aturan penting**

- **Commander intent** hanya dikirim Komandan atau PLH. Modul ini ada di `yafidsa` tetapi tidak ada di diagram T01–T09.
- **Penerima dibekukan saat dikirim:** semua personel di bawah unit sasaran disalin ke `ci_penerima`, lalu statusnya naik dari TERKIRIM sampai DIKONFIRMASI.
- **Pesan Siaga atau lebih dari 50 penerima** wajib dikonfirmasi pengirim sebelum terkirim. Menarik pesan mewajibkan alasan.
- **`lampiran`** dipakai bersama; tepat satu kolom rujukan yang terisi. Laporan dan tiket paling banyak 6 foto.
- **`kunci_idempoten`** mencegah kiriman ganda dari antrean offline (check-in, pengajuan izin, terima tugas, laporan, tiket).
- **`notifikasi`** saat ini hanya di dalam aplikasi, sesuai `yafidsa`.

## Status

| Tabel | Nilai status | Diagram |
|---|---|---|
| `presensi` | MENUNGGU_VERIFIKASI → HADIR, IZIN, SAKIT, atau TIDAK_HADIR; koreksi antar keempatnya oleh Komandan | T01 |
| `penerima_tugas` | BARU → DITERIMA → DIKERJAKAN → LAPORAN_TERKIRIM → SELESAI atau REVISI; DIBATALKAN | T02, T03 |
| `laporan_tugas` | DRAF → TERKIRIM → DITERIMA atau REVISI | T03 |
| `tiket` | BARU → DITERIMA → DIKERJAKAN → MENUNGGU_REVIEW → SELESAI; DITERIMA → DITUTUP | T04 |
| `permohonan_tugas` | MENUNGGU → DISETUJUI atau DITOLAK | T04 |
| `utas` | MENUNGGU_TANGGAPAN ⇄ DITANGGAPI → SELESAI | T06 |
| `commander_intent` | DRAF → AKTIF → KEDALUWARSA atau DITARIK | `yafidsa` |
| `penetapan_peran` | DIJADWALKAN → AKTIF → BERAKHIR atau DICABUT | T09 |

## Pemetaan diagram ke tabel

| Diagram | Tabel utama |
|---|---|
| T01 Absensi Serdik | `presensi`, `unit` (geofence), `lampiran` |
| T02 Penugasan berjenjang | `tugas`, `penerima_tugas`, `tembusan_tugas`, `langkah_tugas`, `matriks_penugasan` |
| T03 Laporan tugas | `laporan_tugas`, `laporan_langkah`, `lampiran` |
| T04 Tiket aspirasi | `tiket`, `catatan_tiket`, `permohonan_tugas` |
| T06 Koordinasi Pokja | `utas`, `pesan_utas`, `baca_utas` |
| T07 Wakil Komandan & Asisten | `tiket.gerbang_wakil`, `kinerja_unit`, sub-tugas di `tugas` |
| T08 Dashboard & data master | `wilayah`, `unit`, `unit_wilayah`, `personel`, `kategori_label`, `impor_master`, `menu`, `peran_menu` |
| T09 Peran & hak akses | `peran`, `izin`, `peran_izin`, `penetapan_peran`, `penyesuaian_izin`, `penyesuaian_menu`, `log_audit` |
| UI Lupa password | `permintaan_reset_password` |

## Penyesuaian saat implementasi

Skema yang diterapkan (`dashboard-satgas-dialog/supabase/migrations/`) berbeda dari diagram di atas pada butir berikut. Diagram belum digambar ulang.

- **Tabel `berkas` (baru).** Isi foto dan lampiran disimpan di basis data (`bytea`), bukan di Supabase Storage, karena backend belum memegang secret key Supabase dan akses berkas harus sepenuhnya dikendalikan backend. Kolom `path` di `lampiran` menjadi `berkas_id`; `presensi.foto_path` menjadi `foto_berkas_id`; `laporan_tugas.pdf_path` menjadi `pdf_berkas_id`. Pindah ke Storage cukup mengganti `src/server/berkas.ts`.
- **Tabel `penomoran` (baru).** Penghitung nomor dokumen per jenis dan periode, dipakai fungsi `nomor_berikut()`.
- **`unit.leluhur` (baru).** Daftar id induk sampai akar, diisi pemicu; dasar perhitungan cakupan data.
- **Salinan nama.** `catatan_tiket.oleh_nama`, `pesan_utas.pengirim_nama`, dan `commander_intent.pengirim_nama` menyimpan nama saat kejadian, supaya riwayat tidak berubah bila personel diganti.
- **Kolom tambahan kecil:** `tiket.nama_lokasi`, `lampiran.urutan`, `penerima_tugas.diperbarui_pada`, `peran.urutan`, `presensi.dibuat_pada`, `laporan_tugas.dibuat_pada`.
- **`peran.unsur_wajib`** bertipe `text[]`, bukan `text`.
- **Pengamanan.** Semua tabel ber-RLS tanpa policy dan hak peran `anon`/`authenticated` dicabut. Seluruh akses data lewat backend, yang menegakkan izin dan cakupan di kode (bukan lewat kebijakan RLS seperti tertulis di kelompok 2).

## Keputusan

Diputuskan pada 15 November 2026.

1. **Wakil Komandan tetap boleh menugaskan Korwil**, di samping Komandan. Lihat isi awal `matriks_penugasan` di kelompok 2.
2. **Komandan boleh menugaskan langsung ke semua jenjang** rantai komando: Wakil Komandan, Korwil, Asisten, Komandan Sektor, Ketua Tim, dan Serdik.
3. **Tiga Katim belum dimasukkan**, tetapi skema sudah menyediakan tempatnya (jenis unit `katim`; peran, izin, dan matriks berupa data).
4. **Penyesuaian izin disediakan di dua tingkat** sambil menunggu arahan: paket default per peran (`peran_izin`, `peran_menu`) dan penyesuaian per personel (`penyesuaian_izin`, `penyesuaian_menu`). Bila kelak hanya satu yang dipakai, tabel yang lain bisa dibuang tanpa memengaruhi modul lain.

## Masih terbuka

1. **Alasan wajib pada jalur langsung.** Mengikuti `yafidsa`, penugasan Komandan yang melewati jenjang mewajibkan alasan. Ini belum dikonfirmasi; mengubahnya cukup lewat kolom `wajib_alasan`.
2. **Label untuk laporan.** T08 menyebut tipe label berlaku untuk tiket, laporan, dan tugas, tetapi `yafidsa` hanya memakainya di tiket. Skema ini memberi `label_id` opsional di `tugas`; laporan belum diberi label.
3. **`kinerja_unit`** diturunkan dari T07 saja. Rumus tiap angka perlu ditetapkan sebelum modul Kinerja dibangun.
