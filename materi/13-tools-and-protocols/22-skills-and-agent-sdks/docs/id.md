# Agent Skills: Kontrak Portabel dan Batas Runtime

> Sebuah skill bukan sekadar prompt panjang dengan nama berkas yang lebih bagus. Skill adalah paket instruksi, sumber daya, dan pembantu yang dapat dieksekusi yang dapat ditemukan, yang masuk ke konteks agen melalui kontrak runtime.

**Jenis:** Build
**Bahasa:** Python (stdlib), TypeScript
**Prasyarat:** Fase 13 · 01 (Antarmuka Alat), Fase 13 · 05 (Desain Skema Alat)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Mendefinisikan skill agen tanpa mengira ia adalah prompt, instruksi repositori, alat, hook, subagen, atau plugin.
- Membaca kontrak `SKILL.md` portabel dan memisahkannya dari ekstensi spesifik runtime.
- Menjelaskan penemuan, seleksi, aktivasi, pemuatan sumber daya, penggunaan alat, dan verifikasi sebagai tahap siklus hidup yang berbeda.
- Memvalidasi paket skill sebelum runtime menempatkannya di katalog agen.
- Memilih antara skill, alat MCP, hook, subagen, atau kode biasa untuk sebuah tugas konkret.

## Keberhasilan Pertama dalam Sepuluh Menit

Lakukan ini sebelum penjelasan panjang. Anda akan membuat skill kecil, memasang bundel pemeriksa lengkap ke host agen sungguhan, memanggilnya, memverifikasi hasilnya, lalu menghapusnya. Ini membuktikan siklus hidup dengan hasil yang dapat diamati.

### Prapemeriksaan untuk laboratorium host sungguhan

Titik pemeriksaan host sungguhan memerlukan Node.js, `npx`, Python 3, satu host yang mendukung skill yang Anda pilih, dan akses tulis ke cakupan proyek atau pengguna yang Anda pilih di pemasang. Verifikasi dulu perintah lokal:

```bash
node --version
npx --version
python3 --version
```

Putuskan host dan cakupan yang akan Anda pakai sebelum pemasangan. Bila ada prasyarat yang tidak tersedia, bacalah pelajaran ini di situs web atau lanjutkan dengan latihan paket manual di bawah. Fallback ini mengajarkan kontrak, tetapi tidak membuktikan penemuan host, pemanggilan, eksekusi skrip bundel, atau perilaku pembongkaran. Tandai pengamatan itu sebagai tertunda.

### 1. Mulai di direktori kerja kosong

Jalankan perintah ini dari direktori induk mana pun tempat Anda menyimpan pekerjaan belajar:

```bash
mkdir -p agent-skills-first-run
cd agent-skills-first-run
TARGET_ROOT="$(pwd -P)"
printf 'TARGET_ROOT=%s\n' "$TARGET_ROOT"
ls -A
```

Perintah terakhir sebaiknya tidak mencetak apa pun. Bila ada berkas yang tercetak, pilih direktori kosong lain agar tinjauan memiliki batas yang jelas.

Buat direktori untuk skill pertama Anda:

```bash
mkdir -p my-first-skill
```

Buat `my-first-skill/SKILL.md` dengan isi ini:

```markdown
---
name: my-first-skill
description: Ubah catatan rapat yang kasar menjadi catatan keputusan ringkas ketika pengguna meminta pencatatan sebuah keputusan teknis.
---

# Catatan keputusan

Ekstrak keputusan, konteks, alternatif, pemilik, dan tanggal tinjauan berikutnya.
Bila catatan tidak memuat keputusan, ajukan satu pertanyaan klarifikasi alih-alih
mengarangnya.
```

Verifikasi bahwa Anda membuat berkas di direktori yang dimaksud:

```bash
test -f my-first-skill/SKILL.md
```

Tidak ada keluaran dan kode keluar 0 berarti berkas itu ada.

### 2. Pasang bundel pemeriksa lengkap

Tetap di `agent-skills-first-run` dan jalankan:

```bash
npx skills add rohitg00/ai-engineering-from-scratch --skill skill-contract-reviewer --full-depth
```

