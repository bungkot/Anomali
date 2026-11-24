---
name: self-improvement-auditor
description: Audit sebuah pipeline perbaikan diri atau constitutional AI yang diusulkan sebelum dijalankan dalam skala besar.
version: 1.0.0
phase: 10
lesson: 9
tags: [alignment, cai, grpo, rlhf, self-improvement, reward-hacking]
---

Diberikan sebuah pipeline pelatihan yang diusulkan dan mengklaim menggunakan Constitutional AI, RLAIF, GRPO, atau bentuk apa pun dari data preferensi yang dihasilkan sendiri, hasilkan sebuah audit dengan:

1. Aturan hadiah. Nyatakan verifier yang tepat (regex, sympy, rangkaian tes, juri LLM). Klasifikasikan sebagai deterministik, LLM-stokastik, atau hibrida. Tolak loop "perbaikan diri" apa pun yang tidak memiliki penyangga eksternal — model tidak bisa menarik sinyal dari ketiadaan.
2. Statistik grup. Untuk pipeline GRPO, konfirmasi ukuran grup, bagaimana keunggulan dihitung (z-skore vs peringkat relatif), dan apa yang terjadi ketika standar deviasi hadiah grup kolaps ke nol. Pipeline harus melewati atau menurunkan bobot grup dengan varians nol, bukan membagi dengan epsilon dan berpura-pura sinyalnya nyata.
3. Anggaran KL. Batas numerik pada KL kumulatif(kebijakan || referensi) selama satu kali proses. Pipeline harus berhenti, mereset, atau beralih ke referensi yang lebih "hangat" ketika batas tercapai. KL tak terbatas berarti pergeseran tak terbatas.
4. Lantai keanekaragaman. Batas bawah terukur pada standar deviasi hadiah per grup, varians panjang respons, atau entropi n-gram, mana pun yang sesuai dengan tugas. Jika lantai dilanggar selama N putaran berturut-turut, pipeline harus mencampur data manusia segar atau distribusi prompt yang lebih luas.
5. Kuota data manusia. Fraksi minimum dari campuran pelatihan yang harus tetap ditulis manusia, biasanya 5-10%. Pipeline yang hanya melakukan distilasi diri kolaps setelah 3-5 putaran. Sebutkan ini secara eksplisit.
6. Pengawas mode collapse. Tandai pemeriksaan otomatis: standar deviasi hadiah antar putaran, jumlah n-gram unik pada prompt yang disisihkan, distribusi panjang, tingkat penolakan. Jika salah satunya melampaui ambang batas, pelatihan dihentikan.
7. Pergeseran konstitusi. Untuk pipeline CAI, wajibkan berkas konstitusi berversi, catatan perubahan, dan "himpunan uji regresi konstitusional" — prompt yang perilaku yang diharapkannya tidak boleh berubah antar suntingan.

Tolak penyetujuan pipeline yang:
- mengklaim "nol data manusia" tanpa verifier eksternal apa pun (aturan, alat, lingkungan).
- menggunakan PRM tanpa probe peretasan hadiah proses (apakah model menulis langkah yang tampak benar tanpa memajukan pembuktian?).
- menjalankan lebih dari 5 putaran fine-tuning rejection sampling tanpa benchmark keanekaragaman yang disisihkan.
- berbagi model referensi dengan kebijakan (tanpa referensi berarti tanpa KL, berarti tanpa jangkar).
- memberi nilai dengan juri LLM yang merupakan model yang sama dengan kebijakan (kontaminasi juri).

Keluaran: audit satu halaman dengan lulus/gagal per gerbang, nilai terukur atau yang dinyatakan, dan langkah tepat dalam pipeline yang menghasilkan setiap sinyal. Jika ada gerbang yang gagal, cantumkan perubahan minimum yang layak yang akan mengubahnya menjadi lulus.
