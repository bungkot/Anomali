# Rekayasa Kepatuhan MCP: Penomoran Versi, Bukti, dan Operasional

> Sebuah server tidak patuh hanya karena jalur normalnya berhasil melalui satu SDK. Kepatuhan hidup di kabel, pada batas versi, melintasi perantara, dan saat pemutaran balik.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 09 (transpor), Fase 13 · 17 (gerbang), Fase 13 · 30 (penerimaan registry)
**Waktu:** ~100 menit

## Tujuan Pembelajaran

- Mengubah aturan normatif MCP menjadi transkrip kabel golden dan negatif.
- Menjaga perilaku ketat `2026-07-28` tetap terpisah dari fallback lawan yang terbatas.
- Membedakan kolom asing yang aditif dari `resultType` asing yang tidak valid.
- Membandingkan bukti JSON-RPC mentah dengan tampilan yang dinormalisasi SDK.
- Membuktikan integritas header dan body melintasi batas proksi sungguhan.
- Menggating rilis dengan bukti transkrip yang diredact, kesehatan, dan pemutaran balik.

## Masalahnya

Klien Anda memanggil `tools/list` melalui SDK dan mendapatkan alat. Uji integrasi lulus.

Hasil itu meninggalkan pertanyaan penting yang belum terjawab:

- Apakah permintaan membawa metadata protokol per permintaan modern?
- Apakah `MCP-Protocol-Version`, `Mcp-Method`, dan `Mcp-Name` cocok dengan body JSON-RPC?
- Apakah respons memuat `resultType` yang valid di kabel, atau apakah SDK yang mensintesiskannya?
- Apakah klien akan mempertahankan kolom aditif di masa depan?
- Apakah galat modern yang dikenal akan memicu jabat tangan lawan secara tidak sengaja?
- Apakah proksi mempertahankan status origin dan galat JSON-RPC?
- Apakah serializer notifikasi mengeluarkan respons yang dilarang?
- Dapatkah operasional membuktikan mengapa sebuah rilis dipromosikan atau diputar balik tanpa menyimpan rahasia?

Kepatuhan adalah sekumpulan invarian yang dapat diamati. Bangunlah harness yang menangkap invarian-invarian itu sebelum lalu lintas produksi yang menemukannya.

```figure
mcp-conformance-operations
```

## Mulai dari Era Versi

MCP `2026-07-28` menggunakan metadata per permintaan yang mandiri. Permintaan modern membawa `params._meta.io.modelcontextprotocol/protocolVersion` dan `params._meta.io.modelcontextprotocol/clientCapabilities`. Kunci bernamespace yang persis penting; alias `protocolVersion` atau `clientCapabilities` yang telanjang bersifat malformed. Ketika header perutean cermin hadir pada batas HTTP, nilainya harus sepakat dengan body JSON-RPC. Hasil sukses modern membawa `resultType`.

Versi hingga `2025-11-25` menggunakan era inisialisasi sebelumnya. Hasil lawan tanpa `resultType` ditafsirkan sebagai lengkap hanya setelah klien memilih era awal tersebut.

Jangan buat satu validator permisif yang menerima kedua bentuk sekaligus. Gunakan dua cabang:

| Cabang | Bukti masuk | `resultType` hilang | Inisialisasi |
|---|---|---|---|
| Modern | `server/discover` yang berhasil atau respons modern yang dikenal | Tidak valid | Bukan jalur default |
| Lawan | Daftar izin yang dikonfigurasi ditambah hasil `initialize` lawan yang valid setelah probe modern yang tidak konklusif | Ditafsirkan sebagai lengkap | Diwajibkan oleh era tersebut |

Pemisahan ini mencegah rekan modern yang malformed mendapat keuntungan berupa validasi yang lebih lemah.

### Mode ketat

Mode ketat mensyaratkan bukti perilaku modern. `server/discover` yang berhasil membuktikan cabang modern. Galat JSON-RPC modern yang dikenal juga membuktikannya. Perbaiki permintaan atau berhenti. Jangan pernah menurunkan versi hanya karena server mengembalikan `-32020`, `-32021`, atau `-32022`.

