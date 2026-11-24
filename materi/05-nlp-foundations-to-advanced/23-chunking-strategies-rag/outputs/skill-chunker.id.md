---
name: chunker
description: Pilih strategi pemotongan, ukuran, dan tumpang tindih untuk korpus dan distribusi kueri tertentu.
version: 1.0.0
phase: 5
lesson: 23
tags: [nlp, rag, chunking]
---

Dengan korpus (jenis dokumen, panjang rata-rata, domain) dan distribusi kueri (faktoid / analitis / multi-lompat), hasilkan:

1. Strategi. Rekursif / kalimat / semantik / induk-dokumen / akhir / kontekstual. Alasan.
2. Ukuran potongan. Jumlah token. Alasan yang terkait jenis kueri.
3. Tumpang tindih. Bawaan 0; beri justifikasi jika >0.
4. Penerapan min/maks. Pengaman `min_tokens`, `max_tokens`.
5. Rencana evaluasi. Recall@5 pada himpunan evaluasi berstrata 50 kueri (faktoid, analitis, multi-lompat).

Tolak strategi pemotongan apa pun tanpa penerapan ukuran potongan min/maks. Tolak tumpang tindih di atas 20% tanpa ablasi yang menunjukkan manfaatnya. Tandai rekomendasi pemotongan semantik tanpa batas bawah jumlah token.
