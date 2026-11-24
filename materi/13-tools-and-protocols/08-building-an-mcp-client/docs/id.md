# Membangun Klien MCP: Penemuan, Perutean, dan Fallback Dua Era

> Klien MCP modern mengulang kontraknya pada setiap permintaan. Keputusan kompatibilitas tersulitnya adalah mengetahui kapan server lama benar-benar lama dan kapan server modern sedang melaporkan kesalahan yang dapat diperbaiki.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 07
**Waktu:** ~85 menit

## Tujuan Pembelajaran

- Bangun setiap permintaan MCP `2026-07-28` dengan metadata terkini.
- Probe server stdio dengan `server/discover` dan pilih versi yang didukung bersama.
- Otorisasi probe lama yang dibatasi hanya untuk rekan (peer) yang dimasukkan dalam daftar izin secara eksplisit.
- Terima era lama hanya setelah memvalidasi hasil `initialize` positif untuk revisi yang didukung.
- Gabungkan daftar alat deterministik tanpa menimpa benturan secara diam-diam.
- Rutekan panggilan ke rekan yang memiliki tiap alat tanpa mengarang sesi protokol.

## Masalah

Host agen biasanya berbicara dengan lebih dari satu server MCP. Host harus menemukan tiap server, menggabungkan katalog alat, menyelesaikan nama ganda, merutekan panggilan, dan pulih dari kegagalan transpor.

Revisi `2026-07-28` menyederhanakan keadaan mantap karena setiap permintaan mandiri. Kompatibilitas membuat startup lebih rumit. Sebuah klien dapat menemui:

- server modern yang mendukung versi pilihan;
- server modern yang mengembalikan versi yang dikenali atau kesalahan header;
- server lama yang tidak pernah mengenal `server/discover`;
- server lama yang tetap diam sampai menerima `initialize`.

Menganggap setiap kesalahan probe sebagai indikasi server lama itu berbahaya. Permintaan modern yang salah bentuk, server yang kelebihan beban, proses yang mati, dan server lama dapat menghasilkan waktu habis atau penutupan koneksi yang sama. Sinyal-sinyal itu ambigu. Klien harus menggabungkan niat eksplisit operator dengan bukti protokol positif sebelum memilih era lama.

## Konsep

### Rekan, bukan sesi protokol

Simpan satu catatan rekan transpor untuk setiap proses server atau endpoint:

- pegangan transpor atau fungsi kirim;
- era dan versi protokol yang terpilih;
- kemampuan server terakhir yang ditemukan;
- daftar alat deterministik terakhir;
- id permintaan tertunda untuk korelasi;
- kesehatan transpor.

Ini adalah pencatatan di sisi klien. Ini bukan keadaan sesi protokol. Pada MCP modern, server tetap menerima versi dan kemampuan terkini pada setiap permintaan.

### Bangun setiap permintaan modern dari awal

```python
def modern_request(request_id, method, params, version, capabilities):
    return {
        "jsonrpc": "2.0",
        "id": request_id,
        "method": method,
        "params": {
            **params,
            "_meta": {
                "io.modelcontextprotocol/protocolVersion": version,
                "io.modelcontextprotocol/clientCapabilities": capabilities,
                "io.modelcontextprotocol/clientInfo": CLIENT_INFO,
            },
        },
    }
```

Jangan melampirkan metadata sekali saja pada objek koneksi lalu berasumsi metadata itu sampai ke kawat. Cap dan periksa permintaan hasil serialisasi akhir.

### Penemuan modern

`server/discover` mengembalikan versi yang didukung, kemampuan server, instruksi, petunjuk penembolokan, dan identitas server yang disarankan. Klien memilih versi modern tertinggi yang didukung bersama.

Penemuan bersifat opsional bagi klien yang hanya modern, tetapi disarankan pada stdio. Beberapa server lama menerima operasi sebelum inisialisasi, sehingga mengirim `tools/list` lebih dulu dapat menghasilkan keberhasilan yang ambigu. `server/discover` menciptakan batas era yang bersih.

### Probe kompatibilitas stdio

Klien stdio dua era mengirim `server/discover` dengan metadata modern pilihannya sebelum permintaan lain. Ada tiga kelas hasil:

1. **DiscoverResult.** Server bersifat modern. Pilih versi yang didukung bersama dan lanjutkan dengan metadata per permintaan.
2. **Kesalahan modern yang dikenali.** Server bersifat modern. Untuk `-32022`, pilih dari `data.supported` dan coba ulang dengan id permintaan baru. Untuk kesalahan header atau kemampuan, perbaiki permintaan. Jangan kirim `initialize`.
3. **Sinyal ambigu.** Kesalahan JSON-RPC yang tidak dikenali, waktu habis, penutupan koneksi, atau respons kosong tidak mengidentifikasi era. Gagal dengan aman (fail closed) kecuali rekan persis itu dikonfigurasi untuk kompatibilitas lama.

Kesalahan protokol modern yang dikenali antara lain:

- `-32020` HeaderMismatch
- `-32021` MissingRequiredClientCapability
- `-32022` UnsupportedProtocolVersion