### Mode fallback

Mode fallback melakukan satu probe modern yang terbatas. Waktu habis, balasan kosong, koneksi tertutup, atau respons yang tidak dikenal bersifat tidak konklusif. Hal itu tidak membuktikan bahwa rekan tersebut lawan. Hanya endpoint yang secara eksplisit dikonfigurasi atau dimasukkan dalam daftar izin untuk kompatibilitas yang boleh menerima probe lawan yang terbatas, dan klien memilih cabang lawan hanya setelah memvalidasi hasil `initialize` dari probe tersebut serta revisi lawan yang dinegosiasi.

Fallback bukanlah "coba lawan setelah galat apa pun." Galat modern yang dikenal memuat informasi koreksi yang berguna. Menurunkan versi setelahnya dapat menyembunyikan ketidaksesuaian header, deklarasi kapabilitas yang hilang, atau versi yang tidak didukung.

Ini mencegah penyerang, pemadaman, atau proksi penyaring memaksa penurunan versi dengan menjatuhkan respons modern. Catatlah kebijakan endpoint, observasi modern yang tidak konklusif, bukti lawan positif yang persis, dan era yang dipilih secara bersamaan.

Catat era yang dipilih di samping setiap transkrip. Tanpa fakta itu, kolom yang hilang dapat tampak dapat diterima pada satu menjalankan pengujian dan tidak valid pada yang lain.

## Bangun Korpus Transkrip

Fixture transkrip mencatat apa yang melintasi batas, bukan hanya panggilan SDK:

```json
{
  "name": "golden-modern-list",
  "era": "modern",
  "headers": {
    "MCP-Protocol-Version": "2026-07-28",
    "Mcp-Method": "tools/list"
  },
  "request": {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list",
    "params": {
      "_meta": {
        "io.modelcontextprotocol/protocolVersion": "2026-07-28",
        "io.modelcontextprotocol/clientCapabilities": {}
      }
    }
  },
  "responseStatus": 200,
  "responseBody": {
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
      "resultType": "complete",
      "tools": []
    }
  }
}
```

Simpan dua kelas fixture.

### Transkrip golden

Transkrip golden membuktikan perilaku yang diterima:

- discovery modern atau permintaan metode dengan metadata dan header yang cocok
- hasil lengkap dengan kolom wajib
- hasil `input_required` ketika metode dapat meminta masukan tambahan
- hasil ekstensi hanya setelah kapabilitas terkait diiklankan
- hasil lawan tanpa `resultType`, tetapi hanya dalam era lawan yang dipilih
- pemrosesan notifikasi tanpa respons JSON-RPC

Transkrip golden bersifat presisi, bukan besar. Jaga id dan cap waktu yang berubah-ubah tetap deterministik atau normalisasi sebelum perbandingan.

### Transkrip negatif

Transkrip negatif membuktikan perilaku penolakan:

- ketidaksesuaian header dan body
- kapabilitas per permintaan yang hilang
- versi protokol yang cocok tetapi tidak didukung
- `resultType` modern yang hilang
- `resultType` yang tidak dikenal atau tidak diiklankan
- `jsonrpc` respons selain `2.0` atau id yang berbeda nilai maupun jenis JSON-nya
- respons yang memuat `result` dan `error` sekaligus, atau keduanya tidak ada
- galat tanpa `code` berupa bilangan bulat dan `message` berupa string
- galat protokol yang dikenal dipetakan ke status HTTP yang salah
- respons yang dikeluarkan untuk sebuah notifikasi
- amplop JSON-RPC yang malformed
- proksi yang meruntuhkan galat protokol

Untuk setiap kasus negatif, tegaskan batas penolakan dan kode galat yang stabil. "Panggilan gagal" terlalu lemah. 500 yang dihasilkan proksi dan `-32020` dari origin dapat sama-sama tampak sebagai kegagalan, namun memberi cerita yang sangat berbeda kepada operator.

