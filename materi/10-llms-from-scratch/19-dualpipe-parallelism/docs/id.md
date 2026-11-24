# Paralelisme DualPipe

> DeepSeek-V3 dilatih pada 2.048 GPU H800 dengan pakar MoE yang tersebar lintas node. Komunikasi all-to-all pakar lintas node menghabiskan 1 jam-GPU komunikasi untuk setiap 1 jam-GPU komputasi. GPU menganggur setengah waktu. DualPipe (DeepSeek, Des 2024) adalah pipeline bidireksional yang menumpangtindihkan komputasi forward dan backward dengan komunikasi all-to-all yang memicunya. Bubble turun, throughput naik, dan menyimpan dua salinan parameter model (sisi "dual" yang memberi nama itu) menjadi murah setelah Expert Parallelism sudah menyebarkan pakar ke seluruh rank. Pelajaran ini adalah penelusuran bertipe Learn tentang apa yang sebenarnya dilakukan DualPipe dan mengapa penyempurnaan DualPipeV dari Sea AI Lab menghilangkan biaya parameter 2x dengan mengorbankan bubble yang sedikit lebih besar.

**Jenis:** Learn
**Bahasa:** Python (stdlib, simulator jadwal), Rust
**Prasyarat:** Fase 10 · 05 (pelatihan terdistribusi, FSDP, DeepSpeed), Fase 10 · 14 (arsitektur model terbuka dan MoE)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Menyebut empat komponen dari chunk forward-backward DualPipe dan mengapa masing-masing mendapat jendela tumpang tindihnya sendiri.
- Menjelaskan masalah bubble pipeline dalam skala besar, dan arti "bebas bubble" secara praktis versus secara pemasaran.
- Melacak jadwal DualPipe dengan tangan untuk 8 rank PP dan 16 micro-batch serta mengonfirmasi bahwa aliran forward dan reverse mengisi slot menganggur satu sama lain.
- Menyatakan trade-off yang diambil DualPipeV (Sea AI Lab, 2025): menghilangkan replikasi parameter 2x dengan biaya bubble yang sedikit lebih besar saat Expert Parallelism tidak aktif.

## Masalahnya

Melatih model MoE 671B pada 2 ribu GPU H800 menabrak tiga hambatan yang saling memperparah:

1. **Tekanan memori.** Setiap GPU menyimpan bagian dari model. Memori aktivasi pada urutan 8k lintas 61 layer dengan 128 head sangat besar.
2. **Bubble pipeline.** Paralelisme pipeline tradisional (GPipe, 1F1B) menganggurkan GPU sambil menunggu masukan atau gradien tahapnya. Pada 8 tahap, sekitar 12% waktu GPU bisa menjadi bubble bahkan dengan penjadwalan 1F1B.
3. **All-to-all lintas node.** MoE dengan paralelisme pakar menyebarkan pakar ke seluruh node. Setiap forward memicu all-to-all untuk mengirim token ke pakarnya, dan satu lagi untuk menggabungkan. Pada 2 ribu GPU ini dengan mudah menjadi rasio komputasi-komunikasi 1:1.

Masing-masing punya solusi terpisah: checkpointing gradien untuk memori, Zero Bubble (Sea AI Lab, 2023) untuk bubble pipeline, kernel komunikasi paralel-pakar untuk all-to-all. Yang dilakukan DualPipe adalah membuat semuanya bekerja bersama. Jadwalnya menumpangtindihkan komputasi dan komunikasi di dalam satu chunk forward-backward, menyuntikkan micro-batch dari kedua ujung pipeline secara bersamaan, dan memakai jadwal yang dihasilkan untuk menyembunyikan all-to-all di dalam jendela komputasi.

Hasil yang dilaporkan: penghapusan bubble pipeline hampir total, utilisasi GPU di atas 95% dalam proses pelatihan 14,8T token DeepSeek-V3.

## Konsepnya

### Pengingat paralelisme pipeline

Bagi model N-layer ke P perangkat. Perangkat `i` menyimpan layer `i * N/P .. (i+1) * N/P - 1`. Sebuah micro-batch mengalir forward melalui perangkat 0 sampai P-1, lalu backward dari P-1 ke 0. Setiap perangkat hanya bisa memulai tahap forwardnya saat perangkat sebelumnya mengirim keluaran, dan hanya bisa memulai backward saat perangkat di bawahnya mengirim gradien ke arah hulu.

