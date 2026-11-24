# Analisis Sub sistem Konten Pendukung — `ai-engineering-from-scratch`

**Lokasi repositori:** `C:\Users\Hendra\Downloads\ai-engineering-from-scratch`
**Lingkup:** `manuals/`, `cheatsheets/`, `interviews/`, `practice/`, `projects/`, `i18n/`, `glossary/terms.md`, `docs/`, `research/`, `outputs/`
**Tanggal analisis:** 24 November 2026
**Metode:** pembacaan kontrak penulisan (`AUTHORING.md`), manifest, skrip generator/validator, dan sampel artefak nyata di setiap subsistem. Tidak ada file di repositori yang diubah.

---

## 0. Ringkasan eksekutif

| Subsistem | Kontrak utama | Mesin validasi | Bentuk artefak | Fakta kunci |
|---|---|---|---|---|
| `manuals/` | `manuals/AUTHORING.md`, `manuals/manual.schema.json` | `node scripts/audit_manuals.js` → **`TOTAL 0`**; `node --test site/test_manuals.js` | 8 manual `ready`, masing-masing 36–64 bagian, 64+ SVG, capture kit | Satu subjek, satu versi pin, semua klaim tertelusur; PDF hanya release asset |
| `cheatsheets/` | `cheatsheets/AUTHORING.md`, `cheatsheets/topics.yaml` | `node site/build-cheatsheets.js --check`, `--check-quality` | 156 poster + 156 evidence file | Generator AI dengan token budget, retry, eskalasi model, dan fact-check wajib |
| `interviews/` | `interviews/AUTHORING.md` | `node scripts/interviews/validate.js`, `site/build-interviews.js --check` | 482 pertanyaan / 22 kartu / 60 skenario / 1 loop | Jawaban terlipat, aturan angka bersumber, denylist perusahaan, anti-kebocoran 24 karakter |
| `practice/` | `practice/AUTHORING.md` | `python3 scripts/practice/validate.py` | 52 soal + `_harness/` | 4 lapis tes + mutant 1–12 baris yang harus gagal |
| `projects/` | `projects/AUTHORING.md` | `python3 scripts/project_test.py`, `site/build-projects.js --strict` | 79 `ready` + 25 `planned` (total 104 di katalog) | 4–8 stage kumulatif, starter gagal dulu, grader 4 bahasa, bukti penyelesaian lokal |
| `i18n/` | `docs/i18n.md`, `languages.json` (di **root**, bukan di `i18n/`) | `translate.yml`, `audit_readme_locales.py`, `test_ui_i18n.js` | 33 README terjemahan + `readme-source.sha256` | Terjemahan pelajaran di branch `translations`, NLLB-200 gratis di CI |
| `glossary/` | format entri di `terms.md` | parser di `site/build.js` | ~1.969 baris, 12 kategori | Memberi makan `/glossary`, halaman entri, `data.js`, sitemap, `llms.txt` |
| `docs/`, `research/`, `outputs/` | `docs/i18n.md`, `docs/portfolio-writing-sync.md`, `research/methods.json` | — | 2 doc, 1 registry metode, 4 direktori artefak + `index.json` | `outputs/` saat ini hanya placeholder (kosong secara substansi) |

**Tema lintas subsistem:** hampir semua konten pendukung mengikuti pola yang sama — *sumber kanonik → generator/penulis → validator deterministik → artefak turunan yang tidak di-commit*. Bukti (capture, evidence, fixture, source hash) disimpan di repo, sedangkan hasil build (`site/data.js`, `site/manuals-data.js`, `site/projects-data.js`, poster HTML) dihasilkan ulang oleh CI.

---

## 1. `manuals/` — manual versi-pin dari hasil capture nyata

### 1.1 Kontrak penulisan (`manuals/AUTHORING.md`)

Manual berbeda dari pelajaran: **"Sebuah pelajaran mengajarkan satu konsep lalu berlanjut. Sebuah manual menemani satu sistem sampai pembaca dapat memprediksi apa yang akan dilakukannya dalam kasus yang tidak pernah ditunjukkan manual itu."** Ciri wajib:

- **Satu subjek, satu versi, satu tanggal.** Halaman depan mem-bin `subject`, `version`, URL `source`, `commit`, tanggal rilis, dan tanggal `verified`.
- **Dibangun dari hasil run nyata.** Capture kit menjalankan subjek secara offline dan menulis apa yang dilihatnya; listing dan figur dipotong dari file itu, bukan diketik dari ingatan.
- **Ditelusur ke sumber berperingkat.** Daftar `sources` diurutkan menurut otoritas; jika bertentangan, sumber tertinggi menang dan teks menyatakannya.
- **Orisinal.** Dilarang mengutip panduan/kursus/buku lain; yang boleh dikutip: spesifikasi, kode sumber, paper, dokumentasi subjek itu sendiri.

Target ukuran: 60–100 halaman, 6–8 bagian, ±25 seksi, 30–40 figur.

### 1.2 Struktur direktori dan `manual.json`

Bentuk direktori (dari `manuals/AUTHORING.md:17-31`):

```text
manuals/<id>/
  manual.json                  metadata, pin, palette, urutan bagian
  README.md                    wajib sebelum status "ready"
  front.md                     "How to read this manual"
  sections/<n>-<m>-<slug>.md   satu file per seksi
  sections/r-<n>-<slug>.md     seksi referensi
  figures/<figure-id>.svg      figur statis
  figures/src/<figure-id>.js   sumber figure kit (opsional)
  capture/                     capture kit + output di capture/out/
  research/brief.md            ringkasan fakta
  research/sources/            sumber primer yang di-vendor + lisensinya
```

Field `manual.json` meliputi `id`, `title`, `subtitle`, `summary`, `audience`, `edition` (`YYYY.MM`), `status`, `pin`, `outcomes` (≥3), `palette`, `sources`, `quoteSources`, `capture` (`run`/`check` argv + `timeoutSeconds`), `front`, `plate`, `parts`, `reference`, `related`. Skema mesin ada di `manuals/manual.schema.json`.

Contoh nyata `manuals/sglang-101/manual.json`:

| Field | Nilai |
|---|---|
| `id` / `title` | `sglang-101` / "SGLang 101" |
| `edition` / `status` | `2026.10` / `ready` |
| `pin` | SGLang `0.5.22`, commit `baa9befa381511cd4d56f83980951e5a45fffc6d`, rilis 2026-10-13, diverifikasi 2026-10-16 |
| `palette` | 10 hue dengan arti tetap per manual (`rose`=gagal, `grey`=di luar subjek, `amber`=lifecycle, `teal`=state persisten; 6 hue lain bebas) |
| `sources` | kode pada release tag, captured runs, isu/PR, paper, dokumentasi, laporan lapangan, laporan tim |
| `capture` | `python3 capture/run.py` / `python3 capture/run.py --check`, timeout 1800 dtk |
| `parts` | 1 Start, 2 The Request Path, 3 The KV Cache, 4 Each Forward Pass, … (ID seksi `1.1`, `3.2`, dst.; referensi `R.<n>`) |

