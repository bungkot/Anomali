# MCP Apps pada Protokol Tanpa Keadaan

> Hasil interaktif tetap merupakan pertukaran alat dan sumber daya MCP. Inti 2026-07-28 membuat pertukaran itu mandiri, sementara ekstensi Apps menambahkan permukaan peramban yang di-sandbox.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 07 (server MCP), Fase 13 · 10 (sumber daya)
**Waktu:** ~75 menit

## Tujuan Pembelajaran

- Iklankan MCP Apps melalui `server/discover` dan kapabilitas ekstensi per permintaan.
- Deklarasikan sumber daya `ui://` pada sebuah alat sebelum alat itu dipanggil.
- Kembalikan hasil alat dan sumber daya yang lengkap pada kawat tanpa keadaan 2026-07-28.
- Pisahkan pesan jembatan `ui/initialize` Apps dari jabat tangan inti MCP yang telah dihapus.
- Terapkan validasi origin, sandboxing, CSP, dan izin paling minimal.

## Masalahnya

Hasil teks dapat mendeskripsikan linimasa. Hasil teks tidak dapat memberi pengguna linimasa yang dapat difilter, diperiksa, atau ditindaklanjuti.

MCP Apps menyelesaikan masalah penyajian dengan ekstensi opsional. Definisi alat menunjuk ke sumber daya `ui://`. Host dapat mengambil dan meninjau sumber daya itu sebelum alat dijalankan, merendernya dalam iframe yang di-sandbox, dan memediasi semua aksi aplikasi melalui jembatan JSON-RPC.

Protokol inti berubah pada 2026-07-28. Jangan membungkus App ke dalam siklus koneksi lama:

- Tidak ada permintaan inti `initialize` maupun notifikasi `notifications/initialized`.
- Tidak ada header `Mcp-Session-Id`.
- Setiap permintaan membawa versi protokol dan kapabilitas klien di `params._meta`.
- Server mengimplementasikan `server/discover` agar klien dapat memeriksa versi, kapabilitas inti, dan ekstensi.
- Setiap hasil sukses memiliki diskriminator `resultType`.
- Streamable HTTP menggunakan satu POST per permintaan. Entrypoint GET dan DELETE modern mengembalikan 405.

Jembatan Apps tetap memiliki metode bernama `ui/initialize`. Metode itu termasuk dialek postMessage iframe. Metode itu tidak menciptakan kembali sesi inti MCP.

## Konsepnya

### Dua protokol, satu fitur

Jaga agar lapisan-lapisan tetap eksplisit:

1. Inti MCP membawa `server/discover`, `tools/list`, `tools/call`, `resources/list`, dan `resources/read`.
2. Ekstensi MCP Apps mendeklarasikan UI dan mendefinisikan jembatan iframe-ke-host.
3. Aturan sandbox peramban membatasi apa yang dapat dijangkau UI.

Identifikator ekstensi adalah `io.modelcontextprotocol/ui`. Kedua pihak harus memilih ikut serta. Klien mengirim dukungan ekstensi di dalam objek kapabilitas pada setiap permintaan:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "server/discover",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "extensions": {
          "io.modelcontextprotocol/ui": {}
        }
      },
      "io.modelcontextprotocol/clientInfo": {
        "name": "timeline-host",
        "version": "1.0.0"
      }
    }
  }
}
```

`clientInfo` disarankan untuk diagnostik. Datanya dilaporkan sendiri, bukan identitas otorisasi.

### Temukan sebelum merender

Hasil penemuan server mengiklankan ekstensi:

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": {
    "tools": {},
    "resources": {},
    "extensions": {
      "io.modelcontextprotocol/ui": {}
    }
  },
  "ttlMs": 300000,
  "cacheScope": "public",
  "_meta": {
    "io.modelcontextprotocol/serverInfo": {
      "name": "timeline-app-server",
      "version": "2.0.0"
    }
  }
}
```

