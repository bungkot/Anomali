# Gerbang MCP Tanpa Keadaan dan Penerimaan Registri

> Gerbang sebaiknya membuat setiap rute menjadi eksplisit. Protokol 2026-07-28 memberinya batas metode, nama, versi, kapabilitas, identitas, tembolok, dan trace tanpa sesi transpor.

**Jenis:** Pelajari
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 15 (keamanan), Fase 13 · 16 (otorisasi)
**Waktu:** ~75 menit

## Tujuan Pembelajaran

- Agregasi beberapa server MCP di balik satu endpoint 2026-07-28 tanpa afinitas sesi.
- Validasi metadata dan header routing per permintaan sebelum kebijakan atau penerusan.
- Gabungkan alat dengan namespace stabil, urutan deterministik, pin deskriptor, RBAC, dan penembolokan privat.
- Perlakukan catatan registri sebagai bukti penemuan yang tetap memerlukan kebijakan penerimaan.
- Routing SSE cakupan permintaan, `subscriptions/listen`, percobaan ulang MRTR, dan panggilan ekstensi Tasks dengan benar.
- Isolasi dukungan jabat tangan dan sesi lawas dari jalur modern.

## Masalahnya

Menghubungkan satu klien langsung ke satu server itu sederhana. Deployment yang lebih besar memerlukan jawaban konsisten untuk pertanyaan yang lebih sulit:

- Server mana yang diizinkan?
- Prinsipal mana yang dapat melihat dan memanggil tiap alat?
- Apa yang terjadi ketika dua backend mengekspos nama yang sama?
- Bagaimana perubahan deskriptor ditinjau?
- Di mana batas laju dan peristiwa audit diterapkan?
- Dapatkah instance mana pun menangani permintaan berikutnya?

Gerbang berada di antara klien dan server MCP backend. Gerbang menyajikan satu endpoint MCP, menerapkan kebijakan lintas, dan meneruskan permintaan yang disetujui.

Desain gerbang lawas sering memultipleks satu sesi klien menjadi beberapa sesi backend dan menulis ulang `Mcp-Session-Id`. Itu adalah desain kompatibilitas lawas. Inti 2026-07-28 tidak memiliki sesi protokol.

## Konsepnya

### Jalur gerbang modern

Untuk setiap permintaan:

1. Autentikasi prinsipal dari otorisasi transpor.
2. Validasi `MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name`, dan `params._meta`.
3. Otorisasi prinsipal, sumber daya, metode, alat, dan argumen.
4. Terapkan kebijakan deskriptor, registri, laju, dan data.
5. Buat permintaan mandiri baru untuk backend yang dipilih.
6. Validasi hasil backend dan kembalikan hasil gerbang.
7. Catat peristiwa audit tanpa mencatat rahasia.

Tidak ada langkah yang memerlukan sesi protokol tersembunyi. Keadaan aplikasi tetap dapat berada dalam basis data, pegangan eksplisit, Tasks, atau keadaan MRTR yang dilindungi integritas.

### Kebijakan runtime adalah keputusan utama gerbang

Penerimaan menentukan versi backend mana yang boleh masuk ke gerbang. Penerimaan tidak mengotorisasi panggilan langsung. Untuk setiap permintaan, gerbang menghitung ulang kebijakan dari prinsipal terautentikasi, penerbit dan sumber daya, tenant, metode dan nama yang cocok, argumen ternormalisasi, pin deskriptor yang diterima, kesehatan backend terkini, irisan kapabilitas, klasifikasi data, keadaan laju, serta persetujuan apa pun yang terikat pada aksi.

Urutan ini penting. Catatan Registri dapat tetap aktif sementara peran pengguna dicabut. Deskriptor dapat tetap dikunci sementara argumen tujuan melintasi batas tenant. Backend dapat tetap disetujui sementara kebijakan insiden mengarantina panggilan yang mengubah keadaan. Karena itu, kebijakan runtime adalah keputusan izin atau tolak utama, dengan bukti Registri dan deskriptor sebagai masukan.

Jangan menembolokkan keputusan izin di bawah koneksi atau identifier sesi yang dihapus. Jika kebijakan tidak tersedia, ikuti kebijakan kegagalan yang dinyatakan per kelas operasi. Nilai bawaan yang aman adalah gagal tertutup untuk perubahan keadaan dan pembacaan sensitif, sementara jalur baca publik yang secara eksplisit disetujui boleh memakai kebijakan terakhir yang diketahui berumur pendek hanya jika model risikonya mengizinkan. Catat versi kebijakan dan jalur kegagalan yang mengambil keputusan, lalu validasi hasil backend sebelum mengembalikannya.

