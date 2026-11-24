# Cakupan Eksplisit dan Elicitation Tanpa Status

> Roots sudah usang (deprecated) dalam MCP 2026-07-28 dan tidak pernah menjadi sandbox keamanan. Letakkan cakupan pada argumen alat yang terlihat atau URI sumber daya, beri otorisasi di server, dan gunakan MRTR ketika sebuah alat benar-benar memerlukan masukan pengguna. Pengguna melihat keputusannya, model melihat handle-nya, dan instance server mana pun dapat memproses percobaan ulang.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 07 (server MCP), Fase 13 · 11 (MRTR tanpa status)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Ganti Roots yang usang dengan parameter ruang kerja eksplisit, URI sumber daya, atau konfigurasi server.
- Pisahkan petunjuk cakupan dari otorisasi, pembatasan jalur (containment), dan sandboxing sistem operasi.
- Sampaikan `elicitation/create` mode formulir melalui hasil MRTR `input_required`.
- Iklankan dukungan elicitation pada kapabilitas klien per permintaan dan tolak mode yang tidak didukung.
- Validasi `accept`, `decline`, dan `cancel` sebagai hasil yang berbeda.
- Ikat konfirmasi destruktif pada prinsipal terautentikasi, argumen asli, himpunan kandidat, dan masa berlaku.

## Dua Masalah yang Terlihat Serupa

Sebuah alat catatan menerima permintaan ini: "Hapus laporan TPS lama."

Server harus menjawab dua pertanyaan yang berbeda.

1. Ruang kerja mana yang boleh disentuh operasi ini?
2. Di antara tiga catatan yang cocok, mana yang dimaksud pengguna?

Yang pertama adalah cakupan dan otorisasi. Yang kedua adalah disambiguasi interaktif. Mencampur keduanya menghasilkan desain berbahaya, misalnya memperlakukan folder yang diberikan klien sebagai bukti bahwa pemanggil boleh menghapus apa pun di dalamnya.

## Roots Adalah Permukaan Migrasi

Revisi MCP sebelumnya memungkinkan klien mengiklankan Roots dan memberi tahu server ketika daftar berubah. Roots hanya bersifat petunjuk informatif. Roots tidak membatasi apa yang dapat dibaca proses server, tidak memberi otorisasi kepada pemanggil, dan tidak menciptakan sandbox sistem operasi.

MCP 2026-07-28 mengusangkan (deprecate) `roots/list` dan `notifications/roots/list_changed` untuk desain baru. Pilih salah satu pengganti eksplisit berikut:

- Argumen alat `workspaceUri` atau `directory` ketika cakupan berubah per panggilan.
- URI sumber daya ketika operasi memang sudah menargetkan sebuah sumber daya.
- Konfigurasi server ketika satu deployment memiliki satu ruang kerja tetap.
- Sandbox proses atau filesystem kurungan ketika kode harus secara teknis tidak dapat keluar.

Jika integrasi 2026-07-28 yang sudah ada masih memerlukan `roots/list` selama masa transisi pengusangan, server menyisipkannya dalam MRTR `inputRequests`. Server tidak boleh mengirim permintaan balik langsung. Itu adalah adapter migrasi; handler baru sebaiknya menerima cakupan eksplisit sebagai gantinya.

Model dapat melihat dan mengulang handle eksplisit. Cakupan sesi transpor yang tersembunyi lebih sulit diperiksa, diputar ulang, diaudit, dan dirutekan.

### Aturan tiga lapis

URI eksplisit tetap tidak memberi otorisasi dengan sendirinya. Terapkan ketiga lapis ini:

1. **Otorisasi:** Apakah prinsipal terautentikasi ini diizinkan menggunakan ruang kerja ini?
2. **Pembatasan (containment):** Apakah URI target yang ternormalkan tetap berada di dalam batas ruang kerja yang diizinkan?
3. **Sandbox:** Dapatkah sistem operasi menghentikan server yang dikompromikan agar tetap tidak keluar?

Server yang dapat dijalankan menyimpan daftar izin (allowlist) URI ruang kerja yang disahkan, menormalkan jalur berkode persen, memeriksa batas komponen jalur yang sebenarnya, dan memeriksa ulang pembatasan tepat sebelum penghapusan.

Pemeriksaan awalan string yang naif itu keliru:

```text
allowed:   file:///work/notes
attacker:  file:///work/notes-evil/secret.md
traversal: file:///work/notes/%2e%2e/private.md
```

Kedua jalur berbahaya itu diawali string yang menyesatkan. Normalkan terlebih dahulu, lalu bandingkan komponen jalur. Server filesystem produksi juga harus bertahan terhadap race symbolic link dan semantik jalur spesifik platform.

