# Analisis Struktur Kurikulum — `ai-engineering-from-scratch`

**Lokasi repositori:** `C:\Users\Hendra\Downloads\ai-engineering-from-scratch`
**Fokus analisis:** direktori `phases/` (20 fase, 542 pelajaran) dan kontrak yang mengaturnya
**Tanggal analisis:** 24 November 2026
**Metode:** inventarisasi filesystem dengan PowerShell/Python, pembacaan `AGENTS.md`, `CONTRIBUTING.md`, `README.md`, `ROADMAP.md`, `LESSON_TEMPLATE.md`, `site/build.js`, `scripts/audit_lessons.py`, serta sampel 4 pelajaran secara mendalam. Tidak ada file di dalam repositori yang diubah.

---

## 1. Ringkasan eksekutif

| Fakta | Angka | Bukti |
|---|---:|---|
| Fase | 20 direktori `00-*` sampai `19-*` | `phases/` |
| Pelajaran | 542 direktori bernomor `NN-*` | `phases/*/[0-9][0-9]-*/` |
| Dokumentasi `docs/en.md` | 542 (100%) | semua pelajaran |
| Direktori `code/` | 540 (99,6%) | 2 pelajaran tanpa `code/` |
| File `main.*` | 942 file, di 490 pelajaran | lihat §5.1 |
| `code/tests/` | 104 pelajaran (19,2%), 114 file, semuanya Python | lihat §5.3 |
| `quiz.json` | 392 file (72,3%) | lihat §5.2 |
| `outputs/` | 478 direktori (473 berisi artefak nyata) | lihat §5.4 |
| `notebook/` | 295 direktori, **semua hanya berisi `.gitkeep`** | lihat §7.4 |
| Fence `figure` | 553 fence di 530 dokumen | lihat §5.5 |

Filosofi inti repositori adalah **"Build It / Use It"**: setiap pelajaran mengharuskan pembaca membangun algoritma dari matematika dasar dengan pustaka standar, lalu menjalankan versi produksi (PyTorch, framework, dll.). Dokumentasi juga menambahkan **"Ship It"** sebagai tahap pengiriman artefak reusable. Kontrak ini tertulis di `AGENTS.md` (baris 21–60, 75–232) dan `CONTRIBUTING.md` (baris 54–110).

---

## 2. Peta level tinggi repositori

Struktur relevan dari `AGENTS.md` (baris 11–45):

```text
phases/
  NN-phase-slug/
    NN-lesson-slug/
      docs/en.md      # penjelasan pelajaran (wajib)
      code/           # implementasi + tests
      quiz.json       # 6 pertanyaan
      outputs/        # artefak reusable (skill / prompt / agent / MCP server)
README.md             # daftar pelajaran; diparser situs
ROADMAP.md            # status per fase & pelajaran; diparser situs
site/build.js         # mengubah README + ROADMAP + glossary -> site/data.js
scripts/audit_lessons.py  # aturan L001-L010
```

Di luar `phases/` terdapat banyak permukaan lain yang **tidak** termasuk kurikulum inti 542 pelajaran: `certifications/`, `manuals/`, `interviews/`, `projects/`, `cheatsheets/`, `book/`, `skills/`, `i18n/`, `web/`, `api/`, `lib/`. Laporan ini hanya menganalisis `phases/` dan kontrak yang menopangnya.

---

## 3. Dua puluh fase kurikulum

Nama fase dan estimasi jam diambil dari header `## Phase N: ... — ✅ (~X hours)` di `ROADMAP.md` (baris 11–563). Fokus topik dirangkum dari `<em>...</em>` pada header fase di `README.md` (mis. baris 449, 469, 1095) dan daftar pelajaran di `ROADMAP.md`.