Kesalahan modern yang dikenali tetap modern meskipun rekan ada dalam daftar izin lama. Begitu server membuktikan bahwa ia memahami kosakata kesalahan modern, mengirim `initialize` berarti menurunkan versi.

Jangan memperlakukan `-32601` sebagai bukti lama yang positif. Kode itu hanya membuat rekan yang secara eksplisit diizinkan memenuhi syarat untuk satu probe lama. Aturan yang sama berlaku untuk waktu habis, penutupan koneksi, atau respons kosong.

### Daftar izin adalah niat operator, bukan bukti

Kompatibilitas lama harus menjadi properti eksplisit dari satu konfigurasi rekan yang ditetapkan:

```python
client.add_server("archive", archive_transport, allow_legacy=True)
```

Ikat pilihan itu pada perintah atau endpoint yang dikonfigurasi. Jangan gunakan wildcard yang memungkinkan server sembarangan memilih sendiri semantik yang lebih lemah. Rekan tanpa `allow_legacy=True` gagal setelah hasil penemuan yang ambigu dan tidak pernah menerima `initialize`.

Daftar izin memberikan izin untuk melakukan probe. Daftar itu tidak memilih era. Klien mengirim satu `initialize` di bawah batas waktu yang ditegakkan transpor, lalu mensyaratkan semua hal berikut:

- respons JSON-RPC `2.0` dengan id permintaan yang cocok;
- tepat satu `result` dan tanpa `error`;
- sebuah `protocolVersion` dalam himpunan revisi lama yang dikonfigurasi klien;
- bidang `capabilities` bernilai objek;
- objek `serverInfo` dengan bidang `name` dan `version` berupa string tidak kosong.

Waktu habis, penutupan koneksi, respons kesalahan, hasil salah bentuk, id tidak cocok, atau revisi yang tidak didukung menyebabkan gagal dengan aman. Hanya hasil positif yang valid secara struktural yang memilih era lama. Kode meneruskan `legacy_probe_timeout_ms` ke adaptor transpor; adaptor stdio atau HTTP sungguhan harus menegakkan batas waktu itu, bukan sekadar mencatatnya.

Simpan era terpilih dalam tembolok untuk rekan transpor. Jangan melakukan probe ulang sebelum setiap panggilan.

### Lama adalah cabang kompatibilitas

Setelah probe yang dibatasi mengembalikan bukti lama positif yang valid, klien menggunakan versi lama terpilih tepat seperti yang ditentukan revisi itu:

1. Verifikasi amplop respons dan id korelasi.
2. Verifikasi bahwa revisi yang disepakati ada dalam himpunan lama yang dikonfigurasi.
3. Catat kemampuan dan identitas server yang telah divalidasi.
4. Kirim `notifications/initialized` hanya setelah semua pemeriksaan lulus.
5. Gunakan bentuk permintaan lama selama masa hidup transpor itu.

Cabang ini ada untuk interoperabilitas dengan rekan yang dikenal. Cabang ini bukan desain bawaan bagi server atau permintaan baru. Jika transpor dimulai ulang atau endpoint-nya berubah, buang tembolok era rekan dan lakukan negosiasi ulang.

### Menemukan dan menembolokkan alat

Untuk setiap rekan aktif, panggil `tools/list`. Hasil modern mencakup `resultType`, `ttlMs`, dan `cacheScope`. Patuhi petunjuk kesegaran dalam konteks otorisasi yang tepat. Ambil ulang setelah kedaluwarsa atau setelah peristiwa perubahan daftar yang dilanggan.

Klien harus memperlakukan `resultType` yang tidak ada dari server lama sebagai `"complete"`. Jangan mensyaratkan bidang tembolok modern pada respons dari era yang disepakati lebih awal.

Server sebaiknya mengembalikan urutan deterministik. Klien juga sebaiknya mengurutkan sebelum menggabung agar urutan registri lokal tidak bergantung pada waktu mulai proses.

### Penggabungan ruang nama yang aman terhadap benturan

Dua server dapat sama-sama mengekspos `search`. Pilih kebijakan yang dinyatakan:

1. **Awalan saat benturan.** Pertahankan nama kanonik pertama dan ekspos benturan berikutnya sebagai `<server>/<tool>`.
2. **Tolak saat benturan.** Jangan memuat duplikat dan tampilkan kesalahan konfigurasi yang jelas.
3. **Timpa diam-diam.** Jangan pernah gunakan ini. Kebijakan ini menyembunyikan server mana yang menerima tindakan yang dipilih model.

Simpan baik nama kanonik maupun nama lokal. Model melihat nama kanonik. `tools/call` yang keluar menggunakan nama lokal yang dinyatakan server pemiliknya.

### Merutekan panggilan

Perutean adalah pencarian murni:

```text
nama alat kanonik
  -> nama rekan + nama alat lokal
  -> id permintaan JSON-RPC baru
  -> metadata permintaan modern atau bentuk lama eksplisit
  -> id respons yang cocok
```