GPipe (Huang dkk., 2019) menjadwalkan satu micro-batch pada satu waktu, yang membuang sebagian besar waktu GPU. 1F1B (Narayanan dkk., 2021) menyelingi forward dan backward untuk beberapa micro-batch. Zero Bubble (Qi dkk., 2023) membelah backward menjadi dua bagian — backward-for-input (B) dan backward-for-weights (W) — dan menjadwalkannya untuk mengisi bubble. Setelah Zero Bubble, pipeline hampir rapat.

DualPipe adalah langkah berikutnya. Ia menambah dua ide di atasnya:

### Ide 1: dekomposisi chunk

Setiap chunk forward dibagi menjadi empat komponen:

- **Atensi.** Proyeksi Q/K/V, atensi, proyeksi keluaran.
- **All-to-all dispatch.** Komunikasi lintas node yang mengirim token ke pakarnya.
- **MLP.** Komputasi pakar MoE.
- **All-to-all combine.** Komunikasi lintas node yang membawa kembali keluaran pakar.

Chunk backward menambah versi gradien dari masing-masing komponen ini. DualPipe menjadwalkannya sehingga all-to-all dispatch terjadi paralel dengan komputasi atensi chunk berikutnya, dan all-to-all combine terjadi paralel dengan komputasi MLP chunk setelahnya.

### Ide 2: penjadwalan bidireksional

Kebanyakan jadwal pipeline menyuntikkan micro-batch dari tahap 0 dan mengalir ke tahap P-1. DualPipe menyuntikkan micro-batch dari KEDUA ujung. Tahap 0 melihat micro-batch forward yang berasal dari sana; tahap P-1 juga melihat micro-batch forward yang berasal dari sana. Kedua aliran bertemu di tengah.

Agar ini berhasil, perangkat `i` harus menyimpan KEDUA layer pipeline awal `i` DAN layer pipeline akhir `P - 1 - i`. Itulah bagian "dual" dari DualPipe: setiap perangkat menyimpan dua salinan layer model yang perlu dilayani (satu untuk setiap arah). Pada skala DeepSeek-V3, ini adalah biaya replikasi parameter 2x. Biaya itu terjangkau karena Expert Parallelism sudah menyebarkan pakar MoE begitu tipisnya sehingga mereplikasi layer non-pakar dua kali hanya masalah kecil.

Yang krusial, aliran forward dalam satu arah dan aliran backward dalam arah lain tumpang tindih tepat di tempat bubble berada pada jadwal satu arah. Bubble pun hilang.

### Jadwal yang dilacak dengan tangan

Perhatikan P = 4 rank, 8 micro-batch, dibagi 4 forward / 4 reverse. Waktu bergerak kiri ke kanan; baris adalah rank perangkat.

```
           Waktu →
rank 0:  F1 F2 F3 F4  F5R F6R F7R F8R  B1 B2 B3 B4  ...
rank 1:     F1 F2 F3  F4/F5R F6R F7R   B1 B2 ...
rank 2:        F1 F2  F3/F5R F4/F6R    B1 ...
rank 3:           F1  F2/F5R F3/F6R    ...
```

Membaca notasi "F4/F5R": rank 1 menjalankan forward micro-batch 4 (bergerak kiri-ke-kanan dalam pipeline) DAN forward micro-batch 5 (bergerak kanan-ke-kiri) pada slot waktu yang sama. Itulah arti "bidireksional" secara operasional.

Pada rank 2 aliran silang tumpang tindih lebih awal, pada rank 0 dan P-1 paling akhir. Pada fase tengah yang stabil dari jadwal, setiap rank menjalankan forward-arah-X yang tumpang tindih dengan backward-arah-Y. Komputasi sibuk. All-to-all dispatch untuk forward disembunyikan di dalam komputasi backward. All-to-all combine disembunyikan di dalam komputasi forward. Bubble diperas keluar.

### Pembukuan bubble

Bubble pipeline 1F1B standar (waktu terbuang per rank):

```
bubble_1F1B = (P - 1) * forward_chunk_time
```

Penyempurnaan Zero Bubble menurunkannya tetapi tidak sampai nol. DualPipe, pada fase stabil, memiliki bubble nol jika jumlah micro-batch habis dibagi dua kali kedalaman pipeline. Di luar fase stabil (warmup dan cooldown), ada sejumlah bubble, tetapi ia tidak tumbuh dengan jumlah micro-batch — sifat utama yang disorot makalah ini.

Dalam bahasa pemasaran: "bebas bubble". Dalam bahasa teknis: bubble tidak tumbuh dengan jumlah micro-batch. Analisis lanjutan Sea AI Lab (DualPipeV / Cut-in-half) menunjukkan bahwa bubble nol penuh hanya terjadi ketika Expert Parallelism bukan hambatan; dengan all-to-all yang digerakkan EP, selalu ada kompromi penjadwalan.