Pilih host agen dan cakupan yang Anda pakai. Pemasang sebaiknya mencantumkan `skill-contract-reviewer` dan tujuan yang ditulisnya. `--full-depth` wajib karena skill pelajaran ini adalah bundel bersarang dengan referensi, skrip, dan aset.

Tetapkan `SKILL_ROOT` sebagai direktori absolut yang dilaporkan pemasang. Direktori itu harus direktori yang berisi `SKILL.md` yang terpasang, bukan direktori sumber pelajaran dan bukan ruang kerja saat ini:

```bash
# Ganti placeholder dengan tujuan yang dicetak pemasang.
SKILL_ROOT="$(cd "/absolute/path/to/skill-contract-reviewer" && pwd -P)"
test -f "$SKILL_ROOT/SKILL.md"
printf 'SKILL_ROOT=%s\n' "$SKILL_ROOT"
```

Bila sesi agen sudah terbuka, mulai sesi baru atau pakai perintah pemindaian ulang skill host itu. Jangan mengira setiap host memuat ulang katalognya secara otomatis.

### 3. Panggil secara eksplisit

Pada agen yang terpasang, dengan `agent-skills-first-run` sebagai direktori kerja, pakai sintaks yang didukung host itu:

| Host | Pemanggilan eksplisit |
|---|---|
| Codex | `skill-contract-reviewer`, atau pilih dari `/skills`, lalu berikan permintaan tinjauan |
| Claude Code | `/skill-contract-reviewer` diikuti permintaan tinjauan |
| Fallback portabel | `Gunakan skill-contract-reviewer untuk meninjau paket target.` |

Pakai nilai absolut yang tercetak untuk `SKILL_ROOT` dan `TARGET_ROOT` dalam permintaan. Minta host mengembangkan keduanya sebelum eksekusi dan menampilkan perintah yang tepat setelah diselesaikan, bukan perintah yang bergantung pada direktori kerja proses:

```text
Gunakan skill-contract-reviewer untuk meninjau <TARGET_ROOT>/my-first-skill. Root bundel yang terpasang adalah <SKILL_ROOT>. Jalankan python3 <SKILL_ROOT>/scripts/check_skill.py <TARGET_ROOT>/my-first-skill. Sebelum menjalankannya, tampilkan argv yang telah diselesaikan sepenuhnya. Kembalikan laporan validasi, primitif yang dipilih, dan satu kalimat untuk setiap pilihan. Sertakan jalur skrip yang telah diselesaikan, jalur target yang telah diselesaikan, cwd, argv, dan kode keluar sebagai bukti eksekusi.
```

Perintah yang telah diselesaikan sebaiknya berbentuk seperti ini, tanpa placeholder yang tersisa:

```bash
python3 "/absolute/install/path/skill-contract-reviewer/scripts/check_skill.py" \
  "/absolute/workspace/path/agent-skills-first-run/my-first-skill"
```

Hasil yang berhasil memiliki ketiga sifat ini:

1. Host menemukan `skill-contract-reviewer` berdasarkan nama.
2. Pemeriksa membaca kontrak paket dan menjalankan validator bundelnya.
3. Respons memuat laporan validasi tanpa error struktural untuk contoh itu, ditambah pemilihan primitif yang beralasan.

Bukti eksekusi juga harus menyebut jalur skrip, jalur target, cwd, vektor argumen yang persis, dan kode keluar. Laporan yang fasih tanpa kolom-kolom itu tidak membuktikan bahwa skrip pendamping yang terpasang benar-benar dijalankan.

Bila host melaporkan skill tidak tersedia, verifikasi tujuan pemasangan, pindai ulang atau mulai ulang sekali, lalu coba lagi permintaan eksplisitnya. Jangan menulis ulang deskripsi skill untuk menyembunyikan kegagalan pemasangan.

### 4. Uji seleksi implisit

Mulai giliran agen baru dan masukkan tugas yang sama tanpa menyebut nama skill:

```text
Tinjau <TARGET_ROOT>/my-first-skill sebagai paket agen yang dapat dipakai ulang dan katakan apakah kontrak paketnya valid.
```