Fixture ketidaksesuaian header harus memuat respons JSON-RPC HTTP 400 asli dari server dengan id permintaan yang cocok dan kode galat `-32020`. Terapkan hal ini secara otomatis setiap kali validator lokal melihat `HeaderMismatch`; jangan jadikan verifikasi respons sebagai flag fixture opsional. Kasus dengan HTTP 500 dan tanpa body gagal meskipun kode penolakan lokal sudah benar. Harness yang berhenti setelah validator permintaan sendiri melempar galat hanya menguji dirinya sendiri, bukan perilaku kabel server.

Proyek kepatuhan MCP resmi berguna sebagai rangkaian eksternal dan referensi berversi. Simpan juga transkrip lokal Anda. Transkrip itu menangkap proksi, SDK, autentikasi, ekstensi, dan jalur rilis Anda, yang tidak dapat diketahui oleh rangkaian umum.

## Nilai Header Harus Cocok dengan Body RPC

Pada Streamable HTTP modern, perantara dapat melakukan perutean atau menegakkan kebijakan menggunakan header cermin. Body JSON-RPC tetap menjadi sumber kebenaran protokol. Ketidaksesuaian adalah kegagalan integritas, bukan petunjuk untuk memilih salah satu nilai.

Validasi dalam urutan ini:

1. Parse dan validasi amplop JSON-RPC serta jenis metadata.
2. Bandingkan `MCP-Protocol-Version` dengan `params._meta.io.modelcontextprotocol/protocolVersion`.
3. Bandingkan `Mcp-Method` dengan `method`.
4. Ketika metode memiliki nama perutean, bandingkan `Mcp-Name` dengan nilai body yang bersesuaian.
5. Setelah kesetaraan terbukti, putuskan apakah versi dan kumpulan kapabilitas yang cocok didukung.

Urutan ini membedakan ketidaksesuaian `-32020` dari versi yang tidak didukung `-32022`. Urutan ini juga mencegah gerbang mengotorisasi nama header sementara origin mengeksekusi nama body yang berbeda.

Nama kolom HTTP tidak peka huruf besar-kecil, sementara nilainya tetap peka huruf besar-kecil. Normalisasi nama header sebelum pencarian dan tolak duplikat yang bertentangan. Untuk `Mcp-Name` yang tidak aman, non-ASCII, atau memiliki spasi di awal maupun di akhir, dekode penanda (sentinel) UTF-8 persis `=?base64?{Base64EncodedValue}?=` sebelum membandingkannya dengan body. Tolak sentinel yang tidak lengkap, Base64 yang tidak valid, UTF-8 yang tidak valid, atau nilai mentah yang tidak aman dengan `-32020`. Spasi mentah di sekitar nilai tidak valid meskipun body memuat karakter yang sama, karena nilai tersebut wajib disandikan dengan sentinel sebelum transpor.

Perantara dapat menolak HTTP yang malformed sebelum permintaan mencapai server MCP, sehingga kegagalannya dapat berupa galat HTTP tanpa JSON-RPC. Catat apakah penolakan berasal dari perantara atau origin. Server MCP origin sebaiknya menggunakan kontrak galat protokol ketika ia menangani permintaan JSON-RPC yang valid.

## Kolom Asing Bukan Hasil Asing

Kompatibilitas maju mensyaratkan dua aturan yang berbeda.

### Kolom asing aditif

Objek hasil dan peta `_meta` dapat memperoleh kolom baru. Validator sebaiknya mempertahankan atau mengabaikan kolom aditif sesuai perannya, kecuali kolom itu melanggar kontrak yang dicadangkan. Contoh menyimpan hasil mentah lengkap dalam bukti dan menerima `futureHint` di samping hasil yang dikenal.

Jika Anda adalah proksi transparan, mempertahankan kolom asing biasanya lebih aman daripada membuangnya. Jika Anda adalah klien aplikasi, mengabaikannya dapat sah. Uji diferensial Anda tetap harus mengungkap bahwa SDK menghilangkannya agar perilaku itu disengaja.