Server wajib mendukung penemuan. Klien tidak diwajibkan memanggil penemuan sebelum setiap aksi, karena setiap aksi membawa kapabilitasnya sendiri.

### Deklarasikan UI pada definisi alat

Kontrak Apps modern mengikat UI ke alat di `tools/list`:

```json
{
  "name": "notes_timeline",
  "description": "Render a timeline of notes.",
  "inputSchema": {
    "type": "object",
    "properties": {}
  },
  "_meta": {
    "ui": {
      "resourceUri": "ui://notes/timeline.html"
    }
  }
}
```

Ini sengaja menjadi metadata sebelum panggilan. Host dapat memuat awal, menembolok, dan meninjau keamanan HTML sebelum hasil meminta agar HTML ditampilkan. Kunci metadata datar lawas boleh diterima oleh kode kompatibilitas, tetapi server baru sebaiknya memancarkan bentuk `_meta.ui.resourceUri` bersarang.

`tools/list` dapat ditembolok dalam inti terkini. Sertakan urutan deterministik, `ttlMs`, dan `cacheScope`. Gunakan `private` ketika alat yang terlihat berbeda menurut pengguna atau token.

### Kembalikan data, lalu biarkan host mengikat tampilan

Panggilan alat mengembalikan konten biasa plus data terstruktur:

```json
{
  "resultType": "complete",
  "content": [
    {"type": "text", "text": "Timeline ready."}
  ],
  "structuredContent": {
    "notes": [
      {"id": "note-1", "title": "Discover", "created": "2026-07-28"}
    ]
  },
  "isError": false
}
```

Host sudah mengetahui tampilan mana yang menjadi milik alat itu. Hindari membuat blok konten baru hanya untuk mengulang URI.

### Sajikan aplikasi sebagai sumber daya

Server mengiklankan `resources` dalam penemuan, sehingga server juga mengimplementasikan operasi `resources/list` yang wajib. Entri daftar deterministiknya memuat URI kanonik, nama yang stabil, deskripsi, dan jenis MIME. Hasil daftar memuat `resultType`, metadata identitas server, `ttlMs`, dan `cacheScope`, seperti daftar alat deterministik.

Host mengirim `resources/read`. Pada Streamable HTTP, permintaannya adalah:

```text
POST /mcp
MCP-Protocol-Version: 2026-07-28
Mcp-Method: resources/read
Mcp-Name: ui://notes/timeline.html
```

Nilai header dan badan JSON-RPC harus cocok. Ketidaksesuaian adalah galat protokol `-32020`.

Hasilnya memuat sumber daya HTML dan petunjuk tembolok:

```json
{
  "resultType": "complete",
  "contents": [
    {
      "uri": "ui://notes/timeline.html",
      "mimeType": "text/html;profile=mcp-app",
      "text": "<!doctype html>...",
      "_meta": {
        "ui": {
          "csp": {
            "connectDomains": [],
            "resourceDomains": [],
            "frameDomains": [],
            "baseUriDomains": []
          },
          "permissions": {}
        }
      }
    }
  ],
  "ttlMs": 60000,
  "cacheScope": "public"
}
```

### Tembolokan sumber daya UI sebagai konten yang dapat dieksekusi

Sumber daya App tidak dapat ditukarkan dengan prosa biasa. Entri temboloknya dapat mengeksekusi kode jembatan, merender data alat, dan meminta aksi yang dimediasi host. Kuncinya adalah URI `ui://` kanonik, identitas dan versi server yang diterima, intisari konten sumber daya, serta konteks otorisasi ketika `cacheScope` bersifat private. Jangan pernah menggunakan kembali sumber daya App privat lintas prinsipal, karena HTML atau metadata kebijakannya dapat berbeda meskipun URI-nya identik.

Batalkan entri ketika `ttlMs`-nya kedaluwarsa, binding `_meta.ui.resourceUri` alat berubah, versi server atau pin deskriptor yang diterima berubah, atau langganan perubahan sumber daya yang diakui menyebut URI itu. Ambil ulang dan terapkan kembali peninjauan CSP serta izin sebelum memasang ulang. iframe lama tidak boleh mempertahankan izin yang lebih luas hanya karena versi sumber daya baru belum dimuat.

