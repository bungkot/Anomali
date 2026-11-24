---
name: skill-structured-outputs
description: Kerangka keputusan untuk memilih strategi keluaran terstruktur yang tepat berdasarkan penyedia, keandalan, dan kompleksitas
version: 1.0.0
phase: 11
lesson: 03
tags: [structured-output, json, schema, constrained-decoding, pydantic, function-calling]
---

# Strategi Keluaran Terstruktur

Saat membangun aplikasi LLM yang memerlukan data terstruktur, terapkan kerangka keputusan ini.

## Kapan memakai tiap pendekatan

**Berdasarkan prompt ("Kembalikan JSON"):** hanya untuk prototyping. Boleh diterima pada alat internal di mana kegagalan parsing sesekali masih bisa ditoleransi. Tambahkan try/except dengan percobaan ulang. Jangan pernah dipakai pada pipeline produksi.

**Mode JSON (flag API):** Anda memerlukan JSON yang dijamin valid tetapi skemanya sederhana atau fleksibel. Cocok bila Anda memvalidasi bentuknya di sisi aplikasi. Tersedia: OpenAI, Anthropic (melalui tool use), Google.

**Mode skema (dekode terbatas):** sistem produksi di mana setiap keluaran harus sesuai skema tertentu. Nol kegagalan parsing. Nol pelanggaran skema. Pakai ini sebagai default untuk tugas ekstraksi atau klasifikasi produksi apa pun. Tersedia: keluaran terstruktur OpenAI, Outlines, Guidance.

**Function calling / tool use:** model perlu memilih fungsi mana yang dipanggil, bukan hanya mengisi parameter. Anda memiliki beberapa skema dan model memilih yang tepat. Pakai juga saat berintegrasi dengan infrastruktur alat/fungsi yang sudah ada.

**Pustaka Instructor:** Anda ingin validasi Pydantic dengan percobaan ulang otomatis lintas penyedia. Pengalaman pengembang (DX) terbaik untuk proyek Python. Membungkus OpenAI, Anthropic, Google, dan model sumber terbuka.

## Panduan khusus penyedia

**OpenAI:** gunakan `response_format` dengan tipe `json_schema`. Dekode terbatas sudah bawaan. Model Pydantic bekerja langsung. Implementasi keluaran terstruktur paling andal.

**Anthropic:** gunakan tool use untuk keluaran terstruktur. Definisikan satu alat dengan skema yang diinginkan. Model mengembalikan argumen panggilan alat yang sesuai skema. Andal, tetapi memerlukan pola API tool use.

**Model sumber terbuka (vLLM, Ollama):** gunakan Outlines atau Guidance untuk dekode terbatas. Pustaka ini mengompilasi JSON Schema menjadi mesin state hingga yang menyaring token tidak valid selama generasi. Memerlukan inferensi lokal.

## Pedoman desain skema

1. Jaga skema tetap datar bila memungkinkan. Objek bersarang lebih dari 2 level meningkatkan error ekstraksi.
2. Gunakan enum untuk kolom kategorikal. Jangan mengandalkan model untuk mengarang string yang tepat.
3. Jadikan kolom ambigu sebagai wajib dengan dukungan null eksplisit, daripada opsional. Ini memaksa model mengambil keputusan.
4. Tambahkan deskripsi pada properti skema. Model membaca deskripsi itu sebagai instruksi.
5. Hindari tipe union (oneOf/anyOf) kecuali memang perlu. Tipe itu meningkatkan kompleksitas dekode.
6. Tetapkan minimum/maksimum pada angka. Ini menangkap nilai ekstrem halusinatif.
7. Gunakan minItems/maxItems pada array untuk mencegah keluaran kosong atau tak terbatas.

## Pola gagal umum dan perbaikannya

- **Model membungkus JSON dalam pagar markdown**: beralih dari berbasis prompt ke mode JSON atau mode skema
- **Sesuai skema tetapi salah secara faktual**: tambahkan langkah validasi LLM-as-judge setelah ekstraksi
- **Nilai enum tidak konsisten**: beralih ke dekode terbatas atau tambahkan normalisasi pasca-pemrosesan
- **Kolom opsional hilang**: jadikan wajib atau tambahkan nilai default dalam kode aplikasi
- **Ekstraksi sangat lambat**: dekode terbatas menambah latensi 5-15%; kurangi kompleksitas skema jika latensi menjadi perhatian
- **Array besar dengan item beragam**: potong masukan menjadi beberapa bagian, ekstrak per bagian, lalu gabungkan hasilnya

## Tangga Keandalan

| Pendekatan | Keberhasilan Parsing | Kesesuaian Skema | Upaya Penyiapan |
|------------|----------------------|------------------|-----------------|
| Berdasarkan prompt | ~90% | ~80% | 1 menit |
| Mode JSON | 100% | ~90% | 5 menit |
| Mode skema | 100% | ~99% | 15 menit |
| Dekode terbatas | 100% | 100% | 30 menit |
| Instructor + percobaan ulang | 100% | ~99,5% | 10 menit |
