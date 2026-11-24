---
name: mcp-reliability-reviewer
description: Tinjau balapan pembatalan, tenggat waktu, percobaan ulang, idempotensi, kontrol alur SSE, dan perilaku penyambungan ulang pada MCP.
version: 1.0.0
phase: 13
lesson: 29
tags: [mcp, reliability, cancellation, idempotency, sse, backpressure]
---

Tinjau satu operasi MCP dari awal permintaan hingga pemulihan dari kegagalan.

Mintalah masukan berikut jika belum ada:

1. Transpor: stdio atau Streamable HTTP.
2. Semantik operasi dan efek samping.
3. Batas waktu diam (idle timeout) dan batas waktu maksimum absolut.
4. Perilaku token progres dan batas laju.
5. Kebijakan percobaan ulang dan penyimpanan idempotensi bisnis.
6. Penyangga SSE, proksi, keepalive, dan pengaturan penyambungan ulang.
7. Perilaku Durable Task (tugas tahan lama), jika operasi menggunakan ekstensi Tasks.

Hasilkan bagian-bagian berikut.

## Kabel pembatalan

- Untuk stdio, wajibkan notifikasi `notifications/cancelled` yang dikirim klien dan merujuk ke id permintaan yang aktif.
- Untuk Streamable HTTP, wajibkan penutupan aliran respons permintaan tersebut. Tolak POST pembatalan biasa.
- Wajibkan tidak ada respons JSON-RPC untuk notifikasi pembatalan.
- Cadangkan `notifications/cancelled` yang dikirim server hanya untuk pembongkaran `subscriptions/listen` pada stdio.
- Nyatakan apa yang terjadi ketika pembatalan tidak dikenal, malformed, terlambat, atau mustahil.

## Tabel balapan

Berikan hasil yang deterministik untuk:

| Urutan peristiwa | Keadaan akhir server | Respons akhir | Perilaku klien |
|-------------|----------------------|----------------|------------------|
| Batalkan lalu selesaikan | Dibatalkan | Ditekan | Lanjutkan pemulihan |
| Selesai lalu batalkan | Selesai | Dipertahankan | Abaikan pembatalan yang terlambat |
| Waktu habis lalu respons terlambat | Pembatalan bergantung transpor | Abaikan atau tekan sesuai urutan yang teramati | Rekonsiliasi sebelum percobaan ulang yang tidak aman |
| Terputus saat progres | Permintaan yang sedang diproses dibatalkan | Tidak ada aliran yang dapat dilanjutkan | Sambung ulang dan ambil ulang |

## Kebijakan tenggat waktu

Wajibkan baik batas waktu diam maupun batas waktu maksimum absolut. Progres boleh mengatur ulang waktu diam hanya jika valid, monoton, dan dibatasi laju. Komentar keepalive tidak dihitung sebagai progres semantik. Batas waktu maksimum tidak pernah diatur ulang.

## Klasifikasi percobaan ulang

Kembalikan satu kelas per operasi:

- `SAFE`: kontrak aplikasi membuktikan tidak ada efek samping.
- `CONDITIONAL`: mutasi memiliki satu kunci idempotensi persisten yang digunakan ulang dengan argumen identik.
- `UNSAFE`: mutasi tidak memiliki deduplikasi otoritatif.

Jangan pernah menganggap id JSON-RPC baru, anotasi alat, atau penyambungan ulang transpor sebagai idempotensi bisnis.

Untuk percobaan ulang bersyarat, wajibkan sidik jari argumen yang tersimpan dan hasil yang telah dikomit. Tolak penggunaan ulang satu kunci dengan argumen yang diubah.

Wajibkan satu batas buku besar (ledger) yang atomik dan persisten untuk klaim kunci,
sidik jari argumen, catatan efek bisnis, dan hasil yang dikomit. Kamus atau pengunci
(lock) lokal proses tidak persisten dan tidak mengoordinasikan replika. Terima transaksi
basis data bersama, outbox transaksional, atau penyedia hulu yang memberlakukan kunci
yang sama. Nyatakan efek mana yang benar-benar berada di dalam batas tersebut.

Kembalikan salinan defensif dari hasil yang telah dikomit. Jangan pernah mengekspos objek
mutable yang disimpan ledger. Buktikan kedua sifat tersebut dengan fixture konkuren
ber-kunci sama, fixture pembukaan ulang, dan fixture alias mutasi.

## Kontrol alur

- Tetapkan kapasitas eksplisit per aliran atau per klien.
- Gabungkan progres yang dapat diganti.
- Tandai progres yang dibuang sebagai memerlukan pengambilan ulang otoritatif.
- Pertahankan respons JSON-RPC akhir.
- Tetapkan `Content-Type: text/event-stream`, `Cache-Control: no-cache`, dan `X-Accel-Buffering: no` untuk SSE.
- Tentukan irama keepalive secara terpisah dari progres operasi.
- Tolak antrean tak terikat.

## Rencana penyambungan ulang

- Buka permintaan baru dengan id baru.
- Pulihkan filter langganan.
- Jangan gunakan `Last-Event-ID` untuk MCP 2026-07-28.
- Ambil ulang sumber daya, daftar, prompt, atau Task yang terdampak.
- Jangan memutar ulang mutasi tidak aman secara otomatis.
- Gunakan backoff eksponensial berbatas dengan jitter.

## Durable Task

Ketika sebuah Task ada, jaga `tasks/cancel` tetap terpisah dari pembatalan permintaan. Sebuah acknowledgement lengkap hanya membuktikan bahwa niat pembatalan diterima. Lakukan polling atau dengarkan hingga Durable Task mencapai keadaan akhir (terminal).

## Fixture wajib

Kembalikan setidaknya skenario-skenario executable berikut:

1. pembatalan stdio sebelum penyelesaian;
2. penyelesaian stdio sebelum pembatalan yang terlambat;
3. penutupan aliran HTTP sebelum penyelesaian;
4. progres yang mengatur ulang batas waktu diam sementara batas waktu maksimum tetap berjalan;
5. mutasi duplikat dengan dua id JSON-RPC dan tanpa kunci idempotensi;
6. mutasi duplikat dengan satu kunci dan argumen identik;
7. penggunaan ulang kunci dengan argumen yang diubah;
8. konsumen lambat yang melewati penyangga progres;
9. aliran terputus diikuti langganan baru dan pengambilan ulang otoritatif;
10. acknowledgement `tasks/cancel` diikuti pembatalan pekerja (worker) yang tertunda;
11. dua koneksi ledger independen yang berbalapan pada satu kunci dan argumen identik;
12. perubahan oleh pemanggil pada hasil yang dikembalikan, diikuti pemutaran ulang yang bersih dari catatan persisten.

Tolak vonis siap produksi ketika kebijakan percobaan ulang hanya dideskripsikan sebagai
"coba ulang saat waktu habis", ledger bersifat lokal proses, efek eksternal berada di luar
batas atomik yang diklaim tanpa outbox atau kunci hulu, catatan yang dikembalikan mengarah
ke penyimpanan mutable, atau kapasitas antrean tidak ditentukan.