### DualPipeV — penyempurnaannya

Sea AI Lab (2025) mengamati bahwa replikasi parameter 2x boros ketika tumpang tindih komunikasi EP bukanlah poin utamanya. Jadwal DualPipeV mereka melipat suntikan bidireksional menjadi jadwal berbentuk "V" yang berjalan pada satu salinan parameter. Bubble sedikit lebih besar daripada DualPipe, tetapi penghematan memori substansial. DeepSeek mengadopsi DualPipeV dalam implementasi DualPipe sumber-terbuka mereka sebagai mode EP-off.

Trade-off-nya:

| Fitur | DualPipe | DualPipeV | 1F1B | Zero Bubble |
|-------|---------|-----------|------|------------|
| Salinan parameter per perangkat | 2 | 1 | 1 | 1 |
| Bubble vs micro-batch | konstan | pertumbuhan kecil | tumbuh | tumbuh |
| Tumpang tindih komputasi-komunikasi | penuh | sebagian | minimal | sebagian |
| Kapan dipakai | MoE EP-heavy | dense atau EP-ringan | baseline | pipeline apa pun |

### Artinya bagi proses 14,8T token

Pra-pelatihan DeepSeek-V3 mengonsumsi 14,8T token pada 2.048 GPU H800 dalam sekitar 2,8 juta jam-GPU. Dengan 1F1B naif, mereka akan kehilangan 12-15% dari itu karena bubble pipeline — 340-420 ribu jam-GPU, cukup untuk melatih satu model 70B penuh. DualPipe menyelamatkan sebagian besar dari itu. Mengkuantifikasi kontribusinya secara langsung sulit tanpa log internal, tetapi klaim dalam makalah adalah utilisasi GPU di atas 95% secara rata-rata sepanjang pelatihan.

Untuk proses yang lebih kecil (di bawah 1 ribu GPU), DualPipe berlebihan — bubble pipeline lebih kecil relatif terhadap total biaya, dan pelatihan model dense jarang menyentuh hambatan all-to-all. Untuk pelatihan MoE frontier dalam skala ribuan GPU, ini pada dasarnya wajib.

### Posisinya dalam stack

- Melengkapi **FSDP** (Fase 10 · 05). FSDP memecah parameter model ke seluruh rank; DualPipe menjadwalkan komputasi ke seluruh rank. Keduanya dapat digabung.
- Kompatibel dengan shard gradien **ZeRO-3**. Pencatatan untuk replikasi dua salinan perlu bekerja sama dengan gradien shard milik ZeRO.
- Memerlukan **kernel all-to-all kustom** yang disetel untuk topologi klaster spesifik. Kernel sumber-terbuka milik DeepSeek adalah implementasi referensi.

```figure
expert-capacity
```

## Pakailah

`code/main.py` adalah simulator jadwal pipeline. Ia menerima `(P, n_micro_batches, schedule)` dan mencetak utilisasi fase stabil untuk masing-masing 1F1B, Zero Bubble, DualPipe, dan DualPipeV. Ini adalah alat pengajar — angkanya cocok dengan klaim kualitatif dalam makalah, bukan klaim percepatan terukur di produksi.

Nilai simulatornya: jalankan dengan P dan jumlah micro-batch berbeda, lalu amati bagaimana fraksi bubble tumbuh pada 1F1B tetapi tidak pada DualPipe.

Pertimbangan integrasi untuk proses pelatihan sungguhan:

- Pilih kedalaman paralelisme pipeline yang habis membagi jumlah micro-batch Anda dengan bersih.
- Pastikan mesh paralelisme pakar Anda mendukung all-to-all bidireksional. Kernel milik DeepSeek adalah referensinya.
- Siapkan sekitar seminggu waktu debugging untuk jadwalnya pada pertama kali. Pencatatannya rumit.
- Pantau utilisasi GPU per rank, bukan hanya agregat. Manfaat DualPipe datang dari merapikan yang paling lambat.

## Kirimkan

Pelajaran ini menghasilkan `outputs/skill-dualpipe-planner.md`. Diberikan spesifikasi klaster pelatihan (jumlah GPU, topologi, interkoneksi, bentuk model), berkas itu merekomendasikan strategi paralelisme pipeline, algoritme penjadwalan yang dipakai, dan perkiraan fraksi bubble pada skala target.

## Latihan

