# Analisis 05 — Automasi, CI, Validasi, dan Tata Kelola Repositori

**Objek analisis:** `C:\Users\Hendra\Downloads\ai-engineering-from-scratch`
**Fokus:** `scripts/` (103 berkas), `.github/workflows/` (12 workflow), `.coderabbit.yaml`, `AGENTS.md`, `CHANGELOG.md`, `FORKING.md`, `CODE_OF_CONDUCT.md`, `BACKERS.md`, `SPONSORS.md`, `LICENSE`, `.gitignore`, `LESSON_TEMPLATE.md`
**Tanggal analisis:** 24 November 2026
**Catatan:** tidak ada berkas di dalam repositori yang diubah; semua temuan bersifat baca-saja.

---

## 1. Ringkasan eksekutif

Repositori ini bukan aplikasi SaaS melainkan **kurikulum 542 pelajaran / 20 fase**, sehingga automasinya berorientasi pada *konsistensi konten*, bukan pada deploy layanan. Tiga lapisan tata kelola terlihat jelas:

1. **Kontrak manusia** di `AGENTS.md` + `LESSON_TEMPLATE.md` + `.github/PULL_REQUEST_TEMPLATE.md`: aturan penulisan, ukuran commit, gaya conventional commit, dependency allowlist, dan kewajiban menjawab review bot.
2. **Gate deterministik** di `scripts/` (103 berkas): audit struktur pelajaran, sertifikasi, manual, kuis, bias jawaban, terjemahan, proyek, praktik, dan bank wawancara; sebagian besar berjalan tanpa dependency berat.
3. **CI GitHub Actions** di `.github/workflows/` (12 workflow): 1 workflow inti (`curriculum.yml`, 6 job), 5 workflow validasi domain, 3 workflow generator bertenaga AI dengan batas model/token/biaya, 3 workflow publikasi buku/manual/cheat sheet PDF, dan 1 workflow terjemahan NLLB-200 lokal tanpa API key.

Temuan penting:

- `curriculum.yml` jauh lebih luas daripada ringkasan tabel “CI gates” di `AGENTS.md`: doc menyebut 4 gate, workflow aktual menjalankan **6 job** dan puluhan tes situs/API.
- Generator AI memakai **Haiku sebagai default, eskalasi Sonnet sekali, Opus hanya manual**, dengan **token budget 600.000** (cheat sheet & wawancara) dan **1.500.000** (praktik), serta **guard PR terbuka** agar tidak menumpuk PR bot.
- Terjemahan sengaja **tidak memakai API berbayar**: NLLB-200 distill 600M berjalan di runner, di-*shard* per `(bahasa, fase)`, di-cache menurut hash, dan dipublikasikan ke cabang `translations`, bukan `main`.
- File yang **dilarang masuk diff PR**: `catalog.json`, `site/data.js` (direbuild bot di `main`), `package-lock.json` (digit ignore), plus README hasil generate dan file turunan situs.
- Ada **ketegangan dokumentasi**: `LESSON_TEMPLATE.md` dan PR template meminta “no comments in code”, sementara `AGENTS.md` mewajibkan header komentar 4–6 baris yang menyitasi `docs/en.md`.
- `check_quiz_bias.py` memiliki baseline `0.84` — artinya **84% soal kuis saat ini terdeteksi “length-biased”**; gate hanya mencegah memburuk, bukan memperbaiki.
- `AGENTS.md` bertanda **“Last reviewed: 2026-10-23”**; `SPONSORS.md` terverifikasi **2026-10-07**; `CHANGELOG.md` enteri terbaru **2026-04** — changelog tertinggal jauh dari aktivitas CI saat ini.

---

## 2. Konteks versi dan tanggal

| Sumber | Penanda waktu | Makna |
|---|---|---|
| `AGENTS.md` (akhir berkas) | `Last reviewed: 2026-10-23` | Kontrak kontributor terakhir ditinjau ~32 hari sebelum tanggal analisis. |
| Konteks sesi | `Today's date: Tue Nov 24 2026` | Basis perbandingan untuk keusangan dokumen. |
| `SPONSORS.md:41` | `All figures were verified 2026-10-07` | Angka reach sponsor memiliki tanggal verifikasi eksplisit. |
| `scripts/check_mcpa_wire.py` | MCP wire shape `2026-07-28` | Transkrip pelajaran MCPA wajib cocok dengan versi protokol ini; CI menyebutnya eksplisit di `curriculum.yml:157`. |
| `scripts/audit_certifications.py:58-69` | guide version `1.0`, effective `July 2026`, rentang skor 100–1000, biaya USD 99–175 | Fakta ujian Claude dipinjamkan ke dalam konstanta auditor. |
| `CHANGELOG.md:15,34` | `2026-04 — Phase 4`, `2026-Q1 and earlier` | Tidak ada entri untuk onChange log April 2026 sampai November 2026. |
| Workflow | contoh tag `v2026.11`, `v2026.10.2` | Pola penamaan rilis bulanan/versi patch. |

---

## 3. Peta `scripts/` — 103 berkas

Distribusi hasil penghitungan direktori:

| Direktori | Jumlah | Peran dominan |
|---|---:|---|
| `scripts/` (akar) | 51 | audit, generator, quality check, scaffold, translator, tes regresi |
| `scripts/decompiler-machine/` | 11 | aset mesin isometrik + model + tangkapan data |
| `scripts/gpu-machine/` | 11 | aset mesin GPU + data SGLang hasil tangkapan |
| `scripts/agent-machine/` | 7 | aset mesin agen + build/ekspor |
| `scripts/interviews/` | 7 | validator + generator bank wawancara + denylist |
| `scripts/practice/` | 7 | validator + harness + generator soal praktik |
| `scripts/kit/` (di bawah `agent-machine`) | 5 | pustaka geometri/ink isometrik bersama + LICENSE |
| `scripts/cheatsheets/` | 4 | generator + prompt + report + tes |
| **Total** | **103** | |

### 3.1 Audit dan invariant checks