| # | Direktori | Lessons | Est. (ROADMAP) | Fokus topik |
|---:|---|---:|---:|---|
| 0 | `00-setup-and-tooling` | 13 | ~16 jam | Lingkungan dev, Git, GPU/cloud, API keys, Jupyter, Docker, editor, data, shell/Linux, debugging, Python untuk AI (`ROADMAP.md:11-27`) |
| 1 | `01-math-foundations` | 22 | ~23 jam | Aljabar linear, kalkulus/chain rule, probabilitas & Bayes, optimasi, teori informasi, PCA/SVD, stabilitas numerik, Fourier, graf, proses stokastik (`ROADMAP.md:29-54`) |
| 2 | `02-ml-fundamentals` | 21 | ~26 jam | ML klasik: regresi, SVM, k-NN, clustering, feature engineering, evaluasi, ensemble, time series, drift, feature store (`ROADMAP.md:56-80`) |
| 3 | `03-deep-learning-core` | 13 | ~15 jam | Perceptron → backprop → optimasi/regularisasi → mini-framework → PyTorch & JAX (`ROADMAP.md:82-98`) |
| 4 | `04-computer-vision` | 28 | ~27 jam | "From pixels to understanding": image, video, 3D, VLM, world models |
| 5 | `05-nlp-foundations-to-advanced` | 29 | ~30 jam | Tokenisasi, embedding, seq2seq, advanced NLP; "Language is the interface to intelligence" |
| 6 | `06-speech-and-audio` | 17 | ~18 jam | ASR, audio representations, TTS — "Hear, understand, speak" |
| 7 | `07-transformers-deep-dive` | 16 | ~14 jam | Self-attention, multi-head, positional encoding, BERT/GPT/T5, MoE, KV cache/FlashAttention |
| 8 | `08-generative-ai` | 15 | ~14 jam | GAN/diffusion, video, audio, 3D, flow matching, visual autoregressive |
| 9 | `09-reinforcement-learning` | 12 | ~13 jam | Fondasi RL & RLHF, policy/value methods — "the foundation of RLHF and game-playing AI" |
| 10 | `10-llms-from-scratch` | 24 | ~26 jam | Tokenizer, arsitektur, pretraining, fine-tuning, inference optimization; studi DeepSeek-V3, Jamba, DualPipe |
| 11 | `11-llm-engineering` | 17 | ~19 jam | Prompting, RAG, evaluasi, caching/biaya, aplikasi produksi, integrasi MCP |
| 12 | `12-multimodal-ai` | 25 | ~65 jam | ViT patches → modality alignment → computer-use agents |
| 13 | `13-tools-and-protocols` | 45 | ~76 jam | Tool interface, function calling, MCP (server/client/transport/auth), A2A, gateway & registry |
| 14 | `14-agent-engineering` | 54 | ~55 jam | Agent loop, ReAct/planning, memory, skill libraries, workflow patterns, tool use |
| 15 | `15-autonomous-systems` | 22 | ~20 jam | Long-horizon agents, self-improvement, safety stack 2026 |
| 16 | `16-multi-agent-and-swarms` | 25 | ~28 jam | Koordinasi, emergensi, kolektif intelligence, A2A, swarm networks |
| 17 | `17-infrastructure-and-production` | 29 | ~34 jam | Serving, observability, A/B testing, deploy & operasional |
| 18 | `18-ethics-safety-alignment` | 30 | ~31 jam | Etika, jailbreak, prompt injection, red-team, alignment |
| 19 | `19-capstone-projects` | 85 | ~620 jam | 17 produk end-to-end + 9 deep-build track; 4–12 pelajaran per track (`README.md:1094-1189`) |

**Jumlah lessons:** 13+22+21+13+28+29+17+16+15+12+24+17+25+45+54+22+25+29+30+85 = **542**, sama dengan jumlah direktori dan dengan klaim README ("Free, MIT-licensed, 542 lessons", `README.md:1477`).

> **Catatan inkonsistensi jam:** `ROADMAP.md:7` menyatakan "Total estimated time: ~332 hours", tetapi penjumlahan 20 header fase menghasilkan **~1.170 jam** (Fase 19 menyumbang ~620 jam). Angka 332 kemungkinan sudah usang.

---

## 4. Anatomi sebuah pelajaran (kontrak)

### 4.1 Kontrak menurut `AGENTS.md` / `CONTRIBUTING.md`

