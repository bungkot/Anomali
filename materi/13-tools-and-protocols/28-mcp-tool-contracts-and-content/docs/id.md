# Kontrak Alat dan Konten MCP

> Sebuah alat aman untuk diotomatisasi hanya ketika penemuan, argumen, hasil, paginasi, dan metadata transpor sepakat pada satu kontrak.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 07, 09, dan 10
**Waktu:** ~120 menit

## Tujuan Pembelajaran

- Tetapkan input dan keluaran alat dengan JSON Schema 2020-12.
- Validasi hasil terstruktur tanpa mengasumsikan bahwa hasil itu objek JSON.
- Pilih antara teks, gambar, audio, tautan sumber daya, dan sumber daya tersemat.
- Tolak definisi `x-mcp-header` yang tidak aman sebelum alat mencapai model.
- Enkodekan nilai header parameter dan verifikasi kesetaraan persis antara header dan isi.
- Telusuri paginasi cursor tanpa menafsirkan nilai cursor.
- Batasi dan otorisasi saran `completion/complete`.

## Masalahnya

Memanggil fungsi Python itu mudah. Memanggil kapabilitas jarak jauh melalui host AI adalah masalah kontrak.

Server menerbitkan deskriptor. Klien mengubah deskriptor itu menjadi konteks model dan antarmuka pengguna. Model membuat argumen. Gerbang mungkin merutekan permintaan berdasarkan header yang dicerminkan. Server menjalankan alat. Klien lalu memutuskan apakah hasil cukup aman dan valid untuk dikembalikan ke model.

Satu batas yang lemah merusak seluruh rantai.

Perhatikan lima kegagalan:

- Deskriptor menyatakan hasil berupa objek, tetapi server mengembalikan array.
- Klien menghentikan paginasi ketika `nextCursor` berupa string kosong.
- Parameter token dicerminkan ke header HTTP dan menjadi terlihat oleh perantara.
- Nilai routing Unicode dikirim sebagai header mentah, lalu gerbang dan asal menafsirkan byte yang berbeda.
- Endpoint completion menyarankan lingkungan produksi kepada pemanggil yang tidak dapat mengaksesnya.

Tidak ada kegagalan ini yang diperbaiki oleh prompting yang lebih baik. Kegagalan itu memerlukan kontrak protokol dan aplikasi yang eksplisit.

## Pipeline Kontrak

Perlakukan setiap pemanggilan alat sebagai lima gerbang:

1. **Temukan.** Baca daftar alat deterministik yang terpagasi.
2. **Terima.** Validasi setiap deskriptor dan terapkan kebijakan keamanan lokal.
3. **Panggil.** Validasi argumen dan susun metadata transpor.
4. **Jalankan.** Jalankan handler dan klasifikasikan kegagalan dengan tepat.
5. **Konsumsi.** Validasi blok konten dan keluaran terstruktur sebelum dipakai model.

```figure
mcp-contract-pipeline
```

Host memiliki gerbang penerimaan dan konsumsi. Server tidak dapat memaksa klien mempercayai annotation, skema, atau keluarannya.

## JSON Schema adalah Batas Runtime

Pada MCP `2026-07-28`, `inputSchema` dan `outputSchema` memakai JSON Schema. Ketika `$schema` tidak ada, dialek default adalah 2020-12.

Skema input harus berupa objek skema. Alat tanpa argumen tetap harus menyatakan persis apa yang diterimanya:

```json
{
  "type": "object",
  "additionalProperties": false
}
```

Ini lebih ketat daripada `{ "type": "object" }`, yang menerima properti sembarang.

Skema keluaran bersifat opsional. Begitu server menerbitkannya, setiap hasil alat lengkap berkomitmen mengembalikan `structuredContent` yang sesuai, termasuk hasil dengan `isError: true`. Flag error mengklasifikasikan hasil eksekusi; flag itu tidak membebaskan dari kontrak keluaran yang dipublikasikan. Klien sebaiknya memvalidasi hasil, bukan mempercayai deskriptor.

### Konten terstruktur adalah nilai JSON apa pun

Jangan mengkodekan `structuredContent` secara tetap sebagai dictionary. Nilainya dapat berupa:

- sebuah objek;
- sebuah array;
- sebuah string;
- sebuah angka;
- sebuah boolean;
- `null`.

Alat ini mengembalikan array:

```json
{
  "name": "tag_catalog",
  "inputSchema": {
    "type": "object",
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "array",
    "items": {"type": "string"}
  }
}
```

Hasil suksesnya valid:

```json
{
  "resultType": "complete",
  "content": [
    {
      "type": "text",
      "text": "[\"contracts\", \"mcp\", \"stateless\"]"
    }
  ],
  "structuredContent": ["contracts", "mcp", "stateless"],
  "isError": false
}
```

Untuk kompatibilitas, hasil terstruktur sebaiknya juga memuat JSON terserialisasi dalam blok teks. Teks itu bukan sumber validasi. `structuredContent`-lah sumbernya.

### Validator kecil tetap mengajarkan batas

Pelajaran ini memakai subset JSON Schema yang disengaja karena tetap berada di dalam pustaka standar Python. Validator itu memeriksa mekanisme yang dipakai alat contoh:

- tipe object, array, string, integer, number, boolean, dan null;
- properti wajib;
- `additionalProperties: false`;
- item array;
- nilai enum;
- panjang minimum string.

Ini bukan pengganti validator produksi lengkap. Pelajaran yang dapat dipakai ulang adalah tempat validasi terjadi: setelah penemuan untuk deskriptor, sebelum eksekusi untuk argumen, dan sebelum konsumsi untuk hasil terstruktur.

## Blok Konten Memiliki Biaya Berbeda

Array `content` dapat menggabungkan beberapa jenis konten.

| Jenis | Digunakan untuk | Batas utama |
|---|---|---|
| `text` | Ringkasan yang dapat dibaca manusia dan model | Perlakukan teks sebagai keluaran tidak tepercaya |
| `image` | Bukti visual yang dienodekan sebagai base64 | Validasi tipe media dan ukuran |
| `audio` | Keluaran suara atau rekaman yang dienodekan sebagai base64 | Validasi tipe media dan batas durasi |
| `resource_link` | URI yang dapat diambil klien kemudian | Otorisasi ulang pembacaan sumber daya berikutnya |
| `resource` | Data yang disematkan langsung dalam hasil | Tegakkan batas payload dan konten sekarang |

Tautan sumber daya bukan bukti bahwa sumber daya itu muncul di `resources/list`. Tautan itu adalah referensi yang dikembalikan oleh pemanggilan alat ini. Klien tetap menerapkan kebijakan sumber dayanya ketika mengikuti URI.

Sumber daya tersemat menghindari satu putaran lagi, tetapi menambah ukuran respons saat ini. Gunakan tautan untuk artefak besar atau yang berubah secara independen. Gunakan sumber daya tersemat untuk bukti kecil yang harus ikut secara atomik bersama hasil.

Hasil `evidence_bundle` pada pelajaran ini memuat kelima jenis itu. Klien memvalidasi setiap blok sebelum menerima hasil.

## `x-mcp-header` adalah Metadata Routing

Sebuah properti di dalam `inputSchema` dapat mendeklarasikan `x-mcp-header`. Melalui Streamable HTTP, klien mencerminkan argumen itu ke dalam `Mcp-Param-{name}`.

```json
{
  "region": {
    "type": "string",
    "x-mcp-header": "Region"
  }
}
```

Dengan `region: "eu-west"`, transpor dapat menghasilkan:

```http
Mcp-Param-Region: eu-west
```

Annotation ada agar penyeimbang beban, gerbang, atau mesin kebijakan dapat merutekan tanpa mengurai isi JSON. Annotation bukan tempat untuk meletakkan kredensial.

Protokol membatasi annotation itu:

- nama header tidak kosong dan mengikuti sintaks token nama kolom HTTP;
- nama header unik tanpa memedulikan huruf besar/kecil;
- tipe properti adalah string, integer, atau boolean;
- `number` tidak diizinkan;
- annotation hanya muncul pada anggota langsung `inputSchema.properties`;
- nilai integer tetap dalam rentang `-9007199254740991` sampai `9007199254740991`.

