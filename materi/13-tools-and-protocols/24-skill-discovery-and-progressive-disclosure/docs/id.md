# Penemuan Skill dan Pengungkapan Bertahap

> Sebuah skill menjadi berguna sebelum isinya dimuat. Nama dan deskripsinya mendapat tempat di katalog; berkasnya yang lebih dalam mendapat konteks hanya bila tugas mencapainya.

**Jenis:** Build
**Bahasa:** Python (stdlib), TypeScript
**Prasyarat:** Fase 13 · 22 (Agent Skills: Kontrak Portabel dan Batas Runtime)
**Waktu:** ~105 menit

## Tujuan Pembelajaran

- Membangun pipeline penemuan sistem berkas yang memisahkan cakupan, validasi, kebijakan tabrakan, dan penerbitan katalog.
- Menjelaskan tiga tingkat pengungkapan: metadata katalog, instruksi aktif, dan sumber daya spesifik tugas.
- Merancang referensi agar agen dapat mencapai detail yang dibutuhkan secara langsung tanpa memuat seluruh paket.
- Menganggarkan ruang katalog secara terpisah dari konteks skill aktif.
- Menolak path traversal dan pelarian symlink ketika sebuah skill membaca sumber dayanya sendiri.

## Masalah

Agen Anda memiliki 200 skill terpasang. Memuat setiap `SKILL.md`, berkas referensi, skrip, dan templat saat sesi dimulai akan mengubur tugas saat ini di bawah prosedur yang tidak relevan. Tidak memuat apa pun akan memaksa pengguna mengingat jalur sistem berkas yang persis.

Kompromi yang biasa adalah katalog: tampilkan kepada model identitas ringkas dan deskripsi perutean untuk setiap skill yang memenuhi syarat, lalu muat isi lengkapnya hanya setelah seleksi. Ini menimbulkan dua masalah rekayasa baru.

Pertama, penemuan bukan sekadar pencarian berkas rekursif. Skill dapat berada pada cakupan proyek, pengguna, administrator, plugin, atau bawaan. Dua paket dapat memiliki nama yang sama. Symlink dapat menunjuk ke luar root yang dipercaya. Paket yang malformed dapat memakan ruang katalog atau menjadi mustahil dipanggil.

Kedua, pengungkapan bertahap dapat berubah menjadi kebingungan bertahap. Bila `SKILL.md` berkata "bacalah panduan yang relevan" dan paket itu memuat dua belas panduan, model harus menebak. Bila setiap panduan menunjuk ke tiga berkas lagi, pemuatan menjadi penjelajahan graf tanpa batas.

Runtime yang baik membuat penemuan deterministik dan pengungkapan disengaja.

## Konsep

### Penemuan adalah pipeline kompilator

Perlakukan sistem berkas sebagai masukan sumber. Jangan menerbitkan jalur mentah langsung kepada model.

```figure
skill-discovery-pipeline
```

Setiap tahap sebaiknya menghasilkan data terstruktur dan kegagalan terstruktur. Log penemuan sebaiknya menjawab:

- Root mana yang dicari?
- Kandidat mana yang ditemukan?
- Kandidat mana yang ditolak, dan mengapa?
- Paket mana yang menang dalam tabrakan?
- Entri katalog mana yang dipendekkan atau dihilangkan karena anggaran?

Tanpa bukti itu, pernyataan "model tidak memakai skill saya" hampir mustahil didiagnosis.

### Cakupan adalah kebijakan runtime

Spesifikasi portabel mendefinisikan paket skill, bukan satu jalur pemasangan universal atau urutan presedensi. Host yang menentukan tempat pencariannya.

Runtime umum mungkin memakai cakupan-cakupan ini:

| Cakupan | Contoh root | Kepemilikan yang dimaksud |
|---|---|---|
| Ruang kerja | `<repo>/.agents/skills/` | Pemelihara proyek |
| Pengguna | `<user-data>/skills/` | Satu pengembang |
| Administrator | `<system>/skills/` | Kebijakan mesin atau organisasi |
| Plugin | Bundel plugin bertanda tangan | Penerbit dan pemasang plugin |
| Bawaan | Paket runtime | Vendor runtime |

Per Agustus 2026, Codex mendokumentasikan penemuan proyek dari `$CWD/.agents/skills` melalui direktori induk hingga root repositori, ditambah lokasi pengguna, administrator, dan bawaan. Codex mendukung direktori skill ber-symlink. Nama yang ganda boleh muncul keduanya alih-alih digabung. Itu adalah perilaku Codex, bukan kewajiban `SKILL.md`; verifikasi [dokumentasi skill Codex](https://learn.chatgpt.com/docs/build-skills) terkini saat menulis adapter.

Jangan pernah mengarang presedensi dari nama direktori. Nyatakan itu sebagai kebijakan dan uji. Laboratorium pelajaran ini memakai peringkat integer eksplisit untuk setiap `Scope` sehingga himpunan kandidat yang sama selalu terselesaikan dengan cara yang sama.

### Tabrakan memerlukan identitas lebih dari `name`

Dua paket bernama `release-readiness` sah-sah saja. Salah satunya bisa berupa pengganti di ruang kerja dan yang lain default pengguna. Karena itu, entri katalog memerlukan setidaknya:

```json
{
  "name": "release-readiness",
  "description": "Periksa kandidat rilis untuk repositori ini.",
  "scope": "workspace",
  "source": "/repo/.agents/skills/release-readiness",
  "selected": true
}
```

Kebijakan tabrakan yang umum meliputi:

| Kebijakan | Manfaat | Risiko |
|---|---|---|
| Simpan semua kandidat | Tidak ada yang disembunyikan | Model melihat nama yang ambigu |
| Cakupan presedensi tertinggi yang menang | Pemanggilan sederhana | Paket lokal dapat menimpa paket yang dipercaya |
| Tolak duplikat | Tidak ada penimpaan diam-diam | Pengganti yang sah berhenti berfungsi |
| Kualifikasi nama menurut sumber | Identitas eksplisit | Nama yang dilihat pengguna menjadi lebih panjang |

Pilih satu kebijakan untuk host. Pertahankan kandidat yang ditolak atau ditimpa dalam diagnostik meskipun kandidat itu tidak muncul di katalog model.

### Tiga tingkat pengungkapan

Spesifikasi Agent Skills menggambarkan pemuatan bertahap. Kuncinya adalah setiap tingkat memiliki tujuan yang berbeda.

```figure
skill-disclosure-levels
```

#### Tingkat 1: metadata katalog

Model memerlukan informasi yang cukup untuk membedakan skill itu dari skill di sekitarnya. Spesifikasi memperkirakan sekitar 100 token per entri katalog, tetapi serialisasi dan tokenisasi aktual menjadi wewenang host.

Deskripsi yang berguna memiliki dua klausa:

```yaml
description: Validasi kandidat rilis dan hasilkan laporan kesiapan. Pakai ketika pengguna bertanya apakah sebuah versi, tag, atau paket siap diterbitkan.
```

Klausa pertama menyatakan kapabilitas. Klausa kedua menyatakan batas pemicu. Pelajaran 25 mengevaluasi batas ini dengan prompt positif dan hampir-terlewat.

#### Tingkat 2: instruksi aktif

Setelah aktivasi, isi skill sebaiknya berfungsi sebagai peta sekaligus prosedur. Spesifikasi menyarankan menjaga `SKILL.md` di bawah 500 baris. Itu adalah sinyal desain, bukan target yang harus dipenuhi.

Isi skill sebaiknya memuat:

- batas tugas;
- alur kerja bawaan;
- kondisi cabang;
- penunjuk langsung ke berkas yang lebih dalam;
- kontrak alat dan skrip;
- perilaku saat gagal dan saat berhenti;
- keluaran yang diharapkan dan verifikasinya.

Jangan memindahkan alur kerja pusat ke referensi hanya agar berkas entri menjadi pendek. Aktivasi harus memberi model konteks yang cukup untuk memulai dengan benar.

#### Tingkat 3: sumber daya pendukung

Referensi menyediakan prosa atau data. Skrip menyediakan komputasi deterministik. Aset disalin, diisi, atau diubah menjadi hasil kerja, bukan diperlakukan sebagai instruksi.

| Direktori | Apakah model membacanya? | Apakah model menjalankannya? | Isi umum |
|---|:---:|:---:|---|
| `references/` | Ya, saat dibutuhkan | Tidak | skema, kebijakan, panduan domain |
| `scripts/` | Boleh memeriksanya | Lewat alat yang diizinkan | validator, konverter, pengumpul |
| `assets/` | Hanya bila berguna | Tidak | templat, fixture, gambar, berkas awal |

Nama-nama ini adalah konvensi, bukan kapabilitas ajaib. Host tetap memerlukan akses berkas dan alat eksekusi.

### Referensi spesifik cabang mengalahkan tumpukan topik

Tulis berkas entri sebagai peta keputusan:

```markdown
## Pilih jalurnya

- Untuk paket Python, baca `references/python-release.md`.
- Untuk image kontainer, baca `references/container-release.md`.
- Untuk rilis yang hanya berisi dokumentasi, baca `references/docs-release.md`.
- Bila rilis menggabungkan beberapa jenis artefak, baca hanya panduan untuk artefak itu.
```

Ini memberi setiap referensi kondisi pemuatan yang dapat diamati. Kalimat "baca `references/` untuk detail lebih lanjut" tidak memberi kondisi seperti itu.

Jaga agar graf referensi tetap dangkal. Panduan resmi menyarankan tautan langsung dari `SKILL.md` dan menghindari rantai panjang. Satu lompatan membuat keterjangkauan dapat diuji dan mengurangi peluang sebuah batasan penting tidak pernah masuk ke konteks.

```figure
skill-reference-map
```

### Anggaran katalog dan konteks aktif adalah anggaran yang berbeda

Misalkan `c_i` adalah biaya katalog terserialisasi dari skill `i`, `B_c` anggaran katalog, `b_j` biaya isi aktif, dan `r_k` sumber daya yang benar-benar dimuat.

```text
catalog_cost = sum(c_i for every published skill)
active_cost = sum(b_j for every activated skill) + sum(r_k for every disclosed resource)
```

Mengurangi satu anggaran tidak otomatis mengurangi anggaran lainnya. Deskripsi pendek dapat menghemat ruang katalog sementara isi aktif sepanjang 900 baris tetap membanjiri tugas. Memecah isi menjadi referensi dapat mengurangi biaya aktif hanya bila runtime dan instruksi benar-benar menghindari pemuatan cabang yang tidak relevan.

Codex saat ini menganggarkan daftar skill awal sebesar 2 persen dari jendela konteks ketika ukuran jendela konteks diketahui. Nilai 8.000 karakter hanya menjadi fallback ketika ukuran itu tidak diketahui; nilai itu bukan batas kedua yang digabungkan dengan aturan 2 persen. Bila katalog melebihi anggaran yang berlaku, deskripsi dapat dipendekkan atau dihilangkan. Perlakukan angka-angka itu sebagai kebijakan Codex terkini, bukan sifat dari standar Agent Skills.

### Jalur sumber daya adalah batas kepercayaan

Skill sebaiknya hanya membaca berkas di dalam paketnya. Pemeriksaan awalan string literal tidak cukup:

```text
references/../../../../.ssh/config
references/external-link -> /private/company-secrets
```

Selesaikan root paket dan kandidat dengan semantika sistem berkas, tolak masukan absolut, dan verifikasi bahwa kandidat yang telah diselesaikan tetap berada di bawah root yang telah diselesaikan. Putuskan apakah symlink diizinkan sebelum penemuan. Bila diizinkan, periksa target yang telah diselesaikan setiap kali.

```figure
skill-resource-containment
```

Keterkandungan jalur tidak menetapkan kepercayaan isi. Referensi yang valid di dalam paket tetap dapat memuat instruksi berbahaya. Pelajaran 26 menangani ancaman itu.

### Pemuatan harus dapat diamati

Catat peristiwa pengungkapan tanpa mencatat rahasia:

```json
{
  "event": "skill.resource.loaded",
  "skill": "release-readiness",
  "resource": "references/python-release.md",
  "reason": "candidate contains pyproject.toml",
  "bytes": 2840
}
```

Alasan mengubah pilihan konteks menjadi bukti yang dapat ditinjau. Alasan itu juga membantu mengidentifikasi instruksi yang menyebabkan agen memuat setiap berkas "jaga-jaga".

## Bangun

`code/main.py` membangun mesin penemuan dan pengungkapan yang deterministik.

Permukaan penemuan meliputi:

- `Scope` untuk metadata sumber dan presedensi;
- `SkillCandidate` untuk kandidat sistem berkas yang belum divalidasi;
- `discover_scope(scope)` untuk mencantumkan direktori skill langsung;
- `resolve_collisions(candidates, precedence)` untuk menerapkan satu kebijakan yang dinyatakan;
- `CatalogEntry` dan `build_catalog(...)` untuk menerbitkan metadata yang terbatas;
- `CatalogBudget` untuk menghitung entri terserialisasi tanpa mengira karakter adalah token universal.

Permukaan pengungkapan meliputi:

- `load_skill_body(entry, ...)` untuk aktivasi Tingkat 2;
- `validate_reference(skill_dir, reference)` untuk keterkandungan jalur;
- `load_reference(...)` untuk pembacaan Tingkat 3 yang terbatas.

Jalankan laboratorium:

```bash
cd "$(git rev-parse --show-toplevel)"
cd phases/13-tools-and-protocols/24-skill-discovery-and-progressive-disclosure
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover -s code/tests -v
```

Blok ini memerlukan klon lokal dan menyelesaikan root repositori dari direktori kerja mana pun di dalam klon itu.

Demo membuat cakupan proyek dan pengguna sementara, menyisipkan tabrakan, membangun katalog di bawah anggaran yang sengaja kecil, mengaktifkan satu skill, dan mencoba pembacaan referensi yang valid sekaligus pelarian traversal. Tidak ada berkas permanen yang dipasang.

### Mengapa penemuan bersifat dangkal

`discover_scope` memeriksa direktori anak langsung untuk menemukan `SKILL.md`. Fungsi ini tidak memperlakukan setiap `SKILL.md` bersarang sebagai paket terpisah secara rekursif. Ini menjaga batas paket dan menghindari penerbitan tidak sengaja atas contoh atau fixture di dalam skill yang terpasang.

### Mengapa laboratorium tidak mengurai YAML sembarangan

Laboratorium mendukung frontmatter skalar yang dibutuhkan katalognya. Runtime produksi sebaiknya memakai parser YAML aman dengan skema eksplisit, batas ukuran, dan konstruksi objek kustom yang dinonaktifkan. "Stdlib saja" adalah batasan pengajaran, bukan izin untuk diam-diam menciptakan dialek YAML yang setengah jadi.

## Pakai

Terapkan daftar periksa ini pada adapter penemuan mana pun:

1. Cantumkan setiap root yang dikonfigurasi dan siapa yang dapat menulis ke sana.
2. Nyatakan apakah paket ber-symlink diizinkan.
3. Validasi nama paket, nama direktori, metadata wajib, dan ukuran isi entri.
4. Pertahankan sumber dan cakupan dalam identitas internal.
5. Nyatakan dan uji perilaku nama ganda.
6. Ukur katalog terserialisasi persis yang dikirim ke model.
7. Catat mengapa sebuah isi atau sumber daya dimuat.
8. Jaga agar pembacaan sumber daya tetap berada di dalam root paket yang telah diselesaikan.
9. Gagal dengan jelas bila berkas yang dirujuk tidak ada.
10. Bangun ulang katalog bila pemasangan atau kebijakan berubah.

## Kirim

Pelajaran ini menghasilkan bundel `skill-catalog-builder`. Bundel itu memindai root yang diurutkan eksplisit, menolak berkas entri ber-symlink dan ketidaksesuaian nama-direktori, menyelesaikan tabrakan lintas cakupan, menolak duplikat berpresedensi sama, dan memasukkan metadata yang dipilih ke dalam anggaran entri, deskripsi, dan karakter terserialisasi yang dinyatakan.

Laporan JSON-nya memuat entri yang dipilih, kandidat yang ditimpa, entri yang dihilangkan, error validasi, presedensi, dan penggunaan anggaran. Pemuatan isi dan referensi tetap menjadi operasi runtime terpisah, sehingga pembangun katalog tidak mengeksekusi skrip maupun memasukkan seluruh paket ke dalam konteks.

## Latihan

1. Tambahkan cakupan plugin dan tempatkan di antara presedensi pengguna dan bawaan. Buktikan hasil tabrakan dengan sebuah uji.
2. Ubah kebijakan tabrakan dari presedensi tertinggi menjadi nama berkualifikasi. Pertahankan kedua entri di katalog.
3. Tambahkan batas ukuran byte pada `load_reference`. Uji berkas yang tepat berada di batas dan berkas yang satu byte di atasnya.
4. Buat dua deskripsi yang terdengar hampir sama. Tulis ulang agar batas pemicunya tidak tumpang tindih.
5. Tambahkan manifest yang memuat hash untuk setiap referensi dan skrip. Deteksi sumber daya yang diubah sebelum memuatnya.
6. Instrumenkan demo agar melaporkan jumlah byte Tingkat 1, Tingkat 2, dan Tingkat 3 secara terpisah.

## Istilah Kunci

| Istilah | Kata orang | Arti sebenarnya |
|---|---|---|
| Penemuan skill | "Temukan setiap SKILL.md" | Cari cakupan yang dikonfigurasi, validasi paket, lampirkan provenansi, dan terapkan kebijakan |
| Katalog skill | "Daftar skill yang terpasang" | Metadata perutean ringkas yang terlihat model untuk paket yang memenuhi syarat |
| Kebijakan tabrakan | "Duplikat mana yang menang" | Aturan yang dinyatakan untuk kandidat bernama sama dari sumber berbeda |
| Pengungkapan bertahap | "Pemuatan malas" | Penerimaan konteks bertahap dari katalog ke isi, lalu ke sumber daya spesifik cabang |
| Graf referensi | "Berkas yang ditautkan skill" | Struktur sumber daya yang dapat dicapai beserta kondisi pemuatannya |
| Keterkandungan jalur | "Tetap di dalam folder" | Verifikasi bahwa target sumber daya yang diselesaikan tetap berada di dalam root paket yang diselesaikan |

## Bacaan Lebih Lanjut

- [Spesifikasi Agent Skills](https://agentskills.io/specification) untuk bentuk paket dan tingkat pengungkapan bertahap.
- [Mengoptimalkan deskripsi skill](https://agentskills.io/skill-creation/optimizing-descriptions) untuk metadata perutean katalog.
- [Praktik terbaik Agent Skills](https://agentskills.io/skill-creation/best-practices) untuk referensi langsung dan ukuran berkas entri.
- [OpenAI: Membangun skill](https://learn.chatgpt.com/docs/build-skills) untuk cakupan penemuan dan batas katalog Codex terkini.