| Berkas | Apa yang diperiksa | Perintah khas |
|---|---|---|
| `scripts/audit_lessons.py` | Aturan `L001`–`L010`: pola direktori `NN-slug`; keberadaan & UTF-8 `docs/en.md`; panjang minimal 200 byte; H1; `code/` tidak kosong; skema `quiz.json` kanonik (`stage/question/options/correct/explanation`), penolakan skema legacy `q/choices/answer`, 2–6 opsi, indeks `correct` valid; tautan internal yang tidak resolve. | `python3 scripts/audit_lessons.py [--phase N] [--json] [--strict]`; CI memakai bentuk polos. |
| `scripts/audit_certifications.py` | Audit kurikulum sertifikasi: ID, fenced code berbahasa, gaya prosa (tanpa em/en dash), kualitas jawaban, 5 heading paritas (`Interactive Lab`, `Practice Lab`, `Shipped Artifact`, `Verify It`, `Capstone Connection`), figure fence terdaftar, 5+ tes & demo runnable, kuis 1 pre + 3 check + 2 post, graf prasyarat, halaman publik, konstanta fakta ujian per provider. | `python3 scripts/audit_certifications.py [--json]` |
| `scripts/audit_manuals.js` | Structured audit manual: batas panjang kalimat per `kind`, larangan dash/semicolon/kontraksi/intensifier/lexicon marketing, SMIL (`begin=indefinite`, `fill=freeze`, `keyTimes`/`keySplines`), batas SVG 40 KB & viewBox ≤760, kelas warna yang diizinkan, provenance listing, verbatim quote, dan **capture drift** dengan menjalankan `manual.capture.check`. Mustahil lulus tanpa `TOTAL 0`. | `node scripts/audit_manuals.js [--manual <id>] [--skip-capture]`; CI: `node scripts/audit_manuals.js`. |
| `scripts/audit_lesson_figures.js` | Audit geometri figure lesson di browser: label utuh, terpisah, dan readable. Konstanta: `MIN_TEXT_PX = 9`, toleransi `0.5`, timeout 20 s, viewport desktop 1280×2400 & phone 375×2400. | `CHROME_PATH=... NODE_PATH=... node scripts/audit_lesson_figures.js --shots <dir>` |
| `scripts/audit_readme_locales.py` | README terjemahan menjaga struktur, tautan, anchor, angka, tabel, dan blok kode Inggris; **bukan** penilaian kelancaran bahasa. | `python3 scripts/audit_readme_locales.py` (diuji via `test_audit_readme_locales.py`) |
| `scripts/check_mcpa_wire.py` | Transkrip pelajaran MCPA terhadap invarian wire MCP `2026-07-28`: request wrapper, error shape, result types. | `python3 scripts/check_mcpa_wire.py` |
| `scripts/backfill_certification_references.py` | Memastikan setiap butir assessment memiliki referensi remediasi berupa jalur pelajaran internal; deterministik & idempoten. | `python3 scripts/backfill_certification_references.py --check` |
| `scripts/check_translation_coverage.js` | Membandingkan catatan cache terjemahan dengan `docs/<lang>.md` yang benar-benar terbit di cabang `translations`; mendeteksi terjemahan tercatat tapi hilang, atau doc tanpa rekaman. | `node scripts/check_translation_coverage.js --ref origin/translations` |
| `scripts/translation_freshness.py` | Membandingkan sha256 sumber Inggris di cache per bahasa/fase dengan `docs/en.md` terkini: stale / missing / orphan. Dipakai matrix untuk memperbaiki leg yang gagal. | `python3 scripts/translation_freshness.py [--ref origin/translations] [--summary FILE]` |
| `scripts/test_agent_readiness.py` | Kontrak permukaan discovery untuk agen (`llms.txt`, manifest, identitas situs). | `python3 scripts/test_agent_readiness.py` |
| `scripts/interviews/validate.js` | Kontrak bank wawancara: front matter, bagian, tautan relatif, angka ≥10/desimal harus bersumber, denylist perusahaan, evidence byte-copy, jawaban terlipat. | `node scripts/interviews/validate.js [--write --check-readmes]` |
| `scripts/practice/validate.py` | Setiap solusi lulus, starter & mutant gagal, fixture mutakhir, pernyataan tidak membocorkan solusi. File wajib: `problem.md`, `starter.py`, `solution.py`, `tests.py`, `hints.md`, `notes.md`, `meta.json`, `fixtures.json`. | `python3 scripts/practice/validate.py --summary <file>` |
| `scripts/project_test.py` | Grader stage proyek kumulatif dengan stdlib runner; solusi referensi **tidak** memenuhi syarat sertifikat peserta. | `python3 scripts/project_test.py --all --solution --strict --report /tmp/project-results.json` |

### 3.2 Generator, auto-sync, dan katalog

| Berkas | Fungsi | Perintah |
|---|---|---|
| `scripts/build_catalog.py` | Memindai semua `phases/NN-slug/MM-slug/` dan menulis `catalog.json` (schema_version 1): fase, pelajaran, file kode, outputs (skill/prompt/agent), totals. | `python3 scripts/build_catalog.py [--out PATH] [--stdout]` |
| `scripts/check_readme_counts.py` | Mencocokkan angka hardcode di `README.md` dengan `catalog.json.totals`; `--fix` menulis ulang README. | `python3 scripts/check_readme_counts.py [--json] [--fix]` |
| `scripts/build_readme_i18n.py` | Menjaga `i18n/<lang>/README.md` sebagai dokumen editorial penuh (bukan render otomatis): cek tautan, anchor, fakta angka, tabel, kode; menyinkronkan statistik. | `python3 scripts/build_readme_i18n.py [--dump] [--check]` |
| `scripts/build_book.py` | Merakit volume buku dari pelajaran dan merender dengan pandoc; EPUB selalu, PDF dengan xelatex saat `--pdf`. | `python3 scripts/build_book.py [--volume language] [--pdf] [--assemble-only]` |
| `scripts/build-favicons.js` | Membangun favicon deterministik dari grid piksel berbasis zlib (tanpa aset eksternal). | `node scripts/build-favicons.js` |
| `scripts/build-og-fonts.js` | Membangun atlas glyph untuk social card `/og/<type>/<id>.png`. **Hanya di mesin developer** dengan Chrome; tidak di CI/Vercel. | `node scripts/build-og-fonts.js` |
| `scripts/print_cheatsheets.js` | Print HTML cheat sheet → PDF memakai Chrome + PDF footer bermerek. | `node scripts/print_cheatsheets.js <print.html> <out.pdf>` |
| `scripts/record_demo.py` | Mengubah transkrip perintah / tangkapan browser menjadi GIF (Pillow + ffmpeg; hanya untuk authoring). | `python3 scripts/record_demo.py ...` |
| `scripts/scaffold-lesson.sh` | Membuat `phases/<phase>/<lesson>/` lengkap dengan `code/`, `notebook/`, `docs/`, `outputs/`, serta skeleton `docs/en.md` dari `LESSON_TEMPLATE.md`. | `scripts/scaffold-lesson.sh <phase-dir> <lesson-slug> [title]` |
| `scripts/scaffold_workbench.py` | Menginstal paket “Agent Workbench” ke repo target: `AGENTS.md`, `docs/`, `schemas/`, `scripts/`, `VERSION`, state seed. | `python3 scripts/scaffold_workbench.py <target_dir> [--force] [--minimal] [--dry-run] [--no-seed]` |
| `scripts/install_skills.py` | Menginstal artifact `phases/**/outputs/{skill,prompt,agent}-*.md` dan bundle `.../<name>/SKILL.md` ke target dengan filter type/phase/tag dan layout `flat/by-phase/skills`. | `python3 scripts/install_skills.py <target_dir> [--type ...] [--phase N] [--tag TAG] [--layout ...]` |
| `scripts/serve-agent-site.js` | Server lokal untuk menguji handler API/MCP (`/api/markdown`, `/api/v1/catalog`, `/api/mcp`, `/api/og`, dll.). | `node scripts/serve-agent-site.js` |

### 3.3 Pemeriksaan kualitas, penghilang bias, dan pemeriksaan konten

