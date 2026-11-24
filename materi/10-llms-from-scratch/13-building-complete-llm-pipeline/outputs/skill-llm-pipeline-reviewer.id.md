---
name: llm-pipeline-reviewer
description: Tinjau manifest pipeline pelatihan LLM ujung-ke-ujung sebelum menjalankan bernilai jutaan dolar
version: 1.0.0
phase: 10
lesson: 13
tags: [pipeline, training, manifest, eval-gate, cost, rollback]
---

Diberikan manifest pipeline pelatihan yang diusulkan (YAML atau JSON yang mendeskripsikan tokenizer, data, pra-pelatihan, SFT, penyelarasan, evaluasi, kuantisasi, dan tahap penyajian), hasilkan tinjauan yang mencakup:

1. Graf tahap. Pastikan setiap tahap memiliki input dan output bertipe. Soroti dependensi yang hilang, keadaan implisit, atau tahap apa pun yang mengonsumsi direktori mentah alih-alih hash artefak bernama.
2. Rantai hash. Verifikasi bahwa output_hash tahap N sama dengan salah satu input_hashes dari setiap tahap hilir. Ketidaksesuaian berarti manifest tidak koheren dan pipeline tidak boleh dimulai.
3. Gerbang evaluasi. Setiap metrik dalam daftar gerbang harus berupa angka, memiliki operator, ambang batas, dan sumber pengukuran. Tolak gerbang mana pun yang subjektif ("terlihat bagus"), tak terikat (tanpa ambang batas), atau diukur pada data pelatihan.
4. Penjaga regresi. Benchmark inti model baru (MMLU, MATH, HumanEval+, GPQA, atau padanan khusus domain) harus memiliki angka baseline yang terlampir. Jalankan tanpa baseline adalah jalankan tanpa deteksi regresi.
5. Anggaran KL. Tahap penyelarasan (RLHF, DPO, CAI, GRPO) harus mendeklarasikan batas KL kumulatif terhadap model referensi. KL tak terikat adalah pergeseran (drift) tak terikat.
6. Pemeriksaan kontaminasi. Potongan data pelatihan dan himpunan evaluasi harus memiliki pemeriksaan tumpang tindih yang terdokumentasi (pencocokan eksak atau 13-gram). Ambang lulus yang wajib: <0,1%.
7. Estimasi biaya. Estimasi prajalankan untuk setiap tahap ditambah totalnya, dibandingkan dengan gerbang anggaran. Jika estimasi > anggaran, pipeline menolak untuk memulai.
8. Rencana pemulihan (rollback). Untuk setiap tahap, sebutkan tindakan bernama saat gagal: jalankan ulang, kembali ke artefak sebelumnya, revisi input dan jalankan ulang tahap hilir. Tahap mahal (pra-pelatihan) harus memiliki strategi checkpoint hangat (warm checkpoint).
9. Penyimpanan artefak. Checkpoint, dataset, tokenizer, dan laporan evaluasi harus dialamati berdasarkan konten (SHA-256). Artefak yang dialamati berdasarkan nama berkas ("latest.pt") ditolak mentah-mentah.
10. Observabilitas. Setiap tahap harus memancarkan log terstruktur dengan ID jejak, nama tahap, hash input, hash output, waktu dinding (wall clock), dan biaya. ID jejak yang hilang berarti jalankan tidak dapat di-debug setelahnya.

Tanda merah yang menghentikan tinjauan:
- gerbang yang tidak memiliki sumber pengukuran (gerbang pada metrik yang tidak dihitung tahap mana pun)
- tahap yang berbagi checkpoint dengan tahap hilir (tidak ada pemisahan tanggung jawab)
- tahap penyelarasan tanpa model referensi (tidak ada jangkar untuk KL)
- evaluasi LLM-sebagai-hakim yang hakimnya berasal dari keluarga model yang sama dengan kebijakan (kontaminasi)
- estimasi biaya yang melebihi anggaran lebih dari 20%
- rencana pemulihan yang hanya terdiri dari "jalankan ulang dari nol"

Output: tinjauan dua halaman dengan status PASS/HOLD per gerbang, kolom manifest yang tepat atau kolom yang hilang yang menghasilkan setiap keputusan, dan perubahan minimum yang diperlukan untuk mengubah HOLD menjadi PASS.
