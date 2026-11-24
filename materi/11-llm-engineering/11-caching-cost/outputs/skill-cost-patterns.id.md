---
name: skill-cost-patterns
description: Kerangka keputusan untuk optimasi biaya LLM -- strategi penembolokan, pembatasan laju, perutean model, dan kontrol anggaran
version: 1.0.0
phase: 11
lesson: 11
tags: [caching, cost-optimization, rate-limiting, model-routing, budget, llm-ops]
---

# Pola Optimasi Biaya LLM

Saat membangun aplikasi LLM yang perlu mengendalikan biaya, terapkan kerangka keputusan ini.

## Kapan harus mengoptimasi

**Optimasi segera ketika:**
- Pengeluaran LLM bulanan melebihi $500 atau 10% dari anggaran infrastruktur
- Biaya per kueri di atas $0.01 untuk produk konsumen
- System prompt Anda lebih dari 1.000 token dan dikirim pada setiap permintaan
- Lebih dari 30% kueri adalah duplikat atau hampir duplikat
- Anda sedang menskalakan dari 100 menjadi 10.000+ pengguna harian

**Jangan mengoptimasi dulu ketika:**
- Anda memiliki kurang dari 100 DAU dan masih memvalidasi product-market fit
- Pengeluaran bulanan di bawah $100 dan pertumbuhannya lambat
- Anda masih mengulang desain prompt (penembolokan mengunci Anda pada sebuah prompt)

## Pemilihan strategi penembolokan

### Exact caching

**Gunakan ketika:** temperature=0, prompt identik berulang, keluaran deterministik diperlukan.

```python
key = sha256(json.dumps({"model": m, "messages": msgs, "temp": 0}))
```

- Implementasi: 30 menit
- Tingkat tembakan: 10-25% pada kebanyakan aplikasi, 40-60% pada bot FAQ
- Latensi: <1ms (pencarian dict)
- Risiko: respons basi jika data yang mendasarinya berubah

**Lewati ketika:** temperature > 0, setiap kueri unik, data real-time diperlukan.

### Semantic caching

**Gunakan ketika:** pengguna mengajukan pertanyaan yang sama dengan kata berbeda, produk yang banyak FAQ, layanan pelanggan.

- Implementasi: 2-4 jam (embedding + kemiripan + penyimpanan)
- Tingkat tembakan: 15-35% di atas exact cache
- Latensi: 10-50ms (embedding + pencarian ANN)
- Risiko: positif palsu (mengembalikan jawaban cache yang salah untuk pertanyaan yang mirip tetapi berbeda)

**Panduan ambang:**
- 0.98+: sangat konservatif, hampir tidak ada positif palsu, tingkat tembakan lebih rendah
- 0.95: keseimbangan baik untuk T&A faktual
- 0.90: agresif, tingkat tembakan lebih tinggi tetapi berisiko jawaban salah
- 0.85: hanya untuk aplikasi berisiko rendah (saran, autocomplete)

**Lewati ketika:** setiap kueri memiliki konteks unik (generasi kode), respons harus mencerminkan data terbaru, ruang kueri tidak terbatas.

### Prompt caching penyedia

**Gunakan ketika:** system prompt > 1.024 token (OpenAI) atau minimum spesifik model, prefix yang sama dikirim berulang.

| Penyedia | Tindakan | Penghematan |
|----------|--------|---------|
| Anthropic | Tambahkan `cache_control: {"type": "ephemeral"}` pada pesan sistem | 90% pada prefix yang di-cache (setelah premium tulis 25%) |
| OpenAI | Tidak ada (otomatis) | 50% pada prefix yang di-cache |
| Google | Gunakan Context Caching API dengan TTL eksplisit | ~75% pada konteks yang di-cache |

**Lewati ketika:** system prompt berubah per permintaan, prompt lebih pendek dari panjang minimum.

## Aturan perutean model

### Berbasis kata kunci (sederhana, cepat)

```
simple:  <= 5 kata OR cocok dengan kata kunci FAQ -> gpt-4o-mini ($0.15/$0.60)
medium:  kueri umum, ringkasan                  -> claude-sonnet ($3/$15)
complex: "analyze", "compare", "debug"           -> gpt-4o ($2.50/$10)
```

- Implementasi: 1 jam
- Akurasi: 70-80%
- Penghematan: 40-60% dari biaya model

### Berbasis embedding (lebih akurat)

Embedding 50-100 kueri berlabel per kategori. Klasifikasikan kueri baru dengan tetangga terdekat.

- Implementasi: 4-8 jam
- Akurasi: 85-92%
- Penghematan: 50-70% dari biaya model
- Biaya tambahan: ~$0.02/1M token untuk embedding klasifikasi (diabaikan)

### Berbasis ML (standar produksi)

Latih klasifikator kecil (regresi logistik atau BERT kecil) pada pasangan kueri/model historis.

- Implementasi: 1-2 minggu
- Akurasi: 90-95%
- Penghematan: 60-75% dari biaya model
- Memerlukan: data latihan berlabel dari lalu lintas produksi

## Konfigurasi pembatasan laju

### Parameter token bucket per tingkatan

| Tingkatan | Ukuran Bucket | Laju Isi Ulang | RPM Maks | Batas Harian |
|------|-------------|-------------|---------|-----------|
| Free | 50K token | 500/detik | 10 | 50K |
| Pro | 500K token | 5K/detik | 60 | 500K |
| Enterprise | 5M token | 50K/detik | 300 | 5M |

