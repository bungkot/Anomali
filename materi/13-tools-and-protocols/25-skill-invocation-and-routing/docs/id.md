# Pemanggilan dan Perutean Skill

> Pemanggilan adalah keputusan wewenang yang diikuti keputusan relevansi. Deskripsi yang baik membantu model memilih; kebijakan yang baik menentukan apakah pilihan itu diizinkan.

**Jenis:** Build
**Bahasa:** Python (stdlib), TypeScript
**Prasyarat:** Fase 13 · 24 (Penemuan Skill dan Pengungkapan Bertahap)
**Waktu:** ~105 menit

## Tujuan Pembelajaran

- Membedakan pemanggilan eksplisit oleh pengguna, pemanggilan implisit oleh model, pemanggilan oleh aplikasi, dan pemanggilan skill-ke-skill.
- Memodelkan visibilitas manusia dan kelayakan model sebagai dimensi kebijakan yang independen.
- Menulis deskripsi perutean dengan pemicu positif dan batas hampir-terlewat.
- Memisahkan kelayakan, seleksi, aktivasi, pengikatan argumen, dan eksekusi dalam jejak serta uji.
- Menyesuaikan kolom pemanggilan spesifik runtime tanpa menyajikannya sebagai frontmatter portabel.

## Masalah

Anda memasang skill `database-migration`. Pengguna dapat menjalankannya berdasarkan nama, tetapi model juga melihat deskripsinya dan memilihnya ketika seseorang mengajukan pertanyaan umum tentang basis data. Skill itu lalu mengusulkan perubahan skema untuk tugas yang hanya memerlukan penjelasan.

Anda menambahkan `user-invocable: false`, berharap menghalangi orang menjalankannya secara manual. Di runtime lain, kolom itu diabaikan. Anda menambahkan `disable-model-invocation: true`, berharap skill itu hilang seluruhnya. Pada runtime yang memahaminya, pengguna masih dapat memanggilnya secara eksplisit.

Tidak ada yang salah dengan nama kolom itu. Yang salah adalah model Anda. "Pengguna dapat melihatnya," "model dapat memilihnya," "aplikasi dapat memuatnya lebih awal," dan "alat di dalamnya dapat dieksekusi" adalah fakta-fakta yang terpisah. Satu boolean bernama `invocable` tidak dapat menyatakan semuanya.

Perutean memiliki mode kegagalan kedua. Bila deskripsi kabur, beberapa skill menjadi mungkin. Bila deskripsi disesaki kata kunci, tugas yang tidak terkait memicunya. Katalog adalah antarmuka probabilistik: cukup ringkas untuk muat, cukup spesifik untuk merutekan.

## Konsep

### Lima kanal dapat memulai siklus hidup

| Pelaku | Bentuk pemanggilan | Penggunaan umum | Risiko utama |
|---|---|---|---|
| Pengguna manusia | Menyebut nama skill di UI atau prompt | Seleksi alur kerja yang disengaja | Pengguna mengira tersedia atau berwenang, padahal host tidak memberikannya |
| Model atau agen otonom | Memilih entri katalog dari konteks tugas | Prosedur pakar otomatis | Perutean positif palsu |
| Aplikasi | Mengaktifkan atau memuat lebih awal sebuah skill lewat kode runtime | Alur kerja produk yang tetap | Kopling tersembunyi pada satu host |
| Skill lain atau subagen | Meminta skill persis sebagai dependensi alur kerja | Komposisi | Siklus, dependensi hilang, atau kebocoran konteks |
| Harness evaluasi | Mengaktifkan skill persis di bawah skenario tetap | Pengukuran yang dapat diulang | Menguji skill sambil tanpa sengaja melewati kebijakan produksi yang sedang diteliti |

Spesifikasi Agent Skills portabel mendefinisikan paketnya. Spesifikasi itu tidak menstandarkan satu UI perintah tunggal universal, flag perutean implisit, API aplikasi, maupun siklus hidup subagen.

### Lima tahap pemanggilan

```figure
skill-invocation-stages
```

Pakai kata-kata ini secara persis:

- **Memenuhi syarat** berarti kebijakan mengizinkan pelaku ini meminta skill tersebut.
- **Dipilih** berarti pengguna menyebutnya atau perute menilainya relevan.
- **Diaktifkan** berarti instruksinya masuk ke konteks kerja.
- **Dieksekusi** berarti agen mulai melakukan kerja model atau alat di bawah instruksi itu.
- **Selesai** berarti keluarannya lulus pemeriksaan keberhasilan independen.

Jejak yang hanya mencatat `skill_used=true` menyembunyikan batas tempat kegagalan terjadi.

### Pemanggilan manusia dan model membentuk matriks 2x2

| Manusia dapat memanggil | Model dapat memanggil | Mode | Contoh yang cocok |
|:---:|:---:|---|---|
| Ya | Ya | Bersama | Penjelasan kode, perencanaan pengujian, tinjauan dokumentasi |
| Ya | Tidak | Khusus manusia | Persiapan publish, ekspor tagihan, rencana pembersihan destruktif |
| Tidak | Ya | Khusus model | Panduan gaya internal, referensi domain, prosedur dukungan otomatis |
| Tidak | Tidak | Dinonaktifkan atau khusus aplikasi | Rollout bertahap, paket yang usang, pramuat terprogram |

Matriks ini adalah model kebijakan, bukan YAML standar.

Satu host terkini memakai `disable-model-invocation: true` untuk baris khusus manusia dan `user-invocable: false` untuk baris khusus model. Nilai bawaan adalah keduanya diizinkan. Host lain memakai `agents/openai.yaml` dengan `allow_implicit_invocation: false` untuk menjaga pemanggilan eksplisit sembari menonaktifkan seleksi implisit. Ini adalah adapter runtime. Host yang tidak dikenal boleh mengabaikannya.

Detail yang membingungkan itu penting: `user-invocable: false` tidak berarti "model tidak dapat memakainya". Kolom itu menghapus pemanggilan langsung oleh pengguna pada host yang mendefinisikannya. `disable-model-invocation: true` tidak berarti "skill dinonaktifkan". Kolom itu menghapus seleksi yang dimulai model sembari menjaga akses eksplisit pengguna.

### Pemanggilan eksplisit mengutamakan identitas

Pemanggilan eksplisit memberikan identitas secara langsung:

```text
/release-readiness v2.4.0
```

atau:

```text
release-readiness check v2.4.0 without publishing
```

Antarmuka Codex terkini mendokumentasikan `/skills` untuk seleksi dan nama skill polos dalam permintaan untuk pemanggilan eksplisit. Claude Code mendokumentasikan `/skill-name` dan ekspansi argumen spesifik host. Sintaks persis, visibilitas menu, aturan pengutipan, dan ekspansi variabel menjadi wewenang host.

Permintaan eksplisit tetap melalui kebijakan. Menyebut nama skill tidak boleh melewati izin yang hilang, batasan ruang kerja, gerbang persetujuan, atau isolasi runtime.

### Pemanggilan implisit mengutamakan deskripsi

Untuk perutean implisit, model pada awalnya melihat metadata katalog, bukan isi lengkap. Karena itu, deskripsi adalah antarmuka perutean skill.

Lemah:

```yaml
description: Membantu dengan rilis.
```

Terlalu luas:

```yaml
description: Pakai untuk rilis, versi, paket, build, deploy, publish, tag, changelog, GitHub, CI, atau tugas perangkat lunak.
```

Terbatas:

```yaml
description: Periksa kandidat rilis yang sudah disiapkan dan hasilkan laporan kesiapan. Pakai ketika pengguna bertanya apakah sebuah versi, tag, paket, atau image siap diterbitkan; jangan pakai untuk kegagalan build biasa atau pengembangan fitur.
```

Versi terbatas memuat:

1. **Kapabilitas:** memeriksa kandidat yang sudah disiapkan.
2. **Keluaran:** laporan kesiapan.
3. **Batas positif:** pertanyaan apakah artefak rilis siap.
4. **Batas negatif:** build dan pengembangan biasa berada di luar cakupan.

