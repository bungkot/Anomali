---
name: mcp-registry-admission
description: Terima, pasang pin, pantau, karantina, dan putar balik rilis MCP Registry dengan bukti rantai pasok.
version: 1.0.0
phase: 13
lesson: 30
tags: [mcp, registry, provenance, admission, drift, rollback]
---

Dengan respons MCP Registry, identitas penerbit terverifikasi, bukti artefak, observasi server langsung, dan kebijakan lokal, hasilkan keputusan penerimaan (admission) dan paket bukti.

## Masukan wajib

- Sumber Registry, catatan server, metadata yang dikelola Registry, dan waktu pengambilan.
- Namespace terverifikasi serta metode autentikasi yang menetapkannya.
- Registry paket, identifier, versi persis, hasil kepemilikan, dan digest artefak yang dihitung.
- Hasil `server/discover` langsung, versi protokol, kapabilitas, informasi server diagnostik, dan deskriptor alat yang lengkap.
- Kapabilitas wajib, properti alat yang dilarang, peninjau, retensi bukti, dan kebijakan pemutaran balik.
- Pin yang sebelumnya diterima, rute saat ini, karantina, dan observasi kesehatan terkini.

## Prosedur

1. Validasi bentuk catatan Registry. Wajibkan `name`, `version`, dan `description` yang tidak kosong serta sedikitnya satu paket atau remote. Perlakukan versi skema Registry dan versi protokol MCP langsung sebagai nilai yang saling independen.
2. Baca status yang dikelola Registry dari `_meta["io.modelcontextprotocol.registry/official"].status` pada tingkat respons, bukan dari `_meta.status` langsung maupun dari catatan publikasi. Tolak penerimaan otomatis kecuali status bernilai `active`.
3. Bandingkan namespace nama secara persis dengan namespace yang ditetapkan oleh autentikasi tepercaya. Tolak awalan yang menyerupai dan slug yang kosong.
4. Gabungkan satu sumber eksekusi yang dideklarasikan dengan bukti terverifikasi. Untuk paket, cocokkan jenis registry, identifier, versi persis, dan transpor. Untuk catatan yang hanya memuat remote, cocokkan URL dan transpor dengan bukti endpoint yang diverifikasi secara independen. Wajibkan digest bukti SHA-256 tepercaya untuk kedua jenis sumber.
5. Hash catatan Registry kanonik. Hash objek provenansi yang menggabungkan sumber Registry, nama server, versi Registry, digest catatan, sumber terpilih, dan digest bukti sumber.
6. Amati endpoint langsung. Wajibkan versi protokol yang diterima, kapabilitas wajib, dan deskriptor alat lengkap. Pertahankan `_meta["io.modelcontextprotocol/serverInfo"]` dari hasil hanya untuk tampilan, log, dan debugging. Jangan pernah gunakan `serverInfo` yang dilaporkan sendiri, termasuk alias yang hanya ada langsung, sebagai otoritas penerimaan maupun keamanan.
7. Normalisasi hanya koleksi yang secara semantik tidak berurutan. Hash seluruh permukaan deskriptor yang dinormalisasi sehingga perubahan nama, deskripsi, skema, anotasi, atau kumpulan alat menimbulkan pergeseran (drift).
8. Terapkan kebijakan otorisasi, data, jaringan, dan peninjauan lokal. Publikasi Registry adalah bukti, bukan persetujuan lokal.
9. Pada persetujuan, simpan pin tak ubah yang memuat digest catatan, sumber eksekusi, provenansi, dan kumpulan alat. Jaga status persetujuan tetap terpisah dari status perutean aktif.
10. Tambahkan keputusan, alasan, referensi bukti, hash ledger sebelumnya, dan hash entri saat ini ke buku besar penerimaan. Sembunyikan kredensial dan argumen sensitif sebelum bukti disimpan.
11. Rekonsiliasikan status Registry dan deskriptor langsung secara terjadwal serta sebelum aktivasi. Karantina dan nonaktifkan pin ketika status berubah menjadi `deprecated` atau `deleted`, bukti sumbernya berubah, atau perilaku langsungnya berbeda. Pin yang dikarantina tidak pernah menjadi target pemutaran balik.
12. Putar balik hanya ke pin yang sebelumnya diterima, masih memenuhi syarat, sehat, dan tidak dikarantina. Catat pemulihan rute sebagai peristiwa baru. Jangan pernah menulis ulang publikasi lama maupun keputusan penerimaan sebelumnya.

## Penolakan keras

- Kepercayaan yang hanya berasal dari nama tampilan yang akrab atau dari keberadaan di Registry.
- Pemeriksaan namespace dengan `startswith`, atau namespace yang hanya diklaim oleh catatan yang dikirimkan.
- `latest`, rentang versi, atau koordinat paket tanpa digest artefak.
- Kolom penerbit yang meniru status atau verifikasi yang dikelola Registry.
- Bukti protokol, kapabilitas, atau deskriptor yang hilang atau malformed.
- Memperlakukan `serverInfo` yang dilaporkan sendiri sebagai otoritas namespace, provenansi, endpoint, atau penerimaan.
- Pembaruan pin deskriptor secara diam-diam setelah drift.
- Aktivasi versi yang `deprecated`, `deleted`, tidak dikenal, atau dikarantina.
- Pemutaran balik ke target yang tidak memiliki bukti penerimaan lengkap.
- Bukti ledger yang memuat token bearer, cookie, kredensial paket, atau argumen alat yang tidak perlu.

## Hasil

Kembalikan bagian-bagian berikut:

1. Keputusan: `approve`, `reject`, atau `quarantine`, disertai kode alasan yang stabil.
2. Bukti Namespace: namespace terverifikasi, sumber autentikasi, dan perbandingan nama secara persis.
3. Pin Publikasi: sumber Registry, nama server, versi persis, status, versi skema, dan digest catatan.
4. Pin Sumber: koordinat paket dan transpor, atau URL remote dan transpor, referensi hasil kepemilikan, serta digest bukti sumber.
5. Pin Runtime: versi protokol yang diterima, kapabilitas wajib, digest kumpulan alat, dan `serverInfo` diagnostik opsional.
6. Gabungan Provenansi: kolom kanonik dan digest provenansi yang dihasilkan.
7. Hasil Kebijakan: setiap kontrol yang lulus, gagal, atau tidak berlaku.
8. Peristiwa Ledger: nomor urut, waktu, peristiwa, hasil, kode alasan, referensi bukti, hash sebelumnya, dan hash entri.
9. Rencana Rekonsiliasi: pemeriksaan status, artefak, discovery, deskriptor, dan kesehatan berikutnya.
10. Rencana Pemutaran Balik: pin sebelumnya yang memenuhi syarat, langkah validasi, perubahan rute, jendela kesehatan, dan bukti yang harus disimpan.

Akhiri dengan satu fakta yang paling berpotensi mengubah keputusan jika masih belum ada. Jika tidak ada fakta wajib yang hilang, nyatakan pemeriksaan drift terjadwal berikutnya.