## Elicitation Tetap Ada, tetapi Penyampaiannya Berubah

Elicitation adalah fitur klien saat ini untuk mengumpulkan masukan pengguna selama `tools/call`, `prompts/get`, atau `resources/read`. Nama metode tetap `elicitation/create`. Yang berubah adalah arah aliran di kabel (wire flow).

Server 2026-07-28 tidak mengirim permintaan JSON-RPC balik. Server mengembalikan `InputRequiredResult`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "delete_choice": {
        "method": "elicitation/create",
        "params": {
          "mode": "form",
          "message": "Choose one matching note and confirm deletion.",
          "requestedSchema": {
            "type": "object",
            "properties": {
              "note_id": {
                "type": "string",
                "enum": ["note-3", "note-7", "note-14"]
              },
              "confirm": {"type": "boolean"}
            },
            "required": ["note_id", "confirm"]
          }
        }
      }
    },
    "requestState": "integrity-protected-delete-state"
  }
}
```

Host merender formulir. Pengguna dapat menerima, menolak secara eksplisit, atau menutupnya. Klien kemudian mencoba ulang `tools/call` asli dengan id baru:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "notes_delete",
    "arguments": {
      "workspaceUri": "file:///Users/alice/Documents/Notes",
      "title": "TPS report"
    },
    "inputResponses": {
      "delete_choice": {
        "action": "accept",
        "content": {"note_id": "note-14", "confirm": true}
      }
    },
    "requestState": "integrity-protected-delete-state",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "elicitation": {"form": {}}
      }
    }
  }
}
```

Tidak ada sesi protokol di antara dua panggilan tersebut. Server memverifikasi state yang digaungkan kembali, memvalidasi respons terhadap skema yang diharapkan, memeriksa bahwa catatan yang dipilih termasuk dalam himpunan kandidat bertanda tangan, mengautorisasi ulang ruang kerja, memeriksa ulang pembatasan, lalu menghapus.

## Negosiasi Kapabilitas Bersifat Per Permintaan

Klien yang mendukung elicitation mode formulir menyatakan:

```json
{
  "io.modelcontextprotocol/clientCapabilities": {
    "elicitation": {"form": {}}
  }
}
```

Kapabilitas elicitation kosong, `"elicitation": {}`, tetap setara dengan dukungan khusus formulir demi kompatibilitas. Deklarasi eksplisit `"elicitation": {"form": {}}` juga mendukung mode formulir. Deklarasi khusus URL, `"elicitation": {"url": {}}`, tidak. Server tidak boleh menyisipkan mode yang tidak ada dalam kapabilitas permintaan saat ini, meskipun permintaan sebelumnya mengiklukannya.

Setiap permintaan juga membawa `io.modelcontextprotocol/protocolVersion`. Versi yang hilang atau bukan string mengembalikan `-32602`. String yang tidak didukung mengembalikan `-32022` dengan data `supported` dan `requested` yang tepat. Dukungan elicitation yang hilang atau khusus URL mengembalikan `-32021` dengan `data.requiredCapabilities` ditetapkan ke `{"elicitation":{"form":{}}}`.

Amplop tanpa `id` JSON-RPC adalah notifikasi. Proseslah tanpa menghasilkan respons sukses maupun error JSON-RPC. Pada Streamable HTTP, notifikasi yang diterima mendapat `202 Accepted` tanpa badan.

`clientInfo` sebaiknya disertakan untuk diagnostik, tetapi bersifat self-reported dan tidak dapat mengidentifikasi pengguna untuk otorisasi.

Server mengimplementasikan `server/discover` dan mengembalikan `supportedVersions`, kapabilitas, `ttlMs`, dan `cacheScope` dengan `resultType: "complete"`. Server tidak mengiklankan Roots untuk desain modern ini. Karena mengiklankan alat, server juga mengimplementasikan `tools/list` yang wajib. Hasil tersebut mengembalikan deskriptor `notes_delete` yang deterministik, `inputSchema` objek yang valid, metadata identitas server, dan petunjuk cache publik.

## Mode Formulir

Mode formulir menggunakan JSON Schema terbatas yang dirancang untuk dialog yang dapat digunakan. Root berupa objek dan propertinya berupa field primitif datar atau array enum yang didukung. Objek bersarang dalam dan skema dokumen serbaguna tidak pantas berada dalam dialog konfirmasi.

Gunakan mode formulir untuk:

- memilih salah satu dari beberapa kandidat;
- mengonfirmasi operasi destruktif;
- mengumpulkan preferensi yang tidak sensitif;
- mengumpulkan sejumlah kecil nilai yang harus diputuskan oleh pengguna, bukan model.