| Berkas | Fungsi | Perintah |
|---|---|---|
| `scripts/check_quiz_bias.py` | Mengukur *length bias*: jawaban benar dianggap bias bila paling panjang dan ≥ `MIN_RATIO = 1.25` kali distractor terpanjang. `BASELINE_RATE = 0.84` adalah ratchet: gate gagal hanya bila angka naik. | `python3 scripts/check_quiz_bias.py [--report] [--check]` |
| `scripts/debias_quizzes.py` | Menyusun ulang posisi jawaban benar secara deterministik ber-seed konten; idempoten; melewati opsi yang saling merujuk (“semua di atas”). | `python3 scripts/debias_quizzes.py [--check]` |
| `scripts/debias_certification_questions.py` | Sama, untuk bank soal sertifikasi; cycle posisi benar per berkas, idempoten. | `python3 scripts/debias_certification_questions.py [--check]` |
| `scripts/check_lesson_math.js` | Memastikan matematika render di luar code span dan uang tetap berupa teks, memakai KaTeX pinned. | `node scripts/check_lesson_math.js --katex <katex.min.js> [phase prefix]` |
| `scripts/find_plain_math.py` | Melaporkan (tanpa mengubah) matematika teks biasa yang bisa jadi LaTeX; opsi `--stray-dollars`. | `python3 scripts/find_plain_math.py [--list] [--phase 01] [--stray-dollars]` |
| `scripts/link_check.py` | HEAD/GET semua URL http(s) di markdown, cache 7 hari di `.link-cache.json` (gitignored), stdlib saja. | `python3 scripts/link_check.py [--phase 14] [--cached] [--ignore ...]` |
| `scripts/lesson_run.py` | Byte-compile seluruh `.py` pelajaran; `--execute` menjalankan dengan timeout 10 s dan melewati berkas ber-header `# requires: pkg`. | `python3 scripts/lesson_run.py [--execute] [--phase N]` |
| `scripts/run_lesson_tests.py` | Menjalankan test suite tiap pelajaran lewat `unittest`; dependency opsional (torch, numpy, scipy, ..., cv2) yang tidak terpasang → **skip**, bukan fail; error impor modul sendiri → fail. | `python3 scripts/run_lesson_tests.py [--timeout 120] [--phase NN-slug]` |

### 3.4 Terjemahan (tanpa API berbayar)

| Berkas | Fungsi |
|---|---|
| `scripts/translate_matrix.py` | Membaca `EVENT/REQUESTED/REQUESTED_PHASE/BEFORE/AFTER/FORCED`, membedah commit range, dan mencetak `lessons=` serta `ui_langs=` untuk `$GITHUB_OUTPUT`. Hanya kode bahasa terdaftar & direktori fase yang ada yang masuk matrix. |
| `scripts/translate_lessons.py` | Terjemahan NLLB-200 untuk prosa saja; code fence, inline code, math, figure/mermaid, tautan, gambar, dan header metadata **dipertahankan byte-for-byte**. Hash-cache per pelajaran; output ke `i18n/<lang>/...` pada cabang `translations`. |
| `scripts/translate_ui_strings.py` | Menerjemahkan `site/ui-strings.json` menjadi `i18n/<lang>/ui.json`; `overrides` mengunci terjemahan label pendek; terjemahan lama dipakai ulang sehingga hanya key baru yang diterjemahkan. |
| `scripts/readme_translations.py` | Peta blok README hasil terjemahan manual sebagai **alat bantu draft**; bukan dasar publikasi (publikasi lewat `build_readme_i18n.py`). |

Dependensi runtime terjemahan berada di `.github/translate-requirements.txt`: CPU wheel index PyTorch, `torch==2.5.1`, `transformers==4.46.3`, `sentencepiece==0.2.0`.

### 3.5 Generator konten bertenaga AI

Ketiga generator memakai pola yang sama: `prompt.md` → generator JS → `summary.json` → `report.js` → gerbang validasi → PR bot. Model dipetakan di kode menjadi `claude-haiku-5-5`, `claude-sonnet-5-5`, `claude-opus-5-5` (`scripts/cheatsheets/generate.js:11-13`, diimpor ulang oleh generator wawancara dan praktik).

| Generator | Input | Default & batas | Escalation | Gerbang sebelum PR |
|---|---|---|---|---|
| `scripts/cheatsheets/generate.js` | topic id dari `cheatsheets/topics.yaml`, sumber topik, 3 contoh | `haiku`, `--limit 5`, `--token-budget 600000` dari workflow (default internal 1.500.000) | `sonnet` sekali bila evaluasi gagal; Opus hanya manual | `site/test_cheatsheets.js`, `test_cheatsheet_topics.js`, `scripts/cheatsheets/test_generate.js`, `node site/build-cheatsheets.js --check`, `--check-quality` |
| `scripts/interviews/generate.js` | `card` + `count` (maks 5), memakai writer cheat sheet | `haiku`, `600000` token, `MAX_ITEMS = 5`, `MAX_FILE_CHARS = 60000`, `MAX_BUNDLE_CHARS = 300000` | mengikuti writer | `validate.js`, `test_validate.js`, `test_generate.js`, `site/test_interviews.js`, `build-interviews.js --check`; menolak README ter-stage |
| `scripts/practice/generate.js` | jalur pelajaran `phases/NN-slug/MM-slug` | `haiku`, `--limit 5`, `--token-budget 1500000` | `sonnet` | `python3 scripts/practice/validate.py`, `node site/build-practice.js --check` |

Pembatas biaya lain: `PRICES` per model (input/output/cache read/cache write), `usageTokens`, `callCost`, `budgetLeft`, dan `output_config.effort = medium` untuk Haiku / `high` untuk model lain (`scripts/cheatsheets/generate.js:254`). `scripts/interviews/denylist.txt` memuat nama perusahaan yang dilarang muncul di bank wawancara.

### 3.6 Mesin isometrik dan aset figure

Direktori `scripts/agent-machine/`, `scripts/decompiler-machine/`, `scripts/gpu-machine/`, dan `scripts/kit/` bukan validator melainkan **sumber model + geometri + runtime browser** untuk figure interaktif:

- `kit/iso-kit.mjs`, `kit/ink.mjs`, `kit/lathe.mjs`, `kit/tube.mjs`: proyeksi isometrik, lathe, tabung, dan helper tinta; `kit/LICENSE` mengatur paket ini.
- `agent-machine/build.mjs`, `export.js`, `model.mjs`, `content.mjs`, `geometry.mjs`, `live.js`, `page.css`: state machine agen → SVG + simulasi halaman.
- `decompiler-machine/*`: `model.mjs` mendefinisikan `STAGES` (header → sections → disasm → cfg → lift → types → structure → pseudocode → ledger), `PROGRAMS`, `SWEEPS`, `OPTS`, `CONFIDENCE`; `fixture.mjs`, `capture.py`, `timing.mjs` menyimpan bukti eksekusi (tanggal mesin: Darwin arm64, Python 3.14.6, best-of-5 `timeit`).
- `gpu-machine/*`: `data.mjs` berisi hasil tangkapan SGLang (`manuals/sglang-101/capture/gpu/summary/*.json`) dan log `KV Cache is allocated...`; `extract.mjs` mengekstrak data itu; `hardware.mjs` menyimpan `SOURCES` spesifikasi NVIDIA; `model.mjs` mendefinisikan `GPUS`, `DTYPES`, `USERS`, `SHARED`, `WORKLOADS`, `CLOCKS`.

