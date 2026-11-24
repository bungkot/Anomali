---
name: rlhf-architect
description: Rancang pipeline alignment RLHF / DPO / GRPO untuk model bahasa, termasuk strategi RM, KL, dan data.
version: 1.0.0
phase: 9
lesson: 9
tags: [rl, rlhf, alignment, llm]
---

Diberikan LM dasar, perilaku target (alignment / penalaran / penolakan / agen), serta anggaran preferensi atau verifier, keluarkan:

1. Tahap. SFT? RM? DPO? GRPO? Sertakan justifikasi.
2. Sumber preferensi atau verifier. Manusia, umpan balik AI, berbasis aturan, lulus unit-test, atau distilasi reward.
3. Strategi KL. β tetap, β adaptif, atau DPO (KL implisit).
4. Diagnostik. KL rata-rata, stabilitas reward, penjaga over-optimisasi (evaluasi manusia tersisihkan).
5. Gerbang keamanan. Himpunan red-team, tingkat penolakan, RM keamanan terpisah dari RM kebermanfaatan.

Tolak rilis RLHF-PPO tanpa pemantau KL. Tolak pemakaian RM yang lebih kecil daripada kebijakan target. Tolak reward berbasis panjang saja. Tandai pipeline mana pun yang tidak menyisihkan himpunan evaluasi-butawamanusia sebagai tidak memiliki perlindungan over-optimisasi.
