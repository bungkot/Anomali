---
name: skill-function-calling-patterns
description: Kerangka keputusan untuk mengimplementasikan function calling di produksi -- desain alat, penanganan galat, keamanan, dan pola penyedia
version: 1.0.0
phase: 11
lesson: 09
tags: [function-calling, tool-use, agents, mcp, security, openai, anthropic]
---

# Pola Function Calling

Saat membangun aplikasi LLM yang menggunakan alat, terapkan kerangka keputusan ini.

## Kapan menggunakan function calling

**Gunakan function calling ketika:**
- Model membutuhkan data real-time (cuaca, harga saham, kueri basis data)
- Tugas memerlukan efek samping (mengirim email, membuat catatan, men-deploy kode)
- Model harus memilih di antara beberapa aksi berdasarkan niat pengguna
- Anda sedang membangun agen yang berinteraksi dengan sistem eksternal

**Gunakan structured outputs sebagai gantinya ketika:**
- Anda membutuhkan ekstraksi data dari teks (tidak perlu panggilan eksternal)
- Keluaran adalah produk akhir, bukan sebuah langkah antara
- Anda hanya memiliki satu skema, bukan beberapa alat untuk dipilih

**Gunakan keduanya ketika:**
- Model memanggil sebuah alat, lalu menyusun hasil alat tersebut ke dalam format keluaran tertentu

## Pedoman desain alat

1. **Satu alat, satu aksi.** Sebuah alat bernama `manage_database` yang menangani kueri, penyisipan, pembaruan, dan penghapusan terlalu luas. Pecah menjadi `query_records`, `insert_record`, `update_record`. Model memilih lebih baik dengan alat-alat yang spesifik.

2. **Deskripsi adalah prompt.** Model membaca deskripsi alat untuk memutuskan pemilihan. Tulislah deskripsi itu seolah Anda menulis instruksi untuk seorang pengembang junior. Sertakan apa yang dikembalikan alat, bukan hanya apa yang dilakukannya.

3. **Batasi dengan enum.** Jika sebuah parameter memiliki 3-10 nilai valid, gunakan enum. Model akan mengarang string -- "celsius", "Celsius", "C", "metric" -- kecuali Anda membatasinya.

4. **Lebih sedikit alat lebih baik.** GPT-4o menangani 5-10 alat dengan baik. Pada 20+ alat, akurasi pemilihan menurun. Pada 50+ alat, perkirakan 10-15% pemilihan alat salah. Kelompokkan fungsi yang terkait atau gunakan lapisan perutean.

5. **Required berarti wajib.** Tandai sebuah parameter sebagai required hanya jika alat itu benar-benar tidak dapat berfungsi tanpanya. Parameter opsional dengan nilai bawaan yang baik mengurangi kegagalan pemanggilan alat.

## Pola khusus penyedia

### OpenAI (GPT-4o, o3, GPT-4o-mini)

```python
tools=[{"type": "function", "function": {"name": ..., "parameters": ...}}]
tool_choice="auto"       # model memutuskan
tool_choice="required"   # harus memanggil setidaknya satu alat
tool_choice={"type": "function", "function": {"name": "specific_tool"}}
```

- Mendukung pemanggilan alat paralel (beberapa `tool_calls` dalam satu respons)
- ID pemanggilan alat harus dikirim kembali bersama hasilnya
- `gpt-4o-mini` 10 kali lebih murah dan menangani perutean alat sederhana dengan baik
- Mode structured outputs bekerja dengan parameter alat untuk menjamin kepatuhan terhadap skema

### Anthropic (Claude 3.5 Sonnet, Claude 4 Opus)

```python
tools=[{"name": ..., "description": ..., "input_schema": ...}]
tool_choice={"type": "auto"}     # model memutuskan
tool_choice={"type": "any"}      # harus memanggil setidaknya satu alat
tool_choice={"type": "tool", "name": "specific_tool"}
```

- Pemanggilan alat muncul sebagai blok konten dengan `type: "tool_use"`
- Hasil dikirim dalam pesan pengguna dengan `type: "tool_result"`
- Nama kolomnya adalah `input_schema`, bukan `parameters` (bug migrasi yang umum)
- Mendukung beberapa pemanggilan alat per respons

### Google (Gemini 2.0 Flash, Gemini 2.0 Pro)

```python
function_declarations=[{"name": ..., "description": ..., "parameters": ...}]
function_calling_config={"mode": "AUTO"}   # atau "ANY" atau "NONE"
```

- Menggunakan `function_declarations` di level teratas
- Hasil dikembalikan melalui bagian `function_response`
- Mendukung function calling paralel

### Model open-source (Llama 3, Hermes, Qwen)