Karena aset ini diverifikasi oleh `scripts/test_agent_audit_lesson_figures.js` dan tes `site/test_*_machine.js`, perubahan model harus sinkron dengan halaman.

### 3.7 Tes regresi di dalam `scripts/`

`test_*.py` / `test_*.js` berikut menguji **infrastruktur**, bukan pelajaran: `test_link_check.py`, `test_book_rendering.py`, `test_check_mcpa_wire.py`, `test_audit_readme_locales.py`, `test_skill_artifact_bundles.py`, `test_translate_ui_strings.py`, `test_translate_workflow.py`, `test_audit_lesson_figures.js`, `test_function_bundles.js`, `test_seo_routes.js`, `test_agent_api.js`, `test_agent_wire.js`, `test_api_state.js`, `test_agent_negotiation.js`, serta tes generator di `scripts/{cheatsheets,interviews,practice}/test_generate.js` dan `scripts/practice/test_harness.py`. `_lib.py` menyediakan parser frontmatter YAML-subset, containment check, dan penemuan bundle skill tanpa dependency luar.

---

## 4. Workflow GitHub Actions

### 4.1 Ringkasan 12 workflow

| Workflow | Trigger | Job utama | Gate / keluaran | Concurrency & guard |
|---|---|---|---|---|
| `.github/workflows/curriculum.yml` | push & PR ke `main` dengan path filter sangat luas | `audit`, `lesson-tests`, `lesson-ports`, `readme-counts-sync`, `site-rebuild`, `readme-counts-drift` | Audit inti + tes situs/API + auto-fix README & data.js | Tidak ada blok `concurrency`; `permissions: contents: read` kecuali job sync (write) |
| `.github/workflows/build-book.yml` | push `main` (phases/book/script), `workflow_dispatch`, `release: published` | `book` | EPUB (push), EPUB+PDF (manual/release), artifact & `gh release upload --clobber` | Pandoc 3.10.1 dengan checksum SHA-256; xelatex hanya manual/release |
| `.github/workflows/cheatsheets.yml` | PR/push path filter, release, dispatch dengan input `tag` | `check`, `pdf` | `--check`, `--check-quality`, PDF ke artifact/release | `group: cheatsheets-${ref}`, cancel PR; timeout 10/20 mnt; validasi regex tag |
| `.github/workflows/cheatsheets-generate.yml` | `schedule: 17 4 * * 1`, `workflow_dispatch` | `generate` | PR `cheatsheets/auto-<run_id>` | `group: cheatsheets-generate`, cancel=false; API key guard; 5 topik; 600k token; timeout 120 mnt |
| `.github/workflows/interviews.yml` | PR/push path filter | `validate`, `readmes` | validasi bank + regenerasi README di `main` | `group: interviews-${ref}`, cancel PR; PR tidak boleh mengubah README hasil generate |
| `.github/workflows/interviews-generate.yml` | `workflow_dispatch` saja | `generate` | PR `interviews/auto-<run>-<attempt>` | `group: interviews-generate`, cancel=false; guard PR terbuka; count ≤5; 600k token; timeout 60 mnt |
| `.github/workflows/lesson-figures.yml` | PR/push pada `docs/en.md`, `site/figures*.js`, audit script | `geometry` | audit label figure + upload screenshot gagal | `group: lesson-figures-${ref}`, cancel=true; timeout 25 mnt |
| `.github/workflows/manuals.yml` | PR/push, release, dispatch `tag` | `audit`, `pdf` | `TOTAL 0`, PDF Paged.js ke artifact/release | `group: manuals-${ref}`, cancel PR; regex tag `^v\d{4}\.\d{2}\.\d+$`; cache Paged.js |
| `.github/workflows/practice.yml` | PR/push path filter | `validate` | solusi lulus / starter & mutant gagal; `--check` | `group: practice-${ref}`, cancel PR; numpy dipasang; timeout 20 mnt |
| `.github/workflows/practice-generate.yml` | `workflow_dispatch` saja | `generate` | PR `practice/auto-<run_id>` | `group: practice-generate`, cancel=false; 5 soal; 1,5 juta token; timeout 120 mnt |
| `.github/workflows/projects.yml` | PR/push path filter | `offline-projects` | `build-projects.js --strict` + `project_test.py --all --solution --strict` | timeout 20 mnt; Python/Node/Go/Rust hadir |
| `.github/workflows/translate.yml` | push `main` pada doc Inggris/languages/ui-strings/script; dispatch `langs`, `phase` | `prepare`, `translate`, `coverage`, `ui-strings` | terbit ke cabang `translations`, cek coverage & freshness | `group: translate`, cancel=false; matrix `max-parallel: 20`, `fail-fast: false`; timeout 330/60 mnt |

### 4.2 `curriculum.yml` — workflow inti

Path filter memicu workflow pada perubahan `phases/**`, `certifications/**`, `skills/**`, `.claude/skills/**`, hampir seluruh `scripts/*.py|js` yang relevan, `site/**`, `api/**`, `lib/**`, `README.md`, `i18n/*/README.md`, `ROADMAP.md`, `glossary/**`, `assets/**`, `CONTRIBUTING.md`, `SPONSORS.md`, `BACKERS.md`, `languages.json`, `vercel.json`, dan file workflow itu sendiri.

**Job `audit` (“pemeriksaan invarian”)** — checkout dengan `persist-credentials: false`, Python 3.12, seluruh action di-*pin* ke SHA (mis. `actions/checkout@34e1148...`, `actions/setup-python@a26af69...`). Step dikelompokkan:

- Audit Python: `audit_lessons.py`, `audit_certifications.py`, `backfill_certification_references.py --check`, semua `certifications/*/lessons/**/code/tests/test_*.py` dan `code/main.py`, `test_check_mcpa_wire.py`, `unittest scripts.test_link_check`, `check_mcpa_wire.py`, `test_skill_artifact_bundles.py`.
- Tes situs/API dengan `node --test`: artifact bundle, asset cache, lesson math, copy commands, home curiosity, skills page, lesson screen, star nudge, certification state, theme menu/overlays/TTS, lesson walkthrough, quiz placement, tiga machine (agent/decompiler/gpu), highlights, practice, interview bank + `build-interviews.js --check`, code render, `llms.txt`, function bundle (`@vercel/nft@1.5.0` di `$RUNNER_TEMP`), API/MCP, SEO routes, agent readiness, negotiation Markdown, static preview routes, clean URLs, portfolio sync, learning paths, sponsor placement, UI i18n.
- Build situs penuh: `node site/build.js && node site/build-projects.js --strict && node site/build-agents.js` (dua kali) plus pemeriksaan `grep -c '^## API and MCP$' site/llms.txt = 1`.
- Quality gates: `debias_quizzes.py --check`, `check_quiz_bias.py --check`, `debias_certification_questions.py --check`, `build_readme_i18n.py --check`, `test_audit_readme_locales.py`, `test_translate_workflow.py`, `test_translate_ui_strings.py`.
- Mirror skill: `diff -r skills .claude/skills` — komentar di workflow menegaskan `skills/` itu kanonik untuk `npx skills add`, `.claude/skills/` hanya mirror untuk Claude Code.

