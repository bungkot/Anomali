---
name: game-rl-designer
description: Rancang pipeline pelatihan game-RL atau reasoning-RL (AlphaZero / MuZero / GRPO) untuk suatu domain tertentu.
version: 1.0.0
phase: 9
lesson: 12
tags: [rl, alphazero, muzero, grpo, self-play]
---

Dengan target tertentu (game informasi-sempurna / informasi-tak-sempurna / Atari / penalaran LLM / kombinatorial), keluarkan:

1. Kecocokan lingkungan. Apakah aturannya diketahui? Markov? Stokastik? Multi-agen? Ini menentukan AlphaZero versus MuZero versus GRPO.
2. Strategi pencarian. MCTS (PUCT dengan prior hasil pembelajaran), sampel-Gumbel, best-of-N, atau tidak ada.
3. Rencana self-play. Self-play simetris / liga / data offline / hasil verifier.
4. Sinyal target. Hasil game / hadiah verifier / preferensi / model hasil pembelajaran. Sertakan rencana ketahanan.
5. Diagnostik. Tingkat kemenangan terhadap baseline, kurva ELO, tingkat kelulusan verifier, KL terhadap referensi.

Tolak AlphaZero pada game informasi-tak-sempurna (alihkan ke CFR). Tolak GRPO tanpa verifier yang tepercaya. Tolak pipeline game-RL apa pun tanpa kumpulan lawan baseline yang tetap (jika tidak, ELO self-play tidak terkalibrasi).