### Satu endpoint POST

Streamable HTTP modern mengirim setiap pesan JSON-RPC melalui POST:

```text
POST /mcp
Authorization: Bearer <gateway-token>
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: notes.search
Accept: application/json, text/event-stream
```

Gerbang dapat mengembalikan JSON atau SSE cakupan permintaan untuk POST itu. GET dan DELETE mengembalikan 405 untuk permintaan modern. `Mcp-Session-Id` dan `Last-Event-ID` tidak menciptakan otoritas, afinitas, maupun perilaku replay.

Nilai header dan badan harus cocok. Tolak ketidaksesuaian dengan `-32020` sebelum mencari backend. Ini memungkinkan penyeimbang beban, gerbang, dan pembatas laju melakukan routing tanpa mengurai badan lengkap, sekaligus menjaga integritas ujung-ke-ujung.

Validasi dalam satu urutan tepat: tipe JSON-RPC dan metadata, kesetaraan header dan badan, lalu dukungan untuk versi yang cocok. Ketidaksesuaian mengembalikan HTTP 400 dengan `-32020`. Jika header dan badan sepakat pada versi yang tidak didukung, kembalikan HTTP 400 dengan `-32022` dan `data` tepat `{"supported":["2026-07-28"],"requested":"<actual>"}`. Metode yang tidak dikenal mengembalikan HTTP 404 dengan `-32601`.

`ProtocolError` membawa `data` opsional, dan gerbang menserialisasinya ke dalam objek galat JSON-RPC. Notifikasi tidak memiliki `id`, sehingga tidak pernah menerima sukses maupun galat JSON-RPC. Notifikasi HTTP yang diterima mengembalikan 202 dengan badan kosong.

### Implementasikan penemuan di setiap lapisan

Gerbang mengimplementasikan `server/discover` untuk klien. Gerbang juga menemukan setiap backend agar mengetahui versi protokol, kapabilitas, dan ekstensi.

Contoh hasil gerbang:

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": {
    "tools": {"listChanged": true}
  },
  "ttlMs": 30000,
  "cacheScope": "private",
  "_meta": {
    "io.modelcontextprotocol/serverInfo": {
      "name": "enterprise-gateway",
      "version": "2.0.0"
    }
  }
}
```

Iklankan hanya irisan kapabilitas yang dapat dipenuhi gerbang sampai ujung. Fitur backend tidak otomatis aman untuk diekspos. Fitur gerbang yang tidak memiliki jalur backend tidak berguna untuk diiklankan.

`serverInfo` adalah data tampilan dan diagnostik yang dilaporkan sendiri. Jangan gunakan sebagai bukti registri maupun penerbit.

### Kapabilitas klien per permintaan

Setiap permintaan yang diteruskan memerlukan amplop `_meta` terkini:

```json
{
  "io.modelcontextprotocol/protocolVersion": "2026-07-28",
  "io.modelcontextprotocol/clientCapabilities": {},
  "io.modelcontextprotocol/clientInfo": {
    "name": "enterprise-gateway",
    "version": "1.0.0"
  }
}
```

Jangan menyalin kapabilitas klien luar ke backend secara membabi buta. Gerbang adalah klien bagi backend. Iklankan hanya fitur yang akan dimediasi gerbang dengan benar.

### Pemberian namespace deterministik

Gabungkan alat backend di bawah nama publik yang stabil:

```text
notes.search
notes.create
issues.list
issues.open
```

Pertahankan pemetaan dari nama publik ke backend dan nama alat asli. Jangan pernah memilih tabrakan pertama atau terakhir. Nama publik adalah bagian dari kontrak persetujuan dan audit, sehingga mengubahnya adalah migrasi.

`tools/list` harus deterministik. Ketika visibilitas berbeda menurut prinsipal, kembalikan `cacheScope: private`. `ttlMs` terbatas mengurangi beban penemuan backend tanpa membiarkan daftar khusus pengguna bocor lintas konteks otorisasi.

Setiap deskriptor alat yang diekspos memuat nama yang stabil, deskripsi, dan `inputSchema` berakar objek. Pemberian namespace tidak dapat menghilangkan field deskriptor yang wajib. Hasil daftar lengkap juga memuat `resultType`, metadata identitas server, dan petunjuk tembolok.

### Kunci deskriptor yang disetujui

Saat penerimaan, kanonicalisasi deskriptor lengkap dan simpan intisarinya di bawah nama publik berkualifikasi. Saat pencantuman dan panggilan, bandingkan deskriptor langsung dengan intisari yang disetujui.

Jika berubah:

- Hapus dari `tools/list`.
- Tolak panggilan langsung.
- Pancarkan peristiwa audit.
- Wajibkan persetujuan ulang oleh kebijakan atau manusia sebelum memperbarui pin.

Gerbang adalah titik penegakan pusat yang berguna, tetapi gerbang tidak mengubah deskriptor yang dilihat pertama kali menjadi aman. Tinjauan awal tetap diperlukan.

### Registri membantu menemukan, bukan memutuskan

`server.json` Registri menyediakan metadata publikasi. Catatan yang didukung paket dapat terlihat seperti ini:

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "com.example/notes",
  "description": "Example notes MCP server.",
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@example/notes-mcp",
      "version": "1.0.0",
      "transport": {"type": "stdio"}
    }
  ]
}
```