**Job `lesson-tests`** menjalankan `python3 scripts/run_lesson_tests.py`. Tidak ada dependency ilmiah yang dipasang, sehingga pelajaran berat di-*skip*; ini sengaja menangkap kelas kerusakan “pelajaran gagal mengimpor modulnya sendiri”.

**Job `lesson-ports`** mengompilasi/menjalankan port:

| Bahasa | Pengaturan |
|---|---|
| Rust | `rustc --edition 2021 -D warnings -O`, timeout 120 s, melewati berkas dengan `Cargo.toml` |
| Julia | `julia-actions/setup-julia` v1, hanya file berpenanda “Julia standard library only”, timeout 300 s |
| Go | Go 1.24, `gofmt -l` harus kosong, `go vet`, `go run`, timeout 120 s |
| TypeScript | Node 22, `tsx@4.19.2`, `typescript@5.6.3`, `@types/node@22.10.2`, `tsc --strict --noEmit --target es2022 --module nodenext`; `lab.ts` & `experiments/*.ts` tidak dijalankan |

**Job auto-sync (hanya push ke `main`):**

- `readme-counts-sync`: build katalog ephemeral → `check_readme_counts.py --fix` → `build_readme_i18n.py` → commit bot `chore(readme): sync counts`. Deteksi perubahan memakai `git status --porcelain` (sengaja, agar README i18n baru yang masih untracked terdeteksi). Ada *loop guard*: bila commit terakhir sudah berawalan prefix yang sama, berhenti. Push retry 5× dengan `git fetch` + `git rebase` + backoff `2n` detik; konflik di-abort dan selesai dengan `exit 0`.
- `site-rebuild`: `needs: readme-counts-sync`, menjalankan `node site/build.js`, commit bot `chore(site): rebuild data.js`, retry loop identik.
- `readme-counts-drift` (hanya PR): menjalankan pengecekan dan mengeluarkan `::warning::README.md counts drift detected. Main will self-heal on merge.` — **advisory, tidak menggagalkan PR**.

### 4.3 Generator workflows — model, token, dan biaya

`cheatsheets-generate.yml`:

- Schedule Senin 04:17 UTC (`17 4 * * 1`) atau dispatch dengan `topics`, `model`, `max_topics`, `token_budget`.
- Langkah `key` mengecek `secrets.ANTHROPIC_API_KEY`; bila kosong → notice, `ready=false`, semua langkah berikut dilewati (tidak gagal).
- `plan`: model default Haiku, whitelist `haiku|sonnet|opus`, limit default 5, budget default 600.000; `node site/build-cheatsheets.js --status` menghasilkan daftar `missing` + `stale`; `gh pr list` menghitung PR cabang `cheatsheets/` yang masih terbuka — bila ada dan bukan dispatch, tidak ada kerja. ID disaring regex `^[a-z0-9-]+$`.
- SDK dipasang di `$RUNNER_TEMP/gen` dengan `--no-save --no-package-lock`; generator dipanggil `--escalate sonnet --limit --token-budget`; `|| true` diikuti `test -s summary.json` sehingga kegagalan per-topik tidak menggagalkan job, tetapi ringkasan wajib ada.
- Gerbang harus lulus sebelum PR dibuka; `git checkout -- site/data.js` mencegah artifact generate ikut ter-commit.

`interviews-generate.yml`: dispatch saja; input `card` wajib dengan regex slug, `count` dipaksa ≤5, budget 600.000; guard PR `interviews/auto-` yang berasal dari repo sendiri; generator memakai ulang `writer` cheat sheet; PR menolak staging `README.md`.

`practice-generate.yml`: dispatch saja; `lessons` wajib dan tiap jalur harus cocok `^phases/[0-9]{2}-[a-z0-9-]+/[0-9]{2}-[a-z0-9-]+/?$`; budget default 1.500.000; memasang numpy dan SDK di luar repo; gerbang `validate.py` + `build-practice.js --check` sebelum PR.

**Ringkasan guard biaya & konkurensi:** API key guard; Haiku default; eskalasi Sonnet satu kali; Opus manual-only; batas item per run; token budget; `concurrency` dengan `cancel-in-progress: false` untuk generator (anh relationships antar-PR tidak saling menimpa); `timeout-minutes` 60–120; dependency di `$RUNNER_TEMP` dengan `--no-save --no-package-lock`; `PUPPETEER_SKIP_DOWNLOAD=1`; cache Paged.js/HuggingFace; dan PR otomatis sehingga manusia tetap dapat meninjau sebelum merge.

### 4.4 Workflow publikasi & validasi domain

- **Buku** (`build-book.yml`): pandoc diunduh dari GitHub release dan diverifikasi SHA-256 `b419369...`; `mermaid-cli` bersifat `continue-on-error`; xelatex hanya saat release/dispatch; artifact EPUB 30 hari, PDF 90 hari; lampiran rilis memakai `gh release upload ... --clobber`.
- **Cheat sheet & manual PDF**: keduanya memasang alat cetak di luar repo, memakai Chrome sistem, menghasilkan PDF lalu meng-upload ke artifact (dispatch) atau attachment rilis. `manuals.yml` menjalankan `audit` sebelum `pdf` dan hanya mempublikasi manual `--ready` saat rilis/dispatch dengan tag.
- **Lesson figures**: audit visual memakai `puppeteer-core@24.10.0` + `CHROME_PATH`; bila gagal, screenshot di-upload sebagai artifact 7 hari.
- **Project contracts**: menjalankan lima tes kontrak situs/proyek, `build-projects.js --strict`, dan seluruh stage proyek offline dengan solusi referensi.
- **Interview & practice validation**: keduanya menolak perubahan pada file hasil generate saat PR; regenerasi README hanya terjadi di `main` oleh bot. `interviews.yml` berbeda dari `curriculum.yml`: setelah 5 kali gagal push, ia keluar dengan `exit 1` (keras), bukan `exit 0`.

### 4.5 `translate.yml` — terjemahan lokal, sharded, dan race-safe

Komentar di bagian atas workflow menjelaskan desainnya: NLLB-200 adalah model open gratis yang jalan di runner — **tanpa API key, tanpa biaya Vercel**. Satu bahasa penuh (~503 pelajaran) memakan ~27 jam CPU, melewati batas job 6 jam; karena itu matrix dipecah per `(bahasa, fase)` sehingga leg terbesar (fase 19, 85 pelajaran) ~4,5 jam. Setiap pelajaran di-cache menurut hash, jadi hanya sumber Inggris yang berubah yang diterjemahkan ulang.

