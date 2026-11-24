---
name: prompt-caching-planner
description: Rancang tata letak prompt yang ramah cache dan pilih mode caching penyedia yang tepat.
version: 1.0.0
phase: 11
lesson: 15
tags: [llm-engineering, caching, cost]
---

Diberikan sebuah prompt (sistem + alat + few-shot + pengambilan + riwayat + pengguna) serta profil penggunaan (permintaan per jam, TTL yang dibutuhkan, penyedia), keluarkan:

1. Tata letak. Bagian-bagian yang diurutkan ulang dengan satu titik putus cache yang ditandai; jelaskan bagian mana yang stabil dan mana yang volatil.
2. Mode penyedia. Anthropic cache_control, OpenAI automatic, atau Gemini CachedContent. Beri justifikasi berdasarkan TTL dan pola penggunaan ulang.
3. Titik impas. Perkiraan jumlah pembacaan per penulisan dalam TTL; biaya bersih dibandingkan tanpa cache, disertai perhitungannya.
4. Rencana verifikasi. Assertion CI bahwa `cache_read_input_tokens > 0` pada permintaan identik kedua; dasbor yang memisahkan token tersimpan-cache dan token tidak tersimpan-cache.
5. Mode kegagalan. Cantumkan tiga alasan paling mungkin mengapa cache akan meleset dalam tata letak ini (stempel waktu dinamis, pengurutan ulang alat, teks yang hampir identik) dan bagaimana Anda mencegah masing-masing.

Tolak untuk merilis rencana cache yang menempatkan kolom dinamis di atas titik putus. Tolak mengaktifkan TTL 1 jam tanpa jumlah penggunaan ulang yang membuat premium penulisan 2x terbayar.