Aturan lokasi bersifat sintaksis dan fail-closed. Telusuri seluruh pohon skema, bukan hanya properti yang kebetulan dipahami validator Anda. Tolak annotation di bawah `properties` objek bersarang, cabang `oneOf`, `items`, definisi yang dicapai `$ref`, atau skema keluaran mana pun. Menyelesaikan referensi tidak mengubah simpul yang dirujuk menjadi properti langsung tingkat atas.

Pelajaran ini menambahkan kebijakan deployment: tolak deskriptor yang mencerminkan nama seperti `password`, `secret`, `token`, `api_key`, atau `authorization`. Spesifikasi resmi menyarankan penulis server agar tidak mencerminkan parameter sensitif. Klien dapat mengubah saran itu menjadi aturan penerimaan yang keras.

Audit nama header, bukan nilainya. Kode contoh mencatat `Mcp-Param-Region` sambil menjaga `eu-west` tetap keluar dari peristiwa audit.

### Enkodekan nilai sebelum menyusun header HTTP

Nilai parameter boleh dikirim sebagai teks polos hanya ketika nilai itu berupa string tidak kosong dari karakter ASCII tampak `!` sampai `~` dan tidak menyerupai penanda enkode. Semua lainnya memakai bentuk persis ini:

```text
=?base64?{Base64UTF8}?=
```

`Base64UTF8` adalah base64 standar atas byte UTF-8 yang persis. Jangan memangkas, menormalkan, atau mengganti nilai itu terlebih dahulu. Enkodekan Unicode, string kosong, spasi, tab, karakter kontrol, CR atau LF, spasi di awal atau akhir, serta nilai apa pun yang diawali `=?base64?`. Mengenkode ulang nilai yang tampak seperti penanda itulah yang memungkinkan penerima memulihkan teks asli secara literal, alih-alih mengurasinya sebagai sintaks transpor.

Boolean dirender sebagai `true` atau `false` huruf kecil. Integer dirender dalam basis 10 dan harus tetap berada di dalam rentang bilangan bulat aman JavaScript. Nilai di luar rentang itu ditolak, bukan dibulatkan oleh perantara.

### Server memeriksa salinan yang dicerminkan

Pembuatan header hanya bagian klien. Pada batas Streamable HTTP, server wajib:

1. menemukan nama `Mcp-Param-*` yang dikenali tanpa memedulikan huruf besar/kecil nama header;
2. mendekode bentuk penanda base64 persis ketika ada;
3. membandingkan teks hasil dekode dengan argumen isi JSON yang bersesuaian secara persis;
4. menolak header dikenali yang hilang, ganda, tak terduga, cacat, atau tidak cocok sebelum pendistribusian.

Penolakan adalah HTTP `400` dengan kode error JSON-RPC `-32020`. Baik nilai isi maupun bentuk header terenkodenya tidak boleh masuk ke catatan audit. Catat hanya nama header yang dikenali dan kategori penolakan.

`code/main.py` memodelkan batas ini secara langsung. [Pelajaran 09](../../09-mcp-transports/) membahas urutan validasi Streamable HTTP yang lebih luas, termasuk kesetaraan metode dan versi protokol.

## Cursor Paginasi Bersifat Buram

Operasi daftar MCP memakai paginasi cursor. Server menentukan ukuran halaman dan format cursor. Klien mendapat satu keputusan:

```python
if result.get("nextCursor") is None:
    break
cursor = result["nextCursor"]
```

Jangan menulis ini:

```python
if not result.get("nextCursor"):
    break
```

String kosong adalah cursor yang valid. Penilaian kebenaran (truthiness) akan berhenti terlalu awal.

Klien tidak boleh mendekode cursor, menaikkannya, membandingkannya dengan cursor sebelumnya untuk pengurutan, maupun menyimpulkan nomor halaman. Server dapat menandatangani cursor, mengikatnya ke versi katalog, atau memetakannya ke state privat. Itu adalah detail implementasi server.

Server contoh sengaja mengembalikan `""` setelah halaman pertama. Klien harus mengirim nilai persis itu pada permintaan kedua. Jejaknya adalah:

```text
<permintaan pertama tanpa cursor>
<permintaan kedua dengan cursor "">
```

Cursor tidak valid menghasilkan parameter tidak valid JSON-RPC, kode `-32602`.

## Completion adalah Permukaan Otorisasi