- `prepare` → `scripts/translate_matrix.py` menentukan leg yang perlu dijalankan, termasuk memperbaiki leg yang source hash-nya tidak cocok lagi.
- `translate` → matrix `lang × phase`, `max-parallel: 20`, `fail-fast: false`, timeout 330 menit, model `facebook/nllb-200-distilled-600M`, cache `~/.cache/huggingface`, runtime dari `.github/translate-requirements.txt`.
- Publikasi memakai **detached worktree** di `$RUNNER_TEMP`, 8 retry dengan jitter acak 3–14 detik, `git add -f` karena `.gitignore` sengaja mengecualikan `i18n/*/phases/` dan `i18n/*/.cache/` di `main`. Hanya slice `i18n/<lang>/phases/<phase>` dan cache-nya yang disalin, sehingga shard paralel tidak saling menimpa.
- `coverage` menjalankan `check_translation_coverage.js` dan `translation_freshness.py`.
- `ui-strings` menerjemahkan `site/ui-strings.json` ke `i18n/<lang>/ui.json` dengan publisher race-safe yang sama; override yang sudah dipin atau terjemahan lama dipakai ulang sehingga sering tidak memuat model sama sekali.

---

## 5. Kontrak validasi per-PR dan perilaku CI

### 5.1 Yang harus dijalankan kontributor (dari `AGENTS.md`)

```bash
python3 scripts/audit_lessons.py
python3 scripts/audit_certifications.py
python3 scripts/check_readme_counts.py        # advisory — CI memperbaiki di main
node --test site/test_manuals.js && node scripts/audit_manuals.js   # bila manuals/ berubah
node site/build-cheatsheets.js --check && node site/build-cheatsheets.js --check-quality
node site/build.js && node --test site/test_*.js   # bila site/ berubah; lalu git checkout -- site/data.js
CHROME_PATH=<chrome> NODE_PATH=<dir puppeteer-core>/node_modules node scripts/audit_lesson_figures.js   # harus TOTAL 0

# untuk setiap pelajaran yang disentuh:
cd phases/NN-phase/MM-lesson/code
python3 main.py && python3 -m unittest discover tests -v
```

Catatan: `TOTAL 0` disebut oleh `AGENTS.md` untuk audit figure, tetapi string `TOTAL ${total}` secara eksplisit ada di `scripts/audit_manuals.js:380`; keduanya memakai konvensi “nol temuan = lulus”.

### 5.2 Gate CI menurut `AGENTS.md` versus realita workflow

| Gate di tabel `AGENTS.md` | Realita di `curriculum.yml` |
|---|---|
| `audit` — push + PR, blocking | Benar, tapi isinya jauh lebih besar: puluhan tes situs/API, bias, i18n, mirror skill, build statis. |
| `readme-counts-sync` — main only | Benar; juga menyinkronkan `i18n/*/README.md` dan punya retry loop + loop guard. |
| `site-rebuild` — main only | Benar; `needs: readme-counts-sync`. |
| `readme-counts-drift` — PR, advisory | Benar; hanya `::warning::`. |
| *(tidak tercantum)* | `lesson-tests`, `lesson-ports`, dan 11 workflow domain lain (buku, cheat sheet, wawancara, figure, manual, praktik, proyek, terjemahan + 3 generator). |

**Rekomendasi dokumentasi:** perbarui tabel CI di `AGENTS.md` agar mencakup 6 job inti dan tautan ke workflow domain; selisih ini bisa menyesatkan kontributor yang hanya mengandalkan doc.

### 5.3 File hasil generate yang tidak boleh di-commit di PR

| Berkas | Status | Penanganan |
|---|---|---|
| `catalog.json` | gitignored (`.gitignore:91`) | Dibangun ephemeral oleh `scripts/build_catalog.py` di CI; jangan di-commit. |
| `site/data.js` | **tidak** gitignored, tetapi dilarang masuk diff PR | CI `site-rebuild` menulisnya di `main` dengan commit bot; PR template mencantumkan “`site/data.js` is not in the diff”. Lokal: `git checkout -- site/data.js` setelah `node site/build.js`. |
| `package-lock.json` | gitignored (`.gitignore:19`) | Semua pemasangan CI memakai `--no-save --no-package-lock`. |
| `site/sitemap*.xml`, `site/llms.txt`, `site/build-meta.js`, `site/*-seo.json`, `site/og-cards.json`, `site/lesson-code/`, `site/interviews/`, `site/practice/`, `site/cheatsheets/`, dll. | gitignored (`.gitignore:93-153`) | Dibangun ulang oleh `site/build.js` atau builder khusus pada deploy. |
| `i18n/*/phases/`, `i18n/*/.cache/`, `i18n/*/ui.json` | gitignored, tapi **di-`git add -f`** ke cabang `translations` | Pengecualian yang disengaja: `main` tetap English-only; terjemahan hidup di cabang terpisah. `i18n/<lang>/README.md` justru *commit* di `main`. |
| README hasil generate (interview) | Dicegah oleh `interviews.yml` saat PR | `validate.js --write --check-readmes` hanya di `main`. |

---

## 6. Konvensi commit, review bot, konflik, dan dependency allowlist

### 6.1 Commit

- Subject **conventional commit**, maksimal **72 karakter**, format utama: `feat(phase-NN/MM): <slug>`.
- Body menjelaskan **mengapa**, bukan daftar perubahan mekanis.
- Ukuran commit adalah keputusan editorial: perbaikan kecil 1–2 commit; perubahan besar beberapa commit logis yang masing-masing bisa dibaca sendiri.
- Contoh prefix lain yang benar-benar dipakai bot/workflow: `chore(readme): sync counts`, `chore(site): rebuild data.js`, `chore(i18n): <lang>/<phase> (NLLB-200)`, `chore(interviews): regenerate the generated READMEs`, `chore(cheatsheets): new and updated sheets`, `feat(practice): new problems`, `feat(interviews): new questions for <card>`.
- Hindari `git push --force` ke branch dengan komentar review terbuka karena akan melepaskan thread.

### 6.2 Review bot

`AGENTS.md` aturan keras 8: **jawab setiap thread CodeRabbit dan Devin** — perbaiki temuan atau balas alasan ketidakberlakuannya, lalu resolve thread. `.github/PULL_REQUEST_TEMPLATE.md:24` mengulang kewajiban ini dalam checklist.

Konfigurasi `.coderabbit.yaml` sengaja minimal dan menekan noise:

| Kunci | Nilai | Efek |
|---|---|---|
| `reviews.high_level_summary` | `false` | tanpa ringkasan otomatis panjang |
| `reviews.poem` | `false` | tanpa “poem” |
| `reviews.changelog` / `release_notes` | `false` | bot tidak menghasilkan changelog/release notes |
| `reviews.collapse_walkthrough` | `true` | walkthrough dilipat |
| `reviews.path_filters` | `!manuals/*/capture/out/**`, `!.../cassettes/**`, `!.../research/sources/**`, `!manuals/*/figures/*.svg` | artifact tangkapan, sumber riset, dan SVG generation tidak direview baris-demi-baris |

### 6.3 Resolusi konflik (dari `AGENTS.md`)

```bash
git fetch origin main
git merge --no-edit origin/main
```