| Elemen | Aturan | Bukti |
|---|---|---|
| `docs/en.md` | H1, hook `>`, frontmatter `**Type:**`, `**Languages:**`, `**Prerequisites:**`, `**Time:**`, lalu `## Learning Objectives` (4–6 bullet diawali kata kerja) | `AGENTS.md:77-94` |
| `code/` | `main.<lang>` untuk setiap bahasa di field Languages; header 4–6 baris berisi path `docs/en.md` dan/atau URL spec/RFC; self-terminating demo | `AGENTS.md:220-225` |
| `code/tests/` | Minimal 5 unit test, dijalankan dengan runner stdlib (`python3 -m unittest discover`, `npx tsx --test`, dll.) | `AGENTS.md:226-231`, `CONTRIBUTING.md:56-65` |
| `quiz.json` | Terdiri atas `lesson`, `title`, `questions[6]`: 1 `pre` + 3 `check` + 2 `post`; `correct` zero-indexed; skema lama `q/choices/answer` dilarang | `AGENTS.md:95-123` |
| `outputs/` | Prompt, skill, agent, atau artefak lain yang dihasilkan pelajaran (opsional tetapi umum) | `CONTRIBUTING.md:56-65` |
| `notebook/` | Notebook Jupyter untuk eksperimen (opsional) | `CONTRIBUTING.md:64` |
| Diagram | Hanya Mermaid atau SVG. Fence `figure` dengan id terdaftar, atau `iso-machine {"machine":"..."}` | `AGENTS.md:53`, `CONTRIBUTING.md:91-110` |
| Blok kode | Semua fence wajib punya tag bahasa; `bash` = perintah, `console` = sesi dengan `$ `, `text` = output, `ansi` = log berwarna | `AGENTS.md:54`, `CONTRIBUTING.md:35-50` |
| Dependensi | Stdlib-first; Python: numpy/torch/h5py/zstandard/safetensors; TS: hono/zod/ws/@hono/node-server; Rust & Julia stdlib only | `AGENTS.md:56`, tabel Dependencies |

Urutan section yang disarankan: **The Problem → The Concept → Build It → Use It → Ship It → Exercises** (`CONTRIBUTING.md:67-72`, `LESSON_TEMPLATE.md`).

### 4.2 Contoh konkret 1 — Fase 3, `03-deep-learning-core/03-backpropagation`

Path: `phases/03-deep-learning-core/03-backpropagation/`

| File | Isi |
|---|---|
| `docs/en.md` (20.207 B) | `# Backpropagation from Scratch`, hook, `**Type:** Build`, `**Languages:** Python, Julia, Rust`, `**Time:** ~120 minutes`. Section: The Problem (baris 17), The Concept (25), **Build It (142)** — 7 langkah dari `Value` node, backward function, sigmoid/loss, layer, XOR, circle classification — **Use It (400)** dengan PyTorch, **Ship It (438)**, Exercises (443). |
| `code/main.py`, `main.jl`, `main.rs`, `main.ts` | Empat implementasi autograd engine. `main.rs` memuat header: `// Lesson: Backpropagation from Scratch (phase 03 / lesson 03)` + `// Rust port of code/main.py. Standard library only.` + `// Source: phases/.../docs/en.md` + perintah build. |
| `outputs/prompt-gradient-debugger.md` | Artefak reusable berupa prompt dengan YAML frontmatter `name`, `description`, `phase: 03`, `lesson: 03`; berisi protokol diagnosa vanishing/exploding/NaN gradient. Disebut eksplisit di `docs/en.md:440-441`. |
| `quiz.json` | **Bentuk array** (55 kasus seperti ini), 5 pertanyaan dengan urutan stage `pre, pre, post, post, post` — tidak mengikuti kontrak 1/3/2. |

Ini adalah contoh paling jelas dari pola **Build It / Use It / Ship It**: tulis backprop manual → bandingkan dengan `loss.backward()` PyTorch → kirim prompt debugger.

### 4.3 Contoh konkret 2 — Fase 7, `07-transformers-deep-dive/02-self-attention-from-scratch`

Path: `phases/07-transformers-deep-dive/02-self-attention-from-scratch/`

- `docs/en.md`: `**Type:** Build`, `**Languages:** Python`, `**Time:** ~90 minutes`. Section: The Problem, The Concept (analogi database lookup, Q/K/V, attention matrix, scaling, softmax, weighted sum), Build It 5 langkah, Use It, Ship It, Exercises.
- Tiga fence `figure`: `attention-matrix`, `softmax-attention-scaling`, dan `iso-machine {"machine":"self-attention"}`. Mesin isometric ini terdaftar di `site/figures-iso.js:17` sebagai kunci `'self-attention'`.
- `code/`: `main.jl`, `main.rs`, `self_attention.py` (perhatikan: file Python tidak bernama `main.py`, meski field Languages hanya "Python"). Header Julia menyitasi makalah asli `https://arxiv.org/abs/1706.03762` (Vaswani et al., *Attention Is All You Need*) — contoh aturan `AGENTS.md:55`: sitasi ke paper/spec, bukan ke repo kurikulum eksternal.
- `outputs/prompt-attention-explainer.md`, `quiz.json` berbentuk objek dengan 5 pertanyaan (`pre, pre, post, post, post`).
- Tidak ada `code/tests/`; ini tipikal untuk pelajaran fase awal/sampel.

