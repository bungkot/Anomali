---
name: mtp-planner
description: Merencanakan integrasi multi-token prediction untuk proses pra-pelatihan baru.
version: 1.0.0
phase: 10
lesson: 18
tags: [mtp, multi-token-prediction, deepseek-v3, pre-training, speculative-decoding]
---

Diberikan spesifikasi proses pra-pelatihan (skala model, ukuran hidden, jumlah layer, anggaran token data, topologi GPU, target deployment) dan tujuan yang dinyatakan (sinyal pelatihan yang lebih padat vs draf dekode spekulatif vs keduanya), hasilkan rencana integrasi MTP.

Hasilkan:

1. Kedalaman D. Pilih 1 atau 2. DeepSeek-V3 menggunakan D=1 dan melaporkan tingkat penerimaan dekode spekulatif pada kedalaman pertama di atas 80%. D=2 adalah wilayah pengembalian yang menurun bagi sebagian besar proses. Justifikasi pilihan terhadap anggaran komputasi — setiap kedalaman tambahan menambah sekitar satu blok transformer dari komputasi per langkah pelatihan.
2. Jadwal lambda. Default: 0,3 untuk 10% pertama pelatihan, 0,1 setelahnya. Naikkan hingga 0,5 pada awal untuk model kecil (di bawah 7B) yang sinyal padatnya lebih penting; turunkan jika Anda melihat loss MTP mendominasi loss utama.
3. Anggaran parameter. Laporkan jumlah parameter per modul terhadap model utama. Konfirmasi overhead di bawah 5% dari parameter utama (dense) atau di bawah 3% (MoE).
4. Overhead memori dan komputasi. Kuantifikasi FLOPs forward-pass tambahan per langkah (sekitar `D * transformer_block_cost`), memori backward-pass tambahan (memori aktivasi untuk D modul), dan puncak VRAM tambahan (embedding dan head yang dibagi tidak dihitung, proyeksi dan blok transformer dihitung).
5. Pengkabelan saat inferensi. Jelaskan cara menggunakan modul MTP sebagai draf dekode spekulatif saat inferensi. Sebutkan jalur integrasi aturan Leviathan dan pencatatan KV-rollback. Konfirmasi kompatibilitas dengan stack inferensi target (vLLM, SGLang, TensorRT-LLM).

Penolakan keras:
- Menambahkan MTP ke model dense yang pra-dilatih tanpanya. Tidak dapat dipasang ulang — modul MTP tidak dilatih.
- D > 2 untuk integrasi pertama. Keuntungan atas D=1 kecil; kompleksitas tumbuh cepat.
- MTP pada model dengan parameter aktif di bawah 1B. Sinyal lebih lemah daripada biaya overhead pada skala itu.
- Menggunakan head paralel (gaya Gloeckle) ketika tujuannya adalah dekode spekulatif. Head-head itu tidak membentuk rantai kausal.

Aturan penolakan:
- Jika data pra-pelatihan didominasi oleh urutan pendek (di bawah 2k), tolak. Keuntungan MTP mengasumsikan urutan yang cukup panjang agar supervisi kedalaman 2 berarti.
- Jika stack inferensi target sama sekali tidak mendukung dekode spekulatif, catat bahwa MTP tetap memberi sinyal pelatihan yang lebih padat dan lanjutkan, tetapi beri tanda pada ketidaksesuaian ini.
- Jika pengguna melanjutkan pra-pelatihan pada checkpoint dense yang sudah ada tanpa MTP, tolak dan rekomendasikan menambahkan MTP hanya pada awal proses pelatihan yang bersih atau pada titik reset batas data yang bersih.

Output: rencana integrasi satu halaman yang mencantumkan D, jadwal lambda, overhead parameter (absolut dan persentase), overhead komputasi (persentase per langkah pelatihan), dan rencana pengkabelan dekode spekulatif saat inferensi. Akhiri dengan paragraf "kriteria keberhasilan" yang menyebut metrik terukur yang membenarkan mempertahankan MTP: tingkat penerimaan pada kedalaman 1 setelah 50B token pelatihan harus di atas 70%, jika tidak, arsitektur harus dikembalikan.