### Tolak ambiguitas kawat sebelum kebijakan fitur

Validasi memiliki urutan yang disengaja. Pertama, validasi bentuk JSON-RPC dan wajibkan metadata protokol berupa string serta peta kapabilitas klien berupa objek. Berikutnya, bandingkan header routing dengan badan. Baru setelah itu tentukan apakah versi protokol yang cocok didukung. Urutan ini mencegah proxy dan server menafsirkan permintaan yang berbeda.

| Kondisi | HTTP | Galat JSON-RPC |
|---------|------|----------------|
| Versi, metode, atau nama pada header dan badan tidak cocok | 400 | `-32020` |
| Header dan badan sepakat pada versi yang tidak didukung | 400 | `-32022`, dengan `data` tepat `{"supported":["2026-07-28"],"requested":"<actual>"}` |
| `resources/read` tidak memiliki kapabilitas ekstensi Apps | 400 | `-32021`, dengan `data.requiredCapabilities.extensions.io.modelcontextprotocol/ui` | 
| Metode tidak dikenal | 404 | `-32601` |

Notifikasi JSON-RPC tidak memiliki `id`, sehingga server tidak pernah mengeluarkan respons JSON-RPC untuknya. Notifikasi HTTP yang diterima mengembalikan 202 dengan badan kosong. Galat dapat mengubah status HTTP, tetapi tetap tidak dapat membuat badan galat JSON-RPC untuk notifikasi.

### Sandbox adalah batas, bukan vonis kepercayaan

Host mengendalikan iframe. App tidak dapat membaca cookie host, local storage, atau DOM halaman secara langsung. Semua pekerjaan istimewa harus melewati jembatan.

Gunakan nilai bawaan ini:

- Biarkan semua daftar domain CSP kosong, lalu tambahkan hanya origin yang dibutuhkan App. Gunakan `connectDomains` untuk fetch, XHR, dan WebSocket; gunakan `resourceDomains` untuk skrip, gaya, gambar, dan font.
- Gabungkan kode dan data bila memungkinkan.
- Jangan meminta izin kamera, mikrofon, atau lokasi kecuali fitur yang terlihat membutuhkannya.
- Kunci `postMessage` ke origin rekan yang tepat dan tolak peristiwa dari semua origin lain.
- Perlakukan argumen alat, hasil alat, teks sumber daya, dan pesan jembatan sebagai masukan yang tidak tepercaya.
- Simpan persetujuan pengguna di host. iframe tidak dapat menyetujui aksi berdampaknya sendiri.

Jangan menyalin atribut `sandbox` tetap dari tutorial ke setiap host. Host harus memilih flag berdasarkan model origin App dan desain isolasinya sendiri.

Domain yang diizinkan tetap merupakan jalur eksfiltrasi. `connectDomains: ["https://api.example.com"]` berarti skrip apa pun yang dieksekusi di dalam App dapat mengirim data yang diizinkan ke sana. Pencocokan origin yang tepat mencegah kebingungan tujuan, tetapi tidak menentukan apakah muatannya pantas. Biarkan akses connect kosong secara bawaan, hindari menaruh token bearer di iframe, mediasi operasi sempit melalui host bila memungkinkan, batasi ukuran respons dan permintaan, serta audit aksi pengguna mana yang menyebabkan setiap permintaan keluar. Perlakukan `resourceDomains` secara terpisah dari `connectDomains`; izin memuat font atau skrip tidak boleh memberikan hak unggah data sembarangan.

### Jembatan Apps memiliki siklus hidupnya sendiri

Jembatan Apps adalah dialek JSON-RPC di atas `postMessage`. Jembatan ini dapat bertukar `ui/initialize` dan notifikasi `ui/*` serta dapat memproksikan metode yang tampak seperti metode inti, misalnya `tools/call`.