Bila host menampilkan skill yang dipilih, catat apakah ia memilih `skill-contract-reviewer`. Bila host tidak memperlihatkan perutean, tandai seleksi implisit sebagai tidak terverifikasi. Pemanggilan eksplisit adalah fallback portabel.

### 5. Bersihkan

Hapus hanya bundel pemeriksa yang terpasang:

```bash
npx skills remove skill-contract-reviewer
```

Pilih host dan cakupan yang sama seperti saat pemasangan. Setelah pemindaian ulang atau sesi baru, permintaan eksplisit untuk `skill-contract-reviewer` sebaiknya melaporkan bahwa skill itu tidak tersedia. Simpan `my-first-skill` untuk pelajaran berikutnya, atau hapus direktori laboratorium setelah Anda menyelesaikan jalur ini.

## Masalah

Misalkan tim Anda memiliki alur rilis yang andal. Alur itu menemukan perubahan yang digabung, memeriksa catatan migrasi, memperbarui changelog, menjalankan perintah pengemasan, dan menghasilkan daftar periksa tinjauan.

Memasukkan alur itu ke dalam satu prompt membuatnya mudah ditempel tetapi sulit dioperasikan. Prompt itu tidak memiliki identitas stabil, aturan penemuan, batas sumber daya, bentuk paket yang dapat diuji, maupun jawaban atas pertanyaan dasar: Siapa yang boleh memanggilnya? Kapan model sebaiknya memilihnya? Skrip mana yang boleh dijalankannya? Berkas mana yang dipercaya? Apa yang tetap ada saat konteks dipadatkan?

Kesalahan sebaliknya adalah memperlakukan setiap instruksi yang dapat dipakai ulang sebagai skill. Konvensi repositori, otomasi deterministik, alat eksternal, hook peristiwa, dan agen yang didelegasikan menyelesaikan masalah yang berbeda. Memasukkan semuanya ke `SKILL.md` menghasilkan direktori yang tampak portabel, tetapi bergantung pada perilaku tak terdokumentasi dari satu host.

Tugas rekayasa pertama adalah klasifikasi. Putuskan apa artefak itu sebelum memutuskan cara mengemasnya.

## Konsep

### Skill mengodekan pengetahuan prosedural

Skill agen adalah direktori yang titik masuknya adalah `SKILL.md`. Berkas entri memuat frontmatter YAML diikuti instruksi Markdown. Direktori itu juga dapat memuat referensi, skrip, dan aset.

```figure
skill-package-anatomy
```

Unit yang dapat diterapkan adalah direktori, bukan berkas Markdown saja. `SKILL.md` yang disalin tanpa referensinya adalah paket rusak meskipun frontmatter-nya dapat diurai.

### Abstraksi di sekitarnya

| Artefak | Tugas utama | Dimuat atau dijalankan kapan | Apa yang tidak boleh ditirunya |
|---|---|---|---|
| Prompt | Membentuk satu interaksi model | Disertakan oleh aplikasi atau pengguna | Paket berversi dengan sumber daya |
| Instruksi repositori | Menjelaskan aturan tetap satu basis kode | Runtime pengkodean memasuki cakupan itu | Alur kerja tugas yang dapat dipakai ulang |
| Skill agen | Menyediakan pengetahuan prosedural yang dapat dipakai ulang | Aktivasi eksplisit atau implisit | Batas otorisasi yang keras |
| Alat MCP | Menyediakan kapabilitas jarak jauh bertipe | Model atau aplikasi memanggilnya | Prosedur operasi yang terperinci |
| Hook | Menjalankan logika deterministik atas suatu peristiwa | Peristiwa yang dinyatakan terjadi | Perutean model probabilistik |
| Subagen | Mendelegasikan kerja dengan konteks dan keadaan terpisah | Orkestrator membuat atau memanggilnya | Bundel instruksi statis |
| Plugin | Mendistribusikan ekstensi runtime yang lebih besar | Host memasang atau mengaktifkannya | Kontrak skill portabel itu sendiri |
| Pustaka skill terpelajar | Menyimpan perilaku yang ditemukan lewat pengalaman | Kebijakan mengambil program atau trajektori sebelumnya | Paket `SKILL.md` berbasis standar |

