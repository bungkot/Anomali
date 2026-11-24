# Rencana Belajar — AI Engineering from Scratch

**Dibuat:** 24 November 2026
**Sumber kurikulum:** `C:\Users\Hendra\Downloads\ai-engineering-from-scratch`
**Titik awal:** Fase 0 (Setup & Tooling) — mulai dari nol, tanpa melewati fase awal
**Tujuan akhir:** Fase 13 (Tools & Protocols), khususnya rute MCP dan Agent Skills

## 1. Rencana fase 0–13

| Fase | Nama | Fokus utama | Pelajaran | Est. waktu | Status | Relevansi |
|---:|---|---|---:|---:|:---:|---|
| 0 | Setup & Tooling | Lingkungan, Git, GPU, API key, Jupyter, Docker, shell, debugging | 13 | ~16 jam | ✅ | Titik awal |
| 1 | Math Foundations | Vektor, turunan, probabilitas, optimasi | 22 | ~23 jam | ✅ | Dasar |
| 2 | ML Fundamentals | Regresi, SVM, tree, evaluasi model | 21 | ~26 jam | ✅ | Dasar |
| 3 | Deep Learning Core | Perceptron, backprop, optimizer, mini framework | 13 | ~15 jam | ✅ | Jembatan ke LLM |
| 4 | Computer Vision | CNN, detection, diffusion, VLM | 28 | ~27 jam | ✅ | Opsional |
| 5 | NLP | Tokenisasi, embedding, attention, retrieval | 29 | ~30 jam | ✅ | Pendukung LLM |
| 6 | Speech & Audio | ASR, Whisper, TTS, audio codec | 17 | ~18 jam | ✅ | Opsional |
| 7 | Transformers Deep Dive | Self-attention, RoPE, MoE, KV cache | 16 | ~14 jam | ✅ | Jembatan ke LLM |
| 8 | Generative AI | VAE, GAN, diffusion, flow matching | 15 | ~14 jam | ✅ | Opsional |
| 9 | Reinforcement Learning | MDP, Q-learning, PPO, RLHF | 12 | ~13 jam | ✅ | Pendukung alignment |
| 10 | LLMs from Scratch | Tokenizer, pre-train GPT, SFT, DPO | 24 | ~26 jam | ✅ | Inti LLM |
| 11 | LLM Engineering | Prompt, RAG, LoRA, tool use, guardrail | 17 | ~19 jam | ✅ | Inti LLM |
| 12 | Multimodal AI | CLIP, LLaVA, video, computer-use | 25 | ~65 jam | ✅ | Opsional |
| 13 | Tools & Protocols | Tool calling, MCP, Agent Skills, reverse engineering | 45 | ~76 jam | ✅ | Tujuan: MCP + Agent Skills |
| | **Total Fase 0–13** | | **297** | **~382 jam** | | |

Semua fase berstatus selesai (`✅`) di repositori, sehingga seluruh materi dapat dipelajari.

## 2. Fokus di dalam Fase 13

Dari 45 pelajaran Fase 13, rencana ini mengambil 22 pelajaran:

| Rute | Nomor pelajaran | Jumlah | Est. waktu |
|---|---|---:|---:|
| MCP | 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 18, 17, 28, 29, 30, 31 | 17 | ~23 jam 15 menit |
| Agent Skills | 22, 24, 25, 26, 27 | 5 | ~9 jam 30 menit |

Catatan penting:

- Deployment MCP publik terkunci sampai checkpoint keamanan di pelajaran 15 selesai.
- Sebelum pelajaran 26 (Agent Skills), harus paham *tool poisoning* / metadata tidak tepercaya; pelajaran 15 bisa dijadikan pengingat.
- Pelajaran 23 (capstone ekosistem tool) opsional setelah kedua rute selesai.
- Sisa pelajaran Fase 13 (tool interface dasar, A2A, OTel, reverse engineering 32–45) tidak masuk rencana ini.

## 3. Cara menjalankan

| Kebutuhan | Perintah / skill | File progres |
|---|---|---|
| Mulai kurikulum dari awal | `start-learning` (Codex) atau `/start-learning` (Claude Code) | `LEARNING.md` |
| Tutor pelajaran per sesi | `learn` / `/learn` | `LEARNING.md` |
| Rute MCP | `learn-mcp` / `/learn-mcp` | `MCP-LEARNING.md` |
| Rute Agent Skills | `learn-agent-skills` / `/learn-agent-skills` | `AGENT-SKILLS-LEARNING.md` |
| Kuis tiap fase | `check-understanding <phase>` | — |

File progres dibuat di direktori kerja peserta dan tidak di-commit ke repositori sumber.

## 4. Setelah Fase 13

Jika ingin melanjutkan:

| Fase | Nama | Alasan |
|---:|---|---|
| 14 | Agent Engineering | Agent loop, memori, orkestrasi, observability, coding agent |
| 15 | Autonomous Systems | Agent long-horizon, budget, kill switch, HITL |
| 16 | Multi-Agent & Swarms | Supervisor, debate, A2A, konsensus |
| 17 | Infrastructure & Production | Serving, KV cache, FinOps, keamanan |
| 18 | Ethics, Safety & Alignment | Prompt injection, red-teaming, regulasi |
| 19 | Capstone Projects | 85 proyek pengunci keahlian |