### 4.4 Contoh konkret 3 — Fase 13, `13-tools-and-protocols/07-building-an-mcp-server`

Path: `phases/13-tools-and-protocols/07-building-an-mcp-server/`

| File | Peran |
|---|---|
| `docs/en.md` | `# Building an MCP Server: Stateless Python and TypeScript`, `**Type:** Build`, `**Languages:** Python, TypeScript`, `**Prerequisites:** Phase 13, Lesson 06`, `~85 minutes`. Section: The Problem, The Concept (dispatch loop, request validation, mandatory discovery, tools/resources/prompts, typed results, no server-initiated requests, legacy compatibility), Use It, Ship It. Fence `t3-dispatch-loop`. |
| `code/main.py`, `code/main.ts`, `code/main.go` | Implementasi stateless MCP server. Header Python/TS menyitasi `https://modelcontextprotocol.io/specification/2026-07-28/` dan path `docs/en.md`; header Go menyebut "Go port of code/main.py". |
| `code/tests/test_main.py` (5.694 B) | Satu-satunya direktori tes di sebagian besar fase 13; menggunakan `unittest`. |
| `assets/server-anatomy.svg` | Diagram SVG sesuai aturan "Mermaid atau SVG only". |
| `outputs/skill-mcp-server-scaffolder.md` | Skill dengan frontmatter `name`, `description`, `version: 2.0.0`, `phase: 13`, `lesson: 07`, `tags`; berisi daftar Produce, Hard rejects, Refusal rules. |
| `notebook/.gitkeep`, `quiz.json` | Quiz berbentuk objek dengan **6 pertanyaan canonical** `pre, check×3, post×2`. |

Contoh ini menunjukkan pelajaran "modern" yang paling patuh kontrak: spec resmi, tes, aset SVG, artefak skill, dan quiz canonical.

### 4.5 Contoh konkret 4 — Fase 19, `19-capstone-projects/05-autonomous-research-agent`

Path: `phases/19-capstone-projects/05-autonomous-research-agent/`

- `docs/en.md`: `# Capstone 05 — Autonomous Research Agent (AI-Scientist Class)`, `**Type:** Capstone`, `**Languages:** Python (agent + sandbox), TypeScript, LaTeX (output)`, field tambahan `**Phases exercised:** P0 → P2 → P3 → P7 → P10 → P14 → P15 → P16 → P18`, `**Time:** 40 hours`. Section menggunakan varian tanpa "The": Problem, Concept, Architecture, Stack, Build It, Use It, Ship It, Exercises.
- Fence `figure`: `ce-experiment-tree`; aset `assets/research-tree.svg`.
- `code/main.py` + `code/main.ts`: scaffold plan/execute/verify tree search dengan budget, sandbox, fungsi novelty × quality × budget; LLM planner dan PyTorch di-*stub* secara sadar (tidak ada API key yang dibutuhkan).
- `outputs/skill-ai-scientist.md`, `notebook/.gitkeep`, `quiz.json` objek dengan 7 pertanyaan (`pre, pre, check×3, post, post`).
- Di `README.md:1104`, barisnya memakai tabel khusus capstone: kolom `| # | Project | Combines | Lang |`, dengan `Combines = P0 P2 P3 P7 P10 P14 P15 P16 P18`. Parser situs membaca ini sebagai `type: "Capstone"` dan memindahkan token prasyarat ke field `combines` (`site/build.js:302-312`).

---

## 5. Statistik kurikulum

### 5.1 Bahasa pada file `main.*`

Distribusi 942 file `main.*` di 540 direktori `code/` (490 di antaranya memiliki minimal satu `main.*`):

| Ekstensi | Jumlah file | Persentase |
|---|---:|---:|
| `.py` | 474 | 50,3% |
| `.ts` | 373 | 39,6% |
| `.rs` | 45 | 4,8% |
| `.jl` | 34 | 3,6% |
| `.go` | 16 | 1,7% |

Kombinasi bahasa per pelajaran (dominan):