View mengirim `ui/initialize` dengan `appInfo` dan objek `appCapabilities`. Host mengembalikan kapabilitasnya dan konteks host. Hanya setelah respons itu View mengirim `ui/notifications/initialized`. Host harus menunggu notifikasi Apps ini sebelum mengirim pesan ke View.

Jabat tangan lokal ini menciptakan jembatan antara satu iframe dan satu frame host. Jabat tangan ini tidak menegosiasi versi protokol MCP, tidak menciptakan keadaan server, dan tidak menerbitkan sesi transpor. Perhatikan awalan yang tepat: `notifications/initialized` inti telah dihapus, sedangkan `ui/notifications/initialized` Apps tetap ada. Permintaan inti yang dihasilkan oleh panggilan alat yang dijembatani adalah permintaan mandiri baru dengan id JSON-RPC baru dan metadata permintaan lengkap.

### Konteks host, aksi, dan pencabutan

Host tetap menjadi otoritas setelah inisialisasi jembatan. View dapat meminta aksi alat, navigasi, penggunaan papan klip, atau efek istimewa lainnya hanya melalui kapabilitas yang diiklankan host. Host memvalidasi permintaan bertipe, pengguna saat ini, target, dan argumen, menerapkan kebijakan persetujuan, serta dapat menolaknya. Klik tombol dan pesan jembatan yang valid hanya menyatakan niat; keduanya tidak memberikan otoritas.

Perlakukan tema, ukuran, dan aksesibilitas sebagai konteks host yang berubah, bukan masukan render sekali pakai:

- Terapkan token warna dan tipografi yang diberikan host, lalu bereaksi ketika preferensi tema atau kontras berubah.
- Biarkan View melaporkan dimensi yang diinginkan, tetapi biarkan host membatasi dan menerapkan ukuran iframe agar konten tidak keluar dari tata letaknya atau membuat overlay yang menipu.
- Pertahankan urutan keyboard, fokus yang terlihat, nama yang dapat diakses, status pembaca layar, kontras yang cukup, zoom, dan perilaku reduced-motion di dalam iframe.
- Uji ulang perpindahan fokus antara kontrol host dan kontrol View setelah pengubahan ukuran dan render ulang.

Kapabilitas dapat dicabut saat App terbuka karena pengguna berganti akun, kebijakan berubah, server dikarantina, atau host mempersempit persetujuan. Periksa kapabilitas dan otorisasi pada saat aksi, bukan hanya selama `ui/initialize`. Saat pencabutan, tolak panggilan istimewa yang tertunda, hentikan aktivitas jaringan yang tidak lagi sesuai kebijakan, bersihkan keadaan render sensitif, dan pasang ulang atau kembalikan ke teks ketika sumber daya UI itu sendiri tidak lagi diterima. View harus memperlakukan penolakan sebagai hasil normal, bukan mencoba lagi sampai host menyerah.

### Fallback adalah bagian dari kontrak

Server yang sadar Apps tetap dapat melayani host yang tidak mengiklankan ekstensi UI:

- Kembalikan alat yang sama tanpa `_meta.ui` di `tools/list`.
- Pertahankan hasil teks yang berguna untuk `tools/call`.
- Tolak `resources/read` untuk UI dengan galat kapabilitas yang hilang.
- Jangan pernah menganggap iframe ada ketika menentukan apakah alat selesai.

```figure
t3-ui-sandbox
```

## Bangunlah

`code/main.py` membangun model protokol kecil dalam proses tanpa SDK. Model ini memvalidasi amplop permintaan terkini dan nilai routing Streamable HTTP, mengiklankan Apps melalui `server/discover`, mencantumkan alat serta sumber daya, menjalankan alat, dan menyajikan sumber daya HTML mandiri.