1. Jalankan `code/main.py` pada `(P=8, micro_batches=16, schedule=dualpipe)` dan `(P=8, micro_batches=16, schedule=1f1b)`. Hitung selisih utilisasi GPU dan nyatakan sebagai jam-GPU yang diselamatkan per juta token pelatihan.

2. Gambar dengan tangan tabel jadwal untuk `(P=4, micro_batches=8, schedule=dualpipe)`. Tandai setiap slot waktu dengan ID micro-batch dan arahnya. Temukan slot waktu pertama yang tidak memiliki bubble.

3. Baca Gambar 5 laporan teknis DeepSeek-V3 (arXiv:2412.19437). Temukan jendela tumpang tindih untuk all-to-all dispatch di dalam chunk forward DualPipe. Jelaskan bagaimana jadwal komputasi menyembunyikannya.

4. Hitung overhead parameter 2x DualPipe untuk model dense 70B dengan P=8 tahap pipeline dan model MoE 671B dengan P=16 tahap pipeline. Tunjukkan mengapa overhead kasus MoE secara proporsional lebih kecil (sebagian besar parameter adalah pakar, yang tersebar di grup EP besar).

5. Bandingkan DualPipe dengan Chimera (penjadwal bidireksional pesaing dari 2021). Temukan dua sifat spesifik yang ditambahkan DualPipe dan tidak dimiliki Chimera, dengan Bagian 3.4 makalah sebagai referensi.

## Istilah Kunci

| Istilah | Yang orang katakan | Arti sebenarnya |
|---------|--------------------|-----------------|
| Bubble pipeline | "Waktu menganggur per rank" | Siklus GPU terbuang karena sebuah tahap pipeline menunggu masukan atau gradiennya |
| 1F1B | "Jadwal pipeline default" | Penjadwalan selang-seling satu forward / satu backward; baseline yang dikalahkan DualPipe |
| Zero Bubble | "Sea AI Lab 2023" | Membelah backward menjadi B (gradien masukan) dan W (gradien bobot); hampir merapikan pipeline sepenuhnya |
| DualPipe | "Jadwal DeepSeek-V3" | Pipeline bidireksional + tumpang tindih komputasi-komunikasi; bubble tidak tumbuh dengan jumlah micro-batch |
| DualPipeV | "Cut-in-half" | Penyempurnaan berbentuk V yang menghilangkan replikasi parameter 2x dengan biaya bubble yang sedikit lebih besar |
| Chunk | "Unit kerja pipeline" | Satu forward atau backward dari satu micro-batch melalui satu tahap pipeline |
| All-to-all dispatch | "Kirim token ke pakar" | Komunikasi lintas node yang merutekan token ke pakar MoE yang ditugaskan |
| All-to-all combine | "Bawa kembali keluaran pakar" | Komunikasi lintas node yang mengumpulkan keluaran pakar setelah MLP |
| Expert Parallelism (EP) | "Pakar lintas GPU" | Memecah pakar MoE ke seluruh rank sehingga GPU berbeda menyimpan pakar berbeda |
| Pipeline Parallelism (PP) | "Layer lintas GPU" | Memecah layer model ke seluruh rank; dimensi yang dijadwalkan DualPipe |
| Fraksi bubble | "Waktu GPU terbuang" | (bubble_time / total_time); fraksi yang didorong DualPipe menuju nol |

## Bacaan Lanjutan

- [DeepSeek-AI — DeepSeek-V3 Technical Report (arXiv:2412.19437), Bagian 3.3.2 dan Gambar 5](https://arxiv.org/abs/2412.19437) — referensi utama DualPipe
- [DeepSeek — repositori GitHub DualPipe](https://github.com/deepseek-ai/DualPipe) — implementasi referensi sumber-terbuka, termasuk mode DualPipeV (Cut-in-half)
- [Qi dkk. — Zero Bubble Pipeline Parallelism (arXiv:2401.10241, Sea AI Lab 2023)](https://arxiv.org/abs/2401.10241) — pendahulu Zero Bubble
- [Sea AI Lab — DualPipe could be better without the Dual](https://sail.sea.com/blog/articles/63) — analisis DualPipeV yang menginformasikan mode EP-off DeepSeek
- [Narayanan dkk. — PipeDream / 1F1B (arXiv:1806.03377, 2018-2021)](https://arxiv.org/abs/1806.03377) — jadwal 1F1B yang menjadi pembanding DualPipe
- [Huang dkk. — GPipe (arXiv:1811.06965, 2018)](https://arxiv.org/abs/1811.06965) — makalah asli paralelisme pipeline dan masalah bubble