Metadata publikasi tidak membawa keputusan keamanan gerbang. Simpan bukti penerbit dan provenans terverifikasi dalam keadaan penerimaan terpisah:

```json
{
  "registryName": "com.example/notes",
  "registryVersion": "1.0.0",
  "publisher": {"namespace": "com.example", "status": "verified"},
  "provenance": {
    "source": "registry.modelcontextprotocol.io",
    "recordId": "com.example/notes@1.0.0"
  },
  "admission": {"status": "approved", "reviewedBy": "gateway-policy"}
}
```

Gerbang memeriksa bentuk `server.json` dan menggabungkannya dengan keadaan eksternal itu. Gerbang tetap memerlukan kebijakan penerimaan.

Untuk setiap backend yang diterima, catat:

- Identifier registri dan catatan yang tepat.
- Bukti namespace atau domain penerbit terverifikasi.
- Transpor dan endpoint yang diizinkan.
- Versi yang dikunci atau kebijakan pembaruan yang disetujui.
- Intisari artefak atau deskriptor.
- Penerbit dan sumber daya otorisasi.
- Peninjau, waktu persetujuan, dan kedaluwarsa.

Jangan menerima server karena nama tampilannya menyerupai produk yang akrab. Jangan menganggap keberadaan di registri sebagai tinjauan keamanan operasional. Server privat dapat diterima melalui skema bukti yang sama meskipun tidak pernah muncul di registri publik.

Pelajaran ini mengimplementasikan sambungan gerbang: menggabungkan bukti publikasi dengan penerimaan lokal sebelum sebuah backend dapat di-routing. [Pelajaran 30: Rantai Pasok Registri MCP, Penerimaan, Pergeseran, dan Pemulihan](../../30-mcp-registry-supply-chain-and-drift/docs/en.md) membangun bidang kontrol penuh untuk bukti namespace yang tepat, provenans artefak, pin yang tidak dapat diubah, pergeseran deskriptor langsung, rekonsiliasi status Registri, buku besar penerimaan yang tahan pemalsuan, serta pemulihan berbasis bukti. Jaga keadaan rantai pasok itu tetap terpisah dari keputusan runtime per permintaan di atas.

### Mediasi kredensial

Gerbang mengautentikasi pemanggilnya dan secara terpisah mengautentikasi ke backend. Kredensial backend tidak pernah dikirim ke klien.

Jaga agar ikatan ini tetap eksplisit:

```text
outer principal -> gateway role and policy
backend issuer + resource -> backend registration and token
```

Jangan pernah meneruskan token gerbang luar ke backend. Jangan pernah menggunakan ulang token backend di penerbit atau sumber daya lain. Jika sebuah alat bertindak atas nama pengguna akhir, pertahankan delegasi itu dengan pertukaran atau model klaim yang dirancang, bukan dengan meniru pengguna menggunakan kredensial layanan bersama.

### Batas laju tanpa sesi

Kunci batas berdasarkan prinsipal terautentikasi, penerbit, sumber daya, alat publik, kelas biaya, dan jendela waktu. Id sesi tidak ada dan akan mudah diputar bahkan jika ada.

Terapkan validasi murah sebelum mengonsumsi pekerjaan mahal. Putuskan apakah panggilan yang ditolak dihitung terhadap batas penyalahgunaan, kuota bisnis, atau keduanya.

### Audit rantai keputusan

Catat cukup informasi untuk merekonstruksi sebuah panggilan:

- Identifier permintaan dan trace.
- Prinsipal dan penerbit terautentikasi.
- Alat publik dan rute backend.
- Versi pin deskriptor.
- Keputusan kebijakan dan alasannya.
- Latensi dan kelas hasil.
- Putaran MRTR atau identifier task bila berlaku.

Redaksi token bearer, kode otorisasi, token penyegaran, rahasia mentah, dan argumen sensitif yang tidak perlu.