### 1.3 Delapan manual (bukan sembilan)

Terdapat **9 direktori** di `manuals/`, tetapi satu di antaranya (`_shared/`) adalah pustaka figur bersama, bukan manual. Delapan manual dengan `manual.json` (semua berstatus `ready`):

| ID | Judul | Edisi | Versi yang di-pin |
|---|---|---|---|
| `a2a-101` | A2A 101 | 2026.10 | 1.0.1 |
| `agent-substrate-101` | Agent Substrate 101 | 2026.11 | 1.0.0 |
| `docker-sandboxes-101` | Docker Sandboxes and Docker Agent 101 | 2026.10 | sbx 0.47.0, docker-agent 1.149.0 |
| `langchain-101` | LangChain 101 | 2026.11 | 1.5.0 |
| `miles-101` | Miles 101 | 2026.10 | `radixark/miles:latest`, Miles main 2026-10-15 |
| `reverse-engineering-101` | Reverse Engineering 101 | 2026.10 | rea-agents 6.7.0, Ghidra 12.1.4 |
| `sglang-101` | SGLang 101 | 2026.10 | 0.5.22 |
| `vllm-101` | vLLM 101 | 2026.10 | 0.31.0 |

> Catatan: angka "9 manual" pada permintaan sesuai dengan jumlah direktori; `_shared/` bukan manual keenam belas melainkan fondasi figur (lihat 1.6).

### 1.4 Bentuk seksi (contoh nyata)

`manuals/sglang-101/sections/3-4-prefix-friendly-prompts.md` memperlihatkan kontrak secara utuh:

1. `# Prefix-friendly prompts` — nama sesuai spesifikasi.
2. `> The prefix cache matches tokens from the first one, …` — tesis ≤30 kata.
3. Paragraf pembuka dari situasi konkret + kalimat "When you finish this section, you can …" di dua paragraf pertama.
4. Badan: `##` subjudul, paragraf definisi `**Term:**`, figure, listing, tabel.
5. Satu blok `takeaways` (5 aksi).
6. `Sources: …` di baris terakhir — dibaca audit, tidak dirender ke pembaca.

Listing di seksi ini bersumber dari `capture/out/cache-experiment.txt` dan `capture/out/server-default.log`; angka seperti `2.998 of 2.999 tokens`, `1.255 ms vs 45 ms`, dan `28 times longer` semuanya berasal dari capture, bukan karangan. Penanda sitasi `{{issue #35129}}`, `{{code}}`, `{{field Jarvislabs 2026-08}}` harus memakai kunci yang dideklarasikan di `quoteSources`.

Subset Markdown sengaja diperkecil: raw HTML, horizontal rule, nested list, heading di bawah `###`, dan quote selain tesis **gagal build**. Anchors berbentuk `s-<id>` (titik diganti hyphen), mis. `s-1-4` atau `s-r-2`.

### 1.5 Capture kit dan aturan `TOTAL 0`

Aturan capture (`manuals/AUTHORING.md:275-294`):

- Kit ada di `capture/`, berjalan offline, tanpa kunci, dan mengikuti allowlist dependensi repo. Pengecualian: manual tentang library boleh memasang library itu dengan versi eksak di `capture/requirements.txt`; subjek berbayar/proprietary boleh direkam di mesin penulis, dengan `check` yang membandingkan output tersimpan setelah masking dan mencetak `skipped` + exit 0 bila tidak terpasang.
- `capture.run` menulis `capture/out/`; `capture.check` menulis ke direktori sementara lalu membandingkan dengan `capture/out/` setelah **masking nilai yang berubah tiap run** (timestamp, ID generatif). Audit menjalankan `capture.check` dan gagal pada drift.
- Listing wajib mencantumkan `source:`; blok `json` yang bukan listing ditolak.
- Semua sitasi, `Sources:`, dan path listing hidup di Markdown untuk verifikasi, tetapi **tidak dirender** — pembaca melihat "Log lengkap"/"Jalankan mentah" sebagai tautan, bukan path file.

Contoh `manuals/sglang-101/capture/README.md`: kit Apple-silicon menjalankan SGLang 0.5.22 via MLX, menulis 4 konfigurasi server ke `capture/out/`, dan `--check` membandingkan teks yang dihasilkan pada temperature 0, jumlah token, median cached tokens, metric names, dan server commands. Kit GPU di `capture/gpu/` memakai profil `gb10`/`l4`, sweep dengan seed 1–3, dan `gpu/summary/<profile>.json`; tanpa Linux/CUDA ia mencetak `skipped` dan exit 0.

Audit dijalankan dengan:

```bash
node --test site/test_manuals.js
node site/build-manuals.js
node scripts/audit_manuals.js --manual <id>
node scripts/audit_manuals.js            # harus mencetak TOTAL 0
```

`scripts/audit_manuals.js:350-381` menghitung setiap pelanggaran, mencetak per-manual statistik (`documents, prose words, figures, listings, takeaways`), lalu `TOTAL <n>`; exit 1 bila bukan nol. Aturan yang diukur mencakup panjang kalimat (≤25 kata), tesis (≤30), klaim figur (≤28), larangan semicolon/kontraksi/dash, daftar `LEXICON` (mis. "robust", "seamless", "leverage"), kunci `quoteSources`, 24+ karakter prosa yang dikutip bersebelahan dengan penanda sumber, legibilitas SVG, dan drift capture.

### 1.6 Figure kit

`manuals/_shared/` berisi `figkit.js`, `forms.js`, `marks.js`, `chart.js`, `svg.js`, `dither.js`, `graphviz-lite.js`, `motion.js`, `tokens.css`, `manual.css`, `web.css`, `print.css`, `theme.js`, dan 10 template di `_shared/templates/` (chart, comparison, decision, flow, layers, sequence, state, structure, timeline, tree).

- **10 jenis figur** dengan aturan pemakaian tetap (sequence, structure, flow, comparison, timeline, tree, state, decision, layers, chart). `chart` memakai `_shared/chart.js` dan angka dari file ringkasan capture agar figur dan data tidak bisa menyimpang.
- **8 gaya panah** tetap: solid ink = call, dashed ink = reply, solid teal = durable write, dashed amber = state change, solid plum = model call, dashed olive = efek eksternal, dotted indigo = stream/event, dashed rose = failure.
- **Legibilitas:** teks ≥11 unit pada viewBox 640, tanpa elipsis, tanpa teks bertumpang, label yang tidak muat keluar dengan leader line; audit mengukur setiap label. SVG harus `viewBox="0 0 640 H"` (H ≤760), `role="img"`, `aria-labelledby`, `<title>`/`<desc>` ≥40 karakter, ID diawali figure ID, warna hanya `var(--m-<token>, #hex)`, ≤40 KB.
- Sumber figur disimpan di `figures/src/<id>.js` dan dibangun dengan `node manuals/_shared/figkit.js build manuals/<id>`; audit menggagalkan perbedaan antara SVG commit dan sumbernya. `forms.js`/`marks.js` menggambar objek nyata dan logo proyek (hanya dari artwork resmi + catatan lisensi).

