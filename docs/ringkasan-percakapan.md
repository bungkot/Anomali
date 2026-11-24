# Ringkasan Percakapan — Analisis & Rencana Belajar AI Engineering from Scratch

Tanggal: 24 November 2026
Repositori sumber: `C:\Users\Hendra\Downloads\ai-engineering-from-scratch`
Semua hasil kini dikumpulkan di `C:\Users\Hendra\Downloads\anomali`.

## 1. Kronologi singkat

1. Diminta menganalisis dan memahami proyek, hasil ditulis ke `../analisa`.
2. Dibuat 7 dokumen analisis (indeks + ringkasan eksekutif + 5 laporan subsistem).
3. Diminta menampilkan kurikulum dalam tabel → tabel 20 phase + anatomi pelajaran + jalur belajar.
4. Diminta menentukan rentang fase untuk latar belakang "sudah Python, sudah ML, fokus LLM & agent, MCP, Agent Skills" → Fase 10–14, dengan Fase 7 opsional.
5. Dikoreksi: ingin satu tabel dari "sudah Python" hingga Agent Skills → tabel 8 langkah.
6. Diminta menampilkan Fase 1–13 → tabel 13 fase.
7. Dikoreksi lagi: mulai dari Fase 0 → tabel Fase 0–13.
8. Rencana dicatat ke `../anomali/rencana-belajar.md`.
9. Diminta mulai Fase 0 → onboarding `start-learning`: 3 pertanyaan dijawab, `LEARNING.md` dibuat.
10. Diminta mengumpulkan semua hasil ke `../anomali` → dokumen ini + salinan seluruh berkas.

## 2. Tabel kurikulum lengkap (20 phase)

| # | Phase | Fokus | Pelajaran | Est. |
|---:|---|---|---:|---:|
| 0 | Setup & Tooling | Lingkungan, GPU, API key, Docker, shell | 13 | ~16 jam |
| 1 | Math Foundations | Vektor, turunan, probabilitas, optimasi | 22 | ~23 jam |
| 2 | ML Fundamentals | Regresi, SVM, tree, evaluasi model | 21 | ~26 jam |
| 3 | Deep Learning Core | Perceptron → backprop → mini framework | 13 | ~15 jam |
| 4 | Computer Vision | CNN, detection, diffusion, VLM | 28 | ~27 jam |
| 5 | NLP | Tokenisasi, embedding, attention, retrieval | 29 | ~30 jam |
| 6 | Speech & Audio | ASR, Whisper, TTS, codec | 17 | ~18 jam |
| 7 | Transformers Deep Dive | Attention, RoPE, MoE, KV cache | 16 | ~14 jam |
| 8 | Generative AI | VAE, GAN, diffusion, flow matching | 15 | ~14 jam |
| 9 | Reinforcement Learning | MDP, Q-learning, PPO, RLHF | 12 | ~13 jam |
| 10 | LLMs from Scratch | Tokenizer, pre-train GPT, SFT, DPO | 24 | ~26 jam |
| 11 | LLM Engineering | Prompt, RAG, LoRA, tool use, guardrail | 17 | ~19 jam |
| 12 | Multimodal AI | CLIP, LLaVA, video, computer-use | 25 | ~65 jam |
| 13 | Tools & Protocols | Tool calling, MCP, Agent Skills, RE | 45 | ~76 jam |
| 14 | Agent Engineering | Agent loop, memori, orkestrasi, observability | 54 | ~55 jam |
| 15 | Autonomous Systems | Long-horizon, budget, kill switch, HITL | 22 | ~20 jam |
| 16 | Multi-Agent & Swarms | Supervisor, debate, A2A, konsensus | 25 | ~28 jam |
| 17 | Infrastructure & Production | Serving, KV cache, FinOps, SRE | 29 | ~34 jam |
| 18 | Ethics, Safety & Alignment | Jailbreak, red-teaming, regulasi | 30 | ~31 jam |
| 19 | Capstone Projects | 17 produk + 9 deep-build track | 85 | ~620 jam |
| | **Total** | | **542** | **~1.170 jam** |

## 3. Anatomi satu pelajaran

