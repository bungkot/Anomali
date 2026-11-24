# Rencana Belajar — AI Engineering from Scratch (Fase Terpilih)

**Dibuat:** 25 November 2026
**Sumber kurikulum:** `C:\Users\Hendra\Downloads\ai-engineering-from-scratch`
**Materi tersalin:** `.\materi` (176 pelajaran, 9 folder fase)
**Titik awal:** Fase 1 (Math Foundations) — Fase 0 dilewati
**Tujuan akhir:** Fase 13 (Tools & Protocols), khususnya rute MCP dan Agent Skills

## 1. Rencana fase terpilih

Hanya fase 1, 2, 3, 5, 7, 9, 10, 11, dan 13 yang diambil.

| Fase | Nama | Fokus utama | Pelajaran | Est. waktu | Status | Relevansi |
|---:|---|---|---:|---:|:---:|---|
| 1 | Math Foundations | Vektor, turunan, probabilitas, optimasi | 22 | ~23 jam | ✅ | Dasar |
| 2 | ML Fundamentals | Regresi, SVM, tree, evaluasi model | 21 | ~26 jam | ✅ | Dasar |
| 3 | Deep Learning Core | Perceptron, backprop, optimizer, mini framework | 13 | ~15 jam | ✅ | Jembatan ke LLM |
| 5 | NLP | Tokenisasi, embedding, attention, retrieval | 29 | ~30 jam | ✅ | Pendukung LLM |
| 7 | Transformers Deep Dive | Self-attention, RoPE, MoE, KV cache | 16 | ~14 jam | ✅ | Jembatan ke LLM |
| 9 | Reinforcement Learning | MDP, Q-learning, PPO, RLHF | 12 | ~13 jam | ✅ | Pendukung alignment |
| 10 | LLMs from Scratch | Tokenizer, pre-train GPT, SFT, DPO | 24 | ~26 jam | ✅ | Inti LLM |
| 11 | LLM Engineering | Prompt, RAG, LoRA, tool use, guardrail | 17 | ~19 jam | ✅ | Inti LLM |
| 13 | Tools & Protocols | Tool calling, MCP, Agent Skills (22 pelajaran terpilih) | 22 | ~32 jam 45 menit | ✅ | Tujuan: MCP + Agent Skills |
| | **Total** | | **176** | **~198 jam 45 menit** | | |

Fase 13 dihitung dari 22 pelajaran yang dipilih di bagian 2, bukan seluruh 45 pelajaran.

Semua fase berstatus selesai (`✅`) di repositori, sehingga seluruh materi dapat dipelajari.

Fase yang tidak diambil: 0 (Setup & Tooling, dilewati karena mulai dari Fase 1), 4 (Computer Vision), 6 (Speech & Audio), 8 (Generative AI), dan 12 (Multimodal AI) — semuanya bersifat opsional.

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