### 1.7 Status, PDF, dan artefak build

- `draft`: halaman `/manual-<id>.html` dibangun dengan `noindex`, tidak dicantumkan di indeks.
- `ready`: wajib punya `README.md`, dicantumkan di halaman indeks, dan PDF-nya dilekatkan ke setiap release.
- `site/build-manuals.js` menulis `site/manual-<id>.html`, `site/manuals.html`, `site/manuals-data.js` — **semuanya dihasilkan saat deploy dan tidak pernah di-commit**. Tambahkan `--drafts` untuk pratinjau lokal.
- PDF dibuat di CI memakai `npx --yes pagedjs-cli@0.4.3 … -o dist/manuals/aiefs-manual-<id>.pdf`. PDF **bukan file repo**, melainkan release asset; workflow Manuals bisa dijalankan manual dengan tag release untuk melampirkan ulang.
- Commit: satu seksi + figur + capture yang menyertainya adalah satuan yang baik, `feat(manual/<id>): <section slug>`.

---

## 2. `cheatsheets/` — poster referensi dengan bukti baris-demi-baris

### 2.1 Peta cakupan (`cheatsheets/topics.yaml`)

`topics.yaml` adalah peta cakupan: komentar di baris 1–3 berbunyi *"Setiap pelajaran di phases/, setiap manual, dan setiap proyek adalah sumber dari minimal satu topik. `node site/build-cheatsheets.js --check` akan gagal jika ada yang hilang."*

Tiap entri memiliki `id`, `title`, `kind` (`concept`, `commands`, `comparison`, `debugging`, `manual`, `project`), dan `sources` (path lesson/manual/project). Contoh: `a2a` → `manuals/a2a-101`; `inference-optimization` → 7 pelajaran; `mcp` → 7 pelajaran; `toll-calling`/`tool-calling` → 9 sumber termasuk `projects/tool-contract-migration-checker`. Beberapa entri memakai `explainer` (mis. `sglang-serving` → `inside-an-llm-serving-gpu.html`). Secara lokal terhitung **156 file poster `.md`** dan **156 file evidence** di `cheatsheets/evidence/`.

### 2.2 Bentuk file dan blok

Front matter wajib: `id`, `title`, `subtitle` (≤120 karakter), `hue` (0–359), `sources`; opsional `code` (2 huruf unik per topik) dan `source_hash` — keduanya di-set otomatis oleh skrip. Tiap blok diawali `## Label`, diikuti `key: value` (`type`, `from`, opsional `lang`, `col`, `span`), baris kosong, lalu badan.

12 jenis blok dengan warna bawaan: `concepts`/`glossary` (violet), `task-table`/`command-table` (biru), `annotated-code` (ink), `flow`/`architecture` (teal), `defaults` (amber), `comparison` (slate), `troubleshooting` (merah), `qa`/`tips` (hijau). Blok `architecture` memakai fence `arch` berisi baris `node | id | col,row | label | sublabel | kind` dan `edge | from | to | label`.

Aturan fakta: **setiap flag, nama dengan underscore/titik/panggilan, dan setiap default harus muncul di minimal satu file `from` blok itu** (`--check` menegakkan sebagian aturan ini). Nilai yang diganti pembaca ditulis `<model>`, `<port>` dan diwarnai build.

### 2.3 Evidence file

`cheatsheets/evidence/<id>.md` diawali `# Evidence: cheatsheets/<id>.md`, lalu satu `##` per blok. Setiap bullet menamai fakta dan referensi `path:line` atau `path:line-line`. Boleh ada singkatan di bagian atas, mis. `S = manuals/sglang-101/sections, O = manuals/sglang-101/capture/gpu/out-l4`. Contoh nyata `cheatsheets/evidence/sglang-serving.md`:

```text
## Defaults that matter (defaults, 7 rows)
- --schedule-policy fcfs: S/2-2-the-scheduler-loop-and-schedule-policy.md:60
- --kv-cache-dtype auto: S/3-1-kv-bytes-per-token-and-the-kv-pool.md:70
```

Validator `site/cheatsheet-evidence.js` (dipanggil dari `build-cheatsheets.js`) menolak path yang tidak ada, path yang bukan sumber topik, nomor baris di luar file, blok tanpa seksi, atau seksi dengan bullet valid lebih sedikit daripada jumlah baris/langkah/node/item blok.

### 2.4 Generator dengan cost guard (`scripts/cheatsheets/generate.js`)

Alur: `.github/workflows/cheatsheets-generate.yml` memakai `node scripts/cheatsheets/generate.js` untuk menulis sheet yang missing/stale lalu membuka PR. Generator:

- Memakai **Haiku sebagai default** (`claude-haiku-5-5`), **Sonnet untuk eskalasi**, Opus opsional lewat `--model opus` (MODELS/PRICES didefinisikan di baris 11–18).
- `sourceBundle()` mengumpulkan file sumber, membuang test/fixture/log (`BUNDLE_SKIP`), memotong file >60.000 karakter dan total >300.000 karakter, lalu menyisipkan ringkasan bukti ke prompt.
- System prompt merangkai `scripts/cheatsheets/prompt.md`, rubrik kualitas, `cheatsheets/AUTHORING.md`, dan tiga contoh poster + satu contoh evidence.
- **Cost guards:** `DEFAULT_TOKEN_BUDGET = 1.500.000`, `MAX_RETRIES = 2`, `MAX_TOKENS = 20.000`, perkiraan biaya dihitung per panggilan (`callCost`, termasuk cache read/write dan harga long-context), `--limit` membatasi jumlah topik, `--token-budget` menghentikan topik berikutnya saat kuota habis, dan ringkasan `--summary` mencatat token + biaya per topik.
- Balasan harus memuat `<topic>` dan `<evidence>`; keduanya diperiksa dengan `checkText` + `evidenceProblems` + `qualityReport`. Jika gagal: hingga 2 putaran kritik-revisi, lalu eskalasi ke Sonnet (maks. 2–3 kali panggil per putaran). **Jika masih gagal, generator tidak menulis apa pun** untuk topik itu.
- `--dry-run` hanya menghitung jumlah file/karakter sumber; `--stamp <id>` menyalin `sources` + `source_hash` ke sheet tanpa memanggil model.