| Kombinasi `main.*` | Pelajaran |
|---|---:|
| `.py,.ts` | 325 |
| `.py` | 71 |
| `.py,.rs` | 28 |
| `.jl,.py,.ts` | 16 |
| `.go,.py,.ts` | 16 |
| `.py,.rs,.ts` | 10 |
| `.jl,.py` | 6 |
| `.jl` | 6 |
| `.ts` | 5 |
| `.jl,.rs` | 4 |
| kombinasi langka lain | 4 |

Catatan: 407 dari 490 pelajaran memiliki lebih dari satu `main.*`. Field `**Languages:**` di dokumen tidak selalu sama dengan file yang ada (misalnya `03-backpropagation` menulis Python, Julia, Rust tetapi juga memiliki `main.ts`).

### 5.2 `quiz.json`

- 392 dari 542 pelajaran memiliki `quiz.json` (72,3%).
- **337** berbentuk objek (`lesson`/`title`/`questions`), **55** berbentuk array polos. Audit `scripts/audit_lessons.py:139-152` memang secara eksplisit menerima kedua bentuk, sehingga ini bukan error parser.
- Total **2.362 pertanyaan**: `pre` 643, `check` 815, `post` 904.
- Jumlah pertanyaan per quiz: 5 → 122 quiz, 6 → 170, 7 → 68, 8 → 32.
- Pola stage paling umum:

| Pola stage | Jumlah quiz |
|---|---:|
| `pre, check, check, check, post, post` (canonical) | 127 |
| `pre, pre, post, post, post` | 119 |
| `pre, pre, check, check, check, post, post` | 63 |
| `pre, pre, check, check, check, post` | 35 |
| `pre, pre, check, check, check, post, post, post` | 29 |
| pola lainnya | 19 |

- 2.272 pertanyaan memakai keys canonical `stage, question, options, correct, explanation`; 90 pertanyaan (di quiz berbentuk array) memiliki key tambahan `id`.
- Belum ada satu pun quiz yang memakai skema legacy `q/choices/answer` — aturan `AGENTS.md:95-123` dan `scripts/audit_lessons.py:157-165` terpenuhi.

### 5.3 Tes

- 104 pelajaran memiliki `code/tests/`; total 114 file, **semua `.py`**.
- Nama paling umum: `test_main.py` (67), `test_lab.py` (5), `__init__.py` (4), sisanya nama spesifik (`test_retrieval.py`, `test_paper_writer.py`, `test_scheduler.py`, dll.).
- 109 dari 114 file memuat `import unittest` / `from unittest` → runner stdlib Python dominan sesuai kontrak. Hanya 2 file `main.ts` dengan `node:test`/`node:assert` yang ditemukan, dan tidak ada tes Rust/Julia inline yang terdeteksi.
- Distribusi tes sangat timpang: Fase 13 (37), Fase 19 (47), Fase 14 (13) = 97 dari 104 tes. Fase 1, 3, 4, 5, 6, 7, 8, 9, 10, 12, 15, 16, 18 memiliki **nol** `code/tests/`.

### 5.4 Artefak `outputs/`

- 478 direktori `outputs/`; 473 di antaranya berisi artefak nyata, 5 hanya `.gitkeep`.
- 552 file artefak, didominasi:
  - `skill-*.md` — 409 file
  - `prompt-*.md` — 100 file
  - `.gitkeep` — 18 file
  - sisanya variasi seperti `career-*`, `feedback-*`, `task-*`, `evidence-*`, `delegation-*`, `workflow-*`, `assumption-*`, `slice-*`.
- Konvensi isi: YAML frontmatter dengan `name`, `description`, `phase`, `lesson`, opsional `version`/`tags`, lalu instruksi yang bisa langsung dipakai (lihat contoh §4.2 dan §4.4).

### 5.5 Diagram, fence `figure`, dan dokumentasi