### `resultType` yang tidak dikenal

`resultType` adalah diskriminator. Hasil inti modern menggunakan `complete` atau `input_required`. Ekstensi dapat menambahkan nilai lain hanya ketika kapabilitasnya diiklankan. Ekstensi Tasks, misalnya, dapat menambahkan `task` dalam konteks kapabilitas yang dinegosiasi itu.

Diskriminator yang tidak dikenal atau tidak diiklankan tidak dapat dengan aman dianggap lengkap. Klien tidak mengetahui siklus hidup yang akan dibuangnya. Tolaklah.

Dengan demikian, respons mentah yang sama dapat memuat kolom asing yang dapat diterima dan jenis hasil asing yang tidak dapat diterima. Ujilah kedua kasus itu.

Diskriminator hanya lapisan pertama. Validasi payload spesifik metode setelahnya. Hasil `tools/list` yang lengkap memerlukan array `tools` yang deskriptornya memiliki nama unik dan tidak kosong, deskripsi yang bermakna, serta nilai `inputSchema` berakar objek. Hasil `task` hanya valid untuk `tools/call` yang memenuhi syarat dengan kapabilitas Tasks dan mensyaratkan `taskId`, status yang dikenal, cap waktu pembuatan dan pembaruan, serta `ttlMs`, ditambah interval polling opsional yang valid. Hasil `completion/complete` yang lengkap memerlukan objek `completion` dengan tidak lebih dari 100 nilai string, `total` berupa bilangan bulat non-negatif opsional yang tidak lebih kecil daripada nilai yang dikembalikan, serta `hasMore` berupa Boolean opsional. `resultType` yang dieja dengan benar tidak dapat membuat payload yang malformed menjadi patuh.

## Invarian Notifikasi

Notifikasi JSON-RPC tidak memiliki `id`. Penerima tidak boleh mengirim respons sukses maupun galat JSON-RPC.

Untuk bentuk notifikasi HTTP yang diterima, harness mengharapkan HTTP `202` dengan body kosong. MCP `2026-07-28` tidak mendefinisikan notifikasi inti dari klien ke server melalui Streamable HTTP. Contoh menggunakan notifikasi ekstensi berkonsep bernamespace hanya untuk menguji invarian serializer searah. Jangan menyajikannya sebagai metode inti baru.

Ujilah serializer, bukan hanya handler. Handler dapat mengembalikan `None` sementara middleware membungkusnya dalam objek sukses JSON. Tangkap byte egress akhir.

## Tambahkan Diferensial SDK

SDK sering mengubah objek kabel menjadi tipe bahasa yang nyaman. Itu berguna, tetapi objek yang dinormalisasi tidak dapat membuktikan apa yang diterima.

Untuk setiap fixture berisiko tinggi, tangkap:

1. Status, header, dan body respons mentah sebelum dekode SDK.
2. Nilai kembalian atau galat yang dinormalisasi SDK.
3. Proyeksi semantik yang diharapkan untuk era yang dipilih.
4. Kolom yang diangkat, disintesiskan, dibuang, atau diubah oleh SDK.

Contoh mengizinkan penghapusan oleh SDK saja terhadap pembukuan kabel yang dikenal seperti `resultType`, `_meta`, `ttlMs`, dan `cacheScope` saat membandingkan payload aplikasi. Contoh melaporkan `futureHint` yang hilang karena kolom semantik asing itu menghilang.

Jangan menganggap setiap perbedaan sebagai bug SDK. Intinya adalah membuat transformasi itu terlihat. Putuskan apakah komponen Anda adalah endpoint aplikasi, yang boleh mengabaikan kolom aditif, atau perantara transparan, yang sebaiknya mempertahankannya.

Jalankan diferensial terhadap setiap SDK dan versi yang Anda sertakan. Jika dua SDK menormalisasi transkrip yang sama secara berbeda, kebijakan rilis sebaiknya menyatakan perilaku mana yang dapat diterima, daripada memilih keluaran yang paling nyaman setelahnya.