Skill rilis dapat memberi tahu agen cara memeriksa sebuah rilis. Server MCP dapat menyediakan registri rilis. Hook dapat melarang push langsung. Subagen dapat mengaudit kandidat secara mandiri. Bagian-bagian ini saling melengkapi karena menjaga tanggung jawab yang berbeda.

### Kata "skill" menyebut dua gagasan berbeda

Sistem riset kadang menyebut program terpelajar, trajektori yang berhasil, atau fragmen kebijakan spesifik lingkungan sebagai skill. Agen dapat membuat artefak ini saat eksplorasi, mengambilnya berdasarkan kesamaan tugas, menjalankannya, dan memperbarui pustaka dari umpan balik. Fase 14 · 10 membangun pustaka pembelajaran seumur hidup seperti itu.

Agent Skill dalam mini-jalur ini berbeda. Ini adalah paket yang ditulis dengan kontrak sistem berkas yang dinyatakan, metadata katalog, pengungkapan bertahap, pemanggilan yang dimediasi runtime, dan alat yang dikendalikan host. Skill ini dapat dibuat atau ditingkatkan oleh agen, tetapi pembelajaran bukanlah syarat formatnya.

| Dimensi | Paket Agent Skill | Pustaka skill terpelajar |
|---|---|---|
| Unit utama | Direktori `SKILL.md` | Program, kebijakan, trajektori, atau catatan memori |
| Pembuatan | Ditulis, dibuat, atau dikurasi | Umumnya ditemukan dari pengalaman lingkungan |
| Seleksi | Deskripsi katalog plus kebijakan runtime | Pengambilan atau kebijakan atas keadaan tugas |
| Eksekusi | Model mengikuti instruksi dan memanggil alat host | Lingkungan menjalankan perilaku atau artefak kode tersimpan |
| Portabilitas | Kontrak paket dapat lintas host yang kompatibel | Sering terikat pada satu lingkungan dan ruang aksi |
| Evaluasi | Perutean, artefak, keamanan, dan kompatibilitas host | Hadiah, tingkat keberhasilan, transfer, dan pertumbuhan pustaka |

Kedua gagasan mengemas kompetensi yang dapat dipakai ulang. Keduanya tidak boleh berbagi klaim implementasi hanya karena berbagi nama.

### Inti portabel

Spesifikasi Agent Skills mewajibkan dua kolom frontmatter:

```yaml
---
name: release-readiness
description: Periksa kandidat rilis ketika pengguna bertanya apakah sebuah versi siap diterbitkan.
---
```

`name` adalah pengenal stabil. Nilai itu harus memenuhi aturan penamaan spesifikasi dan cocok dengan direktori induk. `description` sekaligus dokumentasi dan metadata perutean. Nilai itu sebaiknya menyatakan apa yang dilakukan skill dan kapan skill itu berlaku.

Kolom opsional portabel adalah:

| Kolom | Tujuan | Catatan portabilitas |
|---|---|---|
| `license` | Menyatakan ketentuan paket | Spesifikasi inti |
| `compatibility` | Menyatakan persyaratan lingkungan | Spesifikasi inti |
| `metadata` | Memuat data ekstensi bernilai string | Spesifikasi inti |
| `allowed-tools` | Menyarankan alat yang sudah disetujui | Eksperimental; dukungan host berbeda-beda |

Isi Markdown memuat instruksi operasional. Isi itu sebaiknya mendefinisikan alur kerja, titik keputusan, perilaku saat gagal, dan jalur langsung ke sumber daya pendukung.

```markdown
# Kesiapan rilis

Pakai alur kerja ini untuk kandidat rilis, bukan untuk build pengembangan biasa.

1. Baca `references/release-policy.md`.
2. Jalankan `python3 scripts/inspect_release.py --format json`.
3. Berhenti bila laporan memuat kegagalan yang memblokir.
4. Buat daftar periksa dari `assets/release-checklist.md`.
5. Minta persetujuan sebelum tindakan publish atau tag apa pun.
```