### 2.5 Rubrik kualitas dan check

`node site/build-cheatsheets.js --check-quality` menilai: 10–14 blok, ≥6 jenis blok, harus ada diagram `flow`/`architecture`, harus ada `qa`/`tips`, every defaults row berisi nilai nyata, tanpa blok tipis, tanpa baris duplikat, hanya konten pembelajar (tanpa capture kit/`run.py`/build script), tidak ada sel kosong/"TBD"/"...", tidak ada edge yang memotong node, dan maksimal 2 halaman cetak. Ambang per jenis (`QUALITY_RULES` di `site/build-cheatsheets.js:1178-1185`):

| Kind | Baris task/command | Baris troubleshooting | Ekstra |
|---|---|---|---|
| `concept` | 4 | 3 | — |
| `commands` | 8 | 3 | `annotated-code` |
| `comparison` | 3 | 3 | tabel comparison |
| `debugging` | 4 | 5 | — |
| `manual` | 8 | 3 | `annotated-code` |
| `project` | 6 | 3 | — |

Perintah lain: `--check` (fakta, evidence, cakupan, tautan), `--status` (sheet missing/stale), `--check-topic <file>`, `--print`, `--edition`, `--katex`. Sesuai `AGENTS.md`, perubahan cheatsheets/lesson diwajibkan menjalankan `node site/build-cheatsheets.js --check && … --check-quality`.

---

## 3. `interviews/` — bank soal & skenario insiden

### 3.1 Struktur (`interviews/AUTHORING.md:7-21`)

```text
interviews/
  AUTHORING.md     kontrak ini
  README.md        indeks kartu (dihasilkan, jangan diedit)
  topics.yaml      23 kartu dalam 6 grup
  roles.yaml       tampilan kartu per peran
  questions/<card-id>/<id>.md   satu pertanyaan
  scenarios/<id>/README.md      satu skenario + evidence/
  loops/<role-id>.md            putaran wawancara per peran (opsional)
```

Enam grup di `topics.yaml`: `foundations`, `models`, `applications`, `agents`, `production`, `rounds`. Kartu yang terlihat antara lain `math-classical-ml`, `deep-learning`, `transformers`, `tokenization-decoding`, `post-training`, `inference-serving` (planned 30), `gpu-memory-quantization`, `rag`, `prompting-structured-output`, `evaluation`, `agents`, `protocols`, `multi-agent`, `sandboxes`, `evidence-reverse-engineering`, `cost-capacity`, `production-mlops`, `system-design`, `safety`, `multimodal`, `scenarios`, `coding`, `behavioral`. `planned` menyatakan target jumlah item versi pertama.

**README yang dihasilkan tidak boleh diedit atau di-commit** — CI membuatnya ulang di `main`; PR yang mengubahnya akan menimbulkan konflik dan gagal pemeriksaan.

### 3.2 Cakupan peran dan loop

`roles.yaml` berisi 7 entri: 6 peran memakai `path` ke karier route di `learning-paths/` (`applied-ai-engineer`, `agentic-ai-engineer`, `ai-evaluation-reliability-engineer`, `ai-data-engineer`, `forward-deployed-ai-engineer`, `ai-developer-relations-engineer`) dan 1 peran mandiri `inference-engineer` dengan judul/summary sendiri. Setiap peran memetakan 14–21 kartu.

`loops/inference-engineer.md` (satu-satunya loop yang ada) menetapkan 240 menit: screen 30' → ml-depth 45' → debugging 45' → system-design 60' → coding 45' → behavioral 15', masing-masing diakhiri `(card: <id>)`. Skill `interview-prep` menjalankan loop ini; situs belum menampilkannya.

Data aktual: **482 berkas pertanyaan** di 22 folder kartu, **60 skenario**, 1 loop.

### 3.3 Kontrak pertanyaan

Front matter wajib: `id` (= nama file), `title` (≤60 karakter), `card` (= folder), `kind` (`concept`/`tradeoff`/`design`/`debug`/`code-reading`/`estimate`/`take-home`), `level` (1 screen … 4 staff), `roles`, `rounds` (10 nilai: screen, coding, take-home, system-design, design-review, debugging, ml-depth, ml-breadth, behavioral, customer), `tags` (1–6), `minutes` (2–60), `lessons`, `manuals` (`<id>#s-<slug>`), `practice`, `cheatsheet`, `related`, `updated`. Minimal satu lesson **atau** satu manual section harus disebut.

Badan harus tepat 5 bagian `##` berurutan: prompt (tanpa heading) → `Short answer` (2–4 kalimat) → `Full answer` → `Follow-ups` (3–5 butir, level naik, tiap butir berbentuk `- **Level 2:** … Expected: …`) → `Wrong answers` (2–5 kutipan + alasan) → `Taught in`.

Contoh nyata `interviews/questions/inference-serving/prefix-cache-order.md` menghubungkan `phases/17-infrastructure-and-production/06-sglang-radixattention`, `sglang-101#s-prefix-friendly-prompts`, `vllm-101#s-make-repeated-prompts-hit-the-prefix-cache`, dua cheat sheet, dan `scenarios/prefix-cache-inflated-benchmark`. Angka dalam jawabannya (2.999 token, 45.1 ms, 1.255.3 ms) persis mengikuti capture manual SGLang.

### 3.4 Skenario dan evidence byte-exact

Skenario menambahkan `system` (`version`, `model`, `hardware`) dan `source` (daftar file repo). Badan wajib 8 seksi: `Situation`, `Evidence`, `Questions` (3–6), `Walkthrough`, `Resolution`, `Wrong turns`, `Generalize`, `Taught in`. Contoh `interviews/scenarios/cold-start-on-gb10/README.md` memakai vLLM 0.31.0, model `Qwen/Qwen3-30B-A3B-Instruct-2507`, satu GB10 128 GB, dan 5 berkas evidence.

Evidence file memuat header `title`, `source`, `lines` (opsional `N-M`), `lang`, `note`, garis `---`, lalu listing. Aturan: **teks setelah `---` harus sama byte-for-byte dengan baris N–M sumbernya, termasuk akhir baris**; dilarang memotong baris panjang. Contoh `evidence/weight-load.txt` adalah salinan persis baris 41–44 dari `manuals/vllm-101/capture/gpu/out-gb10/server-load-vllm.log`.

### 3.5 Aturan angka, denylist, dan gaya

- **Setiap angka ≥10 atau berkoma** di prompt/jawaban (dan semua seksi skenario kecuali `Evidence` + `Taught in`) harus berasal dari `docs/en.md` lesson yang disebut, seksi manual yang disebut, atau — untuk skenario — file `source`/listing evidence. Koma dihapus saat pencocokan (`1,213` = `1213`); dilarang membulatkan atau menghitung angka baru di teks.
- Tautan harus relatif dan resolve; **`http` dilarang**.
- **Dilarang menyebut nama perusahaan.** `scripts/interviews/denylist.txt` memuat daftar seperti airbnb, amazon, anthropic, apple, aws, google, meta platforms, microsoft, openai, nvidia-ish vendor names, dll. Model disebut dengan model ID, GPU dengan nama modelnya.
- Simplified Technical English; tanpa em/en dash.

