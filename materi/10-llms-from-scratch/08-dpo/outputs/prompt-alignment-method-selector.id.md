---
name: prompt-alignment-method-selector
description: Pilih metode penyelarasan yang tepat (SFT, RLHF, DPO, KTO, ORPO, SimPO) untuk kasus penggunaan Anda
version: 1.0.0
phase: 10
lesson: 8
tags: [alignment, dpo, rlhf, kto, orpo, simpo, preference-optimization, fine-tuning]
---

# Pemilih Metode Penyelarasan

Saat memilih metode penyelarasan untuk sebuah model bahasa, gunakan kerangka kerja ini untuk mengevaluasi data, komputasi, dan kebutuhan kualitas Anda, lalu pilih metode yang paling sesuai dengan batasan yang ada.

## Persyaratan Masukan

Berikan:
- **Model dasar** (mis. Llama 3 8B, Mistral 7B, Qwen 2.5 72B)
- **Titik awal** (model dasar, atau sudah di-SFT?)
- **Data yang tersedia** (pasangan instruksi, pasangan preferensi, peringkat tanpa pasangan, atau tidak ada)
- **Anggaran komputasi** (jam GPU, jumlah GPU)
- **Target kualitas** (cukup baik untuk prototipe, setara dengan model open-source, state-of-the-art)
- **Linimasa** (hari, minggu, bulan)

## Matriks Keputusan

### Pemilihan Cepat

| Situasi Anda | Metode yang Direkomendasikan | Alasan |
|---------------|-------------------|-----|
| Tidak ada data preferensi, hanya pasangan instruksi | Hanya SFT | Anda tidak dapat menyelaraskan tanpa sinyal preferensi |
| < 5.000 pasangan preferensi, komputasi terbatas | DPO | Pipeline lebih sederhana, bekerja baik dengan data kecil |
| Umpan balik tanpa pasangan (hanya suka/tidak suka) | KTO | Satu-satunya metode yang bekerja tanpa perbandingan berpasangan |
| Ingin penyelarasan dalam satu kali pelatihan | ORPO | Menggabungkan SFT + penyelarasan, tanpa model referensi |
| Terbatas memori (tidak cukup untuk model referensi) | SimPO | Tidak memerlukan model referensi |
| Skala besar, penyelarasan multiobjektif | RLHF (PPO) | Model hadiah terpisah menangkap preferensi kompleks |
| Penyelarasan iteratif dengan data online | RLHF (PPO) | Dapat menghasilkan, menilai, dan melatih ulang dalam satu loop |
| Penyempurnaan pasca-RLHF | DPO | Fine-tuning model RLHF pada preferensi yang ditargetkan |

### Perbandingan Detail

| Metode | Kebutuhan Data | Model di Memori | Loop Pelatihan | Stabilitas | Skala Terbaik |
|--------|-----------------|-----------------|----------------|-----------|------------|
| SFT | Pasangan instruksi (10K+) | 1 | 1 | Tinggi | Semua |
| RLHF | Pasangan preferensi (20K+) | 3-4 | 3 | Rendah | Besar (70B+) |
| DPO | Pasangan preferensi (5K+) | 2 | 2 (SFT + DPO) | Tinggi | Kecil-Menengah (7B-70B) |
| KTO | Peringkat tanpa pasangan (5K+) | 2 | 2 (SFT + KTO) | Tinggi | Semua |
| ORPO | Pasangan preferensi (10K+) | 1 | 1 | Tinggi | Kecil-Menengah |
| SimPO | Pasangan preferensi (5K+) | 1 | 2 (SFT + SimPO) | Tinggi | Kecil-Menengah |

## Konfigurasi per Metode

### SFT

- **Kapan berhenti**: Setelah 1-3 epoch atau ketika loss validasi berhenti menurun
- **Hiperparameter utama**: Laju pembelajaran (1e-5 hingga 5e-5, lebih rendah untuk model yang lebih besar)
- **Detail penting**: Topengi token instruksi dalam loss
- **Jebakan**: Lebih dari 3 epoch menyebabkan hafalan; campurkan 2-5% data pra-pelatihan

### RLHF (PPO)

- **Kapan digunakan**: Anda memiliki 20K+ pasangan perbandingan, memerlukan penyelarasan multiobjektif, atau menginginkan pembelajaran online iteratif
- **Hiperparameter utama**: Koefisien KL (0,01-0,05), rasio klip PPO (0,1-0,3), laju pembelajaran (5e-6 hingga 3e-5)
- **Detail penting**: Model hadiah sebaiknya berukuran >= model kebijakan
- **Jebakan**: PPO tidak stabil; pantau divergensi KL dan kurva hadiah secara terus-menerus

### DPO

- **Kapan digunakan**: Anda memiliki pasangan preferensi dan menginginkan pipeline yang lebih sederhana daripada RLHF
- **Hiperparameter utama**: Beta (0,1-0,5; lebih rendah = lebih banyak penyimpangan dari referensi yang diizinkan)
- **Detail penting**: Model referensi harus berupa salinan beku dari checkpoint SFT
- **Jebakan**: Sangat sensitif terhadap beta; jalankan sapuan atas [0,05, 0,1, 0,2, 0,5]