- 530 dari 542 dokumen memiliki fence `figure`; total **553 fence**.
- Terdapat juga 395 fence `mermaid`.
- ID figure didaftarkan dengan `LF.register` di 78 file `site/figures*.js` (mis. `site/figures-setup.js:507`, `site/figures-iso.js:17` untuk `'self-attention'`).
- Tiga dokumen memakai `iso-machine {"machine":"..."}`; sisanya ID statis seperti `backprop-vanishing`, `attention-matrix`, `t3-dispatch-loop`, `s0-env-stack`, `ce-experiment-tree`.
- Field frontmatter: `**Type:**` 542, `**Prerequisites:**` 542, `**Time:**` 542, `**Languages:**` 525 (17 dokumen tidak memilikinya), `## Learning Objectives` 376.
- Nilai `**Type:**`: `Build` 331, `Learn` 136, `Learn + Build` 47, `Capstone` 18, plus varian langka (`Build + Learn`, `Learn + Use`, `Build + Use`, `Use + Build`, `Build (Capstone)`, `Learn + lab`, `Learn (capstone)`).
- Keberadaan section: `Use It` 491, `Exercises` 491, `Ship It` 449, `The Problem` 458, `The Concept` 438, `Build It` 384. Banyak pelajaran memakai varian `Problem`/`Concept` tanpa "The" (terutama capstone).
- Panjang `docs/en.md`: median 167 baris, rata-rata 247, minimum 77, maksimum 1.536 baris.

### 5.6 Header komentar kode

Dari 942 file `main.*`: 479 dimulai dengan marker komentar (`#`, `//`, `--`, `/*`), dan 510 menyebut `docs/en.md` dalam 8 baris pertama. Dua gaya yang terlihat:

```text
// Lesson: Backpropagation from Scratch (phase 03 / lesson 03)
// Rust port of code/main.py. Standard library only.
// Source: phases/03-deep-learning-core/03-backpropagation/docs/en.md
// Build: rustc --edition 2021 -O code/main.rs -o /tmp/lesson
```

```text
"""
Phase 13 Lesson 07: a stateless MCP server over stdio.
Lesson: phases/13-tools-and-protocols/07-building-an-mcp-server/docs/en.md
Specification: https://modelcontextprotocol.io/specification/2026-07-28/
Implements discovery, three server primitives, and per-request validation.
Run: python3 main.py --demo
"""
```

Fase-fase awal (terutama `01-math-foundations`) memakai gaya lama: `# Stdlib only. Sources:` diikuti URL dokumentasi resmi/arxiv.

### 5.7 Cakupan per fase (ringkasan)

| Fase | quiz | tests | outputs | notebook | code | punya `main.*` |
|---|---:|---:|---:|---:|---:|---:|
| 00 | 13 | 1 | 6 | 0 | 11 | 2 |
| 01 | 22 | 0 | 22 | 0 | 22 | 4 |
| 02 | 21 | 3 | 21 | 0 | 21 | 7 |
| 03 | 13 | 0 | 13 | 0 | 13 | 10 |
| 04 | 28 | 0 | 28 | 0 | 28 | 28 |
| 05 | 29 | 0 | 29 | 29 | 29 | 29 |
| 06 | 0 | 0 | 17 | 16 | 17 | 17 |
| 07 | 1 | 0 | 16 | 13 | 16 | 16 |
| 08 | 0 | 0 | 15 | 14 | 15 | 15 |
| 09 | 0 | 0 | 12 | 12 | 12 | 12 |
| 10 | 11 | 0 | 24 | 11 | 24 | 24 |
| 11 | 17 | 2 | 17 | 2 | 17 | 12 |
| 12 | 0 | 0 | 25 | 25 | 25 | 25 |
| 13 | 37 | 37 | 45 | 23 | 45 | 45 |
| 14 | 54 | 13 | 54 | 30 | 54 | 54 |
| 15 | 0 | 0 | 22 | 22 | 22 | 22 |
| 16 | 2 | 0 | 25 | 23 | 25 | 24 |
| 17 | 29 | 1 | 29 | 28 | 29 | 29 |
| 18 | 30 | 0 | 30 | 30 | 30 | 30 |
| 19 | 85 | 47 | 28 | 17 | 85 | 85 |

---

## 6. Bagaimana `README.md` dan `ROADMAP.md` memberi makan situs

Penjelasan di `CONTRIBUTING.md:12-33` dan implementasinya di `site/build.js`:

1. **Parser `ROADMAP.md`** (`site/build.js:131-162`):
   - Regex `^##\s+Phase\s+(\d+).*?—\s*(✅|🚧|⬚)` membaca status fase.
   - Regex `^\|\s*\d+\s*\|\s*(.+?)\s*\|\s*(✅|🚧|⬚)\s*\|` membaca status tiap pelajaran.
   - Glyph dipetakan ke `complete` / `in-progress` / `planned`. Itulah sebabnya `CONTRIBUTING.md:27-28` melarang mengganti glyph dengan teks.