## Tangkap Bukti Proksi

Kebanyakan kegagalan MCP produksi terjadi melintasi lebih dari satu proses. Catat tiga tampilan:

| Tampilan | Bukti minimum |
|---|---|
| Ingress | header permintaan, body JSON-RPC, jenis konten, rute terautentikasi, waktu penerimaan |
| Origin | header yang diteruskan dan digest body, status origin, header dan body respons |
| Egress | status, header, dan body yang terlihat klien, serta waktu pengiriman |

Contoh mendeteksi dua transformasi umum:

- galat JSON-RPC HTTP 400 atau 404 dari origin berubah menjadi 500 proksi umum
- body JSON-RPC egress berbeda dari body origin

Tambahkan pernyataan spesifik deployment untuk jenis konten, `Accept`, kompresi, SSE per permintaan, header cache, dan korelasi jejak. Tangkap kedua sisi penghentian TLS ketika kebijakan mengizinkan. Jangan pernah mencatat kredensial hanya untuk membuktikan jalur.

## Redact Sebelum Bukti Keluar dari Memori

Redaksi adalah bagian dari operasional kepatuhan, bukan pekerjaan pembersihan belakangan. Terapkan sebelum serialisasi, hashing, log, artefak uji, atau unggahan kegagalan.

Contoh menyamakan huruf pada nama kunci dan menghapus pemisah sebelum mencocokkan, lalu secara rekursif mengganti nilai di bawah kunci seperti `Authorization`, `Cookie`, `Set-Cookie`, `X-Api-Key`, `accessToken`, `clientSecret`, `registrationAccessToken`, `token`, `password`, `secret`, dan `api_key`. Kanonisasi dan daftar hitam harus menggunakan bentuk yang sama agar varian camelCase, bertanda hubung, bergaris bawah, dan bertitik tidak dapat saling melewati kebijakan. Kolektor produksi sebaiknya menambahkan kebijakan argumen spesifik metode, karena kunci yang tampak tidak berbahaya seperti `query` tetap dapat memuat data pribadi atau data yang diatur.

Lakukan hash pada paket bukti yang telah diredact. Simpan tangkapan mentah hanya di sistem berumur pendek yang disetujui ketika penyelidikan tertentu memerlukannya. Digest membuktikan paket bukti mana yang mendorong keputusan; digest itu tidak mengungkap nilai yang dihapus.

## Jadikan Kesehatan dan Pemutaran Balik Bagian dari Gerbang

Kepatuhan protokol bersifat perlu tetapi tidak cukup untuk rilis. Kandidat yang patuh tetap dapat mengalami waktu habis, kebocoran memori, atau membebani sebuah dependensi.

Tentukan jendela kesehatan sebelum rollout:

- jumlah sampel minimum
- laju galat maksimum
- persentil latensi maksimum
- saturasi atau batas sumber daya
- durasi observasi
- perbandingan dengan baseline yang diterima

Tentukan juga bukti pemutaran balik sebelum rollout:

- versi sebelumnya yang persis
- digest bukti penerimaan
- pin artefak dan deskriptor SHA-256
- status Registry saat ini
- hasil kesehatan saat ini
- prosedur pemulihan rute
- attestation atas kolom-kolom persis tersebut dari identitas pengontrol rilis tepercaya

Wajibkan target pemutaran balik itu terverifikasi dan sehat sebelum promosi, bukan hanya setelah kandidat gagal. Rilis yang berhasil tanpa jalur pemulihan yang dapat digunakan belum siap produksi.

Jika kandidat gagal dan target pemutaran balik tidak memiliki bukti itu, tahan lalu lintas daripada menebak. "Putar balik ke apa pun yang ada sebelumnya" bukanlah kontrol operasional.