### KTO

- **Kapan digunakan**: Anda hanya memiliki label "bagus" atau "buruk" tanpa perbandingan berpasangan
- **Hiperparameter utama**: Beta (sama seperti DPO), pengali aversi rugi (1,5x pada respons buruk)
- **Detail penting**: Membutuhkan contoh bagus/buruk yang kurang lebih seimbang (pembagian 40-60%)
- **Jebakan**: Tanpa pasangan, sinyal gradien lebih lemah; mungkin memerlukan lebih banyak data daripada DPO

### ORPO

- **Kapan digunakan**: Anda ingin melewati SFT sepenuhnya dan langsung dari model dasar ke model terselaraskan
- **Hiperparameter utama**: Lambda (bobot suku preferensi terhadap suku SFT)
- **Detail penting**: Membutuhkan label instruksi DAN pasangan preferensi dalam satu dataset
- **Jebakan**: Objektif gabungan bisa sulit diseimbangkan; jika loss SFT mendominasi, penyelarasan menjadi lemah

### SimPO

- **Kapan digunakan**: Pengaturan dengan memori terbatas di mana Anda tidak dapat memuat model referensi
- **Hiperparameter utama**: Beta, gamma (eksponen normalisasi panjang)
- **Detail penting**: Normalisasi panjang mencegah model memihak respons pendek
- **Jebakan**: Tanpa jangkar model referensi, model dapat menyimpang lebih jauh; pantau dengan saksama

## Templat Pipeline

### Templat 1: Prototipe Cepat (1-2 hari)

```
Model Dasar -> SFT (1 epoch, 10K contoh) -> DPO (3 epoch, 5K pasangan)
```

Komputasi: ~4 jam-GPU untuk model 7B pada A100
Kualitas: Pengikutan instruksi yang solid, penyelarasan preferensi dasar

### Templat 2: Kualitas Produksi (1-2 minggu)

```
Model Dasar -> SFT (2 epoch, 50K contoh) -> DPO (5 epoch, 20K pasangan) -> Eval -> Iterasi
```

Komputasi: ~40 jam-GPU untuk 7B, ~200 jam-GPU untuk 70B
Kualitas: Setara dengan model RLHF open-source

### Templat 3: State-of-the-Art (1-3 bulan)

```
Model Dasar -> SFT (2 epoch, 100K+ contoh) -> RLHF (PPO, 50K+ pasangan) -> DPO (penyempurnaan terarah) -> Eval -> Iterasi
```

Komputasi: ~500+ jam-GPU untuk 70B
Kualitas: Mendekati penyelarasan model frontier

### Templat 4: Data Minimumnya (1-2 hari)

```
Model Dasar -> SFT (1 epoch, 5K contoh) -> KTO (suka/tidak suka tanpa pasangan dari pengguna)
```

Komputasi: ~2 jam-GPU untuk 7B
Kualitas: Lebih baik daripada hanya SFT dengan overhead pengumpulan data yang minimal

## Protokol Evaluasi

Setelah penyelarasan, evaluasilah pada dimensi-dimensi berikut:

1. **Tingkat kemenangan preferensi**: Bandingkan model terselaraskan dengan model SFT pada 200+ prompt uji dengan juri manusia. Target: > 60% tingkat kemenangan.
2. **Retensi benchmark**: MMLU, HumanEval, atau benchmark spesifik domain. Tidak boleh turun > 5% dari baseline SFT.
3. **MT-Bench atau AlpacaEval**: Benchmark kualitas penyelarasan standar. Bandingkan dengan baseline yang dipublikasikan.
4. **Evaluasi keamanan**: Uji terhadap prompt adversarial, jailbreak, dan kategori permintaan berbahaya.
5. **Keanekaragaman respons**: Ukur entropi respons atas 100 prompt. Entropi rendah = mode collapse.

## Mode Kegagalan Umum

| Gejala | Penyebab | Perbaikan Spesifik Metode |
|---------|-------|-------------------|
| Respons panjang bertele-tele | Model hadiah / hadiah implisit memihak panjang | DPO: naikkan beta. RLHF: tambahkan penalti panjang. SimPO: sesuaikan gamma. |
| Model menyetujui segalanya | Sopan berlebihan (sycophancy) dari bias data preferensi | Tambahkan pasangan preferensi di mana respons yang benar tidak setuju dengan pengguna |
| Menolak permintaan yang tidak berbahaya | Penyelarasan berlebih pada data keamanan | Kurangi proporsi contoh keamanan, tambahkan lebih banyak pasangan penolakan yang tidak perlu |
| Keluaran hampir identik dengan SFT | Beta terlalu tinggi (DPO/KTO) atau koefisien KL terlalu tinggi (PPO) | Turunkan beta / koefisien KL; model tidak sedang belajar |
| Loss pelatihan berosilasi | Laju pembelajaran terlalu tinggi atau data tidak cukup | Turunkan lr 2-3x; tambahkan data preferensi |
