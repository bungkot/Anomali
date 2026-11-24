# Rantai Pasok MCP Registry: Penerimaan, Pergeseran, dan Pemutaran Balik

> Sebuah entri registry memberi tahu Anda apa yang dinyatakan penerbit. Penerimaan produksi membuktikan apa yang Anda ambil, apa yang Anda amati, apa yang Anda setujui, dan apa yang dapat Anda pulihkan dengan aman.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 17 (gerbang dan registri), Fase 13 · 18 (autentikasi produksi)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Memisahkan publikasi Registry, provenansi paket, discovery runtime, dan persetujuan lokal.
- Memverifikasi namespace server MCP tanpa mempercayai nama di dalam catatannya sendiri.
- Memasang pin bukti publikasi, sumber eksekusi, provenansi, dan deskriptor langsung yang tak ubah.
- Mendeteksi perubahan status registry dan drift runtime setelah penerimaan.
- Memutar balik perutean ke versi yang sebelumnya diterima tanpa menulis ulang riwayat.
- Mempertahankan buku besar penerimaan yang membuktikan tamper dan menjelaskan setiap keputusan.

## Masalahnya

Anda menemukan `com.example/inventory` di sebuah registry. Deskripsinya tampak tepat. Paketnya ada. Server menjawab `server/discover`.

Itu bukan satu fakta. Itu adalah rantai fakta dari otoritas yang berbeda:

1. Seorang penerbit yang terautentikasi untuk sebuah namespace mengirimkan catatan.
2. Sebuah registry paket menyajikan artefak dengan identitas dan digest tertentu.
3. Sebuah endpoint yang berjalan melaporkan versi protokol, kapabilitas, alat, dan informasi server diagnostik.
4. Organisasi Anda memutuskan bahwa kombinasi persis ini diizinkan.

Menyederhanakan fakta-fakta itu menjadi "ia ada di registry, jadi percayalah" menciptakan titik buta rantai pasok. Publikasi yang valid tetap dapat berstatus `deprecated`. Sebuah tag paket dapat menunjuk ke artefak yang tidak terduga jika Anda tidak memasang pin digest-nya. Sebuah server dapat menambahkan alat destruktif setelah peninjauan. Pemutaran balik dapat diam-diam memilih versi yang tidak pernah diterima.

Perbaikannya adalah pengontrol penerimaan (admission controller) dengan bukti di setiap batas.

## Registry Adalah Indeks, Bukan Sistem Persetujuan Anda

Registry MCP resmi menyimpan metadata server. Catatan `server.json`-nya menamai sebuah versi server dan mendeklarasikan satu atau lebih paket maupun endpoint remote. Aturan publikasi menambahkan autentikasi namespace, pemeriksaan kepemilikan paket, aturan registry terbatas, dan lokasi metadata penerbit yang sempit.

Kontrol-kontrol itu menjawab pertanyaan publikasi. Kebijakan produksi Anda tetap menjawab pertanyaan deployment:

| Batas | Pertanyaan | Pemilik bukti |
|---|---|---|
| Namespace | Apakah penerbit diizinkan menggunakan nama ini? | Autentikasi Registry ditambah masukan namespace terverifikasi Anda |
| Catatan | Apa yang dinyatakan penerbit untuk versi ini? | Digest `server.json` yang tak ubah |
| Sumber eksekusi | Paket atau endpoint remote mana yang akan dieksekusi? | Kolom sumber yang dideklarasikan, hasil kepemilikan terverifikasi, transpor, dan digest tepercaya |
| Runtime | Apa yang diekspos endpoint sekarang? | `server/discover` dan deskriptor alat |
| Penerimaan | Apakah kebijakan Anda menyetujui rangkaian persis ini? | Pin lokal dan entri ledger |
| Operasional | Apakah masih aman, dan apa yang dapat menggantikannya? | Pemeriksaan drift, sinkronisasi status, kesehatan, dan rute pemutaran balik |

Versi skema Registry dan versi protokol MCP bersifat independen. Sebuah catatan dapat menggunakan skema server `2025-12-11` yang dipublikasikan sementara server langsung mendukung MCP `2026-07-28`. Jangan pernah menyimpulkan salah satu dari yang lain.

```figure
mcp-registry-admission
```

## Tujuh Kontrol dalam Satu Keputusan Penerimaan

### 1. Verifikasi namespace

Nama Registry resmi menggunakan namespace terautentikasi. Sebuah domain terverifikasi dapat dipetakan ke awalan domain yang dibalik. Misalnya, kendali atas `example.com` dapat menetapkan `com.example/*`.