### Daftar periksa implementasi

1. Simpan bucket di Redis (bukan di memori) untuk aplikasi multi-instansi
2. Gunakan operasi atomik (MULTI/EXEC) untuk mencegah kondisi balapan
3. Kembalikan header `Retry-After` pada respons penolakan
4. Lacak permintaan yang ditolak sebagai metrik (>5% penolakan = batas tingkatan terlalu ketat)
5. Terapkan degradasi anggun: tolak permintaan model mahal lebih dulu, pertahankan akses model murah

## Kontrol anggaran

### Pemutus arus tiga ambang

| Ambang | Tindakan | Dapat dibatalkan |
|-----------|--------|------------|
| 70% dari anggaran bulanan | Catat peringatan, beri tahu tim via Slack/PagerDuty | Ya (otomatis) |
| 85% dari anggaran bulanan | Arahkan seluruh lalu lintas ke model termurah | Ya (otomatis, siklus penagihan berikutnya) |
| 95% dari anggaran bulanan | Hanya sajikan respons dari cache, tolak panggilan LLM baru | Ya (reset manual atau siklus berikutnya) |

### Pelacakan biaya per pengguna

Lacak biaya kumulatif per pengguna. Tandai pengguna yang melebihi 10 kali median. Penyebab umum:
- Pengguna daya yang sah (tingkatkan tingkatannya)
- Loop injeksi prompt (bot mengirim permintaan otomatis)
- Integrasi yang tidak efisien (klien mencoba ulang pada setiap galat)

## Kolom pelacakan biaya

Catat setiap panggilan API dengan kolom-kolom ini:

```json
{
  "timestamp": "2026-04-02T10:30:00Z",
  "model": "gpt-4o",
  "input_tokens": 1523,
  "output_tokens": 487,
  "cached_input_tokens": 1024,
  "latency_ms": 1847,
  "cost_usd": 0.006142,
  "user_id": "user_abc123",
  "cache_status": "partial_hit",
  "request_category": "customer_support",
  "complexity_class": "medium",
  "routed_from": "gpt-4o"
}
```

### Metrik utama untuk dasbor

- **Biaya per kueri** (P50, P95, P99) -- per model, per fitur, per tingkatan pengguna
- **Tingkat tembakan cache** -- exact versus semantic, tren dari waktu ke waktu
- **Distribusi model** -- persentase lalu lintas per model, biaya per model
- **Laju pembakaran anggaran** -- pengeluaran saat ini versus proyeksi bulanan pada laju saat ini
- **Tingkat penolakan** -- persentase permintaan yang dibatasi lajunya, per tingkatan

## Kesalahan umum

| Kesalahan | Mengapa merugikan | Perbaikan |
|---------|-------------|-----|
| Cache dengan temperature > 0 | Keluaran non-deterministik, cache basi memberi variasi yang salah | Hanya cache panggilan temp=0, atau terima bahwa respons cache kehilangan keacakan |
| Ambang semantic cache terlalu rendah | Mengembalikan jawaban salah untuk kueri yang tampak mirip | Mulai dari 0.95, turunkan hanya setelah mengukur laju positif palsu |
| Tidak ada invalidasi cache | Respons menjadi basi saat data yang mendasarinya berubah | Tetapkan TTL (1 jam untuk data dinamis, 24 jam untuk statis), invalidasi saat data diperbarui |
| Mengarahkan semua lalu lintas ke model termurah | Kualitas turun, pengguna menyadarinya | Arahkan berdasarkan kompleksitas, ukur kualitas per tingkatan, tetapkan ambang kualitas minimum |
| Tidak ada batas per pengguna | Satu pengguna yang menyalahgunakan membakar seluruh anggaran | Selalu terapkan kuota per pengguna, meski besar |
| Mengabaikan token keluaran | Biaya keluaran 2-5 kali lebih mahal per token dibanding masukan | Tetapkan max_tokens secara tepat, gunakan stop sequence, ringkas keluaran |
| Cache sebelum prompt stabil | Cache terisi respons dari prompt lama | Aktifkan cache hanya setelah prompt final, siram cache saat prompt berubah |

## Referensi harga (per April 2026)

| Model | Masukan ($/1M) | Keluaran ($/1M) | Masukan Cache ($/1M) | Terbaik Untuk |
|-------|-------------|--------------|--------------------|---------| 
| gpt-4.1-nano | $0.10 | $0.40 | $0.025 | Tugas sederhana volume tinggi |
| gpt-4o-mini | $0.15 | $0.60 | $0.075 | Perutean sederhana, klasifikasi |
| gemini-2.5-flash | $0.15 | $0.60 | $0.0375 | Multimodal hemat anggaran |
| claude-haiku-3.5 | $0.80 | $4.00 | $0.08 | Tugas menengah yang cepat |
| o4-mini | $1.10 | $4.40 | $0.275 | Penalaran dengan anggaran terbatas |
| gemini-2.5-pro | $1.25 | $10.00 | $0.3125 | Konteks panjang, multimodal |
| gpt-4o | $2.50 | $10.00 | $1.25 | Penggunaan umum, function calling |
| claude-sonnet-4 | $3.00 | $15.00 | $0.30 | Keseimbangan kualitas/biaya |
| claude-opus-4 | $15.00 | $75.00 | $1.50 | Kualitas maksimal, penalaran kompleks |