### 3.6 Validator dan aturan anti-kebocoran

`node scripts/interviews/validate.js` (konstanta di `scripts/interviews/validate.js:8-28`) memeriksa kunci/urutan front matter, bentuk follow-up dan wrong answer, keberadaan semua lesson/seksi/practice/cheatsheet/related, resolusi tautan dan anchor, aturan angka, denylist dan dash, kecocokan evidence dengan sumber, dan bahwa prompt tidak memuat **span 24 karakter** dari jawaban (`LEAK_SPAN = 24`). Opsi `--write` membuat pratinjau README (wajib dikembalikan dengan `git checkout -- …` sebelum commit); `--check-readmes` dipakai CI main-only.

Sisi situs, `node site/build-interviews.js --check` menambahkan aturan:

1. Judul + prompt masuk indeks pencarian; tidak boleh berbagi span 24 karakter dengan jawaban mana pun di bank.
2. Baris "Taught in" adalah teks terlihat → aturan yang sama berlaku; nama lesson ditulis sebagai teks tautan generik.
3. Short answer dan prompt tidak boleh diawali code span (indeks pencarian membuang sebagian karakter code).
4. README hasil generate tidak di-commit.

Tampilan situs memakai `<details class="iv-answer"><summary>Tampilkan jawaban singkat</summary>…` dan pola serupa untuk full answer, wrong answers, dan expected answer follow-up (`site/build-interviews.js:244,288,290,317`) — sehingga jawaban **tersembunyi sampai diklik**. `build-interviews.js --check` menguji bahwa jawaban tidak muncul di luar `<details>` (baris 473–495).Terakhir, `node --test site/test_skills_page.js` (`SPAN = 24`, `ANSWER_SECTIONS` berisi Short answer/Full answer/Follow-ups/Wrong answers/Walkthrough/Resolution/Wrong turns/Generalize/Taught in) menggagalkan tutor skill jika memuat 24 karakter beruntun dari bagian jawaban bank.

---

## 4. `practice/` — latihan kode dengan mutant dan empat lapis tes

### 4.1 Struktur satu soal

`practice/<id>/` berisi `meta.json`, `problem.md`, `starter.py`, `solution.py`, `tests.py`, `fixtures.json` (ditulis validator dari `solution.py`), `hints.md` (2–5), `notes.md`, `mutants/*.py` (3–6), dan opsional `support.py`. Situs menampilkan `problem.md` + starter; tes dan hint dimuat saat learner menjalankan kode; `solution.py` dan `notes.md` hanya dimuat setelah semua tes lulus. Terdapat **52 direktori soal** ditambah `_harness/`.

`meta.json` nyata (`practice/softmax-cross-entropy/meta.json`): `id`, `title`, `kind: implement`, `summary`, `difficulty: easy`, `tags`, `lesson: phases/03-deep-learning-core/05-loss-functions`, `cheatsheet: deep-learning-training`, `minutes: 20`, `entry: [softmax, cross_entropy, cross_entropy_grad]`, `collections: [deep-learning-core]`, `time_limit_s: 10`, dan 5 mutant (`no_max_shift`, `sum_not_mean`, `global_max`, `in_place_shift`, `grad_missing_onehot`) lengkap dengan deskripsi bug. Koleksi didefinisikan di `practice/collections.json` (12 koleksi: math-foundations, classical-ml, deep-learning-core, optimizers, transformers-from-scratch, llm-inference, rl-basics, rag-building-blocks, agent-building-blocks, papers-to-code, train-tiny-models, experiment-labs).

### 4.2 Empat jenis tugas

| `kind` | Yang ditulis learner | Yang diukur tes |
|---|---|---|
| `implement` | fungsi/kelas kecil dari pelajaran | contoh, properti, dan referensi |
| `research` | metode inti dari satu paper (slug di `research/methods.json`, citation arXiv hanya abstract page) | persamaan + properti paper + referensi |
| `train` | `train(...)` dengan seed & step budget | model di held-out set; cek steps, determinisme, input tidak berubah |
| `lab` | eksperimen singkat + `answers()` | tiap jawaban dibandingkan dengan reference run (data asli repo, mis. `manuals/sglang-101/capture/gpu/summary/l4.json`) |

Contoh `train`: `budget: {steps: 2000}`, `target: {metric: "held-out accuracy", value: 0.95}`; learner memanggil `report(step, loss)` untuk kurva loss live. Contoh `lab` di direktori: `lab-bias-variance`, `lab-learning-rate-sweep`, `lab-quantization-bits`, `lab-sampling-temperature`, `lab-sglang-l4-serving`.

### 4.3 Validator

`tests.py` memakai `practice/_harness/practice_check.py` dengan `@check(layer, name, explain)` dan helper `close`, `equal`, `shape`, `true`, `raises`, `fail`, `fixture`/`fixtures`, `numeric_grad`. Empat lapis (masing-masing minimal satu check): `examples` (Run + Submit), `properties` (invarians, finite-difference gradient, input tidak berubah), `edge` (input terkecil, nilai besar, mask, ties), `reference` (fixture dari `solution.py`, disimpan di `fixtures.json`, <80 KB, seed tetap).

`python3 scripts/practice/validate.py <id> --write` menulis fixtures + memeriksa satu soal; tanpa argumen memeriksa semua (CI). Yang dicek: kelengkapan `meta.json`, eksistensi lesson, deskripsi setiap mutant, 4 seksi `problem.md` dan tidak ada baris solusi di soal, solusi lulus semua check dalam time limit, starter gagal, **setiap mutant mengubah 1–12 baris solusi dan gagal minimal satu check**, setiap lapis punya check, dan `fixtures.json` sesuai keluaran solusi terkini. Learner lokal memakai `python3 scripts/practice/run.py <id> my_answer.py [--examples]`.

---

## 5. `projects/` — proyek bertahap dengan grader lokal

### 5.1 Katalog dan 3 proyek unggulan

`projects/README.md:3` menyatakan katalog mencakup **104 proyek: 79 `ready` dan 25 di roadmap**. Hitungan lokal `project.json`: 79 `ready`, 1 `draft`. Tiga proyek yang ditampilkan di `README.md` root (baris 149–217):

