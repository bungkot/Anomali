# Model pemanggilan

| Kanal | Pemrakarsa | Seleksi | Penggunaan umum |
|---|---|---|---|
| Manusia eksplisit | Pengguna | Nama yang ditemukan persis | Pilihan alur kerja yang disengaja |
| Model atau agen implisit | Model atau agen otonom | Relevansi deskripsi plus kebijakan host | Perutean yang sadar konteks |
| Aplikasi terprogram | Runtime produk | Nama yang dikonfigurasi persis dan daftar izin target | Alur kerja produk yang deterministik |
| Komposisi skill | Skill lain atau subagen | Target persis, identitas pemanggil, dan kebijakan kedalaman | Dependensi alur kerja terbatas |
| Harness terprogram | Runtime evaluasi | Nama yang dikonfigurasi persis dan daftar izin target | Evaluasi yang deterministik |

Aktivasi oleh manusia dan model membentuk matriks 2x2: keduanya tidak, manusia saja, model saja, atau keduanya. Aktivasi oleh aplikasi, komposisi, dan harness adalah kanal terpisah dengan kebijakan target, pemanggil, dan kedalaman masing-masing.

Untuk perutean implisit, terapkan kelayakan aktor dan ekstensi host sebelum peringkat relevansi. Skill ber skor tinggi yang diblokir bukanlah pemenang; singkirkan dari himpunan seleksi dan nilai kandidat tersisa yang memenuhi syarat. Lakukan abstain bila himpunan yang memenuhi syarat kosong atau skor terbaiknya tidak mencapai ambang batas.

Kolom seperti `user-invocable` atau `disable-model-invocation` mungkin bermakna bagi host tertentu. Adapter boleh memberlakukannya, tetapi dokumentasi portabel tidak boleh mengklaim bahwa setiap runtime mengenali kolom atau nilai yang sama.