Jangan terima pemeriksaan awalan string:

```python
server_name.startswith("com.example")
```

Itu juga menerima `com.exampleevil/tool`. Pisahkan nama pada `/`, wajibkan slug yang tidak kosong, dan bandingkan segmen namespace secara persis. Lebih penting lagi, teruskan namespace terverifikasi ke dalam penerimaan dari hasil autentikasi. Jangan turunkan kepercayaan dari catatan yang tidak tepercaya.

Namespace berbasis GitHub dan namespace berbasis domain menggunakan jalur autentikasi yang berbeda. Normalisasi kedua jalur itu menjadi satu masukan penerimaan: string namespace terverifikasi yang persis.

### 2. Gabungan provenansi

Untuk catatan paket, deklarasi dan artefak yang diambil harus digabungkan berdasarkan kolom eksplisit:

- jenis registry paket
- identifier paket
- versi paket
- hasil kepemilikan terverifikasi
- digest artefak yang diunduh

Validasi juga transpor paket yang dideklarasikan. Catatan yang hanya memuat endpoint remote tetap valid dan tidak boleh ditolak karena tidak memiliki paket. Untuk sumber remote, gabungkan URL dan jenis transpor yang dideklarasikan dengan kepemilikan endpoint yang diverifikasi secara independen serta digest dari bukti koneksi atau deployment yang tepercaya.

Kode pelajaran mendukung kedua jenis sumber dan melakukan hash pada sumber terpilih bersama sumber Registry, nama server, versi Registry, digest catatan, dan digest bukti. Digest provenansi yang dihasilkan adalah penunjuk ringkas ke seluruh rangkaian bukti. Ia bukan pengganti penyimpanan bukti itu sendiri.

Jangan pernah menerima digest yang hanya disediakan oleh artefak yang sedang Anda verifikasi. Hitunglah di batas pengambilan tepercaya, atau terimalah dari layanan paket yang hasil verifikasinya Anda validasi.

### 3. Pasang pin pada keputusan, bukan hanya versi

Versi Registry adalah identifier publikasi yang unik. Metadata yang dipublikasikan bersifat tak ubah. Catatan yang diubah memerlukan versi baru. Semantic versioning disarankan, tetapi Registry tidak mewajibkannya dan tidak menerima rentang versi.

Ini berarti `^1.4` bukan pin penerimaan. "latest" juga bukan. Pin yang berguna memuat:

```json
{
  "server": "com.example/inventory",
  "version": "1.0.0",
  "recordDigest": "...",
  "source": {"kind": "package", "registryType": "pypi"},
  "sourceDigest": "...",
  "toolsetDigest": "...",
  "provenanceDigest": "...",
  "registryStatus": "active"
}
```

Memasang pin pada beberapa lapisan memungkinkan Anda mengidentifikasi batas mana yang berubah. Perubahan digest catatan pada versi Registry yang sama adalah kegagalan integritas Registry. Perubahan digest sumber pada koordinat paket atau deployment remote yang sama adalah kegagalan integritas sumber eksekusi. Perubahan digest kumpulan alat adalah drift runtime.

### 4. Deteksi drift langsung

Penerimaan sebaiknya mengamati server yang benar-benar akan menerima lalu lintas. Panggil `server/discover`, daftarkan atau peroleh dengan cara lain deskriptor alat yang diekspos melalui jalur tepercaya Anda, dan verifikasi:

- `2026-07-28` ada dalam `supportedVersions`
- semua kapabilitas yang diwajibkan secara lokal hadir
- setiap deskriptor alat memiliki identitas dan permukaan skema yang wajib
- digest deskriptor yang dinormalisasi cocok dengan pin yang diterima pada pemeriksaan berikutnya

Nilai opsional `_meta["io.modelcontextprotocol/serverInfo"]` pada hasil adalah konteks tampilan, log, dan debugging yang dilaporkan sendiri. Catatlah sebagai bukti diagnostik, tetapi jangan pernah gunakan untuk menetapkan namespace, kepemilikan paket, kepemilikan endpoint, penerimaan, atau keputusan keamanan lainnya. Alias `serverInfo` langsung di luar `_meta` bukanlah kolom kontrak dan tidak boleh dinaikkan menjadi bukti diagnostik.

Normalisasi hanya kolom yang urutannya tidak bermakna. Contoh mengurutkan daftar alat berdasarkan nama yang stabil sebelum hashing, sehingga perubahan urutan daftar yang tidak berbahaya tidak menyebabkan drift. Contoh itu tidak membuang kolom deskriptor. Alat baru, skema yang diubah, deskripsi yang diubah, atau anotasi baru mengubah pin.