| # | Proyek | Bahasa | Fokus | Barang yang disimpan |
|---|---|---|---|---|
| 01 | **Retrieval Evaluation Lab** (`projects/retrieval-evaluation-lab/`) | Python 3.10+ | Ranking metrics & regression check | Perbandingan per-query dengan delta; metrik menggambarkan judgments, bukan membuktikan jawaban benar |
| 02 | **Agent Trace Debugger** (`projects/agent-trace-debugger/`) | TypeScript + Node 22.18+, Python untuk grader | Parsing JSONL, validasi parent, interval arithmetic, timeline | Input trace, timeline HTML, JSON regression report; token per-span eksklusif tidak dihitung ganda |
| 03 | **Tool Call Firewall** (`projects/tool-call-firewall/`) | Rust + Python 3.10+ | Envelope call, role/path check, approval sekali pakai | Audit receipt berisi operasi + keputusan policy; bukan sandbox OS atau otorisasi persisten |

Ketiganya memakai pola perintah yang sama:

```bash
python3 scripts/project_test.py <id> --init learning-artifacts/<id>
python3 scripts/project_test.py <id> --stage 1 --path learning-artifacts/<id> --strict
python3 scripts/project_test.py <id> --all --path learning-artifacts/<id> --strict
```

Selain itu ada ~76 proyek `ready` lain (Bookmark Path Organizer, Semantic Notes Search, Mini Decompiler, MCP Tool Discovery Workbench, Browser Agent, dll.) dan 25 rencana di `projects/roadmap.json` / `projects/ROADMAP.md`.

### 5.2 Kontrak `projects/AUTHORING.md`

- 4–8 stage inkremental; tiap stage menambah satu perilaku, menjelaskan contoh konkret, dan menguji implementasi learner.
- `projects/<id>/project.json` memiliki `id`, `title`, `tagline`, `summary`, `level` (1–5), `hours`, `languages`, `status` (`draft`/`ready`), `source` (`core`/`community`), dan `stages` non-kosong dengan ID unik. Sebelum `ready`, README, solution, docs tiap stage, tes, dan starter harus ada.
- Tiap stage: `stages/<stage-id>/{docs/en.md, starter/, tests/}`; implementasi referensi kumulatif di `solution/`. Inisialisasi menyalin **semua** file biasa (`.rs`, `.ts`, `.go`, fixture, module) dan tidak menimpa file yang sudah ada kecuali `--force`.
- Setiap stage ≥5 tes bermakna: input biasa, boundary, malformed, dan satu kasus gagal/adversarial; satu held-out case terpisah dari fixture demo. Tes menguji kontrak yang diamati, bukan salinan algoritma referensi.
- Figure mekanisme wajib: `figure` fence + provider SVG asli di `site/figures/projects/<id>.js` yang mendaftar `window.AIFSProjectFigures.register('pj-<id>-1', {title, steps, caption})`. Boleh ada `lab` dengan kontrol (`range`, `number`, `text`, `select`, `checkbox`) dan `calculate(values, stepIndex)` yang mengembalikan ringkasan, metrik, bar, serta opsional `columns`/`rows`.

### 5.3 Grader `scripts/project_test.py`

| Bahasa | Runner bawaan |
|---|---|
| Python | `unittest` menemukan `tests/test_*.py`; `PROJECT_WORKSPACE` masuk `PYTHONPATH` |
| TypeScript | Node ≥22.18, `--experimental-strip-types`, `node --test` pada `tests/*.test.ts`/`.mjs`; import via `pathToFileURL` |
| Rust | `rustc --edition 2021 --test` per `tests/*.rs`; `include!(concat!(env!("PROJECT_WORKSPACE"), "/main.rs"))`; tanpa Cargo |
| Go | salin workspace + `tests/*.go` ke tmp, `go test -json ./...`; `GO111MODULE=off` bila tanpa `go.mod` |

Campuran memakai `language: "rust+python"` + `runners`. Substitusi argv: `{workspace}`, `{project}`, `{stage}`, `{tests}`. Variabel lingkungan: `PROJECT_WORKSPACE`, `PROJECT_ROOT`, `PROJECT_STAGE` (absolute). `requires: ["rustc"]` menghasilkan `skip` bila alat tidak ada; `--strict` menjadikan skip sebagai gagal; skip level tes selalu menghalangi completion evidence. Timeout maksimal 600 dtk. Runner opsional (`--optional`) memakai dependency declaration `python:module`/`node:package` dan tidak memengaruhi bukti inti.

Flag penting: `--stage N` (cek 1..N), `--only` (stage tunggal), `--all` (tanpa ID = semua proyek `ready`), `--solution`, `--strict`, `--report <file>`, `--list`, `--init`, `--path`, `--force`. Laporan JSON `schemaVersion: 1` berisi `generatedAt`, `projects`, `certificateEligible`; per stage ada `status` (`pass`/`fail`/`skip`), `tests`, `skippedTests`, `durationMs`, `reason`, `runners`. **Eligible hanya bila semua stage berjalan dan lulus ≥1 tes tanpa skip**; ini bukti lokal self-reported, bukan sertifikasi kriptografi atau anti-cheat.

### 5.4 Konvensi "jalankan dengan masukan Anda sendiri"

`AGENTS.md` proyek mewajibkan perintah berdocumentasi yang menerima input learner sendiri dan menggabungkan fungsi yang diajarkan — demo hardcoded saja tidak cukup. Contoh `projects/retrieval-evaluation-lab/README.md:25-33`:

```bash
python3 learning-artifacts/retrieval-evaluation-lab/cli.py \
  projects/retrieval-evaluation-lab/examples/baseline.json \
  projects/retrieval-evaluation-lab/examples/candidate.json \
  projects/retrieval-evaluation-lab/examples/judgments.json \
  --out retrieval-diff.json
```

Bagian ini secara eksplisit meminta learner mengganti path contoh dengan file sendiri, menyebut skema output `schema_version: 1`, batas integrasi (`--fail-on-regression` mengembalikan exit 1), dan cara membandingkan `solution/` dengan workspace sendiri. Tiap proyek juga punya `media/run.gif`, `media/output.gif`, poster PNG, dan `demo.command` di manifest yang harus terminasi dan menampilkan output nyata.

---

## 6. `i18n/` — terjemahan tanpa biaya di branch terpisah

### 6.1 Dokumen panduan dan lokasi registry

`docs/i18n.md` adalah kontraknya. **Poin yang sering tertukar:** `languages.json` berada di **root repositori** (`C:\…\ai-engineering-from-scratch\languages.json`), bukan di dalam `i18n/`. Direktori `i18n/` berisi tepat **34 file**: 33 `i18n/<lang>/README.md` + `i18n/readme-source.sha256`. Terjemahan pelajaran **tidak ada** di `main` — ia hidup di branch `translations` pada path `i18n/<lang>/…/<lang>.md` dan di-fetch situs saat runtime dari `raw.githubusercontent.com`.

