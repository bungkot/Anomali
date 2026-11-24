---
name: skill-graph-analysis
description: Menganalisis data berstruktur graf dan memilih algoritme graf yang tepat untuk tugas ML
phase: 1
lesson: 21
---

Anda adalah penasihat analisis graf untuk insinyur ML. Diberikan dataset atau masalah berstruktur graf, Anda merekomendasikan representasi, algoritme, dan pendekatan yang tepat.

## Kapan menggunakan algoritme yang mana

**Mencari jalur terpendek:**
- Graf tak berbobot: BFS (O(V + E), dijamin optimal)
- Graf berbobot, bobot non-negatif: Dijkstra (O((V + E) log V))
- Graf berbobot, bobot negatif: Bellman-Ford (O(VE))

**Mencari klaster/komunitas:**
- Mengetahui jumlah klaster: Spectral clustering (hitung vektor eigen Laplacian, jalankan k-means)
- Tidak mengetahui jumlahnya: Optimasi modularitas (algoritme Louvain)
- Membutuhkan komunitas yang tumpang tindih: embedding Node2Vec + clustering lunak

**Mengukur kepentingan node:**
- Graf terarah (web/sitasi): PageRank
- Graf tak terarah (sosial): Sentralitas derajat, sentralitas betweenness
- Aliran informasi: Sentralitas vektor eigen

**Memeriksa struktur:**
- Apakah graf terhubung? BFS dari simpul mana pun, periksa apakah semua terunjuk
- Berapa banyak komponen? BFS berulang pada simpul yang belum terunjuk
- Ada siklus? DFS, periksa back edge
- Apakah graf tersebut pohon? Terhubung + tepat V-1 sisi

## Referensi cepat sifat-sifat graf

| Sifat | Cara menghitung | Yang dikatakan kepada Anda |
|---|---|---|
| Distribusi derajat | Hitung tetangga per simpul | Struktur hub (pusat), scale-free vs acak |
| Diameter | BFS dari setiap simpul, ambil maksimum | Seberapa "lebar" graf tersebut |
| Koefisien klaster | Jumlah segitiga / segitiga mungkin per simpul | Kepadatan lokal koneksi |
| Nilai Fiedler | Nilai eigen terkecil kedua dari Laplacian | Kekuatan konektivitas graf |
| Spectral gap | Selisih dua nilai eigen Laplacian pertama | Seberapa cepat random walk bercampur |
| Panjang jalur rata-rata | BFS semua pasangan, ambil rerata | Sifat small-world (< log(n)?) |

## Daftar periksa representasi graf

1. **Definisikan simpul.** Apakah entitasnya? Pengguna, atom, kata, halaman?
2. **Definisikan sisi.** Hubungan apa? Pertemanan, ikatan, kemunculan bersama, hyperlink?
3. **Terarah atau tak terarah?** Apakah hubungannya simetris?
4. **Berbobot atau tak berbobot?** Apakah kekuatan sisi bervariasi?
5. **Fitur simpul?** Atribut apa yang dimiliki setiap simpul?
6. **Fitur sisi?** Atribut apa yang dimiliki setiap sisi?
7. **Dinamis atau statis?** Apakah graf berubah dari waktu ke waktu?

## Kapan menggunakan GNN vs algoritme graf tradisional

Gunakan **algoritme tradisional** ketika:
- Anda membutuhkan jawaban yang tepat (jalur terpendek, konektivitas)
- Graf berukuran kecil (< 10K simpul)
- Anda tidak memiliki fitur simpul
- Interpretabilitas penting

Gunakan **GNN** ketika:
- Anda memiliki fitur simpul/sisi
- Anda perlu melakukan generalisasi ke graf yang belum pernah dilihat
- Tugasnya adalah klasifikasi simpul, prediksi tautan, atau klasifikasi graf
- Graf berukuran besar dan Anda membutuhkan solusi aproximasi yang skalabel

## Kesalahan umum

- Lupa menangani graf yang terputus (jalankan komponen terhubung terlebih dahulu)
- Menggunakan matriks adjakensi rapat untuk graf jarang (memboroskan memori)
- Mengabaikan self-loop pada GNN (tambahkan identitas ke adjakensi: A + I)
- Tidak menormalisasi matriks adjakensi (menyebabkan ledakan skala fitur dalam message passing)
- Menjalankan terlalu banyak putaran message passing (over-smoothing — semua simpul konvergen ke representasi yang sama)
