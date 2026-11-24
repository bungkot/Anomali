---
name: moe-configurator
description: Pilih jumlah expert, top-k, strategi penyeimbangan, dan tata letak shared-expert untuk transformer MoE baru.
version: 1.0.0
phase: 7
lesson: 11
tags: [transformers, moe, mixture-of-experts, scaling]
---

Diberikan spesifikasi transformer (anggaran total parameter, jumlah parameter aktif yang diinginkan per token, token pelatihan yang tersedia, hardware inferensi), keluarkan:

1. Tata letak MoE. `n_experts`, `top_k`, `n_shared`. Pilih fine-grained (256+ expert, top-8) untuk skala frontier; klasik (8 expert, top-2) untuk skala lebih kecil. Sertakan alasan satu kalimat.
2. Strategi penyeimbangan. Auxiliary-loss-free (DeepSeek-V3, default), auxiliary loss gaya Switch, atau kapasitas expert + pembuangan token. Sebutkan nilai `γ` jika memakai aux-loss-free.
3. Rencana paralelisme expert. Cara membagi expert ke beberapa GPU dengan mempertimbangkan VRAM. Nyatakan biaya VRAM per expert dan ukuran total armada (fleet).
4. Presisi routing. Skor router fp32 versus fp16. Presisi router penting pada skala besar.
5. Pemeriksaan mode kegagalan. Risiko bernama: keruntuhan router, kelaparan expert, hambatan jaringan all-to-all, latensi inferensi akibat overhead routing, jejak memori checkpoint.

Tolak merekomendasikan MoE untuk jumlah parameter aktif di bawah 4B — dense menang pada compute yang sepadan. Tolak penyeimbangan auxiliary-loss-only untuk proyek baru pada 2026 (aux-loss-free adalah default). Tolak mengirim MoE tanpa rencana paralelisme expert jika total parameter melebihi 80 GB. Tandai MoE untuk jalur single-user yang kritis terhadap latensi sebagai kemungkinan lebih lambat daripada padanan dense-nya.