### Ekstensi runtime adalah lapisan kedua

Beberapa host menerima frontmatter tambahan atau konfigurasi pendamping. Kolom itu bisa berguna, tetapi tidak otomatis portabel.

| Perilaku | Contoh ekstensi host | Inti portabel? |
|---|---|---|
| Menyembunyikan skill dari perutean model sambil mempertahankan pemanggilan langsung pengguna | `disable-model-invocation` | Tidak |
| Menyembunyikan skill dari menu perintah pengguna sambil mengizinkan perutean model | `user-invocable` | Tidak |
| Menampilkan bantuan argumen di menu perintah | `argument-hint` | Tidak |
| Menjalankan skill dalam konteks yang didelegasikan | `context`, `agent` | Tidak |
| Mengunci pengaturan model atau penalaran | `model`, `effort` | Tidak |
| Mendaftarkan otomasi siklus hidup | `hooks` | Tidak |
| Menonaktifkan pemanggilan implisit di Codex | kebijakan `agents/openai.yaml` | Tidak |

Perlakukan setiap ekstensi sebagai adapter. Jaga agar alur kerja inti tetap valid tanpanya, dokumentasikan fallback-nya, dan uji host yang mengonsumsinya. Runtime boleh mengabaikan kolom yang tidak dikenal, menolaknya, atau menyimpannya tanpa menerapkan perilakunya.

### Frontmatter adalah metadata yang dapat dieksekusi

Metadata mengubah perilaku sistem sebelum isi skill dibaca.

- `name` yang malformed dapat membuat penemuan gagal.
- `description` yang kabur dapat merutekan permintaan yang salah.
- Flag khusus manusia dapat menghilangkan skill dari katalog model.
- Izin alat dapat mengubah apakah host meminta izin.
- Pengaturan konteks dapat memindahkan eksekusi ke sesi agen terpisah.

Tinjau frontmatter seperti kode konfigurasi. Validasi, versi, dan sertakan perilakunya dalam eval.

### Siklus hidup skill

```figure
skill-runtime-lifecycle
```

Setiap panah adalah batas dengan mode kegagalannya sendiri.

1. **Penemuan** menemukan paket yang mungkin di lokasi yang dikonfigurasi.
2. **Validasi** menolak paket yang malformed atau tidak aman sebelum diterbitkan ke katalog.
3. **Pengkatalogan** menampilkan `name` dan `description` ringkas, bukan paket lengkap.
4. **Seleksi** memutuskan apakah skill itu relevan.
5. **Aktivasi** memuat isi ke konteks yang terlihat model.
6. **Pengungkapan** membaca referensi atau aset hanya bila sebuah cabang memerlukannya.
7. **Eksekusi** memakai alat host di bawah aturan izin dan isolasi host.
8. **Verifikasi** memeriksa artefak yang dihasilkan secara independen dari klaim model.

Menggabungkan tahap-tahap ini menimbulkan model mental yang keliru. Skill yang ditemukan belum aktif. Skill yang aktif belum berwenang melakukan semua yang dijelaskannya. Panggilan alat yang diizinkan bukan bukti bahwa hasilnya benar.

### Skill dan alat bersifat ortogonal

MCP menjawab, "Kapabilitas apa yang dapat dipanggil aplikasi ini, dan apa skemanya?" Sebuah skill menjawab, "Bagaimana sebaiknya agen mendekati kelas tugas ini?"

```figure
skill-tool-orthogonality
```

Skill boleh menyebut alat, tetapi host memiliki registri kapabilitas yang sebenarnya. Bila alat itu tidak ada, skill sebaiknya menyatakan fallback atau gagal dengan jelas. Skill tidak boleh menyiratkan bahwa menyebut kapabilitas akan menciptakannya.

### Skill dan instruksi repositori adalah cakupan yang berbeda

Instruksi repositori menggambarkan lingkungan yang sudah Anda masuki: perintah, konvensi, berkas yang dihasilkan, dan batas. Skill menyediakan prosedur yang dapat dipakai ulang untuk tugas yang dapat muncul di banyak repositori.