Contoh memperlakukan deskriptor yang malformed dan setiap perubahan digest deskriptor sebagai drift, mengarantina pin, menghapus rute aktifnya, dan memblokir versi tersebut sebagai target pemutaran balik. Kebijakan produksi dapat mengizinkan perubahan redaksional hanya melalui peninjauan baru, karena deskripsi memengaruhi pemilihan alat oleh model. Metadata yang "kosmetik" dapat mengubah perilaku agen.

### 5. Status Registry adalah keadaan langsung

API Registry melampirkan objek `_meta` pada tingkat respons di samping setiap catatan server. Kolom yang dikelola Registry berada di bawah `_meta["io.modelcontextprotocol.registry/official"]`. Teruskan objek `_meta` dari respons ke penerimaan dan bacalah `_meta["io.modelcontextprotocol.registry/official"].status`. Nilai `_meta.status` langsung bukanlah bentuk kabel resmi. Jangan campuradukkan metadata respons dengan `_meta` milik catatan publikasi. Status dapat berupa:

- `active`: dikembalikan secara default dan memenuhi syarat untuk penerimaan lokal
- `deprecated`: tetap dapat ditemukan dengan peringatan, tetapi bukan lagi pilihan otomatis yang aman
- `deleted`: disembunyikan secara default sementara catatan historisnya tetap tersedia melalui tampilan deleted atau inkremental

Sinkronkan status setelah penerimaan. Jika sebuah versi aktif menjadi `deprecated` atau `deleted`, karantina pin-nya dan hentikan perutean kerja baru ke sana. Simpan buktinya. Penghapusan dari daftar default bukan izin untuk menghapus jejak audit Anda.

Metadata kustom yang disediakan penerbit hanya boleh berada di bawah `_meta.io.modelcontextprotocol.registry/publisher-provided` dalam catatan publikasi. Metadata respons yang dikelola Registry terpisah. Jangan biarkan penerbit menetapkan status resminya sendiri.

### 6. Pemutaran balik berarti pemulihan rute

Publikasi yang tak ubah tidak disunting saat pemutaran balik. Pemutaran balik memilih pin yang sebelumnya diterima dan masih memenuhi syarat, lalu mengubah rute aktif.

Target yang aman harus:

1. Memiliki catatan penerimaan yang lengkap.
2. Masih memiliki status Registry `active` sesuai kebijakan Anda.
3. Tidak dikarantina oleh bukti runtime maupun keamanan.
4. Masih mengarah ke paket yang dipasangi pin dan kumpulan deskriptor langsung.
5. Lulus pemeriksaan kesehatan saat ini.

Contoh berfokus pada tiga kondisi pertama. Reconciler sungguhan sebaiknya mengambil ulang paket dan memeriksa ulang endpoint langsung sebelum aktivasi.

### 7. Tambahkan buku besar penerimaan

Basis data penerimaan menyatakan apa yang aktif. Buku besar menjelaskan mengapa.

Setiap entri contoh memuat nomor urut, waktu, peristiwa, server, versi, hasil, alasan, bukti, hash entri sebelumnya, dan hash entri itu sendiri. Mengubah hasil lama merusak verifikasi entri tersebut dan setiap tautan setelahnya.

Ini bersifat membuktikan tamper (tamper-evident), bukan secara ajaib kebal tamper (tamper-proof). Jangkarlah kepala buku besar secara berkala di domain kepercayaan terpisah, misalnya metadata rilis bertanda tangan atau penyimpanan write-once. Batasi siapa yang dapat menambah entri. Jauhkan token otorisasi, kredensial paket, argumen alat, dan data endpoint pribadi dari bukti.

## Bangun

Pengontrol yang dapat dijalankan ada di `code/main.py`. Pengontrol ini hanya menggunakan pustaka standar Python.

Mulailah dengan demonstrasi berbatas (finite):

```bash
cd phases/13-tools-and-protocols/30-mcp-registry-supply-chain-and-drift
python3 code/main.py
npx tsx code/main.ts
```

Demonstrasi melakukan lima operasi:

1. Terima `1.0.0` dengan namespace, provenansi paket, protokol, kapabilitas, dan alat yang cocok.
2. Terima `1.1.0` dan jadikan aktif.
3. Amati alat delete yang tidak terduga saat runtime.
4. Amati status Registry untuk `1.1.0` berubah menjadi `deprecated`.
5. Pulihkan perutean ke pin `1.0.0` yang masih diterima.

