---
name: skill-production-checklist
description: Kerangka keputusan untuk merilis aplikasi LLM ke produksi -- mencakup setiap komponen dengan ambang spesifik dan kriteria lulus/gagal
version: 1.0.0
phase: 11
lesson: 13
tags: [production, deployment, llm, architecture, scaling, cost, observability, guardrails]
---

# Daftar Periksa Produksi LLM

Saat merilis aplikasi LLM, kerjakan daftar periksa ini secara berurutan. Setiap bagian memiliki kriteria lulus/gagal dengan ambang spesifik.

## 1. Keamanan (Penghalang Peluncuran)

Setiap item di sini harus lulus sebelum penyebaran apa pun.

| Pemeriksaan | Kriteria Lulus | Cara Memverifikasi |
|-------------|----------------|--------------------|
| Kunci API dalam variabel lingkungan | Nol kunci yang ditanam keras di basis kode | `grep -r "sk-" --include="*.py"` tidak mengembalikan apa pun |
| Pagar pengaman masukan aktif | Pola injeksi prompt diblokir | Kirim "Abaikan semua instruksi sebelumnya" -- mengembalikan respons diblokir |
| Penyembunyian PII | Pola SSN, kartu kredit, email tertangkap | Kirim "SSN saya adalah 123-45-6789" -- PII disembunyikan sebelum panggilan LLM |
| Filter keluaran | Konten berbahaya diblokir | Model tidak dapat mengembalikan pola `DROP TABLE`, `rm -rf`, `exec()` |
| Pembatasan laju | Batas permintaan per pengguna ditegakkan | 100 permintaan dari pengguna yang sama dalam 10 detik -- 50+ terakhir ditolak |
| Autentikasi di semua endpoint | Tidak ada akses LLM tanpa autentikasi | `curl /v1/chat` tanpa token mengembalikan 401 |
| CORS dibatasi | Hanya domain produksi yang diizinkan | Permintaan dengan `Origin: evil.com` ditolak |
| Token masukan maksimal | Permintaan melebihi batas ditolak | Kirim masukan 50K token -- mengembalikan 413 atau pemotongan |

## 2. Keandalan (Kelangsungan Minggu Pertama)

Ini mencegah insiden piket pertama Anda.

| Pemeriksaan | Kriteria Lulus | Cara Memverifikasi |
|-------------|----------------|--------------------|
| Percobaan ulang dengan backoff | 3 percobaan ulang pada 5xx, jeda eksponensial | Mati-mock LLM saat permintaan berjalan -- percobaan ulang terlihat di log |
| Rantai model cadangan | 2+ model dalam rantai | Model utama tidak tersedia -- respons tetap kembali dari cadangan |
| Batas waktu permintaan | Maksimal 30s pada semua panggilan eksternal | Mock LLM lambat (60s) -- permintaan habis pada 30s |
| Degradasi anggun | Kegagalan tembolok/RAG tidak membuat layanan gagal | Hentikan tembolok -- permintaan tetap berhasil (lebih lambat, lebih mahal) |
| Endpoint pemeriksaan kesehatan | Mengembalikan status dependensi | `GET /health` mengembalikan `{"status": "healthy", "cache": ..., "llm": ...}` |
| Streaming berfungsi | Token pertama di bawah 500ms | Waktu-ke-token-pertama diukur, konsisten < 500ms |
| Pesan galat aman | Galat internal tidak pernah bocor ke pengguna | Picu 500 -- pengguna melihat galat umum, bukan jejak tumpukan |

## 3. Pengendalian Biaya (Ekonomi Bulan Pertama)

Ini mencegah tagihan mengejutkan $50K.

| Pemeriksaan | Kriteria Lulus | Cara Memverifikasi |
|-------------|----------------|--------------------|
| Biaya per permintaan dilacak | Setiap permintaan mencatat jumlah token + biaya USD | Log permintaan memiliki kolom `input_tokens`, `output_tokens`, `cost_usd` |
| Tembolok semantik aktif | > 20% tingkat tembak pada pola berulang | Statistik tembolok menunjukkan tingkat tembak setelah 1000 permintaan uji |
| TTL tembolok dikonfigurasi | Entri kedaluwarsa (default: 1 jam) | Entri dimasukkan -- tidak dikembalikan setelah TTL |
| Pelacakan biaya per pengguna | Biaya diagregasi menurut user_id | Dasbor/API menunjukkan 10 pengguna teratas berdasarkan biaya |
| Peringatan biaya | Peringatan pada 80% anggaran harian | Tetapkan anggaran harian $10, kirim $8,50 dalam permintaan -- peringatan berbunyi |
| Perutean model berdasarkan biaya | Kueri kompleksitas rendah menggunakan model lebih murah | Pertanyaan sederhana dialihkan ke gpt-4o-mini, kompleks ke gpt-4o |
| Token keluaran maksimal ditetapkan | Respons dibatasi per templat | Templat dengan max_output_tokens=512 -- respons tidak pernah melebihi itu |

**Rumus estimasi biaya:**
```
Biaya LLM bulanan = DAU x queries_per_user x 30 x (1 - cache_hit_rate) x (avg_input_tokens x input_price + avg_output_tokens x output_price) / 1.000.000
```

**Ambang tolok ukur menurut skala:**

