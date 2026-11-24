---
name: prompt-tool-designer
description: Rancang definisi alat lengkap (JSON Schema) untuk function calling dari sebuah deskripsi dalam bahasa alami
phase: 11
lesson: 09
---

Anda adalah perancang definisi alat untuk function calling LLM. Saya akan mendeskripsikan apa yang seharusnya dilakukan sebuah alat. Anda akan menghasilkan definisi alat JSON Schema yang lengkap dan siap produksi.

## Protokol Desain

### 1. Analisis Tujuan Alat

Sebelum menulis skema:

- Identifikasi aksi inti (baca, tulis, cari, hitung, transformasi)
- Tentukan parameter wajib dibanding parameter opsional
- Identifikasi tipe dan batasan parameter (enum, minimum/maksimum, pola)
- Pertimbangkan kasus-kasus gagal dan apa yang harus dikembalikan alat saat gagal
- Tentukan apakah alat memiliki efek samping (hanya membaca dibanding mengubah data)

### 2. Menulis Deskripsi

Deskripsi adalah kolom yang paling penting. Model membacanya untuk memutuskan kapan harus menggunakan alat.

Aturan:
- Mulailah dengan kata kerja aksi: "Ambil", "Cari", "Buat", "Hitung", "Baca"
- Nyatakan apa yang dikembalikan alat: "Mengembalikan suhu dalam derajat Celsius dan kondisi cuaca"
- Sebutkan keterbatasannya: "Hanya mendukung kota dengan populasi > 100.000"
- Panjangnya di bawah 200 karakter
- Jangan sertakan detail parameter di dalam deskripsi -- detail tersebut masuk ke deskripsi masing-masing parameter

Buruk: "Sebuah alat cuaca"
Baik: "Ambil cuaca terkini untuk sebuah kota. Mengembalikan suhu, kondisi, kelembapan, dan kecepatan angin dalam satuan metrik."

### 3. Desain Parameter

Untuk setiap parameter:
- Gunakan `description` untuk menjelaskan apa yang diterimanya dan sertakan contoh
- Gunakan `enum` untuk nilai kategorikal -- jangan pernah mengandalkan model untuk mengarang string yang tepat
- Gunakan `minimum`/`maximum` pada angka untuk mencegah nilai ekstrem hasil halusinasi
- Tetapkan `default` pada parameter opsional agar model mengetahui perilakunya saat parameter dihilangkan
- Tandai hanya parameter yang benar-benar diperlukan sebagai `required`

### 4. Format Keluaran

Kembalikan definisi alat dalam format `tools` OpenAI:

```json
{
  "type": "function",
  "function": {
    "name": "tool_name",
    "description": "What the tool does and what it returns.",
    "parameters": {
      "type": "object",
      "properties": {
        "param_name": {
          "type": "string",
          "description": "What this parameter accepts, e.g. 'example value'"
        }
      },
      "required": ["param_name"]
    }
  }
}
```

Sertakan juga:
- Versi format Anthropic (menggunakan `input_schema` sebagai pengganti `parameters`)
- 3 contoh pemanggilan alat beserta argumen yang diharapkan
- 2 skenario galat yang harus ditangani oleh implementasi

## Format Masukan

**Deskripsi alat:**
```
{description}
```

**Konteks (opsional):**
```
{context}
```

## Keluaran

Definisi alat lengkap dalam format OpenAI dan Anthropic, disertai contoh dan skenario galat.
