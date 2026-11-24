---
name: vit-configurator
description: Pilih varian ViT, ukuran patch, dan sumber pra-pelatihan untuk suatu tugas visi baru.
version: 1.0.0
phase: 7
lesson: 9
tags: [transformers, vit, vision]
---

Diberikan tugas visi (klasifikasi / segmentasi / deteksi / pengambilan), resolusi gambar, ukuran dataset (berlabel + tak berlabel), dan target deployment, keluarkan:

1. Backbone. Salah satu dari: DINOv2 ViT-L/14 (default untuk pengambilan/klasifikasi), encoder SAM 3 (segmentasi), SigLIP (vision-language), ConvNeXt (kritis terhadap latensi). Sertakan alasan satu kalimat.
2. Ukuran patch. 16 untuk klasifikasi standar pada 224, 14 untuk DINOv2, 8 untuk prediksi padat pada resolusi tinggi. Tandai panjang urutan `(H/P)^2 + 1` dan biaya atensi `O(N^2)`.
3. Sumber pra-pelatihan. Nama checkpoint. Untuk himpunan berlabel kecil (<10k): fitur DINOv2 dibekukan + linear probe. Untuk >100k: setel halus blok-blok terakhir. Jelaskan alasannya.
4. Resep pelatihan. Pengoptimal (AdamW), lr, augmentasi (RandAug, MixUp, Random Erasing), label smoothing (0.1 lazim), EMA.
5. Catatan risiko. Risiko rezim data (terlalu sedikit data untuk fine-tuning penuh), ketidakcocokan resolusi (pra-pelatih 224 → deploy 1024 tanpa interpolasi posisi), tidak adanya register token (dapat merusak fitur DINOv2).

Tolak merekomendasikan melatih ViT dari awal dengan kurang dari 1 juta gambar — baseline CNN akan menang. Tolak merekomendasikan ukuran patch yang menghasilkan panjang urutan > 4096 tanpa diskusi eksplisit tentang Flash Attention + varian hierarkis (Swin). Tandai setiap deployment yang mengubah resolusi input tanpa menginterpolasi embedding posisi.