2. **Parser `README.md`** (`site/build.js:164-332`):
   - Menerima dua bentuk header fase: `### Phase 0: Setup & Tooling \`13 lessons\`` **atau** `<summary><b>Phase 1 — Math Foundations</b> <code>22 lessons</code> ... <em>deskripsi</em></summary>` (baris 185-190).
   - Deteksi tabel pelajaran via `| # | Lesson | Type | Lang |` (baris 224); tabel capstone via `| # | Project | Combines | Lang |` (baris 323-328).
   - Kolom `Lang` boleh teks biasa atau emoji bendera `🐍 🟦 🦀 🟣 ⚛️`; keduanya dinormalisasi (baris 249-266).
   - **URL pelajaran diambil dari link markdown** `[Title](phases/NN-phase/MM-lesson/)` (baris 268-277). Ini alasan `AGENTS.md` menyebut kebiasaan umum "README baris polos tanpa link" sebagai bug: `site/data.js` kehilangan URL.
   - Status dari ROADMAP dicocokkan fuzzy dengan judul (baris 280-295); jika ada link tetapi status masih `planned`, dipaksa `complete` (baris 297-300).
   - Deskripsi fase diambil dari blockquote baris berikutnya untuk header `###` (baris 196-203) atau dari `<em>` pada `<summary>`.
3. **Keluaran:** `site/build.js` menulis `site/data.js` (baris 2292 dst.) dan `site/certification-data.js`. Juga membaca `glossary/terms.md` dan graf dependensi Mermaid canonical di README (baris 663-669).
4. **Pipeline CI:** `site/data.js` **tidak pernah di-commit**; di-build ulang oleh job `site-rebuild` pada push ke `main` (`AGENTS.md` "Automation contract", baris 262+). Kontributor menjalankan `node site/build.js`, melihat perubahan hanya pada timestamp, lalu `git checkout -- site/data.js` (`CONTRIBUTING.md:30-33`).
5. **Kartu sosial:** setiap halaman memakai `/og/<type>/<id>.png`; teks kartu berasal dari `lesson-seo.json`, `certification-seo.json`, dan `site/og-cards.json` yang ditulis `site/build.js` (`AGENTS.md` "Social cards").

---

## 7. Anomali dan ketidakteraturan yang ditemukan

### 7.1 Cakupan file tidak merata
- **392/542** pelajaran memiliki `quiz.json`; lima fase sama sekali tanpa quiz: **06, 08, 09, 12, 15**. Fase 07 hanya 1/16, Fase 16 hanya 2/25.
- **104/542** pelajaran memiliki tes; 438 pelajaran tanpa `code/tests/`. Kontrak `AGENTS.md:226-231` ("5+ unit tests minimum") belum terpenuhi secara universal.
- Dua pelajaran tidak punya `code/` sama sekali: `00-setup-and-tooling/02-git-and-collaboration` dan `00-setup-and-tooling/11-linux-for-ai`.
- 50 pelajaran punya `code/` tetapi **tanpa file `main.*`**; mereka memakai nama file bertema, mis. `gpu_check.py`, `trees.py`, `clustering.ts`, `fourier.jl`, `eval_framework.py` (contoh: `01-math-foundations/20-fourier-transform/code/fourier.py`).

### 7.2 Bentuk dan isi quiz menyimpang dari kontrak
- 55 quiz berupa **array polos** tanpa `lesson`/`title`; 2 quiz objek hanya punya `lesson` + `questions`; 65 objek hanya punya `questions`.
- Hanya **127 quiz** yang mengikuti pola canonical 6 soal `1 pre + 3 check + 2 post`; 119 quiz memakai `pre,pre,post,post,post`; ada quiz 5, 7, bahkan 8 soal.
- 90 pertanyaan memiliki field `id` tambahan.
- Penyebabnya terlihat jelas di `scripts/audit_lessons.py:129-194`: audit hanya memeriksa JSON valid, keys canonical, panjang `options`, dan rentang `correct`. **Tidak** memeriksa keberadaan quiz, jumlah 6, atau komposisi stage.

### 7.3 Penomoran direktori tidak kontigu
| Fase | Jumlah dir | Nomor ditemukan | Keterangan |
|---|---:|---|---|
| `08-generative-ai` | 15 | 01–14, **19** | Lonjakan ke 19; nomor 15–18 tidak ada |
| `10-llms-from-scratch` | 24 | 01–22, **25**, **34** | Nomor 23–24 dan 26–33 kosong |
| `19-capstone-projects` | 85 | 01–17, **20–87** | Celah 18–19 disengaja: 17 produk lalu track 20–87 |