- Konflik `catalog.json` (branch legacy): `git rm catalog.json && git commit --no-edit` — berkas sekarang gitignored.
- Konflik count README: `git checkout --theirs README.md`, `python3 scripts/build_catalog.py`, `python3 scripts/check_readme_counts.py --fix`, `git add README.md && git commit --no-edit`.
- Konflik `site/data.js`: `git checkout --theirs site/data.js`, `node site/build.js`, `git add site/data.js && git commit --no-edit`.
- Lanjut `git push origin <branch>`; hindari force-push bila ada review thread terbuka.

### 6.4 Dependency allowlist

| Bahasa | Diizinkan | Rasional |
|---|---|---|
| Python | `numpy`, `torch`, `h5py`, `zstandard`, `safetensors`, stdlib | Mendukung latihan numerik & format model tanpa menjadikan manajer paket bagian dari kurikulum. |
| TypeScript | `hono`, `zod`, `ws` (hanya bila WebSockets diperlukan), `@hono/node-server`, Node 20+ stdlib | Server HTTP & validasi skema cukup; `ws` bersifat kondisional. |
| Rust | **stdlib saja**, satu berkas, `rustc --edition 2021` | Port harus tetap dapat dikompilasi tanpa Cargo. |
| Julia | `Random`, `Statistics`, `LinearAlgebra`, `Printf` (stdlib) | Numerik dasar tanpa ekosistem paket tambahan. |

Aturan tambahan: **stdlib-first**; bila review bot menyarankan dependency terlarang, lewati dengan alasan persis *“tetap mengutamakan stdlib demi kejelasan edukatif.”*

Perlu dibedakan: dependency **tooling repositori** tidak sama dengan dependency pelajaran. CI memakai `tsx`, `typescript`, `@types/node`, `@vercel/nft`, `puppeteer-core`, `pdf-lib`, `katex`, `pagedjs-cli`, `@anthropic-ai/sdk`, `numpy` untuk practice, dan torch/transformers/sentencepiece untuk terjemahan — semuanya dipasang di runner atau `$RUNNER_TEMP`, bukan menjadi deklarasi pelajaran. Namun `lesson-ports` mengompilasi port **Go** dengan stdlib Go 1.24/1.23, padahal tabel allowlist `AGENTS.md` tidak mencantumkan Go — ini inkonsistensi kecil yang sebaiknya ditambahkan ke doc.

### 6.5 Kontrak pelajaran dan ketegangan dokumentasi

`AGENTS.md` mewajibkan frontmatter `Type`, `Languages`, `Prerequisites`, `Time`, serta `## Learning Objectives` (4–6 bullet), header komentar 4–6 baris pada `main.*` yang menyitasi `docs/en.md`, dan minimal 5 tes. `LESSON_TEMPLATE.md` memakai struktur naratif (`The Problem`, `The Concept`, `Build It`, `Use It`, `Ship It`, `Exercises`, `Key Terms`, `Further Reading`) dan menyatakan **“No comments — code should be self-explanatory”** (`LESSON_TEMPLATE.md:96`); `.github/PULL_REQUEST_TEMPLATE.md:19` mengulanginya. Dua aturan ini bertentangan dengan header wajib di `AGENTS.md` — perlu diputuskan mana yang berlaku (misalnya: “tidak ada komentar selain header sitasi”). Selain itu `audit_lessons.py` tidak memeriksa kesesuaian `**Languages:**` dengan file `main.*`; pemeriksaan mendalam semacam itu hanya ada untuk pelajaran sertifikasi di `audit_certifications.py`.

---

## 7. Lisensi, berkas komunitas, dan changelog

### 7.1 Lisensi

`LICENSE` = **MIT License**, Copyright (c) **2026 Rohit Ghumare**, dengan klausul standar (permisi tanpa batasan + notice + “AS IS” tanpa garansi). `FORKING.md` menegaskan konsekuensinya:

- Tim: fork, buang fase yang tidak perlu, tambahkan contoh/data internal dan integrasi alat, **pertahankan atribusi**.
- Sekolah/universitas: petakan fase ke kalender semester, tambahkan rubric & tugas, kontribusi balik opsional.
- Bootcamp berbayar: dibolehkan MIT; struktur untuk cohort, video/live/mentoring; dianjurkan sponsor atau kontribusi balik.
- Bahasa pemrograman lain: fork + re-implementasi contoh, pertahankan struktur lesson, ajukan PR untuk ditautkan di README.
- Sinkron fork: `git remote add upstream https://github.com/rohitg00/ai-engineering-from-scratch.git`, `git fetch upstream`, `git merge upstream/main`.
- Atribusi tidak wajib secara hukum, tapi dianjurkan: “Based on AI Engineering from Scratch” + URL repo.

### 7.2 Tata kelola komunitas

| Berkas | Isi |
|---|---|
| `CODE_OF_CONDUCT.md` | Janji bebas pelecehan; standar perilaku positif & negatif; pelaporan ke `ghumare64@gmail.com`; diadaptasi dari Contributor Covenant **2.1**. |
| `BACKERS.md` | Ucapan terima kasih; sponsor SerpApi (Web Search API) dan CodeRabbit (maintenance kurikulum); dukungan infrastruktur dari Vercel Open Source Program; daftar dikelola maintainer, detail di `SPONSORS.md`. |
| `SPONSORS.md` | *Rate card* sponsorship: kurikulum gratis MIT, 542 pelajaran/20 fase; sponsor dikelola maintainer, **tidak** diterima lewat PR kontributor; pembayaran lewat GitHub Sponsors (`github: [rohitg00]` di `.github/FUNDING.yml`) atau email untuk invoice; **tanpa token/coin/NFT/kripto**. Berisi angka reach terverifikasi 2026-10-07 (page views 521.690, subscriber newsletter 11.000+, star 65.223, fork 11.242, dll.). |
| `.github/FUNDING.yml` | `github: [rohitg00]` + `custom: [SPONSORS.md]`. |
| `.github/ISSUE_TEMPLATE/bug_report.md`, `new_lesson_proposal.md` | Intake terstruktur untuk bug dan usulan pelajaran (disebut di changelog Unreleased). |
| `.github/PULL_REQUEST_TEMPLATE.md` | Checklist: kode jalan, built-from-scratch-first, struktur sesuai template, baris ROADMAP berupa markdown link, commit by judgment, thread review dijawab, `site/data.js` tidak masuk diff. |
| `.gitignore` | Mengabaikan artifact Python/Node/Rust/Julia, `dist/`, `build/`, credential, model/checkpoint, `catalog.json`, seluruh turunan situs, dan sebagian `.claude/` dengan **allowlist mirror skill** (`!.claude/skills/<nama>/**`) agar clone segar memuat tutor. |

### 7.3 Ruang lingkup `CHANGELOG.md` (100 baris pertama)