Jangan gunakan mode formulir untuk kata sandi, kunci API, token akses, atau kredensial pembayaran. Rahasia tersebut akan melewati klien MCP dan dapat masuk ke log atau konteks model.

Server memvalidasi kembali konten yang dikembalikan. Validasi formulir di sisi klien meningkatkan UX tetapi tidak menciptakan kepercayaan.

## Mode URL

Mode URL mengirim URL web aman untuk interaksi di luar jalur (out-of-band):

```json
{
  "method": "elicitation/create",
  "params": {
    "mode": "url",
    "message": "Connect the report service to continue.",
    "url": "https://mcp.example.com/connect/report-service"
  }
}
```

Gunakan mode ini ketika informasi sensitif harus masuk langsung ke alur web yang dikendalikan server, misalnya otorisasi pihak ketiga. Klien menampilkan tujuan lengkap dan meminta persetujuan sebelum membukanya. Klien tidak boleh mengambil URL terlebih dahulu (prefetch).

Respons `accept` berarti pengguna setuju membuka URL. Respons itu tidak membuktikan alur eksternal selesai. Saat mencoba ulang, server memeriksa state-nya sendiri lalu menyelesaikan tugas atau mengembalikan hasil `input_required` lagi.

Elicitation URL bukan pengganti otorisasi antara klien MCP dan server MCP. Mode ini untuk interaksi eksternal yang perlu dilakukan server MCP atas nama pengguna. Server harus mengikat pengguna browser ke prinsipal terautentikasi yang sama yang memulai operasi MCP.

## Cabang Respons

Perlakukan aksi-aksi ini sebagai keputusan produk, bukan alias:

| Aksi | Makna | Perilaku server yang aman |
|--------|---------|----------------------|
| `accept` | Pengguna mengirimkan interaksi | Validasi konten dan lanjutkan |
| `decline` | Pengguna secara eksplisit menolak | Kembalikan hasil penolakan yang lengkap dan bukan error |
| `cancel` | Pengguna menutup atau tidak dapat menyelesaikan | Hentikan dengan aman dan izinkan percobaan ulang kemudian |

Jangan pernah menafsirkan konten yang hilang sebagai persetujuan. Jangan pernah mengubah penolakan menjadi loop prompt berulang.

## Melindungi State MRTR Destruktif

Daftar kandidat tidak boleh hanya berada dalam prompt atau nilai Base64 tanpa tanda tangan. Klien mengendalikan semua yang dikirim balik.

Pelajaran ini menandatangani payload state yang berisi:

- prinsipal terautentikasi;
- metode asal;
- digest `workspaceUri` dan `title`;
- id catatan yang diizinkan dan ditampilkan dalam formulir;
- fase operasi;
- masa berlaku singkat.

Sebelum mutasi, server juga memeriksa rekaman catatan aktual. Ini menangkap race penghapusan dan target yang dipindahkan ke luar ruang kerja setelah formulir ditampilkan.

Untuk tindakan finansial sekali jalan atau yang tidak dapat dibatalkan, HMAC saja tidak mencegah state valid diputar ulang dalam masa berlakunya. Simpan dan konsumsi nonce tepat sekali dalam penyimpanan replay yang dibagikan oleh setiap instance handler. Pelajaran ini menyuntikkan penyimpanan terbatas yang dipangkas berdasarkan TTL dan menahan klaim atomiknya saat melakukan penghapusan dalam memori. Database produksi sebaiknya menggabungkan klaim nonce dan mutasi dalam satu transaksi atau batas penulisan bersyarat yang setara.

Validasi interaksi sebelum mengklaim nonce. Respons yang malformed atau `cancel` tidak melakukan mutasi dan membiarkan state dapat dicoba ulang hingga kedaluwarsa. `decline` eksplisit bersifat terminal, sehingga pelajaran ini mengonsumsi nonce tanpa menghapus apa pun.

```figure
t3-roots-boundary
```

## Bangunlah

`code/main.py` mendemonstrasikan alat `notes_delete` modern:

- `tools/list` mengembalikan deskriptor deterministik yang dapat ditembolok dengan skema ruang kerja dan judul yang wajib.
- Cakupan adalah argumen `workspaceUri` eksplisit.
- Konfigurasi server mengautorisasi ruang kerja tersebut untuk prinsipal pelajaran.
- Normalisasi URI menolak kebingungan awalan dan traversal terenkode.
- Setiap penghapusan destruktif memerlukan elicitation mode formulir.
- Elicitation dikirim di dalam `resultType: "input_required"`.
- `requestState` bertanda tangan mengikat daftar kandidat dan argumen asli secara tepat.
- Penyimpanan replay yang disuntikkan menolak state yang sama yang diterima atau ditolak lintas instance server.
- Percobaan ulang menggunakan id permintaan baru dan mengembalikan `resultType: "complete"`.