Batas negatif berguna bila dua skill yang berdekatan berbagi kosakata. Batas itu bukan pengganti eval hampir-terlewat.

### Perutean adalah klasifikasi dengan opsi abstain

Untuk sebuah skill `s` dan permintaan `x`, bayangkan skor perute:

```text
score(s, x) = capability_match + trigger_match + context_match - exclusion_match - ambiguity_penalty
```

Penilaian persisnya boleh berupa keputusan LLM, bukan aritmetika. Prinsip rekayasa tetap berlaku: seleksi sebaiknya mengalahkan ambang batas dan skill pesaing. Bila bukti lemah, lakukan abstain.

```figure
skill-routing-abstention
```

Untuk skill berdampak tinggi, perutean implisit mungkin tidak tepat meskipun deskripsinya kuat. Pakai kebijakan khusus manusia bila biaya positif palsu melebihi kenyamanan seleksi otomatis.

### Kelayakan harus mendahului peringkat

Jangan menilai semua skill yang ditemukan, memilih yang paling cocok, lalu memeriksa kebijakan skill itu belakangan. Kecocokan teratas yang diblokir akan salah mencegah kandidat yang memenuhi syarat dengan skor lebih rendah untuk dipertimbangkan.

Pakai urutan ini untuk perutean implisit:

1. Saring skill yang ditemukan berdasarkan pelaku peminta dan adapter host aktif.
2. Nilai hanya kandidat yang memenuhi syarat.
3. Pilih kecocokan tertinggi yang memenuhi syarat bila ia melewati ambang dan aturan ambiguitas.
4. Lakukan abstain bila tidak ada kandidat yang memenuhi syarat atau tidak ada skor yang cukup kuat.

Misalkan `incident-triage` memperoleh `0.80`, tetapi ekstensi host-nya menonaktifkan pemanggilan model. `incident-review` memperoleh `0.55` dan mengizinkan pemanggilan model. Perute sebaiknya menilai `incident-review` sebagai kandidat terbaik yang memenuhi syarat. Perute tidak boleh memilih `incident-triage`, menolaknya, lalu berhenti.

Urutan ini juga mencegah perubahan kebijakan mengubah makna skor relevansi. Kelayakan menentukan himpunan seleksi. Relevansi memberi peringkat pada himpunan itu.

### Eval perutean memerlukan hampir-terlewat

Kasus positif membuktikan recall:

```json
{"prompt":"Apakah versi 2.4.0 siap diterbitkan?","expected":"release-readiness"}
```

Negatif yang jelas membuktikan presisi dasar:

```json
{"prompt":"Jelaskan rotary position embeddings.","expected":null}
```

Hampir-terlewat mengungkap kualitas batas:

```json
{"prompt":"Mengapa build paket hari ini gagal?","expected":"build-diagnostics"}
```

Hampir-terlewat berbagi kata `package` dan `build` dengan skill rilis, tetapi termasuk alur kerja lain. Himpunan perutean yang hanya berisi positif yang jelas dan negatif yang tidak terkait akan melebih-lebihkan kualitas.

### Argumen memiliki tiga representasi

Sebuah argumen pemanggilan melintasi beberapa batas:

```figure
skill-argument-boundaries
```

Pada setiap batas, pertahankan maksud tanpa memperlakukan teks sebagai kode.

- Parser host menentukan sintaks perintah dan pengutipan.
- Skill menerima teks atau variabel yang terikat menurut aturan host.
- Instruksi memvalidasi nilai wajib dan nilai bawaan.
- Panggilan alat mengubah nilai menjadi skema bertipe dan memvalidasi ulang.

Jangan menyisipkan argumen mentah ke perintah shell. Utamakan skrip yang dipanggil dengan vektor argumen atau alat MCP bertipe.

### Pemanggilan aplikasi adalah orkestrasi eksplisit

Produk dapat mengaktifkan skill karena alur kerjanya sudah mengetahui jenis tugas. Misalnya, layanan tinjauan pull request dapat memuat lebih awal `pull-request-risk-review` setelah pengguna menekan Review.