Bila keduanya berlaku, permintaan pengguna yang aktif dan aturan repositori membatasi skill. Skill refaktor umum tidak boleh mengabaikan aturan repositori yang melarang pengeditan berkas hasil generasi.

### Skill tidak saling mengimpor

Satu skill dapat mengarahkan agen untuk memanggil skill lain, tetapi ini bukan import tingkat bahasa. Skill kedua tetap melalui penemuan, kelayakan, aktivasi, izin, dan penanganan konteks runtime.

Tulis dependensi antarskill sebagai tepian alur kerja yang dapat diamati:

```markdown
Setelah menghasilkan changelog kandidat, panggil skill `release-risk-review`.
Teruskan jalur kandidat dan minta verdict yang memblokir atau tidak memblokir.
Bila skill itu tidak tersedia, berhenti dan laporkan dependensi yang hilang.
```

Ini membuat dependensi dapat diuji dan memberi host kesempatan untuk memberlakukan kebijakan.

## Bangun

`code/main.py` mengimplementasikan validator kecil berorientasi standar dan pemilih artefak. Kode ini memakai stdlib saja agar setiap aturan terlihat.

Validator menyediakan:

- `parse_frontmatter(text)` untuk memisahkan metadata dari isi.
- `validate_skill_text(text, directory_name, allowed_runtime_extensions=())` untuk memeriksa kolom wajib, penamaan, ekstensi yang tidak dikenal, keberadaan isi, dan batas portabel.
- `ValidationIssue` dan `SkillReport` untuk mengembalikan bukti terstruktur, bukan satu boolean yang buram.
- `FrontmatterSyntaxError` untuk masukan yang tidak dapat diartikan dengan aman.

Pemilih menyediakan `TaskShape` dan `select_primitives(task)`. Pemilih itu memetakan kebutuhan tugas ke kode biasa, instruksi repositori, skill, hook, subagen, atau alat MCP.

Jalankan laboratorium:

```bash
cd "$(git rev-parse --show-toplevel)"
cd phases/13-tools-and-protocols/22-skills-and-agent-sdks
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover -s code/tests -v
```

Blok perintah ini memerlukan klon lokal dan harus dimulai dari mana pun di dalam klon itu agar `git rev-parse --show-toplevel` dapat menyelesaikan root repositori.

Demo mencetak JSON untuk satu skill portabel yang valid, satu skill dengan ekstensi host, satu paket yang tidak valid, dan beberapa keputusan bentuk tugas. Periksa kode isu. Validator paket sebaiknya menjelaskan cara memperbaiki artefak tanpa menebak atas nama penulis.

### Urutan validasi penting

Validasi fakta struktural yang murah sebelum aturan isi yang lebih dalam:

```figure
skill-validation-order
```

Urutan ini mencegah error sekunder menutupi invarian pertama yang rusak.

## Pakai

Sebelum menulis skill, isi kartu keputusan ini:

| Pertanyaan | Bila ya | Primitif yang mungkin |
|---|---|---|
| Apakah ini membutuhkan penilaian model yang dapat dipakai ulang lintas beberapa langkah? | Prosedurnya stabil, tetapi keputusannya beragam | Skill |
| Apakah ini harus terjadi setiap kali peristiwa terjadi? | Tidak boleh ada satu eksekusi yang terlewat | Hook atau kode aplikasi |
| Apakah model membutuhkan kapabilitas eksternal dengan masukan bertipe? | Operasi itu berada di luar konteks model | Alat atau server MCP |
| Apakah pekerjaan itu memerlukan konteks, keadaan, atau kepemilikan yang terisolasi? | Pekerja terpisah mengembalikan hasil terbatas | Subagen |
| Apakah panduan ini spesifik untuk satu repositori? | Panduan itu menjelaskan perintah dan batas lokal | Instruksi repositori |
| Apakah satu interaksi sudah cukup? | Tidak perlu siklus hidup paket | Prompt |

Banyak alur kerja produksi memakai lebih dari satu baris. Kartu ini mencegah satu artefak berpura-pura menyediakan semua sifat.

## Kirim