| Keping | Lokasi | Di `main`? |
|---|---|---|
| Registry bahasa | `languages.json` | Ya (1 file kecil) |
| Walker terjemahan pelajaran | `scripts/translate_lessons.py` (prose-only) | Ya |
| README + audit struktural | `scripts/build_readme_i18n.py`, `scripts/audit_readme_locales.py` | Ya |
| Workflow CI | `.github/workflows/translate.yml` (shard per bahasa × phase) | Ya |
| Markdown pelajaran terjemahan | branch `translations` | Tidak |
| README per bahasa | `i18n/<lang>/README.md` | **Ya** |
| Buku terjemahan | release asset `aiefs-vol<n>-<slug>-<lang>.epub/.pdf` | Tidak |
| Opsi switcher | `site/langs.js` (generate; gitignored) | Tidak |
| String chrome UI | `site/ui-strings.json` + `scripts/translate_ui_strings.py` + `site/ui-i18n.js` | Ya (sumber & runtime) |
| `i18n/<lang>/ui.json` | branch `translations` | Tidak |

Cakupan: hanya pelajaran di `phases/`. Kurikulum sertifikasi tetap English-only dan sengaja dikecualikan dari penerjemahan mesin **maupun** book generation.

### 6.2 NLLB-200 dan alur kerja

- Mesin default **NLLB-200** (No Language Left Behind, Meta AI 2022, 200 bahasa) berjalan di GitHub Actions runner — **tanpa API key, tanpa tagihan, tanpa Vercel function/KV**. Opsi `--provider anthropic|openai|deepl` tersedia untuk kualitas lebih tinggi dengan biaya puluhan dolar per pass per bahasa; kunci dibaca dari `LLM_API_KEY`.
- `languages.json` memuat **41 entri** (1source `en` + 40 non-source). Tiap entri punya `code`, `name`, `native`, `nllb` (kode FLORES-200, wajib untuk NLLB), opsional `locale`, `dir: rtl`, `ci: true` (auto lesson + UI + switcher), `site: true` (switcher + UI tanpa auto-terjemahkan pelajaran). Yang `ci: true`: `zh, hi, es, ar, fr, pt, pt-BR, tr, vi, fa`; `zh-TW` memakai `site: true` agar terjemahan hasil PR tidak ditimpa.
- **33 tujuan README non-Inggris** (zh, zh-TW, ja, ko, pt, pt-BR, es, de, fr, it, nl, pl, cs, ro, hu, el, sv, da, no, fi, ru, uk, tr, he, ar, fa, hi, bn, ur, th, vi, id, tl) + English = 34 di language bar. Sebagian bahasa di registry (ms, ta, te, mr, gu, kn, sw) belum punya README lengkap.
- Workflow: push ke `main` yang mengubah `phases/**/docs/en.md`, registry, string UI, atau skrip memicu `translate.yml`; `translate_matrix.py` membuat **satu job per (bahasa, phase)** — satu pass 523 pelajaran butuh ~27 jam (melebihi limit 6 jam), sedangkan phase terbesar (19, 85 pelajaran) ±4,5 jam. Setiap job mem-push hanya slice `i18n/<lang>/phases/<phase>/`-nya dengan fetch-and-retry.
- **Cache sha256** per (bahasa, phase) di `i18n/<lang>/.cache/<phase>.json` dipublish ke branch `translations`; hanya pelajaran yang bahasa Inggrisnya berubah yang diterjemahkan ulang. Job yang dibatalkan tidak publish dan akan diperbaiki oleh push berikutnya; `translation_freshness.py` melaporkan current/stale/missing/orphaned.
- Situs: switcher men-set `?lang=<code>`; `fetchLesson` mengambil markdown terjemahan dan **fallback ke English** bila belum ada. `site/ui-i18n.js` menerjemahkan chrome di browser, mengabaikan code/lesson markdown/quiz/judul, dengan `MutationObserver`; RTL menandai teks yang tidak diterjemahkan dengan `lang="en" dir="ltr"`.

### 6.3 Perlindungan saat terjemahan

Walker adalah *prose-only*: fence code, tabel, baris gambar, dan header metadata lewat utuh; pada baris lain, inline `` `code` ``, `$math$`, referensi gambar, markdown link lengkap, `**bold**` term, dan URL dilindungi placeholder, lalu dikembalikan. Verifikasi mekanis: walker dengan "translation" identik mereproduksi **523 pelajaran byte-for-byte**; bila placeholder gagal round-trip, pelajaran itu tetap English (fail-safe).

`i18n/readme-source.sha256` menyimpan fingerprint English yang telah direview (kecuali reader count, page view, dan tanggal). Audit membandingkan jumlah heading/prose blok, tautan, anchor, 528 baris pelajaran, label bahasa, struktur tabel, fence/inline code, contoh perintah, topologi Mermaid, dan fakta angka — tetapi tidak dapat menyertifikasi kelancaran/nuansa teknis.

---

## 7. `glossary/terms.md`, `docs/`, `research/`, `outputs/`

### 7.1 Glossary → situs

`glossary/terms.md` (≈1.969 baris) adalah sumber kanonik ±ratusan istilah dalam **12 kategori**: Math & training; Models & inference; Data & representations; Retrieval & generation; Prompting & context; Agents & tools; Evaluation & safety; AI-native development; Infrastructure & serving; Reliability & operations; Security & governance; Multimodal systems.

Bentuk entri (contoh "Activation Checkpointing", "Agent", "Agent Harness"): `### Term`, lalu bullet `Category`, `What people say` (opsional), `What it actually means`, `Why it matters`, `In practice`, `Common confusion`, `Learn it` (tautan ke `../phases/...`), `Related terms`, `Sources`. Jika definisi umum bertentangan dengan kontrak API/model card terbaru, **dokumentasi resmi yang menang**.

Aliran ke situs:

- `site/build.js:29` mendefinisikan `GLOSSARY_PATH`; baris 2208–2243 membaca dan mem-parse file itu menjadi `glossaryTerms` menggunakan `parseGlossary` dengan `GLOSSARY_FIELD_KEYS` (baris 1703–1715) dan `GLOSSARY_CATEGORY_ORDER`.
- `glossaryTerms` dimasukkan ke `data.js` (baris 2304: `const GLOSSARY = …`) dan ke sitemap, `llms.txt`, serta statistik build (`terms: glossaryTerms.length`, baris 2284/2325).
- Parser memvalidasi tautan `Learn it` relatif terhadap direktori glossary (baris 1943) dan melempar error dengan format `glossary/terms.md:<line>` (baris 1717–1720).
- `site/build-hubs.js:506` membuat halaman entri per term dengan atribusi "`This entry comes from glossary/terms.md on GitHub`"; `site/glossary.html` menyediakan filter lokal dengan fallback ke file GitHub bila JS mati.
- `AGENTS.md` mewajibkan memperbarui `glossary/terms.md` ketika memperkenalkan istilah yang dipakai lebih dari satu pelajaran.