Model menerima badan yang telah diurai dan header routing. Model ini bukan adapter HTTP lengkap dan tidak mengurai `Content-Type` maupun `Accept`. Gunakan Pelajaran 09 untuk adapter Streamable HTTP lengkap yang mewajibkan `Content-Type: application/json` dan nilai `Accept` yang memuat baik `application/json` maupun `text/event-stream`.

Jalankan:

```bash
cd phases/13-tools-and-protocols/14-mcp-apps
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover code/tests -v
```

Periksa lima hal pada keluaran:

1. Setiap panggilan bersifat independen.
2. Setiap permintaan memiliki kapabilitas `_meta`.
3. `resources/list` mengembalikan deskriptor yang stabil sebelum ada pembacaan sumber daya.
4. Setiap hasil memiliki `resultType` dan metadata identitas server.
5. Tidak ada identifier sesi inti yang muncul.

## Gunakanlah

Mulai dengan `server/discover`. Pastikan `io.modelcontextprotocol/ui` muncul di peta ekstensi server. Lalu panggil `tools/list` dua kali, sekali dengan kapabilitas Apps dan sekali tanpa kapabilitas itu. Respons pertama mendeklarasikan sumber daya. Respons kedua tetap menjadi alat teks-saja yang dapat digunakan.

Baca `ui://notes/timeline.html`. Cari `hostOrigin` dan penjaga `event.origin` di HTML. Kedua baris itu adalah bukti terlihat minimal bahwa jembatan tidak menggunakan target wildcard.

## Kirimkan

Pelajaran ini mengirimkan `outputs/skill-mcp-apps-spec.md`. Gunakan berkas itu untuk meninjau kontrak App sebelum menulis kode kerangka kerja. Berkas itu memaksa penulis menyatakan amplop inti terkini, negosiasi ekstensi, fallback, sumber daya UI, kebijakan tembolok, CSP, izin, metode jembatan, dan batas persetujuan.

## Latihan

1. Ubah kapabilitas klien menjadi peta ekstensi kosong. Pastikan `tools/list` mempertahankan alat tetapi menghapus binding UI.
2. Kirim `Mcp-Name: ui://notes/other.html` dengan badan yang membaca linimasa. Pastikan galat `-32020`.
3. Ubah sumber daya menjadi `cacheScope: private`. Jelaskan kondisi spesifik pengguna yang membenarkannya.
4. Pindahkan skrip ke `https://static.example.com/app.js`. Tambahkan origin itu ke `resourceDomains` dan jelaskan risiko rantai pasok baru.
5. Tambahkan alat `notes_open` dan routing klik tombol melalui host. Simpan persetujuan pengguna di host.

## Istilah Kunci

| Istilah | Makna |
|---------|-------|
| MCP Apps | Ekstensi opsional untuk HTML interaktif yang dirender oleh host MCP |
| `io.modelcontextprotocol/ui` | Identifikator ekstensi yang diiklankan oleh kedua pihak |
| `ui://` | Skema sumber daya untuk templat UI sebuah App |
| `text/html;profile=mcp-app` | Jenis MIME untuk HTML MCP App |
| `server/discover` | RPC terkini untuk penemuan protokol dan kapabilitas |
| `resources/list` | Metode pencantuman sumber daya wajib ketika server mengiklankan sumber daya |
| `resultType` | Diskriminator wajib untuk hasil sukses modern |
| `ui/initialize` | Permintaan jembatan Apps pertama, terpisah dari inisialisasi inti yang dihapus |
| `ui/notifications/initialized` | Notifikasi kesiapan View Apps yang dikirim setelah host merespons |
| CSP | Kebijakan peramban yang membatasi skrip, gaya, gambar, dan origin jaringan |
| Fallback teks | Perilaku alat yang dipertahankan untuk host tanpa dukungan Apps |

## Bacaan Lanjutan

- [Protokol dasar MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic)
- [Ringkasan MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview)
- [Panduan membangun MCP Apps](https://modelcontextprotocol.io/extensions/apps/build)
- [Matriks dukungan ekstensi resmi](https://modelcontextprotocol.io/extensions/client-matrix)