Pelajaran ini menghasilkan bundel `skill-contract-reviewer` di bawah `outputs/`. Bundel itu memuat:

- `SKILL.md` portabel yang meninjau paket skill yang diusulkan;
- daftar periksa referensi untuk kontrak portabel dan seleksi primitif;
- skrip validasi deterministik;
- fixture bentuk tugas yang mencakup prompt, skill, alat, hook, kode biasa, dan subagen.

Pasang bundel lengkap, bukan hanya berkas entrinya:

```bash
cd "$(git rev-parse --show-toplevel)"
python3 scripts/install_skills.py /tmp/aiefs-skills --phase 13 --type skill
```

Pemasang kursus melaporkan setiap skill Fase 13 yang disalin dan menulis `/tmp/aiefs-skills/manifest.json`. Tujuan yang bersih ini memeriksa bentuk paket; loop keberhasilan pertama di atas memeriksa penemuan dan pemanggilan pada host sungguhan.

Pelajaran berikutnya memperdalam setiap tahap siklus hidup. Pelajaran 24 membangun penemuan dan pengungkapan bertahap. Pelajaran 25 membangun kebijakan pemanggilan dan perutean. Pelajaran 26 memisahkan izin dari sandboxing. Pelajaran 27 mengubah seluruh paket menjadi artefak rilis yang dievaluasi.

## Latihan

1. Klasifikasi lima alur kerja dari tim Anda sendiri memakai `TaskShape`. Pertahankan setiap kasus yang memilih lebih dari satu primitif.
2. Tambahkan uji batas yang membuktikan nilai `compatibility` 500 karakter lolos dan nilai 501 karakter gagal sebagai error spesifikasi.
3. Tambahkan satu ekstensi runtime ke daftar izin. Tulis uji yang membuktikan berkas yang sama masih dapat dibedakan dari skill portabel saja.
4. Pecah prompt 400 baris menjadi `SKILL.md`, satu referensi, satu kontrak skrip, dan satu templat keluaran. Jaga agar setiap berkas hanya bertanggung jawab atas satu jenis informasi.
5. Rancang respons kegagalan untuk skill yang merujuk alat MCP yang tidak tersedia. Jangan diam-diam mengganti alat dengan alat yang memiliki izin lebih luas.
6. Tinjau skill yang sudah ada dan beri label setiap kalimat sebagai perutean, prosedur, kebijakan, penunjuk referensi, atau kontrak keluaran. Pindahkan apa pun yang tidak semestinya.

## Istilah Kunci

| Istilah | Kata orang | Arti sebenarnya |
|---|---|---|
| Skill agen | "Prompt yang disimpan" | Direktori instruksi prosedural yang dapat ditemukan dengan sumber daya opsional |
| Inti portabel | "Kolom yang sama di semua runtime" | Kontrak yang didefinisikan spesifikasi Agent Skills |
| Ekstensi runtime | "Frontmatter tambahan" | Konfigurasi spesifik host yang perilakunya memerlukan adapter kompatibel |
| Aktivasi | "Skill itu berjalan" | Isi skill masuk ke konteks yang terlihat model; eksekusi bisa terjadi kemudian |
| Dependensi skill | "Impor skill lain" | Tepian pemanggilan yang dimediasi runtime dengan pemeriksaan ketersediaan dan kebijakan |
| Kontrak alat | "Skema fungsi" | Masukan, keluaran, izin, efek samping, error, dan bukti untuk sebuah kapabilitas |

## Bacaan Lebih Lanjut

- [Spesifikasi Agent Skills](https://agentskills.io/specification) untuk kontrak direktori dan frontmatter portabel.
- [Praktik terbaik Agent Skills](https://agentskills.io/skill-creation/best-practices) untuk cakupan, instruksi, dan pengorganisasian sumber daya.
- [OpenAI: Membangun skill](https://learn.chatgpt.com/docs/build-skills) untuk perilaku penemuan dan pemanggilan Codex terkini.
- [Skill Claude Code](https://code.claude.com/docs/en/skills) untuk ekstensi pemanggilan, argumen, alat, dan konteks terdelegasi dari satu runtime.