Jangan mengirim panggilan saat transpor pemiliknya tidak tersedia. Hubungkan ulang atau mulai ulang transpor, lalu jalankan lagi penemuan dan `tools/list`. Permintaan modern yang sedang berjalan dan hilang akibat transpor rusak dapat dicoba ulang dengan id JSON-RPC baru bila kebijakan keamanan operasi mengizinkan.

### Notifikasi dan langganan

Perubahan daftar dan sumber daya modern hanya tiba pada aliran `subscriptions/listen` yang dibuka klien. Klien mengirim filter notifikasi, menunggu `notifications/subscriptions/acknowledged`, dan mengorelasikan peristiwa dengan id permintaan listen pada metadata notifikasi.

Saat koneksi terputus, buka permintaan listen baru dan ambil ulang daftar atau sumber daya yang relevan. Aliran modern tidak dilanjutkan dengan `Last-Event-ID`.

### Tidak ada permintaan yang dimulai server

Server modern tidak memanggil klien dengan permintaan JSON-RPC independen untuk sampling, elicitation, atau roots. Server mengembalikan `input_required`, lalu klien mencoba ulang permintaan asli setelah memenuhi permintaan input yang tertanam.

Jangan memblokir pembaca respons rekan saat memenuhi input. Pertahankan korelasi dan buat id JSON-RPC baru untuk percobaan ulang.

```figure
tp-client-merge
```

## Gunakan

`code/main.py` menggunakan fungsi rekan dalam proses agar keputusan protokol tetap terlihat. Kode ini terhubung ke dua rekan modern dan satu rekan lama yang sengaja dimasukkan dalam daftar izin, lalu menggabungkan dan merutekan alat-alat mereka. Callable transpor menerima anggaran waktu habis agar cabang kompatibilitas tidak dapat menyembunyikan probe tanpa batas.

```bash
cd code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Uji-uji ini membuktikan batas-batas yang terlewat oleh demo biasa:

- permintaan modern mengulang metadata;
- `-32022` mencoba ulang penemuan modern tanpa inisialisasi;
- kesalahan modern yang dikenali tidak pernah menurunkan versi, bahkan untuk rekan yang diizinkan;
- waktu habis, penutupan koneksi, respons kosong, dan kesalahan yang tidak dikenali tidak memicu `initialize` tanpa daftar izin;
- rekan yang diizinkan menjadi lama hanya setelah hasil `initialize` yang valid dan didukung;
- hasil lama yang salah bentuk dan tidak didukung membiarkan rekan tetap tidak tersedia;
- era yang berhasil dipilih disimpan dalam tembolok selama masa hidup transpor.

## Kirim

Pelajaran ini menyertakan `outputs/skill-mcp-client-harness.md`. Kerangka ini menyediakan cap permintaan modern, negosiasi era stdio, penggabungan ruang nama deterministik, perutean, dan cabang kompatibilitas lama yang gagal dengan aman.

## Latihan

1. Buat server palsu mengembalikan `-32022` tanpa versi yang didukung bersama. Pastikan klien gagal alih-alih mengirim `initialize`.
2. Masukkan server lama palsu dalam daftar izin, buat probe `initialize` yang dibatasinya mengalami waktu habis, dan buktikan rekan tetap `unknown` serta tidak tersedia.
3. Tambahkan daftar alat dengan `cacheScope: "private"` untuk dua konteks otorisasi. Pastikan klien tidak pernah membagikan hasil tembolok satu konteks kepada konteks lainnya.
4. Ubah kebijakan benturan menjadi penolakan dan buat startup gagal dengan kedua nama rekan disertakan dalam kesalahan.
5. Tambahkan simulator `subscriptions/listen` yang berhenti dalam waktu tertentu. Saat aliran hilang, lakukan listen ulang dengan id permintaan baru dan ambil ulang alat.

## Istilah Kunci

| Istilah | Makna |
|---------|---------|
| Rekan (peer) | Catatan di sisi klien untuk satu transpor server beserta data yang ditemukan |
| Era protokol | Metadata per permintaan modern atau semantik inisialisasi lama |
| Probe penemuan | `server/discover` awal yang digunakan untuk mengidentifikasi era stdio |
| Kesalahan modern yang dikenali | Kesalahan yang membuktikan perilaku modern dan melarang fallback lama |
| Daftar izin lama | Konfigurasi operator yang mengizinkan satu probe kompatibilitas terbatas untuk rekan yang ditetapkan |
| Bukti lama positif | Hasil `initialize` yang valid dan berkorelasi untuk revisi lama yang secara eksplisit didukung |
| Ruang nama gabungan | Nama alat kanonik dari semua rekan aktif |
| Kebijakan benturan | Aturan awalan atau penolakan untuk nama alat ganda |
| Tembolok era | Perilaku modern atau lama terpilih yang disimpan untuk satu rekan transpor |
| Pemulihan transpor | Mulai ulang atau hubungkan ulang, temukan ulang, daftar ulang, dan coba ulang dengan aman menggunakan id baru |

## Bacaan Lanjutan

- [Spesifikasi MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/)
- [Penemuan Server MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [Transpor stdio MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [Penomoran Versi MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)
- [Alat MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