Jangan mereduksi kesiapan menjadi pemeriksaan truthiness seperti versi yang tidak kosong, `healthy: "yes"`, atau string bukti sembarangan. Contoh mensyaratkan jenis yang persis, status `active`, tiga digest SHA-256, penandatangan tepercaya, serta attestation HMAC-SHA-256 yang valid atas seluruh payload pemutaran balik. Kunci demo deterministiknya adalah fixture yang bukan rahasia. Suntikkan kunci terlindungi, hasil verifikasi KMS, atau verifikator attestation kunci publik pada batas rilis di produksi.

Gerbang rilis juga menolak transkrip, diferensial SDK, atau bukti proksi yang kosong. Setiap sumber harus membawa digest bukti yang valid. Jendela kesehatan yang hijau tidak dapat mengisi batas yang tidak pernah diamati.

## Bangun

Jalankan harness pustaka standar:

```bash
cd phases/13-tools-and-protocols/31-mcp-conformance-versioning-and-operations
python3 code/main.py
npx tsx code/main.ts
```

Demo menjalankan tepat lima belas transkrip golden dan negatif, termasuk hasil completion yang valid dan malformed, membandingkan hasil mentah dengan tampilan SDK, memeriksa proksi yang meruntuhkan galat origin, mengevaluasi kesehatan, mengautentikasi bukti pemutaran balik, dan memilih target tersebut.

Bentuk yang diharapkan:

```json
{
  "transcriptsPassed": 15,
  "transcriptsTotal": 15,
  "sdkDroppedFields": ["futureHint"],
  "proxyIssues": [
    "proxy collapsed a protocol error into HTTP 500",
    "proxy changed the origin JSON-RPC body"
  ],
  "releaseAction": "rollback",
  "evidenceDigest": "..."
}
```

Baca `code/main.py` dalam urutan ini:

1. `validate_request()` memberlakukan aturan permintaan dan header spesifik era.
2. `validate_result()` memisahkan diskriminator lawan yang hilang, nilai modern yang valid, ekstensi, dan nilai yang tidak dikenal.
3. `select_era()` mengimplementasikan kebijakan ketat dan fallback terbatas.
4. `run_transcript()` mengevaluasi fixture golden dan negatif.
5. `compare_sdk_view()` mengungkap perbedaan normalisasi.
6. `inspect_proxy()` membandingkan bukti ingress, origin, dan egress.
7. `redact()` menghapus rahasia yang jelas sebelum hashing bukti.
8. `rollback_evidence_ready()` memvalidasi kolom pin yang persis dan attestation rilis tepercaya.
9. `ReleaseGate.evaluate()` menggabungkan bukti kepatuhan, SDK, proksi, kesehatan, dan pemutaran balik yang tidak kosong.

## Gunakan

Jalankan harness pada empat titik:

1. Pada setiap perubahan implementasi dengan adapter uji dalam proses.
2. Terhadap biner klien dan server yang dibangun melalui transpor sungguhan.
3. Melalui proksi atau gerbang yang dideploy di lingkungan staging.
4. Selama rollout canary dengan bukti kesehatan dan pemutaran balik langsung.

Gunakan nama kasus stabil yang sama lintas lapisan. `negative-header-body-mismatch` sebaiknya berarti invarian yang sama dalam laporan unit, end-to-end, proksi, dan canary. Digest bukti akan berbeda karena batasnya berubah; persyaratannya tidak boleh.

Simpan skema fixture dalam kontrol versi. Simpan bukti menjalankan yang telah diredact dalam sistem rilis Anda. Simpan tangkapan mentah berumur pendek hanya di bawah kontrol akses insiden.

## Laboratorium Interaktif

### Lab A: membuktikan batas era

Dari direktori `code`, buka Python:

```bash
cd phases/13-tools-and-protocols/31-mcp-conformance-versioning-and-operations/code
python3 -q
```

Jalankan:

```python
from main import *
validate_result({"tools": []}, "legacy")
validate_result({"tools": []}, "modern")
```

Panggilan lawan menyimpulkan `complete`. Panggilan modern melempar `ProtocolViolation`. Sekarang uji fallback:

```python
select_era({"kind": "timeout"}, "fallback")
select_era(
    {"kind": "timeout"},
    "fallback",
    legacy_allowed=True,
    legacy_evidence={"kind": "initialize_success", "protocolVersion": LEGACY_VERSION},
)
select_era({"kind": "jsonrpc_error", "code": -32021}, "fallback")
```

Waktu habis pertama gagal dengan aman (fail closed) karena keheningan bukan bukti lawan. Panggilan kedua memilih lawan hanya karena konfigurasi mengizinkannya dan hasil inisialisasi lawan yang valid teramati. Galat kapabilitas-hilang yang dikenal membuktikan cabang modern.

### Lab B: kolom aditif versus diskriminator

```python
validate_result({"resultType": "complete", "tools": [], "futureHint": True}, "modern")
validate_result({"resultType": "future_mode", "tools": []}, "modern")
```

Hasil pertama mempertahankan `futureHint`. Hasil kedua ditolak karena diskriminator siklus hidupnya tidak dikenal.

### Lab C: memeriksa transformasi SDK

```python
compare_sdk_view(
    {"resultType": "complete", "tools": [], "futureHint": {"mode": "new"}},
    {"tools": []},
)
```

Putuskan apakah komponen Anda boleh mengabaikan `futureHint` atau wajib meneruskannya. Tulis pilihan itu ke dalam kebijakan rilis. Jangan menghapus diferensial secara diam-diam.

### Lab D: memperbaiki proksi

Ubah pertukaran demo agar egress mempertahankan status dan body origin. Jalankan `python3 main.py` lagi. Masalah proksi sebaiknya hilang, tetapi diferensial SDK tetap memblokir promosi. Lalu sertakan `futureHint` dalam tampilan SDK dan amati aksi berubah menjadi `promote` ketika setiap sumber bukti lulus.

## Laboratorium Latihan

Tambahkan transkrip SSE per permintaan ke harness.

Persyaratan:

- Tangkap status respons, jenis konten, peristiwa SSE berurutan, dan penghentian aliran.
- Buktikan setiap peristiwa JSON-RPC memiliki hasil atau galat yang valid sesuai era.
- Tambahkan kasus negatif untuk proksi yang menyangga seluruh aliran sebelum meneruskannya.
- Tambahkan kasus negatif untuk peristiwa SSE yang id JSON-RPC-nya berbeda dari permintaan.
- Redact data peristiwa sebelum menulis bukti.
- Sertakan durasi aliran, latensi peristiwa pertama, dan jumlah peristiwa dalam jendela kesehatan.
- Buat gerbang rilis hanya memilih target pemutaran balik yang dibuktikan ketika aliran gagal.

Berhasil berarti kasus yang sama berjalan langsung dan melalui proksi, dengan laporan yang mengidentifikasi batas persis yang mengubah perilaku.

## Artefak yang Disertakan

Pelajaran ini menyertakan `outputs/skill-mcp-conformance-release-gate.md`. Gunakan artefak itu untuk mengubah perubahan server, klien, gerbang, atau SDK menjadi matriks kepatuhan berversi dan keputusan rilis. Artefak ini mensyaratkan bukti kabel mentah, kasus negatif, pemilihan era eksplisit, diferensial SDK, bukti proksi, redaksi, ambang kesehatan, dan bukti pemutaran balik.

## Verifikasi

Jalankan demo dan rangkaian deterministik:

```bash
cd phases/13-tools-and-protocols/31-mcp-conformance-versioning-and-operations
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover -s code/tests -v
```

Verifikasi sebaiknya membuktikan:

- setiap transkrip golden dan negatif yang disertakan mencapai hasil yang diharapkan
- permintaan modern mensyaratkan kunci metadata bernamespace yang persis
- nama header HTTP dicocokkan tanpa memedulikan huruf besar-kecil dan nilai `Mcp-Name` yang disandikan didekode secara persis
- ketidaksesuaian header dan body mengembalikan kode ketidaksesuaian modern
- versi respons, id, keeksklusifan hasil atau galat, bentuk galat, dan pemetaan HTTP divalidasi
- persyaratan payload spesifik metode untuk tool-list, task, dan completion ditegakkan
- setiap `HeaderMismatch` yang teramati mensyaratkan respons JSON-RPC `-32020` HTTP 400 yang sungguhan
- spasi `Mcp-Name` mentah ditolak sementara spasi hasil penyandian sentinel yang persis berhasil melakukan round-trip
- `resultType` yang hilang hanya valid dalam era lawan yang dipilih
- kolom aditif bertahan dalam validasi mentah sementara jenis hasil yang tidak dikenal gagal
- jenis hasil ekstensi mensyaratkan kapabilitas yang diiklankan
- galat modern yang dikenal tidak pernah menyebabkan fallback lawan
- notifikasi tidak menghasilkan respons JSON-RPC
- penghapusan pembukuan SDK dan kehilangan kolom semantik dibedakan
- peruntuhan galat proksi terdeteksi dan kredensial diredact secara rekursif lintas varian camelCase dan pemisah
- promosi mensyaratkan bukti transkrip, SDK, proksi, dan operasional yang sehat serta tidak kosong
- promosi maupun pemutaran balik sama-sama mensyaratkan target pemutaran balik yang terautentikasi, dipasangi pin, aktif, dan sehat

## Mode Kegagalan Produksi

| Kegagalan | Apa yang dilaporkan uji yang lemah | Apa yang harus dibuktikan harness |
|---|---|---|
| SDK menyintesiskan diskriminator yang hilang | "tools/list lulus" | Hasil modern mentah tidak memiliki `resultType` dan tidak valid |
| Klien menurunkan versi setelah `-32021` | "percobaan ulang lawan berhasil" | Galat modern yang dikenal melarang fallback |
| Jenis hasil yang tidak dikenal dianggap lengkap | "respons ter-parse" | Diskriminator siklus hidup yang tidak diiklankan ditolak |
| Proksi mengotorisasi satu alat dan origin mengeksekusi alat lain | "permintaan mencapai server" | `Mcp-Name` sama dengan nama perutean body di setiap lompatan |
| Harness melempar galat sebelum membaca respons server | "uji ketidaksesuaian header lulus" | Respons HTTP 400 dan JSON-RPC `-32020` ditangkap dan divalidasi |
| Proksi mengubah 400 origin menjadi 500 umum | "galat hulu" | Status origin dan egress serta body JSON-RPC dipertahankan |
| Middleware notifikasi mengeluarkan `{result: null}` | "handler mengembalikan none" | Body egress akhir kosong dan tidak ada respons JSON-RPC |
| SDK membuang kolom aditif | "objek bertipe cocok" | Tampilan mentah dan ternormalisasi menunjukkan kolom yang hilang secara persis |
| Artefak kegagalan membocorkan token bearer | "paket debug diunggah" | Redaksi terjadi sebelum hashing, logging, atau unggahan |
| Gaya kunci kredensial melewati redaksi | "daftar hitam memuat api_key" | Varian camelCase dan pemisah menggunakan satu bentuk daftar hitam kanonik |
| Canary tidak memiliki sampel tetapi tampak sehat | "nol galat" | Jumlah sampel minimum ditegakkan |
| Pemutaran balik memilih build yang tidak dikenal | "deployment sebelumnya dipulihkan" | Versi target, digest penerimaan, pin, status, dan kesehatan hadir |

## Aturan Operasional

Ujilah byte yang Anda kirim, byte yang diteruskan setiap perantara, semantik yang diekspos tiap SDK, serta bukti yang akan digunakan operasional di bawah tekanan. Kompatibilitas adalah cabang eksplisit. Pemutaran balik adalah tindakan rilis yang didukung bukti. Keduanya tidak boleh menjadi efek samping tidak sengaja dari parser yang permisif.

## Bacaan Lanjutan

- [Protokol dasar MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic)
- [Negosiasi versi MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)
- [MCP Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [Proyek kepatuhan MCP resmi](https://github.com/modelcontextprotocol/conformance)