- Format “loosely follows Keep a Changelog”, urutan terbaru lebih dulu; tiap entri menyebut fase, pelajaran, dan perubahan agar pembaca bisa langsung membuka delta.
- `## [Unreleased]` → `### Added`: `scripts/scaffold-lesson.sh`, `.github/PULL_REQUEST_TEMPLATE.md`, dua issue template, dan `CHANGELOG.md` itu sendiri.
- `## 2026-04 — Phase 4: Computer Vision complete`: 28 pelajaran fase 4, baris ROADMAP berlink, serta precision pass 15+ pelajaran (contoh: `phase-4/06` memakai `AP@0.5`, `phase-4/25` `_merge` memunculkan `ValueError`, `phase-4/28` `rope_3d` memvalidasi split dimensi).
- `## 2026-Q1 and earlier`: Phase 0 (12), Phase 1 (22), Phase 2 (18), Phase 3 inti, skill Claude Code `find-your-level` & `check-understanding`, situs dengan 277 istilah glossary, scaffolding 20 fase, dan file dasar `LESSON_TEMPLATE.md`/`CONTRIBUTING.md`/`ROADMAP.md`/`README.md`.
- Penutup: link compare `[Unreleased]: .../compare/HEAD...HEAD`.

**Observasi:** changelog berhenti di April 2026 sementara workflow dan konstanta auditor sudah merujuk ke Juli 2026 (MCPA wire) dan November 2026 (contoh tag). Ini bukan kegagalan CI, melainkan utang dokumentasi rilis.

---

## 8. Temuan lintas-lapisan dan rekomendasi ringkas

| # | Temuan | Bukti | Rekomendasi |
|---|---|---|---|
| 1 | Tabel gate CI di `AGENTS.md` tertinggal dari implementasi. | `AGENTS.md` 4 baris vs `curriculum.yml` 6 job + 11 workflow lain | Perbarui tabel & sebutkan workflow domain. |
| 2 | Aturan komentar kode bertentangan. | `LESSON_TEMPLATE.md:96` & PR template:19 vs `AGENTS.md` header 4–6 baris | Tegaskan pengecualian header sitasi. |
| 3 | Allowlist tidak mencantumkan Go, tetapi CI mengompilasi port Go. | `AGENTS.md` Dependencies vs `curriculum.yml:292-304` | Tambahkan baris Go stdlib ke allowlist. |
| 4 | Bias panjang jawaban masih tinggi. | `check_quiz_bias.py:30` `BASELINE_RATE = 0.84` | Turunkan baseline secara bertahap sesuai komentar kode. |
| 5 | `audit_lessons.py` tidak memvalidasi frontmatter `Languages` vs `main.*`. | `audit_lessons.py:85-222` hanya L001–L010 | Tambahkan aturan audit bila kontrak ingin dipaksakan otomatis. |
| 6 | Pinning action tidak seragam. | `curriculum.yml` memakai SHA; `build-book.yml` & sebagian workflow lain memakai tag `@v4`/`@v5` | Pin semua action ke SHA untuk hardening supply chain. |
| 7 | `site/data.js` tidak di-gitignore tetapi dilarang di PR. | `.gitignore` tidak memuatnya; PR template & `curriculum.yml:394-427` mengaturnya | Pertahankan, tapi dokumentasikan alasan “generated-but-committed-by-bot” lebih tegas. |
| 8 | Changelog usang. | `CHANGELOG.md:15` terakhir 2026-04 vs today 2026-11-24 | Tambahkan entri untuk gelombang automation 2026-07 s.d. 2026-11. |
| 9 | Generator toleran gagal per-item (`|| true`), aman karena ada `summary.json` + gate. | `cheatsheets-generate.yml:116-127`, `interviews-generate.yml:114-125`, `practice-generate.yml:111-121` | Pertahankan; pastikan report selalu dibaca pengulas PR. |
| 10 | Terjemahan memakai `git add -f` karena ignore rule `main`. | `translate.yml:192-203`, `.gitignore:112-115` | Dokumentasikan pengecualian ini di `docs/i18n.md` bila belum ada. |

---

### Lampiran A — Daftar berkas `scripts/` menurut kategori

**Audit & validasi (akar):** `audit_lessons.py`, `audit_certifications.py`, `audit_manuals.js`, `audit_lesson_figures.js`, `audit_readme_locales.py`, `check_mcpa_wire.py`, `check_translation_coverage.js`, `backfill_certification_references.py`, `test_agent_readiness.py`, `link_check.py`, `check_lesson_math.js`, `check_quiz_bias.py`, `check_readme_counts.py`, `find_plain_math.py`, `lesson_run.py`, `run_lesson_tests.py`, `project_test.py`.

**Generator/build/sync:** `build_catalog.py`, `build_readme_i18n.py`, `build_book.py`, `build-favicons.js`, `build-og-fonts.js`, `print_cheatsheets.js`, `record_demo.py`, `preview-writing.js`, `scaffold-lesson.sh`, `scaffold_workbench.py`, `install_skills.py`, `serve-agent-site.js`, `debias_quizzes.py`, `debias_certification_questions.py`, `readme_translations.py`, `_lib.py`.

**Terjemahan:** `translate_matrix.py`, `translate_lessons.py`, `translate_ui_strings.py`, `translation_freshness.py`.

**Tes infrastruktur (akar):** `test_link_check.py`, `test_book_rendering.py`, `test_check_mcpa_wire.py`, `test_audit_readme_locales.py`, `test_skill_artifact_bundles.py`, `test_translate_ui_strings.py`, `test_translate_workflow.py`, `test_audit_lesson_figures.js`, `test_function_bundles.js`, `test_seo_routes.js`, `test_agent_api.js`, `test_agent_wire.js`, `test_api_state.js`, `test_agent_negotiation.js`.

**Subdirektori:** `cheatsheets/{generate.js,prompt.md,report.js,test_generate.js}`; `interviews/{denylist.txt,generate.js,prompt.md,report.js,test_generate.js,test_validate.js,validate.js}`; `practice/{generate.js,prompt.md,report.js,run.py,test_generate.js,test_harness.py,validate.py}`; `agent-machine/{build.mjs,content.mjs,export.js,geometry.mjs,live.js,model.mjs,page.css}`; `decompiler-machine/{build.mjs,capture.py,clock.mjs,content.mjs,fixture.mjs,geometry.mjs,live.js,model.mjs,page.css,parts.mjs,timing.mjs}`; `gpu-machine/{build.mjs,content.mjs,data.mjs,extract.mjs,geometry.mjs,hardware.mjs,live.js,materials.css,model.mjs,page.css,parts.mjs}`; `kit/{ink.mjs,iso-kit.mjs,LICENSE,lathe.mjs,tube.mjs}`.

### Lampiran B — Daftar workflow

`.github/workflows/curriculum.yml`, `build-book.yml`, `cheatsheets.yml`, `cheatsheets-generate.yml`, `interviews.yml`, `interviews-generate.yml`, `lesson-figures.yml`, `manuals.yml`, `practice.yml`, `practice-generate.yml`, `projects.yml`, `translate.yml`. Pendukung: `.github/FUNDING.yml`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/*`, `.github/translate-requirements.txt`, `.coderabbit.yaml`.

---

*Laporan disusun dari pembacaan langsung berkas; tidak ada perubahan pada repositori sumber.*