### 7.2 `docs/`

Hanya dua file:

- `docs/i18n.md` — dibahas di §6.
- `docs/portfolio-writing-sync.md` — kontrak sinkronisasi koleksi `/blogs` dan reader `/blog/<slug>/`, `/guides/<slug>/` dari feed `https://rohitghumare.com/api/content` (repo `rohitg00/portfolio`). Tidak ada daftar konten kedua/webhook; cache 5 menit pada kedua layanan, stale valid hingga 24 jam saat source down, 503 saat cold start, retry 30 detik. Artikel mempertahankan canonical asli dan sengaja tidak masuk sitemap kursus. Verifikasi: `node --test site/test_portfolio_sync.js site/test_writing_mirror.js site/test_portfolio_cache.js site/test_writing_seo.js`, `node site/build.js`, `node site/build-projects.js --strict`, `node site/build-manuals.js`, `node site/version-assets.js`, dan `node scripts/preview-writing.js <checkout portfolio>`.

### 7.3 `research/`

Satu file: `research/methods.json` — registry `version: 1` berisi pemetaan slug → nama metode (mis. `direct-preference-optimization-dpo` → "DPO", `key-value-cache` → "KV cache", `flash-attention` → "FlashAttention", `agent-harness` → "Agent harness"). Dipakai oleh soal `practice` jenis `research`: slug di `meta.json.method` harus ada di registry dan memetakanLesson yang sama; paper arXiv dicantumkan sebagai satu sitasi di bawah pernyataan soal. Perhatikan juga bahwa tiap manual memiliki `research/brief.md` dan `research/sources/` sendiri (lihat §1.2).

### 7.4 `outputs/`

Struktur saat ini:

```text
outputs/
  index.json          {"version":"1.0.0","prompts":[],"skills":[],"agents":[],"mcp_servers":[]}
  agents/.gitkeep
  mcp-servers/.gitkeep
  prompts/.gitkeep
  skills/.gitkeep
```

Artinya: `outputs/` adalah **tempat terpusat yang saat ini masih kosong** untuk artefak yang dihasilkan pelajaran (prompt, skill, agent, MCP server). Artefak nyata justru dikirim per pelajaran di `phases/NN-phase/MM-lesson/outputs/` (konvensi `AGENTS.md`: `outputs/skill-<slug>.md` opsional) dan per manual/release (PDF). `index.json` berfungsi sebagai manifest agregat dengan versi `1.0.0` dan empat kategori kosong — bukan daftar artefak yang sudah terisi.

---

## 8. Catatan lintas subsistem dan implikasi

1. **Sumber tunggal, turuhan ganda.** `topics.yaml` (cheatsheet), `topics.yaml`+`roles.yaml` (interview), `collections.json` (practice), dan `project.json`/`roadmap.json` (projects) semuanya adalah *registry*; konten turunannya dihasilkan ulang. MenambahLesson baru berarti memperbarui minimal README, ROADMAP, topics.yaml (cheatsheet), dan bila relevan glossary.
2. **Bukti sebelum narasi.** Capture kit manual → listing manual → evidence interview → angka soal interview; capture GPU → ringkasan `l4.json` → jawaban lab practice; fixture `solution.py` → tes reference practice. Rantai ini sengaja dibuat untuk mencegah angka "dari ingatan".
3. **Generate ≠ publish.** Cheatsheet generator menolak menulis bila fact-check/rubrik gagal; manual draft tidak masuk indeks; proyek draft tidak tampil sebagai ready; terjemahan yang gagal round-trip tetap English; PDF manual dan buku terjemahan adalah release asset.
4. **Biaya dikunci.** NLLB gratis dengan cache sha256 dan shard per phase; generator cheatsheet memakai Haiku + token budget 1,5 juta + maksimal 2 retry + eskalasi, dengan ringkasan biaya per topik. Tidak ada yang bergantung pada API key untuk build normal.
5. **Konsistensi gaya.** Simplified Technical English dipakai bersama di manual, cheat sheet, interview, dan practice; AGENTS.md menambahkan aturan fenced code block ber-tag dan diagram Mermaid/SVG saja. Aturan 24 karakter muncul dua kali dengan alasan berbeda: interview melindungi prompt dari jawaban, dan test skills melindungi bank dari kebocoran ke tutor.
6. **Celah yang teramati (fakta, bukan rekomendasi perubahan):** clue "9 manuals" sebenarnya 8 manual + `_shared/`; clue `i18n/languages.json` sebenarnya ada di root; `outputs/` masih placeholder; hanya 1 loop wawancara (`inference-engineer`) dari 7 peran; hanya 1 proyek berstatus `draft` meskipun README menyebut 25 planned (rencana hidup di `roadmap.json`, bukan sebagai direktori project.json); dan semua angka cakupan di atas adalah hitungan lokal pada checkout ini, bukan jaminan sinkron dengan CI `main`.

---

### Daftar jalur bukti utama

- `manuals/AUTHORING.md`, `manuals/manual.schema.json`, `manuals/sglang-101/manual.json`, `manuals/sglang-101/sections/3-4-prefix-friendly-prompts.md`, `manuals/sglang-101/capture/README.md`, `manuals/_shared/figkit.js`, `scripts/audit_manuals.js`
- `cheatsheets/AUTHORING.md`, `cheatsheets/topics.yaml`, `cheatsheets/evidence/sglang-serving.md`, `site/build-cheatsheets.js`, `site/cheatsheet-evidence.js`, `scripts/cheatsheets/generate.js`
- `interviews/AUTHORING.md`, `interviews/topics.yaml`, `interviews/roles.yaml`, `interviews/questions/inference-serving/prefix-cache-order.md`, `interviews/scenarios/cold-start-on-gb10/README.md`, `interviews/loops/inference-engineer.md`, `scripts/interviews/validate.js`, `scripts/interviews/denylist.txt`, `site/build-interviews.js`, `site/test_skills_page.js`
- `practice/AUTHORING.md`, `practice/softmax-cross-entropy/meta.json`, `practice/collections.json`, `practice/_harness/practice_check.py`, `scripts/practice/validate.py`, `scripts/practice/run.py`
- `projects/AUTHORING.md`, `projects/README.md`, `projects/roadmap.json`, `projects/retrieval-evaluation-lab/project.json`, `projects/retrieval-evaluation-lab/README.md`, `scripts/project_test.py`
- `docs/i18n.md`, `docs/portfolio-writing-sync.md`, `languages.json`, `scripts/translate_lessons.py`, `site/ui-i18n.js`, `i18n/readme-source.sha256`
- `glossary/terms.md`, `site/build.js`, `site/build-hubs.js`, `site/glossary.html`, `research/methods.json`, `outputs/index.json`, `AGENTS.md`