### SSE cakupan permintaan

POST biasa dapat mengembalikan SSE cakupan permintaan ketika pekerjaan mengalir selama satu permintaan itu. Menutup aliran respons membatalkan permintaan HTTP modern yang sedang berjalan itu.

Jangan buat aliran GET terpisah dan jangan menjanjikan replay Last-Event-ID. Itu adalah asumsi transpor lawas.

### Notifikasi perubahan berumur panjang

Untuk notifikasi perubahan daftar dan sumber daya, klien terkini mengirim `subscriptions/listen` melalui POST dan menerima respons SSE. Filter notifikasi menggunakan field datar yang tepat `toolsListChanged`, `promptsListChanged`, `resourcesListChanged`, dan `resourceSubscriptions`:

```json
{
  "jsonrpc": "2.0",
  "id": "listen-tools",
  "method": "subscriptions/listen",
  "params": {
    "notifications": {
      "toolsListChanged": true
    },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {}
    }
  }
}
```

Peristiwa pertama mengakui himpunan bagian yang didukung. Identifier langganannya adalah id JSON-RPC permintaan yang membuka aliran:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/subscriptions/acknowledged",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/subscriptionId": "listen-tools"
    },
    "notifications": {
      "toolsListChanged": true
    }
  }
}
```

Gerbang kemudian hanya meneruskan jenis perubahan yang diakui. Setiap notifikasi pada aliran itu membawa `io.modelcontextprotocol/subscriptionId` yang sama di `params._meta`. Tidak ada replay otomatis maupun listen ulang otomatis. Saat reconnect, klien membuka kembali langganan dan menyegarkan daftar yang menjadi sandarannya. Penutupan anggun yang dimulai server mengembalikan hasil lengkap akhir yang ditandai dengan id langganan yang sama.

Jalur modern menggantikan `resources/subscribe`, `resources/unsubscribe`, dan streaming GET mandiri yang tidak diminta. Pertahankan itu hanya dalam jalur lawas yang dijaga versi.

### MRTR melalui gerbang

Ketika backend mengembalikan `resultType: input_required`, gerbang dapat meneruskan hasil itu hanya jika klien luar mendukung permintaan masukan yang dibutuhkan. Pertahankan `requestState` byte demi byte kecuali gerbang dengan sengaja mengakhiri dan menerbitkan ulang interaksi itu.

Klien mencoba ulang alat publik asli dengan id JSON-RPC baru dan `inputResponses`. Gerbang mengotorisasi ulang percobaan itu, memeriksa rute publik yang sama, lalu meneruskan permintaan backend baru. Gerbang tidak boleh menganggap putaran sebelumnya memberikan persetujuan tanpa batas.

### Routing ekstensi Tasks

Tasks adalah ekstensi resmi yang diidentifikasi oleh `io.modelcontextprotocol/tasks`. Tasks bukan pengganti sesi inti.

Klien menyatakan ekstensi itu di dalam kapabilitas klien per permintaan, dan gerbang mengiklankannya dalam penemuan hanya jika dapat mempertahankan siklus hidupnya sampai ujung. Untuk `tools/call` yang didukung, backend sendirilah yang memutuskan apakah akan mengembalikan hasil biasa atau `resultType: task`. Hasil task membawa `taskId`, `status`, stempel waktu, `ttlMs`, dan `pollIntervalMs` opsional langsung di hasil. Task harus sudah dapat dibaca secara tahan lama sebelum hasil itu dikirim.

Gerbang mencatat prinsipal terautentikasi dan rute backend untuk identifier task yang buram. Panggilan `tasks/get`, `tasks/update`, dan `tasks/cancel` berikutnya menggunakan `params.taskId` sebagai `Mcp-Name`, yang memberi perantara kunci routing. `tasks/get` mengembalikan `resultType: complete` dengan keadaan task terkini dan menyisipkan hasil akhir atau galat protokol dalam keadaan terminal. `tasks/update` mengirim `inputResponses` berbasis kunci untuk masukan task yang tertunda dan mengembalikan acknowledgment complete kosong. `tasks/cancel` adalah niat kooperatif dengan acknowledgment complete kosong, bukan jaminan bahwa pekerjaan berhenti.

Jangan implementasikan metode `tasks/list` atau `tasks/result` baru. Keduanya termasuk model eksperimental lawas. Task yang memerlukan masukan menyingkap permintaan tersemat lengkap melalui `tasks/get`; klien menjawabnya melalui `tasks/update`, bukan dengan mencoba ulang panggilan alat asli. Klien tetap melakukan polling pada interval yang disarankan; pembuatan task tetap diarahkan oleh server.

Keadaan rute task yang tahan lama adalah data aplikasi yang dikunci berdasarkan pegangan task, bukan sesi protokol.

### Batas kompatibilitas

Jika gerbang harus melayani klien atau backend lawas:

- Deteksi era secara eksplisit.
- Simpan inisialisasi, sesi transpor, aliran GET, langganan sumber daya, dan kosakata task lama di dalam adapter lawas.
- Jangan pernah membocorkan id sesi lawas ke routing atau otorisasi modern.
- Utamakan probe penemuan terbatas dan kebijakan fallback eksplisit daripada downgrade diam-diam.

```figure
t3-gateway-funnel
```

## Bangunlah

`code/main.py` mengimplementasikan gerbang protokol dalam proses dan dua server backend. Setiap backend menerima permintaan protokol terkini yang segar. Gerbang menyediakan penemuan, `tools/list` deterministik yang difilter pengguna, routing ber-namespace, `server.json` Registri plus keadaan penerimaan eksternal, pin deskriptor, RBAC, batas laju berbasis prinsipal, keputusan audit, serta acknowledgment SSE `subscriptions/listen` yang dimodelkan.

Model menerima badan permintaan yang telah diurai, header routing, dan identitas bearer terautentikasi. Model ini bukan adapter HTTP lengkap dan tidak mengurai `Content-Type` maupun kontrak `Accept` lengkap. Hubungkan ke adapter Streamable HTTP Pelajaran 09, yang mewajibkan `Content-Type: application/json` dan nilai `Accept` yang memuat baik `application/json` maupun `text/event-stream`.

Jalankan:

```bash
cd phases/13-tools-and-protocols/17-mcp-gateways-and-registries
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover code/tests -v
```

Demo mencetak id permintaan luar dan id permintaan backend yang segar agar lompatan tanpa keadaan terlihat.

## Gunakanlah

Ganti objek backend dalam proses dengan klien protokol terkini yang nyata. Pertahankan sambungan yang sama:

- Catatan penerimaan sebelum koneksi.
- Penemuan backend sebelum eksposur kapabilitas.
- Nama publik berkualifikasi sebelum otorisasi.
- Pin deskriptor sebelum pencantuman atau panggilan.
- Metadata per permintaan yang segar sebelum penerusan.
- Validasi hasil sebelum pengembalian.

## Kirimkan

Pelajaran ini mengirimkan `outputs/skill-gateway-bootstrap.md`. Berkas itu menghasilkan desain gerbang modern yang mencakup ingress, penemuan, penerimaan, namespace, otorisasi, penembolokan, streaming, langganan, MRTR, Tasks, observabilitas, dan isolasi lawas.

## Latihan

1. Tambahkan konteks trace ke metadata permintaan luar dan yang diteruskan, lalu catat korelasinya dalam peristiwa audit.
2. Tambahkan backend yang mendukung Tasks dan routing `tasks/get` berdasarkan id task di `Mcp-Name`.
3. Ubah satu deskriptor backend dan buktikan baik penemuan maupun panggilan langsung diblokir.
4. Tambahkan kapabilitas server khusus prinsipal dan jelaskan mengapa penemuan harus tetap ditembolokkan secara privat.
5. Tulis antarmuka adapter lawas tanpa menambahkan keadaan lawas apa pun ke kelas `Gateway` modern.

## Istilah Kunci

| Istilah | Makna |
|---------|-------|
| Gerbang MCP | Server kebijakan dan routing antara klien dan server MCP backend |
| Catatan penerimaan | Bukti dan keputusan kebijakan yang mengizinkan satu backend masuk ke gerbang |
| Nama alat berkualifikasi | Rute publik stabil seperti `notes.search` |
| Pin deskriptor | Intisari yang disetujui dan diperiksa saat penemuan serta dispatch |
| Cakupan tembolok privat | Hasil yang ditembolok hanya untuk satu konteks otorisasi |
| SSE cakupan permintaan | Respons streaming yang melekat pada satu permintaan POST |
| `subscriptions/listen` | Aliran SSE yang dibuka klien untuk notifikasi perubahan berumur panjang tertentu |
| Rute task | Pemetaan aplikasi dari id task yang buram ke backend-nya |
| Adapter lawas | Batas yang dijaga versi secara eksplisit untuk jabat tangan dan perilaku sesi lawas |

## Bacaan Lanjutan

- [Transpor Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [Penemuan server](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [Persyaratan server.json Registri resmi](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/official-registry-requirements.md)
- [Ekstensi MCP Tasks](https://tasks.extensions.modelcontextprotocol.io/specification/draft/tasks)