Penyimpanan data berada di memori agar perilaku protokol mudah diperiksa. Aturan keamanan tetap sama dengan database.

## Gunakanlah

Dari root repositori:

```bash
cd phases/13-tools-and-protocols/12-mcp-roots-and-elicitation/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Titik pemeriksaan yang diharapkan:

- Discovery mengiklankan alat tanpa Roots.
- Discovery alat mengembalikan `notes_delete` dengan `resultType`, identitas server, dan petunjuk cache.
- Permintaan id `1` mengembalikan formulir di `inputRequests.delete_choice`.
- Permintaan id `2` menggaungkan state bertanda tangan dan menyelesaikan penghapusan.
- Jalur awalan dan jalur traversal terenkode keduanya gagal dalam pembatasan.
- Judul yang diubah tidak dapat menggunakan ulang state konfirmasi asli.
- Penolakan membiarkan catatan tidak berubah.
- Dua objek server yang berbagi state catatan dan replay tidak boleh menjalankan satu konfirmasi secara bersamaan.
- Deklarasi formulir kosong dan eksplisit keduanya berfungsi, sementara dukungan khusus URL mengembalikan persyaratan formulir `-32021` yang tepat.
- Kegagalan versi yang tidak didukung menggunakan bentuk data `-32022` yang tepat.
- Notifikasi tanpa id tidak menghasilkan respons JSON-RPC.

## Terapkanlah

`outputs/skill-elicitation-form-designer.md` merancang cakupan eksplisit, pemeriksaan otorisasi, formulir MRTR, cabang respons, dan pengikatan state. File ini menolak memperlakukan Roots yang usang sebagai sandbox atau mengumpulkan rahasia melalui mode formulir.

## Latihan

1. Ganti penyimpanan replay dalam memori dengan SQLite. Gunakan satu transaksi untuk mengklaim nonce dan menghapus catatan, lalu buktikan dua proses tidak dapat melakukan commit secara bersamaan.
2. Tambahkan negosiasi kapabilitas `url` dan alur penyiapan di luar jalur. Jauhkan kredensial pihak ketiga dari `inputResponses`.
3. Ganti peta catatan dalam memori dengan database SQLite sementara. Periksa ulang otorisasi dan pembatasan di dalam transaksi mutasi.
4. Tambahkan kebijakan symbolic link untuk implementasi filesystem nyata. Jelaskan mengapa pembatasan leksikal URI saja tidak dapat menghentikan pelarian symlink.
5. Rancang adapter 2025-11-25 yang memetakan keluaran handler MRTR modern ke elicitation lama yang dimulai server. Jaga agar adapter tetap terisolasi dari handler saat ini.

## Istilah Kunci

| Istilah | Makna dalam 2026-07-28 |
|--------|------------------------|
| Roots | Petunjuk ruang kerja informatif yang usang, bukan otorisasi atau sandboxing |
| Cakupan eksplisit | Ruang kerja, direktori, atau handle sumber daya yang terlihat dalam argumen permintaan |
| Pembatasan (containment) | Pemeriksaan komponen jalur ternormalkan yang menjaga target tetap di dalam batas |
| Elicitation | Fitur klien untuk mendapatkan masukan pengguna selama operasi MCP |
| Mode formulir | Masukan pengguna terstruktur dalam jalur dengan skema datar terbatas |
| Mode URL | Interaksi di luar jalur untuk alur kerja sensitif atau eksternal |
| MRTR | Hasil input_required tanpa status yang diikuti percobaan ulang baru |
| `requestState` | State buram yang digaungkan tepat dan diperiksa integritasnya oleh server |
| Penolakan (Decline) | Penolakan eksplisit pengguna |
| Pembatalan (Cancel) | Penutupan atau interaksi tidak lengkap tanpa persetujuan |

## Kompatibilitas Warisan

Untuk rekan yang terkunci pada 2025-11-25, `roots/list`, `notifications/roots/list_changed`, dan `elicitation/create` langsung yang dimulai server mungkin masih ada. Beri label warisan (legacy) pada adapter tersebut. Jangan izinkan daftar Root warisan melewati otorisasi server, dan jangan membawa asumsi sesi protokol ke dalam handler modern.

## Bacaan Lanjutan

- [Elicitation MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation)
- [Permintaan Multi Perjalanan Bolak-Balik MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [Pengusangan Roots MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/client/roots)
- [Discovery server MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
