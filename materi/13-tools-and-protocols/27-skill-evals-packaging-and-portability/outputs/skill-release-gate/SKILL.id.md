---
name: skill-release-gate
description: Evaluasi bundel Agent Skill untuk integritas struktural, kualitas pemicuan, perbaikan artefak, kebenaran skrip, keamanan, integritas pohon terpasang, dan portabilitas host target sebelum rilis.
license: MIT
metadata:
  lesson: "27"
---

# Gerbang rilis skill

Gunakan skill ini sebelum menerbitkan atau mendistribusikan bundel direktori Agent Skill.

## Alur kerja

1. Selesaikan `SKILL_ROOT` menjadi direktori absolut yang memuat `SKILL.md` yang terpasang ini. Jangan mengasumsikan cwd proses adalah bundel yang terpasang.
2. Selesaikan `TARGET_ROOT` dari direktori kerja ruang kerja asli, lalu selesaikan kandidat yang diberikan pengguna sebagai `TARGET_BUNDLE` absolut.
3. Baca `references/eval-contract.md` dari `SKILL_ROOT`.
4. Periksa kasus pemicu positif dan nyaris-melenceng dalam `evals/cases.json` di bawah `TARGET_BUNDLE`.
5. Periksa assertion baseline dan dengan-skill bersama dalam `evals/artifacts.json` di bawah `TARGET_BUNDLE`.
6. Periksa hasil skrip dan keamanan yang eksplisit dalam `evals/evidence.json` di bawah `TARGET_BUNDLE`.
7. Periksa kapabilitas runtime yang dinyatakan dalam `assets/hosts.json` di bawah `TARGET_BUNDLE`, lalu verifikasi hash berkas target terhadap `assets/manifest.json`-nya.
8. Untuk produksi, ganti prediksi, artefak, bukti, dan kapabilitas host yang deterministik dengan hasil yang ditangkap; tetapkan keempat mode tangkap; dan ikat setiap observasi pemicu mentah, kedua artefak, seluruh rangkaian bukti, dan matriks host yang tidak kosong ke sumber yang tidak kosong serta digest provenance SHA-256 yang cocok. Pemeriksaan lokal ini dapat menetapkan `localEvidenceReady`, tetapi hash yang dapat dihitung ulang secara lokal tidak membuktikan penangkapan.
9. Peroleh attestation JSON eksternal yang `evidenceRoot`-nya cocok dengan laporan, ditambah SHA-256 dari byte persisnya dari kebijakan atau kanal rilis tepercaya yang terpisah. Attestation harus berupa berkas regular di luar bundel target.
10. Sebelum eksekusi, tampilkan argv yang telah diselesaikan secara persis. Evaluator yang terpasang adalah `scripts/evaluate_skill.py` di bawah `SKILL_ROOT`. Untuk fixture pelajaran yang dikirim, susun argv dari `python3`, jalur evaluator absolut tersebut, `--fixture-demo`, dan `TARGET_BUNDLE` absolut. Untuk produksi, gunakan skrip terpasang yang sama dengan `--attestation`, `--trusted-attestation-sha256`, dan `TARGET_BUNDLE` absolut, tanpa `--fixture-demo`.
11. Kembalikan `checksPassed`, `fixturePassed`, `localEvidenceReady`, `trustAnchorValid`, `productionReady`, dan `passed` beserta akar bukti, mode evaluasi, pemeriksaan yang gagal, presisi, recall, setiap observasi pemicu mentah, laju eksekusi berulang per kasus, perbandingan artefak, bukti skrip dan keamanan, verifikasi pohon terpasang, dan matriks portabilitas. Sertakan jalur skrip yang diselesaikan, jalur target yang diselesaikan, cwd, argv persis, dan kode keluar. Tandai observasi yang tidak tersedia sebagai unverified.

## Kontrak keluaran

Kembalikan laporan evaluasi JSON lengkap. Pertahankan setiap pemeriksaan spesifik-lapis dan buktinya agar agregat yang lulus tidak dapat menyembunyikan kegagalan pada routing, artefak, skrip, keamanan, pohon terpasang, atau portabilitas. `fixturePassed` melaporkan fixture pengajaran yang berhasil. `localEvidenceReady` hanya melaporkan integritas digest lokal. `passed` bernilai true hanya ketika `productionReady` juga memiliki jangkar kepercayaan yang valid di luar bundel.

## Perilaku kegagalan

Jika konfigurasi tidak valid, provenance tidak ada atau tidak cocok, attestation tepercaya hilang atau tidak valid, hash berkas berbeda, kapabilitas wajib tidak ada, atau gerbang produksi mana pun gagal, berhentilah dengan hasil nonzero dan laporkan lapisan yang gagal. Jalur `--fixture-demo` yang eksplisit hanya boleh keluar dengan sukses ketika `fixturePassed` bernilai true, dan jalur itu tidak pernah menyatakan klaim rilis. Jangan pernah menerbitkan, memasang di tempat lain, memperbaiki bukti, membuat keputusan kepercayaan, atau melemahkan ambang batas secara otomatis.

Jangan menerbitkan bundel hanya karena SKILL.md dapat diurai atau satu prompt positif mengaktifkannya. Jangan menyebut paket portabel ketika sebuah host menggugurkan berkas pendamping wajib atau mengabaikan ekstensi runtime wajib.