Ini menghilangkan ketidakpastian perutean, tetapi menimbulkan ketergantungan pada API runtime. Jaga adapter itu tetap di luar isi portabel:

```figure
skill-host-adapter
```

Skill sebaiknya tetap dapat dipahami ketika dibuka oleh klien patuh yang berbeda.

### Pemanggilan skill-ke-skill adalah tepian seperti alat

Misalkan `release-readiness` meminta `security-change-review` ketika berkas dependensi berubah.

Pemanggil sebaiknya menyediakan:

- identitas skill target;
- tugas dan jalur artefak yang terbatas;
- kontrak respons yang diharapkan;
- alasan pemanggilan;
- fallback bila tidak tersedia;
- kedalaman maksimum atau aturan siklus.

```json
{
  "target_skill": "security-change-review",
  "task": "Review dependency changes in the candidate diff",
  "inputs": ["artifacts/release.diff"],
  "expected": "risk-report.json",
  "max_depth": 2
}
```

Skill kedua tidak ditempelkan secara buta ke skill pertama. Host yang memutuskan cara mengaktifkannya, apakah ia berbagi konteks, berjalan dalam fork, atau kembali lewat hasil alat.

### Siklus hidup konteks spesifik host

Setelah aktivasi, isi skill dapat tetap berada dalam percakapan, diringkas saat pemadatan, atau berjalan dalam konteks terdelegasi. Izin alat boleh berlangsung satu giliran sementara instruksi bertahan lebih lama. Subagen dapat menerima skill tanpa seluruh riwayat induknya.

Jangan menulis skill yang bergantung pada anggaran umur yang tidak terlihat. Letakkan keluaran yang tahan lama di berkas atau keadaan bertipe, buat entri ulang aman, dan nyatakan apa yang harus dimuat ulang setelah gangguan.

```markdown
Saat dilanjutkan, baca `artifacts/release-readiness.json` bila berkas itu ada.
Validasi ulang commit kandidat sebelum melanjutkan.
Jangan ulangi penulisan eksternal yang kunci idempotensinya sudah tercatat.
```

## Bangun

`code/main.py` mengimplementasikan kebijakan dan perutean sebagai adapter terpisah.

Model meliputi:

- `Actor` untuk pemanggil manusia, model, agen otonom, aplikasi, skill, dan harness;
- `SkillMetadata` untuk identitas perutean;
- `InvocationPolicy` untuk matriks manusia/model;
- `InvocationRequest` dan `InvocationDecision` untuk masukan dan hasil yang dapat dilacak;
- `CorePolicyAdapter` untuk perilaku portabel tanpa ekstensi host;
- `ExtensionPolicyAdapter` untuk kolom runtime yang dikenal;
- `build_invocation_matrix(policy)` untuk tampilan 2x2;
- `route_request(skills, request, adapter)` untuk penyaringan kelayakan sebelum peringkat relevansi, seleksi, dan penolakan.

Jalankan:

```bash
cd phases/13-tools-and-protocols/25-skill-invocation-and-routing
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover -s code/tests -v
```

Demo mencetak satu matriks dan keputusan untuk pemanggilan manusia eksplisit, model implisit, agen otonom, aplikasi, komposisi skill, dan harness. Hasil adapter ekstensi menunjukkan kecocokan leksikal teratas yang diblokir dan disingkirkan sebelum alternatif yang memenuhi syarat diberi peringkat. Demo juga memuat daftar izin nama persis. Tidak diperlukan API model. Perute deterministik ada untuk membuat batas kebijakan dapat diperiksa, bukan untuk mengklaim bahwa pencocokan leksikal mereproduksi perutean model produksi.

### Mengapa adapter inti dan ekstensi terpisah

Bila satu parser memberikan makna kepada setiap kolom frontmatter yang dilihatnya, parser itu diam-diam mengangkat konvensi runtime menjadi standar palsu. Adapter terpisah memaksa pemanggil menyebutkan semantika host mana yang aktif.