`completion/complete` memberi saran untuk argumen prompt dan argumen template sumber daya. Fitur ini berguna untuk formulir interaktif, tetapi dapat membocorkan nama yang dilindungi oleh metode daftar biasa.

Permintaan completion menyebut referensi dan argumen yang sedang dilengkapi:

```json
{
  "method": "completion/complete",
  "params": {
    "ref": {
      "type": "ref/prompt",
      "name": "deployment_review"
    },
    "argument": {
      "name": "environment",
      "value": "st"
    }
  }
}
```

Hasil mengembalikan paling banyak 100 nilai dan dapat melaporkan `total` serta `hasMore`.

Terapkan batas otorisasi yang sama seperti yang dipakai prompt atau sumber daya yang dirujuk. Seorang analis dalam contoh menerima `development` dan `staging`. Hanya operator yang dapat menerima `production`.

Completion produksi juga memerlukan:

- validasi input;
- pemfilteran sadar-pemanggil;
- debounce permintaan di klien;
- pembatasan laju di server;
- jumlah hasil yang terikat batas;
- log yang tidak menampilkan nilai saran sensitif.

Completion adalah bantuan, bukan jalan pintas penemuan.

## Dua Lapisan Error

Jaga agar error protokol terpisah dari error eksekusi alat.

Gunakan error JSON-RPC ketika permintaan MCP tidak dapat didistribusikan dengan benar:

- nama alat tidak dikenal;
- bentuk permintaan cacat;
- metadata permintaan hilang;
- cursor tidak valid.

Gunakan hasil alat lengkap dengan `isError: true` ketika pemanggilan mencapai alat dan alat melaporkan kegagalan yang dapat ditindaklanjuti:

- sumber laporan tidak tersedia;
- tanggal di luar rentang yang didukung;
- aturan bisnis menolak operasi yang diminta.

Model sering kali dapat memperbaiki error eksekusi alat. Model tidak dapat memperbaiki server yang melanggar skema keluarannya sendiri.

Jika alat mendeklarasikan skema keluaran, modelkan kegagalan yang dapat ditindaklanjuti di dalam skema itu. Kegagalan `route_report` pada contoh mengembalikan wilayah yang dimintanya dengan `accepted: false`, bersama teks error yang dapat dibaca manusia dan `isError: true`.

## Bangunlah

`code/main.py` membangun kedua sisi batas dengan pustaka standar Python.

Server mengimplementasikan:

- validasi metadata MCP per permintaan;
- `server/discover` dengan kapabilitas tools dan completions;
- paginasi `tools/list` deterministik;
- empat deskriptor alat, termasuk satu yang harus ditolak;
- keluaran terstruktur berupa array;
- setiap jenis blok konten alat terkini;
- gerbang kesetaraan Streamable HTTP yang mendekode header parameter dikenali dan mengembalikan HTTP `400` plus JSON-RPC `-32020` saat terjadi ketidaksesuaian;
- completion yang terautorisasi dan terbatasi lajunya.

Klien mengimplementasikan:

- penerimaan deskriptor;
- validasi penempatan `x-mcp-header` pada seluruh pohon dan kebijakan kolom sensitif;
- enkode nilai ASCII tampak-polos persis atau base64 UTF-8;
- loop cursor buram yang mengikuti string kosong;
- validasi argumen dan hasil;
- validasi blok konten;
- peristiwa audit header yang memuat nama, bukan nilai.

Deskriptor yang sengaja tidak aman adalah data pengajaran. Deskriptor itu membuktikan bahwa satu alat yang ditolak tidak mencegah alat valid untuk dimuat.

## Gunakanlah

Dari akar repositori:

```bash
cd phases/13-tools-and-protocols/28-mcp-tool-contracts-and-content/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Demo mencetak alat yang diterima, deskriptor yang ditolak, kedua permintaan paginasi, konten array terstruktur, jenis blok konten, nama header yang dicerminkan, apakah nilai memerlukan enkode, status kesetaraan HTTP, serta nilai completion yang tersaring berdasarkan pemanggil.

## Laboratorium Interaktif

Buka `code/main.py` dan temukan `TOOLS`.

1. Ubah `tag_catalog.outputSchema.type` dari `array` menjadi `object`.
2. Jalankan demo. Klien seharusnya menolak array yang dikembalikan.
3. Pulihkan skema.
4. Pertahankan `nextCursor` halaman pertama sebagai `""`, lalu buat halaman akhir mengembalikan `nextCursor: None`, alih-alih menghilangkan kolom itu.
5. Jalankan uji dan bandingkan jejak cursor.
6. Tambahkan `x-mcp-header: "Authorization"` ke sebuah properti string.
7. Pastikan penerimaan deskriptor menolaknya sebelum pemanggilan.
8. Coba nilai `region` yang memuat Unicode, baris baru, spasi di sekitarnya, dan teks literal `=?base64?SGVsbG8=?=`. Dekode setiap header yang dihasilkan dan buktikan nilai asli tetap utuh persis.
9. Pindahkan annotation ke bawah `oneOf`, `items`, atau definisi `$ref`. Pastikan setiap deskriptor ditolak meskipun cabang itu tidak pernah dipakai demo.
10. Hapus header yang dikenali atau ubah nilai hasil dekodenya. Pastikan batas HTTP mengembalikan status `400` dan kode JSON-RPC `-32020`.

Intinya bukan menghafal bentuk JSON. Intinya adalah menyaksikan setiap gerbang gagal pada batas yang memilikinya.

## Laboratorium Praktik

Perluas laboratorium kontrak dengan alat `search_evidence`.

Persyaratan:

1. Skema inputnya menerima `query`, `limit`, dan kolom routing `region` yang aman.
2. Skema keluarannya adalah array objek dengan `uri`, `title`, dan `score`.
3. Hasil memuat teks kompatibilitas dan satu tautan sumber daya per butir.
4. Argumen menolak properti yang tidak dikenal.
5. `limit` dibatasi oleh validasi aplikasi.
6. Pemanggil tanpa akses ke suatu URI tidak pernah melihat URI itu melalui completion maupun keluaran alat.
7. Uji mencakup skor yang tidak sesuai, annotation header yang tidak valid, dan daftar dua halaman.
8. Uji nilai header mencakup ASCII tampak, Unicode, karakter kontrol, spasi, teks tampak seperti penanda, serta kedua batas bilangan bulat aman JavaScript.
9. Fixture HTTP menerima nama header tanpa memedulikan huruf besar/kecil, tetapi menolak nilai dikenali yang hilang atau tidak cocok dengan status `400` dan kode `-32020`.

## Artefak yang Dikirim

`outputs/skill-mcp-contract-reviewer.md` adalah skill tinjauan datar yang dapat dipakai ulang. Berikan deskriptor alat, contoh hasil, perilaku paginasi, dan kebijakan completion. Skill itu mengembalikan keputusan penerimaan, rencana validasi hasil, kebijakan header, dan uji kegagalan konkret.

## Verifikasikan

Pelajaran selesai ketika pernyataan-pernyataan ini benar:

- `tools/list` mengembalikan urutan logis yang sama pada pemanggilan berulang.
- Klien melakukan permintaan kedua ketika `nextCursor` adalah `""`.
- Deskriptor header sensitif yang tidak aman dikecualikan, sementara alat lain tetap tersedia.
- Sebuah array lolos skema keluaran array-nya.
- Sebuah objek gagal pada skema array yang sama.
- Hasil error tidak boleh menghilangkan atau melanggar skema keluaran yang dipublikasikan.
- Blok teks, gambar, audio, tautan sumber daya, dan sumber daya tersemat tervalidasi.
- Peristiwa audit header memuat nama dan tidak memuat nilai.
- ASCII tampak polos tetap polos; nilai Unicode, kontrol, ber-padding, kosong, dan tampak seperti penanda melakukan pergi-balik melalui enkode base64 UTF-8 yang persis.
- Integer yang dicerminkan di luar rentang aman JavaScript ditolak.
- Annotation di bawah `oneOf`, `items`, objek bersarang, definisi `$ref`, atau skema keluaran ditolak saat penerimaan.
- Nama header dikenali tanpa memedulikan huruf besar/kecil lolos hanya ketika nilai hasil dekode persis cocok dengan isi; salinan yang hilang atau tidak cocok menghasilkan HTTP `400` dan JSON-RPC `-32020`.
- Completion analis tidak pernah mengembalikan `production`.
- Kegagalan alat memakai `isError: true`; panggilan protokol cacat memakai `error` JSON-RPC.

## Mode Kegagalan Produksi

| Kegagalan | Yang dilihat pembelajar | Respons yang benar |
|---|---|---|
| Klien mengasumsikan keluaran objek | Array valid gagal atau dibungkus secara senyap | Validasi terhadap skema yang dipublikasikan tanpa tipe hanya-objek |
| Cursor kosong diperlakukan sebagai false | Halaman akhir hilang | Lanjutkan selama `nextCursor` ada dan tidak null |
| Nilai sensitif dicerminkan | Rahasia muncul di proxy, WAF, atau data jejak | Tolak deskriptor dan simpan rahasia dalam data permintaan yang dilindungi |
| Unicode atau spasi mentah dicerminkan | Gerbang dan asal tidak sepakat atau nilai dinormalkan | Gunakan enkode penanda base64 UTF-8 persis dan bandingkan setelah dekode |
| Annotation tersembunyi dalam cabang skema | Klien melewatkan metadata routing saat penerimaan | Telusuri seluruh pohon skema dan izinkan hanya properti langsung tingkat atas |
| Integer besar dicerminkan | Perantara JavaScript membulatkan nilai routing | Tolak nilai di luar rentang bilangan bulat aman JavaScript |
| Header dan isi tidak sepakat | Gerbang merutekan satu target sementara asal menjalankan target lain | Tolak sebelum pendistribusian dengan HTTP `400` dan JSON-RPC `-32020` |
| Skema keluaran diabaikan | Kode hilir mengonsumsi struktur rusak | Validasi sebelum dipakai model atau aplikasi |
| Tautan sumber daya dipercaya otomatis | Pemanggil mengikuti URI tanpa izin | Otorisasi ulang setiap pembacaan sumber daya |
| Completion membagi saran global | Nama tenant tersembunyi bocor | Saring berdasarkan pemanggil, referensi, dan otorisasi |
| Annotation alat diperlakukan sebagai kebijakan | Operasi destruktif melewati konfirmasi | Tegakkan otorisasi dan persetujuan di luar annotation |
| Satu alat cacat merusak penemuan | Seluruh server menjadi tidak tersedia | Tolak deskriptor buruk dan terima alat valid secara independen |

## Kaitan Proyek Akhir

Proyek akhir Fase 13 memerlukan gerbang yang dapat menggabungkan alat dari beberapa server. Pelajaran ini menyediakan inti penerimaannya.

Gunakan artefak itu untuk menilai empat bagian bukti proyek akhir:

- penemuan terpagasi yang deterministik dan lengkap;
- validasi deskriptor sebelum paparan ke model;
- keluaran terstruktur tervalidasi plus blok konten yang terikat batas;
- completion dan metadata routing yang menjaga batas otorisasi.

Jangan menyatakan kompatibilitas gerbang hanya dari satu `tools/call` yang berhasil. Tangkap deskriptor, jejak halaman, himpunan alat yang diterima, himpunan alat yang ditolak, dan satu hasil tervalidasi.

## Istilah Kunci

| Istilah | Arti |
|---|---|
| `inputSchema` | Objek JSON Schema yang mendefinisikan argumen alat yang diterima |
| `outputSchema` | JSON Schema opsional yang mendefinisikan `structuredContent` |
| `structuredContent` | Nilai JSON apa pun yang dihasilkan oleh hasil alat |
| Blok konten | Teks, gambar, audio, tautan sumber daya, atau sumber daya tersemat yang bertipe |
| `x-mcp-header` | Annotation skema yang mencerminkan argumen primitif ke metadata Streamable HTTP |
| Cursor buram | Token paginasi yang diterbitkan server dan nilainya tidak ditafsirkan klien |
| Referensi completion | Nama prompt atau URI/template sumber daya yang argumennya sedang dilengkapi |
| Penerimaan | Keputusan klien untuk mengekspos atau menolak deskriptor yang ditemukan |

## Bacaan Lanjutan

- [Alat MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
- [Completion MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/completion)
- [Paginasi MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/pagination)
- [Header Parameter Streamable HTTP MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#custom-headers-from-tool-parameters)