Meskipun demikian, jumlah baris tabel README dan ROADMAP per fase **selalu sama** dengan jumlah direktori (542 total), sehingga parser situs tidak meleset. Phase 19 memakai dua tabel berturut-turut; ini didukung parser (`site/build.js:302-328`).

### 7.4 `notebook/` adalah direktori kosong
295 pelajaran memiliki `notebook/`, tetapi **semua** berisi tepat satu file `.gitkeep` — tidak ada satu pun `.ipynb` nyata. Kontrak `CONTRIBUTING.md:64` menyebut notebook sebagai opsional, jadi direktori ini tampaknya dibuat oleh scaffold lalu dibiarkan kosong.

### 7.5 Bahasa dan dependensi
- Ada **16 file `main.go`** (terutama Fase 13: 8 file, Fase 14: 2, Fase 16: 2, Fase 19: 2), padahal tabel "Dependencies" di `AGENTS.md` hanya mencantumkan Python, TypeScript, Rust, Julia — **Go tidak ada dalam allowlist**.
- Beberapa field `**Languages:**` menyebut bahasa tanpa file `main.*` corresponding (mis. `Python, C, TypeScript` di 3 dokumen; tidak ada `main.c`).
- Kombinasi `Type` tidak mengikuti enum sederhana `Learn | Build | Reference` dari `AGENTS.md:81` (muncul `Learn + Build`, `Capstone`, `Learn + lab`, dll.). Parser situs tetap menerimanya sebagai teks.

### 7.6 Inkonsistensi estimasi waktu
`ROADMAP.md:7` menyebut total **~332 jam**, tapi jumlah per-fase adalah **~1.170 jam**, dengan Fase 19 menyumbang ~620 jam. README tidak mengulang angka 332.

### 7.7 Variasi struktur dokumen
- 17 dokumen tanpa `**Languages:**`; 166 tanpa `## Learning Objectives`.
- Beberapa pelajaran memakai `## Problem` / `## Concept` alih-alih `## The Problem` / `## The Concept` (capstone dan pelajaran modern).
- `**Phases exercised:**` adalah field khas capstone yang tidak ada di kontrak umum.

---

## 8. Kesimpulan

`phases/` adalah kurikulum yang sangat terstruktur pada level **direktori dan dokumentasi** (542/542 memiliki `docs/en.md`, pola Build It/Use It/Ship It jelas, 530/542 memakai figure fence, artefak `outputs/` melimpah). Akan tetapi, **kedisiplinan kontrak menurun tajam pada lapisan penunjang**: quiz hanya 72% hadir dan hanya ~32% dari semua quiz mengikuti pola 1/3/2; tes hanya 19% hadir dan hampir seluruhnya Python; `notebook/` yang 295 jumlahnya seluruhnya kosong; penomoran meloncat di tiga fase; dan allowlist dependensi tidak mencakup Go yang ternyata dipakai.

Situs tidak menderita karena ini: `site/build.js` hanya bergantung pada **bentuk tabel README + glyph ROADMAP + link folder**, dan ketiga hal itu konsisten untuk ke-542 pelajaran. Anomali di atas lebih berdampak pada **pengalaman belajar dan jaminan kualitas** daripada pada generasi `site/data.js`.

### Daftar bukti utama
- `AGENTS.md:53-56, 75-123, 220-232, 262+` — aturan diagram, kontrak pelajaran, quiz, kode/tes, automasi CI.
- `CONTRIBUTING.md:12-33, 54-110` — relasi README/ROADMAP → situs, struktur pelajaran, figure fence.
- `ROADMAP.md:1-9, 11-563` — glyph status, estimasi jam, 20 header fase.
- `README.md:449-466, 468-484, 1094-1189, 1456-1477` — format tabel, tabel capstone, audit, total lessons.
- `site/build.js:125-332, 663-669, 2206-2296` — parser status, parser README, graf dependensi, penulisan `data.js`.
- `scripts/audit_lessons.py:129-194` — alasan variasi bentuk quiz masih lolos audit.
- `phases/03-deep-learning-core/03-backpropagation/`, `phases/07-transformers-deep-dive/02-self-attention-from-scratch/`, `phases/13-tools-and-protocols/07-building-an-mcp-server/`, `phases/19-capstone-projects/05-autonomous-research-agent/` — empat contoh anatomi pelajaran.