Bentuk yang diharapkan:

```json
{
  "admitted": [true, true],
  "driftAllowed": false,
  "rollbackAllowed": true,
  "activeVersion": "1.0.0",
  "ledgerValid": true
}
```

Baca implementasi dalam urutan ini:

1. `namespace_for_domain()` dan `namespace_matches()` menetapkan otoritas penamaan yang persis.
2. `digest()` dan `normalized_tools()` menghasilkan bukti deterministik.
3. `RegistryAdmissionController.admit()` menggabungkan publikasi, provenansi, runtime, dan kebijakan.
4. `check_live()` membandingkan observasi baru dengan pin.
5. `observe_registry_status()` mengarantina versi yang status Registrynya berubah.
6. `rollback()` mengaktifkan hanya target yang sebelumnya diterima dan masih memenuhi syarat.
7. `AdmissionLedger.verify()` mendeteksi perubahan pada riwayat yang tercatat.

## Gunakan

Letakkan pengontrol di antara discovery dan perutean:

```text
Registry sync -> artifact verifier -> live discovery -> admission controller -> route table
                                               |                 |
                                               v                 v
                                          evidence store    admission ledger
```

Gunakan identitas terpisah untuk tugas-tugas ini. Pekerja sinkronisasi Registry memerlukan akses baca terhadap metadata. Verifikator artefak memerlukan akses pengambilan paket. Reconciler rute memerlukan izin untuk mengaktifkan pin yang disetujui. Tidak ada di antara mereka yang memerlukan semua kredensial.

Nyatakan status rollout secara eksplisit. "Disetujui" berarti bukti lulus kebijakan. "Aktif" berarti rute saat ini memilihnya. "Dikarantina" berarti ia tidak dapat menerima kerja baru. "Digantikan" berarti versi lain yang diterima sedang aktif. Jangan menyandikan keempat makna itu dalam satu nilai Boolean.

Jalankan penerimaan sebelum mengekspos server dalam `tools/list`. Jika tidak, klien dapat menemukan alat pada celah antara publikasi dan evaluasi kebijakan.

## Laboratorium Interaktif

Anda akan melihat satu batas gagal pada satu waktu.

### Lab A: tabrakan namespace

Buka shell Python dari direktori kode:

```bash
cd phases/13-tools-and-protocols/30-mcp-registry-supply-chain-and-drift/code
python3 -q
```

Lalu jalankan:

```python
from main import namespace_matches
namespace_matches("com.example/inventory", "com.example")
namespace_matches("com.exampleevil/inventory", "com.example")
```

Hasil pertama adalah `True`; hasil kedua `False`. Ganti perbandingan persis dengan `startswith` secara lokal dan amati mengapa nama kedua melintasi batas. Pulihkan perbandingan persis sebelum melanjutkan.

### Lab B: drift deskriptor

```python
from main import *
times = iter(f"2026-08-21T12:00:{n:02d}+00:00" for n in range(10))
c = RegistryAdmissionController(clock=lambda: next(times))
meta = {OFFICIAL_META_KEY: {"status": "active"}}
c.admit(sample_record("1.0.0"), meta, "com.example", evidence_for("1.0.0"), sample_live("1.0.0"))
c.check_live("com.example/inventory", "1.0.0", sample_live("1.0.0", True))
```

Periksa alasan dan status rute. Paket dan catatan Registry tidak berubah. Permukaan alat runtime berubah, sehingga pengontrol mengarantina dan menonaktifkan pin. Inilah sebabnya kontrol rantai pasok harus berlanjut setelah pemasangan.

### Lab C: status dan pemutaran balik

Terima `1.1.0`, tandai sebagai `deprecated`, lalu coba kedua target pemutaran balik:

```python
c.admit(sample_record("1.1.0"), meta, "com.example", evidence_for("1.1.0"), sample_live("1.1.0"))
c.observe_registry_status("com.example/inventory", "1.1.0", "deprecated")
c.rollback("com.example/inventory", "1.1.0", "unsafe retry")
c.rollback("com.example/inventory", "1.0.0", "restore known release")
c.ledger.verify()
```

Target yang dikarantina ditolak. Pin aktif sebelumnya diterima. Ledger tetap valid.

## Laboratorium Latihan

Kembangkan pengontrol dengan gerbang persetujuan dua orang.

Persyaratan:

- Simpan persetujuan sebagai referensi bukti bertanda tangan, bukan nama yang dapat diubah dalam pin.
- Wajibkan dua identitas peninjau yang berbeda untuk kumpulan alat yang memuat alat dengan `destructiveHint: true`.
- Tolak identitas peninjau yang ganda.
- Pertahankan upaya penerimaan awal dalam ledger ketika persetujuan belum lengkap.
- Tambahkan pengujian untuk nol, satu, duplikat, dan dua persetujuan yang berbeda.
- Jangan mencatat tanda tangan, kredensial, atau argumen alat pribadi secara lengkap.

Berhasil berarti alat destruktif tidak dapat menjadi aktif sebelum kedua identitas menyetujui digest catatan, paket, dan kumpulan alat yang persis.

## Artefak yang Disertakan

Pelajaran ini menyertakan `outputs/skill-mcp-registry-admission.md`. Gunakan sebagai runbook datar yang dapat dipakai ulang ketika meninjau versi Registry baru atau menyelidiki drift. Artefak ini mendefinisikan masukan, aturan penolakan, paket bukti, rekonsiliasi status, dan bukti pemutaran balik tanpa bergantung pada nama kelas contoh.

## Verifikasi

Jalankan demonstrasi dan rangkaian deterministik:

```bash
cd phases/13-tools-and-protocols/30-mcp-registry-supply-chain-and-drift
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover -s code/tests -v
```

Verifikasi sebaiknya membuktikan:

- batas namespace yang persis menolak awalan yang menyerupai
- hanya status Registry resmi yang bernamespace yang dapat membuat sebuah versi memenuhi syarat
- bukti paket dan remote yang tidak terverifikasi atau tidak cocok ditolak
- metadata penerbit tidak dapat menyamar sebagai metadata yang dikelola Registry
- urutan alat dinormalisasi tanpa menyembunyikan perubahan deskriptor
- struktur paket dan alat yang malformed ditolak dengan aman
- `serverInfo` tetap bersifat diagnostik dan tidak pernah menjadi otoritas penerimaan
- drift deskriptor mengarantina, menonaktifkan, dan memblokir pemutaran balik ke pin
- perubahan status mengarantina pin aktif
- pemutaran balik tidak dapat memilih versi yang dikarantina atau tidak dikenal
- tamper pada ledger terdeteksi

## Mode Kegagalan Produksi

| Kegagalan | Mengapa terjadi | Respons wajib |
|---|---|---|
| Nama tampak valid tetapi namespace tidak pernah diautentikasi | Kebijakan mempercayai teks catatan | Tolak sampai verifikator namespace tepercaya memberikan awalan yang persis |
| Koordinat paket yang sama mengembalikan byte baru | Hulu yang dapat diubah atau distribusi yang disusupi | Hentikan aktivasi, simpan kedua digest, selidiki batas pengambilan |
| "Latest" berubah tanpa peninjauan | Pemilihan mengambang lolos dari pin | Resolusi hanya versi dan digest persis yang diterima |
| Alat baru muncul setelah persetujuan | Drift runtime atau deployment yang berbeda | Karantina rute dan ambil observasi deskriptor baru |
| Versi `deprecated` tetap aktif | Sinkronisasi status tidak ada atau tertunda | Rekonsiliasi status secara terjadwal dan sebelum aktivasi |
| Catatan `deleted` hilang dari sinkronisasi default | Klien hanya meminta catatan aktif | Gunakan rekonsiliasi inkremental atau yang sadar-`deleted` dan pertahankan riwayat lokal |
| Target pemutaran balik tidak pernah diterima | Kontrol rute dan status persetujuan terputus | Tolak pemutaran balik dan jalankan penerimaan baru untuk target |
| Ledger terverifikasi secara lokal setelah penyerang menulis ulang semua entri | Rantai hash tidak memiliki jangkar eksternal | Publikasikan kepala ledger bertanda tangan ke domain kepercayaan terpisah |
| Bukti memuat token bearer atau argumen alat | Logging menyalin seluruh permintaan | Redact saat pengumpulan dan simpan hanya bukti minimum |

## Aturan Operasional

Publikasi menjawab "dapatkah identitas ini menerbitkan nama ini?" Penerimaan menjawab "apakah kita akan mengeksekusi artefak persis ini dan mengekspos perilaku persis ini?" Jaga agar keputusan-keputusan itu terpisah, pasang pin pada setiap penggabungan, dan buat pemutaran balik memilih bukti, bukan ingatan.

## Bacaan Lanjutan

- [Persyaratan `server.json` Registry resmi](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/official-registry-requirements.md)
- [Kontrak OpenAPI Registry resmi](https://registry.modelcontextprotocol.io/openapi.yaml)
- [Discovery server MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