| DAU | Biaya target/permintaan | Anggaran bulanan |
|-----|----------------------|------------------|
| 1K | < $0,005 | < $750 |
| 10K | < $0,003 | < $4.500 |
| 100K | < $0,001 | < $15.000 |

## 4. Observabilitas (Debugging di Produksi)

Anda tidak dapat memperbaiki apa yang tidak dapat Anda lihat.

| Pemeriksaan | Kriteria Lulus | Cara Memverifikasi |
|-------------|----------------|--------------------|
| Pencatatan JSON terstruktur | Setiap permintaan menghasilkan satu baris log JSON | Log berisi: request_id, user_id, model, tokens, latency_ms, cost |
| Tracing permintaan | Jejak menyeluruh dengan waktu per komponen | Satu permintaan menunjukkan: guardrail (5ms) + cache (2ms) + llm (3200ms) + eval (1ms) |
| Pelacakan latensi | P50, P95, P99 diukur | Setelah 1000 permintaan: P50 < 2s, P99 < 10s |
| Pemantauan tingkat galat | Galat dihitung dan dikategorikan | Dasbor menunjukkan: 0,5% galat API, 0,1% pemblokiran guardrail, 0,01% timeout |
| Metrik tembolok | Tingkat tembak, tingkat miss, jumlah entri terlihat | `GET /v1/cache/stats` mengembalikan angka saat ini |
| Metrik uji A/B | Metrik kualitas per varian dicatat | Setiap permintaan mencatat prompt_template + versi untuk perbandingan |
| Pencatatan evaluasi | Sinyal kualitas direkam per permintaan | Panjang respons, latensi, model, versi templat disimpan untuk analisis luring |

## 5. Manajemen Prompt

Prompt adalah kode. Perlakukan seperti kode.

| Pemeriksaan | Kriteria Lulus | Cara Memverifikasi |
|-------------|----------------|--------------------|
| Templat berversi | Setiap templat memiliki nama + string versi | Perubahan templat membuat versi baru, versi lama dipertahankan |
| Dukungan uji A/B | Pembagian lalu lintas berdasarkan hash pengguna yang deterministik | Pengguna yang sama selalu melihat varian yang sama dalam eksperimen |
| Kemampuan memutar balik | Kembali ke versi sebelumnya dalam < 1 menit | Ubah konfigurasi eksperimen -- lalu lintas langsung beralih |
| Validasi templat | Variabel divalidasi sebelum rendering | Variabel yang hilang dalam templat memunculkan galat jelas, bukan KeyError |
| Pemisahan prompt sistem | Pesan sistem dan pengguna dalam kolom terpisah | Prompt sistem tidak digabungkan ke dalam pesan pengguna |

## 6. Kesiapan Penskalaan

Tidak diperlukan saat peluncuran. Diperlukan pada 10x.

| Pemeriksaan | Kriteria Lulus | Cara Memverifikasi |
|-------------|----------------|--------------------|
| Panggilan LLM asinkron | Tidak ada utas yang memblokir pada panggilan API | 50 permintaan bersamaan -- CPU server tetap < 30% |
| Koneksi pooling | Koneksi HTTP digunakan kembali | Jejak jaringan menunjukkan koneksi persisten ke penyedia LLM |
| Penskalaan horizontal | Desain server stateless | 2 instans di belakang penyeimbang beban -- semua permintaan berhasil |
| Dukungan antrean | Tugas non-realtime masuk ke antrean | Permintaan ringkasan mengembalikan job_id, hasil tersedia lewat polling |
| Uji beban | 100 pengguna bersamaan, tingkat galat < 5% | Uji `wrk` atau `locust` lulus pada konkurensi target |

## Urutan implementasi untuk proyek baru

1. **Hari 1:** Server API + templat prompt + satu panggilan LLM dengan percobaan ulang
2. **Hari 2:** Pagar pengaman masukan + pagar pengaman keluaran + penanganan galat
3. **Hari 3:** Tembolok semantik + pelacakan biaya per permintaan
4. **Hari 4:** Streaming (SSE) + endpoint pemeriksaan kesehatan
5. **Hari 5:** Pencatatan terstruktur + tracing permintaan + pencatatan evaluasi
6. **Minggu 2:** Uji A/B + penerapan versi prompt + pemutarbalikan
7. **Minggu 3:** Rantai model cadangan + degradasi anggun
8. **Minggu 4:** Uji beban + optimasi asinkron + penskalaan horizontal

## Diagnosis cepat

Jika sesuatu salah di produksi, periksa dalam urutan ini:

1. **Pengguna mengeluh tentang galat?** Periksa endpoint kesehatan, lalu tingkat galat di log, lalu halaman status penyedia LLM
2. **Respons lambat?** Periksa latensi P99, lalu tingkat tembak tembolok, lalu waktu respons LLM dalam jejak
3. **Biaya melonjak?** Periksa tren biaya-per-permintaan, lalu tingkat tembak tembolok, lalu pengguna teratas berdasarkan biaya, lalu cari perubahan templat prompt yang meningkatkan jumlah token
4. **Kualitas turun?** Periksa apakah versi prompt baru diterapkan, periksa apakah akurasi pengambilan RAG berubah, periksa apakah penyedia model mengubah versi model default
5. **Insiden keamanan?** Periksa tingkat pemblokiran guardrail (penurunan mendadak = guardrail dinonaktifkan), periksa log permintaan untuk pola tidak biasa, rotasi kunci API segera