`CorePolicyAdapter` hanya memakai kebijakan yang diberikan aplikasi. `ExtensionPolicyAdapter` mengenali sekumpulan kolom host yang eksplisit dan mencatat kolom mana yang mengubah keputusan.

## Pakai

Tulis kontrak pemanggilan sebelum menerbitkan skill:

```yaml
actors:
  human: allow
  model: deny
  application: allow
  skill: deny
explicit_name: release-readiness
arguments:
  candidate: required
  publish: fixed_false
ambiguity: ask_user
missing_dependency: stop
context:
  durable_state: artifacts/release-readiness.json
  max_composition_depth: 2
```

Kontrak ini adalah dokumentasi desain untuk adapter dan uji. Kontrak ini bukan frontmatter `SKILL.md` portabel kecuali sebuah standar secara eksplisit mengadopsinya.

## Kirim

Pelajaran ini menghasilkan bundel `skill-invocation-router`. Bundel itu memuat referensi model pemanggilan, contoh kebijakan host, dan CLI non-eksekutor yang menilai satu permintaan manusia, model, agen otonom, aplikasi, komposisi skill, atau harness serta mengembalikan keputusan JSON berisi kanal, adapter, skor, dan alasan.

CLI satu permintaan adalah probe kebijakan, bukan evaluasi pemicu lengkap. Pakai desain positif dan hampir-terlewat berlabel dari Pelajaran 27 untuk menghitung jumlah confusion, presisi, recall, dan stabilitas pengulangan.

## Latihan

1. Buat keempat baris matriks manusia/model dan tulis satu kasus penggunaan yang sah untuk masing-masing.
2. Tambahkan aktivasi khusus aplikasi ke `CorePolicyAdapter`. Buktikan pemanggil manusia dan model tetap ditolak.
3. Tulis sepuluh hampir-terlewat untuk skill deployment. Setiap prompt harus berbagi kosakata dengan skill itu, tetapi termasuk alur kerja yang berbeda.
4. Tambahkan margin ambiguitas antara dua skor perutean teratas. Kembalikan `ask` bila margin terlalu kecil.
5. Tambahkan kedalaman komposisi maksimum pada permintaan skill-ke-skill dan deteksi siklus dua skill.
6. Jalankan himpunan berlabel yang sama lewat adapter inti dan ekstensi. Jelaskan setiap keputusan yang berubah.

## Istilah Kunci

| Istilah | Kata orang | Arti sebenarnya |
|---|---|---|
| Pemanggilan eksplisit | "Perintah slash" | Pelaku memberikan identitas skill secara langsung, tunduk pada kebijakan |
| Pemanggilan implisit | "Model yang memilih" | Perute memilih dari metadata katalog yang memenuhi syarat berdasarkan konteks tugas |
| User-invocable | "Manusia dapat memakainya" | Properti menu atau pemanggilan langsung spesifik host, bukan kolom inti |
| Model-invocable | "Agen dapat memakainya" | Kelayakan untuk seleksi implisit oleh model di bawah kebijakan host |
| Adapter pemanggilan | "Parser frontmatter" | Kode yang memetakan kolom dan API host ke model kebijakan yang dinyatakan |
| Hampir-terlewat | "Negatif sulit" | Permintaan yang tidak memicu skill tetapi menyerupai masukan yang diharapkannya |
| Abstain | "Tidak ada skill yang dipilih" | Hasil perutean yang disengaja ketika bukti tidak ada atau ambigu |

## Bacaan Lebih Lanjut

- [Mengoptimalkan deskripsi skill](https://agentskills.io/skill-creation/optimizing-descriptions) untuk pemicu positif, spesifisitas, dan evaluasi.
- [Mengevaluasi skill](https://agentskills.io/skill-creation/evaluating-skills) untuk desain eval pemicu dan keluaran.
- [OpenAI: Membangun skill](https://learn.chatgpt.com/docs/build-skills) untuk kontrol pemanggilan eksplisit dan implisit Codex terkini.
- [Skill Claude Code](https://code.claude.com/docs/en/skills) untuk `user-invocable`, `disable-model-invocation`, argumen, dan konteks terdelegasi dari satu host.