- Tidak ada format standar -- berbeda menurut model dan kerangka penyajian (serving framework)
- Format Hermes (NousResearch) adalah konvensi fine-tuning yang paling umum
- vLLM mendukung pemanggilan alat yang kompatibel dengan OpenAI untuk model yang didukung
- Ollama mendukung pemanggilan alat dasar dengan model yang kompatibel
- Uji akurasi pemilihan alat sebelum produksi -- model terbuka 15-30% kurang akurat dibanding GPT-4o di Berkeley Function Calling Leaderboard

## Pola penanganan galat

### Kembalikan galat terstruktur

```json
{"error": true, "message": "City 'Toky' not found. Did you mean 'Tokyo'?", "code": "NOT_FOUND", "suggestions": ["Tokyo"]}
```

Sertakan informasi yang dapat ditindaklanjuti. "Tidak ditemukan" itu buruk. "Tidak ditemukan, maksud Anda X?" itu baik. Model menggunakan pesan galat untuk memperbaiki diri sendiri.

### Strategi percobaan ulang

1. Pemanggilan alat gagal dengan galat yang dapat diperbaiki (salah ketik, nilai enum salah)
2. Kirim galat kembali ke model sebagai hasil alat
3. Model menyesuaikan dan mencoba lagi
4. Maksimal 3 kali percobaan ulang per pemanggilan alat
5. Setelah 3 kegagalan, kembalikan galat kepada pengguna

### Penanganan waktu habis

Tetapkan batas waktu pada setiap eksekusi alat. 30 detik adalah nilai bawaan yang wajar. Jika sebuah alat mengalami waktu habis, kembalikan galat waktu habis terstruktur agar model dapat memberi tahu pengguna, bukan menggantung.

## Daftar periksa keamanan

| Pemeriksaan | Mengapa | Cara |
|-------|-----|-----|
| Buat daftar putih fungsi | Mencegah eksekusi kode sembarangan | Hanya daftarkan alat yang dibutuhkan pengguna |
| Validasi tipe argumen | Mencegah serangan kebingungan tipe | Periksa tipe sebelum eksekusi |
| Bersihkan argumen string | Mencegah injeksi | Tolak atau escape karakter khusus |
| Parameterisasi kueri basis data | Mencegah injeksi SQL | Jangan pernah meneruskan SQL hasil model secara langsung |
| Filter hasil alat | Mencegah kebocoran data | Hapus kunci API, PII, galat internal |
| Batasi laju pemanggilan alat | Mencegah loop tak terkendali | Maksimal 10-20 panggilan per percakapan |
| Catat semua pemanggilan alat | Jejak audit | Simpan nama alat, argumen, hasil, dan stempel waktu |
| Blokir path traversal | Mencegah akses sistem berkas | Tolak `..` dan path absolut pada alat berkas |
| Sandbox eksekusi kode | Mencegah akses sistem | Gunakan kontainer atau builtin yang dibatasi |
| Validasi ukuran keluaran | Mencegah context stuffing | Potong hasil di atas 10KB |

## Optimasi performa

- **Panggilan paralel:** Saat model meminta beberapa alat yang saling bebas, jalankan secara konkuren dengan `asyncio.gather()` atau `concurrent.futures`
- **Penembolokan (caching):** Simpan hasil alat untuk argumen yang identik dalam sesi yang sama (cuaca tidak berubah dalam 60 detik)
- **Streaming:** Alirkan respons akhir model sementara hasil alat masih diambil
- **Pemangkasan alat:** Jika konteks mepet, sertakan hanya definisi alat yang relevan dengan kueri saat ini (gunakan klasifikator untuk memfilter)
- **Model lebih kecil untuk perutean:** Gunakan `gpt-4o-mini` atau `claude-haiku-4-5` untuk pemilihan alat, lalu serahkan hasilnya ke model yang lebih kuat untuk disintesis

## Pola kegagalan umum

| Kegagalan | Penyebab | Perbaikan |
|---------|-------|-----|
| Alat salah dipilih | Deskripsi ambigu | Tulis ulang deskripsi dengan kata pemicu yang spesifik |
| Argumen wajib hilang | Model lupa sebuah parameter | Tambahkan contoh jelas di deskripsi parameter |
| Loop alat tak berhenti | Model terus memanggil alat yang sama | Tetapkan iterasi maksimal (5-10) dan deteksi panggilan berulang |
| Argumen halusinasi | Model mengarang nilai yang masuk akal tetapi salah | Gunakan enum, validasi terhadap nilai yang diketahui |
| Hasil alat terlalu besar | API mengembalikan 100KB data | Potong atau ringkas sebelum dikirim kembali |
| Model mengabaikan hasil alat | Format hasil membingungkan | Kembalikan JSON bersih dengan nama kolom yang jelas |