| Elemen | Isi |
|---|---|
| `docs/en.md` | The Problem → The Concept → Build It → Use It → Ship It → Exercises |
| `code/main.<lang>` | Implementasi dari nol + demo self-terminating |
| `code/tests/` | Minimal 5 unit test (baru 104/542 pelajaran) |
| `quiz.json` | 1 pre + 3 check + 2 post (baru 392/542 pelajaran) |
| `outputs/` | Skill, prompt, agent siap pakai (478 pelajaran) |
| `notebook/` | 295 direktori, semuanya hanya `.gitkeep` |
| Figure | 553 fence `figure`, 395 fence Mermaid |

## 4. Jalur yang dipilih: Fase 0 → Fase 13

| Fase | Nama | Pelajaran | Est. | Relevansi |
|---:|---|---:|---:|---|
| 0 | Setup & Tooling | 13 | ~16 jam | Titik awal |
| 1 | Math Foundations | 22 | ~23 jam | Dasar |
| 2 | ML Fundamentals | 21 | ~26 jam | Dasar |
| 3 | Deep Learning Core | 13 | ~15 jam | Jembatan ke LLM |
| 4 | Computer Vision | 28 | ~27 jam | Opsional |
| 5 | NLP | 29 | ~30 jam | Pendukung LLM |
| 6 | Speech & Audio | 17 | ~18 jam | Opsional |
| 7 | Transformers Deep Dive | 16 | ~14 jam | Jembatan ke LLM |
| 8 | Generative AI | 15 | ~14 jam | Opsional |
| 9 | Reinforcement Learning | 12 | ~13 jam | Pendukung alignment |
| 10 | LLMs from Scratch | 24 | ~26 jam | Inti LLM |
| 11 | LLM Engineering | 17 | ~19 jam | Inti LLM |
| 12 | Multimodal AI | 25 | ~65 jam | Opsional |
| 13 | Tools & Protocols | 45 | ~76 jam | Tujuan: MCP + Agent Skills |
| | **Total Fase 0–13** | **297** | **~382 jam** | |

Fokus di Fase 13:

| Rute | Nomor pelajaran | Jumlah | Est. |
|---|---|---:|---:|
| MCP | 06–16, 18, 17, 28–31 | 17 | ~23 jam 15 menit |
| Agent Skills | 22, 24, 25, 26, 27 | 5 | ~9 jam 30 menit |

## 5. Onboarding (start-learning)

Jawaban interview:

| Pertanyaan | Jawaban |
|---|---|
| Mengapa belajar AI engineering? | Mau ship produk AI |
| Waktu per minggu | ~10 jam |
| Ingin membangun apa? | Sebuah agent |

Isi `LEARNING.md` yang dibuat di akar repositori (salinan ada di folder ini):

- Mission: ingin ship produk AI, membangun agent, mulai Fase 0 sampai Fase 13 (MCP + Agent Skills), lalu lanjut Fase 14–19.
- Placement: tanggal 2026-11-24, `Score: self-selected`, entry point Fase 0 — Setup & Tooling, pace ~10 jam/minggu.
- Path: ke-20 fase berstatus `Do`, total ~1.170 jam.
- Progress log kosong, review queue kosong.

Handoff:

- `Use learn to start my first lesson.` — memulai pelajaran pertama dan membaca progres dari `LEARNING.md`.
- `Use course-guide <topic> to jump to a specific topic instead.`

## 6. Daftar berkas hasil percakapan di folder ini

| Berkas | Asal |
|---|---|
| `README.md`, `00-ringkasan-eksekutif.md`, `01-struktur-kurikulum.md`, `02-arsitektur-situs.md`, `03-sistem-pembelajaran.md`, `04-konten-pendukung.md`, `05-automasi-dan-tata-kelola.md` | disalin dari `../analisa` |
| `rencana-belajar.md` | rencana Fase 0–13 |
| `LEARNING.md` | salinan state progres dari akar repositori |
| `ringkasan-percakapan.md` | dokumen ini |

Catatan: `LEARNING.md` yang **aktif** tetap berada di akar repositori karena skill `learn` membacanya dari direktori kerja saat ini. Salinan di folder ini hanya arsip.
