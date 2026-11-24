# Ekstensi Tasks MCP: Pekerjaan Persisten di atas Inti Tanpa Status

> MCP tanpa status tidak berarti setiap operasi harus selesai dalam satu permintaan. Ekstensi Tasks resmi memberikan pekerjaan yang berjalan lama sebuah handle persisten yang eksplisit. Server dapat mengembalikan handle tersebut dari `tools/call`, instance mana pun dapat menjawab `tasks/get`, dan masukan klien masuk melalui `tasks/update` tanpa menghidupkan kembali sesi protokol.

**Jenis:** Build
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 09 (transpor), Fase 13 · 11 (MRTR tanpa status), Fase 13 · 12 (elicitation)
**Waktu:** ~90 menit

## Tujuan Pembelajaran

- Bedakan transpor protokol tanpa status dari state tugas aplikasi yang persisten.
- Negosiasi ekstensi `io.modelcontextprotocol/tasks` pada kapabilitas per permintaan dan `server/discover`.
- Kembalikan `CreateTaskResult` yang diarahkan server dengan `resultType: "task"` hanya setelah penciptaan persisten.
- Lakukan polling dengan `tasks/get`, penuhi masukan tugas dengan `tasks/update`, dan minta pembatalan kooperatif dengan `tasks/cancel`.
- Singkirkan asumsi lama tentang `tasks/status`, `tasks/result`, dan `tasks/list`.
- Berlangganan notifikasi tugas opsional melalui `subscriptions/listen` pada aliran SSE respons POST.
- Permodelkan kedaluwarsa tugas, pemulihan setelah restart, deduplikasi kunci masukan, dan error eksekusi dengan benar.

## Mengapa Tasks Menjadi Ekstensi

Tasks pertama kali muncul sebagai fitur inti eksperimental pada 2025-11-25. Desain ulang Juli 2026 memindahkannya ke ekstensi resmi `io.modelcontextprotocol/tasks` agar klien dan server dapat mengikutsertakan siklus hidup tambahan tanpa memperluas protokol inti untuk semua pihak.

Spesifikasi ekstensi tetap berstatus draf meskipun menjadi rumah resmi Tasks saat ini. Kunci versi ekstensi yang didukung SDK Anda, jalankan skenario kepatuhan (conformance), dan isolasi adapter kabel dari domain pekerja dan penyimpanan Anda.

Gunakan tugas ketika operasi memiliki satu atau lebih sifat berikut:

- Operasi dapat bertahan lebih lama dari batas waktu permintaan biasa.
- Antrean pekerja atau sistem pekerjaan eksternal sudah mengelola eksekusi.
- Klien perlu pulih setelah restart sendiri.
- Operasi berhenti sementara untuk masukan pengguna atau model saat eksekusi.
- Pembatalan dan pengambilan hasil persisten merupakan kebutuhan produk.

Jangan buat tugas untuk pencarian deterministik yang murah. Handle, persistensi, polling, kedaluwarsa, dan pembatalan adalah kompleksitas nyata.

## Inti Tanpa Status, Aplikasi Berstate

MCP 2026-07-28 menghapus `initialize`, `notifications/initialized`, sesi protokol, dan `Mcp-Session-Id`. Hal itu tidak melarang produk yang berstate.

Id tugas adalah state aplikasi yang eksplisit:

- Server mempersistennya sebelum mengembalikannya.
- Klien dapat menyimpannya dan melakukan polling lagi setelah restart.
- Id dapat diroute ke replika mana pun yang menggunakan penyimpanan persisten yang sama.
- Otorisasi diperiksa pada setiap metode tugas.
- Kedaluwarsa dan penghapusan ditentukan oleh field tugas, bukan umur transpor.

Ini secara operasional berbeda dari state tersembunyi yang melekat pada koneksi.

Jaga agar empat masa hidup tetap terpisah:

| State | Masa hidup | Tempatnya |
|---|---|---|
| Metadata protokol | Satu permintaan | `params._meta`, divalidasi ulang pada setiap panggilan |
| Pekerjaan transpor | Satu permintaan stdio atau respons HTTP | Koordinator in-flight dengan batas waktu terbatas |
| Kelanjutan MRTR | Satu urutan percobaan ulang | `requestState` dilindungi integritas, plus kontrol replay bila diperlukan |
| Tugas persisten | Lintas permintaan, replika, restart, dan reconnect | Penyimpanan aplikasi bersama yang dikunci oleh `taskId` terautorisasi |

Memindahkan rekaman tugas ke memori proses tidak membuat MCP menjadi berstate. Hal itu membuat aplikasi tidak andal. Protokol tetap tanpa status, tetapi `tasks/get` berikutnya yang diroute ke replika lain tidak dapat memulihkan rekaman tersebut. Persistenkan sebelum mengembalikan handle, lalu pastikan setiap metode tugas menyelesaikan rekaman bersama yang sama di bawah pemeriksaan tenant dan prinsipal.

## Negosiasi Kapabilitas

Klien mengiklankan dukungan pada setiap permintaan yang memenuhi syarat:

```json
{
  "_meta": {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientCapabilities": {
      "extensions": {
        "io.modelcontextprotocol/tasks": {}
      }
    },
    "io.modelcontextprotocol/clientInfo": {
      "name": "lesson-client",
      "version": "1.0.0"
    }
  }
}
```

Server mengembalikan `supportedVersions`, kapabilitas, `ttlMs`, dan `cacheScope` yang tepat dari `server/discover`, dengan ekstensi yang sama di bawah kapabilitas. Karena mengiklankan alat, server juga mengimplementasikan `tools/list` yang wajib. Hasil tersebut mengembalikan deskriptor `generate_report` yang deterministik, `inputSchema` objek valid, `resultType: "complete"`, metadata identitas server, dan petunjuk cache publik.

Metode tugas dari klien yang tidak mendeklarasikan ekstensi mengembalikan `-32021`, Missing Required Client Capability, dengan `data.requiredCapabilities` ditetapkan ke `{"extensions":{"io.modelcontextprotocol/tasks":{}}}`. String protokol yang tidak didukung mengembalikan `-32022` dengan data `supported` dan `requested` yang tepat; versi yang hilang atau bukan string mengembalikan `-32602`.

Amplop tanpa `id` JSON-RPC adalah notifikasi. Penerima dapat memprosesnya, tetapi tidak menghasilkan hasil maupun error JSON-RPC. Adapter Streamable HTTP mengembalikan `202 Accepted` tanpa badan untuk notifikasi yang diterima.

Saat ini, hanya `tools/call` yang mendukung eksekusi dengan augmentasi tugas. Rancang abstraksi internal Anda agar jenis permintaan di masa depan tidak perlu menulis ulang penyimpanan.

## Penciptaan Tugas yang Diarahkan Server

Flag klien lama `params._meta.task.required` sudah hilang. Klien mendeklarasikan dukungan ekstensi, lalu server memutuskan apakah suatu `tools/call` tertentu menjadi tugas.

Permintaan:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "generate_report",
    "arguments": {"size": "large"},
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "extensions": {
          "io.modelcontextprotocol/tasks": {}
        }
      }
    }
  }
}
```

Respons:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "task",
    "taskId": "tsk_786512e29e0d",
    "status": "working",
    "statusMessage": "Preparing report outline.",
    "createdAt": "2026-08-21T10:30:00Z",
    "lastUpdatedAt": "2026-08-21T10:30:00Z",
    "ttlMs": 900000,
    "pollIntervalMs": 1000
  }
}
```

Server tidak boleh mengembalikan handle ini sebelum `tasks/get` untuk id tersebut dapat menyelesaikannya. Pada penyimpanan yang konsisten akhirnya (eventually consistent), tunggu hingga data dapat dibaca sebelum menjawab. Jika tidak, klien dapat menerima id yang tampak valid lalu langsung mendapat "not found".

Respons tugas bersifat tanpa permintaan (unsolicited) dalam arti klien tidak meminta mode tugas. Namun bukan berarti tanpa negosiasi: permintaan saat ini tetap harus mengiklankan ekstensi.

## Bentuk Tugas

Setiap tugas membawa:

- `taskId`: identifier stabil yang dihasilkan server;
- `status`: `working`, `input_required`, `completed`, `cancelled`, atau `failed`;
- `createdAt` dan `lastUpdatedAt`: stempel waktu ISO 8601;
- `ttlMs`: durasi kedaluwarsa sejak penciptaan, atau `null` jika tidak ada batas yang diiklankan;
- `pollIntervalMs` opsional: jeda polling minimal yang saat ini disarankan server;
- `statusMessage` opsional: konteks untuk pengguna atau model.

Field khusus status hanya muncul ketika relevan:

- `input_required` memuat `inputRequests`.
- `completed` memuat bentuk `result` dari permintaan asli.
- `failed` memuat objek `error` JSON-RPC.

Klien sebaiknya mematuhi `pollIntervalMs`. Server dapat membatasi laju polling yang lebih agresif dan dapat mengubah interval selama masa hidup tugas.

## Polling dengan `tasks/get`

Klien meminta cuplikan terkini:

```http
POST /mcp HTTP/1.1
Content-Type: application/json
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tasks/get
Mcp-Name: tsk_786512e29e0d
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tasks/get",
  "params": {
    "taskId": "tsk_786512e29e0d",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "extensions": {
          "io.modelcontextprotocol/tasks": {}
        }
      }
    }
  }
}
```

`tasks/get` sendiri telah selesai, sehingga hasilnya selalu memiliki `resultType: "complete"`. Tugas bersarang tetap dapat ber-`status: "working"` atau `status: "input_required"`.

Pembedaan ini mencegah bug parser yang umum:

```text
result.resultType = complete    berarti RPC tasks/get telah selesai
result.status = working        berarti pekerjaan yang diwakili masih berjalan
```

Tidak ada panggilan `tasks/result`. Saat tugas selesai, respons `tasks/get` berikutnya menyisipkan `CallToolResult` asli di bawah `result`:

```json
{
  "resultType": "complete",
  "taskId": "tsk_786512e29e0d",
  "status": "completed",
  "createdAt": "2026-08-21T10:30:00Z",
  "lastUpdatedAt": "2026-08-21T10:34:12Z",
  "ttlMs": 900000,
  "result": {
    "resultType": "complete",
    "content": [
      {"type": "text", "text": "Generated large report with approved outline."}
    ],
    "structuredContent": {"size": "large", "approved": true},
    "isError": false,
    "_meta": {
      "io.modelcontextprotocol/serverInfo": {
        "name": "tasks-demo",
        "version": "1.0.0"
      }
    }
  },
  "_meta": {
    "io.modelcontextprotocol/serverInfo": {
      "name": "tasks-demo",
      "version": "1.0.0"
    }
  }
}
```

`resultType` luar menyatakan RPC `tasks/get` selesai. `result.resultType` bersarang menyatakan panggilan alat asli selesai. Diskriminator bersarang itu wajib. `CallToolResult` bersarang SEBAIKNYA juga membawa `io.modelcontextprotocol/serverInfo`-nya sendiri; pelajaran ini menyertakannya alih-alih menyimpan payload tanpa tipe.

Tidak ada `tasks/list`. Server tanpa sesi tidak dapat menyimpulkan dengan aman tugas mana yang termasuk dalam daftar berlingkup koneksi. Aplikasi yang memerlukan riwayat sebaiknya mengekspos alat domain terautorisasi dengan filter dan aturan kepemilikan yang eksplisit.

## Masukan Saat Eksekusi Tugas

Masukan tugas dan MRTR inti tampak serupa tetapi menggunakan kelanjutan yang berbeda.

### Masukan yang diperlukan sebelum penciptaan tugas

Kembalikan `resultType: "input_required"` inti dari `tools/call` asli. Klien memenuhinya dan mencoba ulang panggilan asli tersebut. Ciptakan tugas hanya setelah putaran MRTR sinkron itu selesai.

### Masukan yang diperlukan setelah penciptaan tugas

Tetapkan tugas ke `input_required`. `tasks/get` mengekspos `inputRequests` yang tertunda, dan klien mengirim respons melalui `tasks/update`. Klien tidak mencoba ulang `tools/call` asli.

Cuplikan:

```json
{
  "resultType": "complete",
  "taskId": "tsk_786512e29e0d",
  "status": "input_required",
  "createdAt": "2026-08-21T10:30:00Z",
  "lastUpdatedAt": "2026-08-21T10:31:00Z",
  "ttlMs": 900000,
  "inputRequests": {
    "approve_outline": {
      "method": "elicitation/create",
      "params": {
        "mode": "form",
        "message": "Approve the generated report outline?",
        "requestedSchema": {
          "type": "object",
          "properties": {"approved": {"type": "boolean"}},
          "required": ["approved"]
        }
      }
    }
  }
}
```

Pembaruan:

```http
POST /mcp HTTP/1.1
Content-Type: application/json
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tasks/update
Mcp-Name: tsk_786512e29e0d
```

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tasks/update",
  "params": {
    "taskId": "tsk_786512e29e0d",
    "inputResponses": {
      "approve_outline": {
        "action": "accept",
        "content": {"approved": true}
      }
    },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "extensions": {
          "io.modelcontextprotocol/tasks": {}
        }
      }
    }
  }
}
```

Respons sukses berupa acknowledged kosong ditambah `resultType: "complete"`. Perubahan state mungkin bersifat konsisten akhirnya, sehingga klien terus melakukan polling atau mendengarkan.

Setiap kunci `inputRequests` harus unik selama masa hidup tugas. Cuplikan `tasks/get` berulang dapat menampilkan kunci tertunda yang sama; klien melakukan deduplikasi UI dan server mengabaikan respons untuk kunci yang tidak dikenal, sudah digantikan, atau sudah dipenuhi. Pembaruan parsial dapat membiarkan tugas tetap `input_required` hingga semua kunci wajib dijawab.

## Pembatalan Bersifat Kooperatif

`tasks/cancel` sinyal niat dan mengembalikan acknowledged lengkap yang kosong. Acknowledged itu tidak menjamin pekerja telah berhenti. Pekerjaan dapat selesai lebih dulu, mengabaikan pembatalan, atau bertransisi kemudian.

```http
POST /mcp HTTP/1.1
Content-Type: application/json
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tasks/cancel
Mcp-Name: tsk_786512e29e0d
```

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "tasks/cancel",
  "params": {
    "taskId": "tsk_786512e29e0d",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "extensions": {
          "io.modelcontextprotocol/tasks": {}
        }
      }
    }
  }
}
```

Untuk ketiga metode tugas, `Mcp-Name` mencerminkan `params.taskId`. Header itu tidak mengulang nama metode JSON-RPC. `code/main.py` memusatkan aturan ini dalam `make_http_request`.

Pekerja dalam pelajaran ini mematuhi pembatalan segera, sehingga panggilan berulang bersifat idempoten. Klien produksi tetap harus memperlakukan pembatalan sebagai kooperatif alih-alih menyimpulkan status akhir tugas dari acknowledged.

Jangan gunakan `notifications/cancelled` untuk membatalkan tugas. Notifikasi itu termasuk pembatalan permintaan, bukan Tasks persisten.

Pembedaan ini penting di batas routing. Pembatalan permintaan menargetkan satu operasi JSON-RPC yang sedang berjalan atau respons HTTP yang berlingkup permintaan. Jika `tools/call` sudah mengembalikan `resultType: "task"`, permintaan itu selesai dan menutup transpornya tidak dapat menyebut atau menghentikan pekerjaan persisten. `tasks/cancel` adalah RPC berautorisasi baru. RPC ini membawa `params.taskId`, mencerminkan id itu di `Mcp-Name`, menyelesaikan backend pemilik tugas, mencatat niat pembatalan kooperatif, dan mengembalikan acknowledged tanpa menyatakan pekerja telah berhenti.

Karena itu, gateway harus menyimpan koordinator permintaan dan rute tugas dalam tabel yang berbeda. Tabel permintaan dapat hilang saat respons selesai. Rute tugas harus bertahan hingga status terminal dan kedaluwarsa retensi. [Pelajaran 29: Keandalan, Pembatalan, dan Kontrol Alur MCP](../../29-mcp-reliability-cancellation-and-flow-control/docs/en.md) membangun aturan race, timeout, idempotensi, backpressure, dan percobaan ulang untuk kedua jalur.

## Notifikasi Opsional

Polling adalah dasarnya. Klien yang menginginkan pembaruan push mengirim `subscriptions/listen` dengan id tugas. Pada Streamable HTTP, ini adalah POST yang responsnya berupa aliran SSE berlingkup permintaan. Tidak ada aliran peristiwa GET mandiri dan tidak ada sesi protokol yang harus dijaga tetap hidup.

Server mengakui id yang diterima dengan `notifications/subscriptions/acknowledged`, lalu dapat mengirim cuplikan lengkap melalui `notifications/tasks`. Acknowledged dan setiap notifikasi tugas membawa `io.modelcontextprotocol/subscriptionId` dalam `_meta`, sama dengan id permintaan `subscriptions/listen`. Selain itu, setiap notifikasi tugas setara dengan apa yang akan dikembalikan `tasks/get` pada saat itu.

Klien tetap harus mendeklarasikan ekstensi Tasks. Klien sebaiknya tersambung ulang dan melanjutkan dari id tugas persisten alih-alih bergantung pada pemutaran ulang peristiwa atau `Last-Event-ID`.

## Semantik Kegagalan

Gunakan dua lapis error dengan benar.

### Error protokol

Parameter metode yang tidak valid atau id tugas yang tidak dikenal mengembalikan error JSON-RPC, umumnya `-32602`. Dukungan ekstensi yang hilang mengembalikan `-32021` dengan objek kapabilitas yang wajib.

### Hasil eksekusi tugas

- Hasil alat biasa dengan `isError: true` tetap merupakan tugas `completed` karena panggilan alat menghasilkan hasil yang ditentukan.
- Error JSON-RPC selama eksekusi tertunda membuat tugas menjadi `failed` dan menyimpan error JSON-RPC itu di bawah `error`.
- Penolakan pengguna dapat menghasilkan `cancelled`, hasil penolakan yang selesai, atau hasil aman lain yang spesifik domain. Dokumentasikan pilihannya.

## Persistensi, Kedaluwarsa, dan Kepemilikan

Persistenkan setidaknya id tugas, status, stempel waktu, ttl, interval polling, kepemilikan operasi asli, hasil atau error, permintaan masukan tertunda, dan semua kunci masukan yang diterbitkan.

Kunci penyimpanan harus memuat atau menyelesaikan tenant dan prinsipal yang otoritatif. Mengetahui id tugas tidak boleh memberi akses. Periksa kepemilikan pada setiap `tasks/get`, `tasks/update`, `tasks/cancel`, dan langganan.

`ttlMs` diukur sejak penciptaan dan dapat berubah. Klien dapat memperlakukannya sebagai jaring pengaman ketika tugas berhenti menghasilkan pembaruan yang dapat diamati. Server dapat gagal lalu menghapus tugas yang kedaluwarsa. Jangan menggambarkannya sebagai janji untuk menyimpan hasil yang selesai selama sekian milidetik setelah penyelesaian.

Gunakan penulisan atomik atau transaksi. Pelajaran ini menulis file sementara lalu mengganti namanya secara atomik. Layanan multi-replika sebaiknya menggunakan penyimpanan persisten bersama dan lease pekerja atau kontrol konkurensi yang setara.

```figure
tp-task-lifecycle
```

## Bangunlah

`code/main.py` mengimplementasikan layanan tugas deterministik:

- `server/discover` mengembalikan `supportedVersions`, petunjuk cache, dan ekstensi Tasks.
- `tools/list` mengembalikan deskriptor `generate_report` deterministik yang dapat ditembolok dengan skema masukan yang valid.
- `tools/call` menciptakan dan mempersistenkan tugas sebelum mengembalikan `resultType: "task"`.
- Instance layanan baru memuat ulang tugas yang sama, mendemonstrasikan pemulihan setelah restart.
- `tasks/get` mengembalikan cuplikan tugas lengkap.
- Pekerja bertransisi dari `working` ke `input_required`.
- `tasks/update` menerima respons formulir dan mengembalikan acknowledged lengkap yang kosong.
- Pekerja menyimpan `CallToolResult` bersarang dengan `resultType` dan identitas servernya sendiri, lalu bertransisi ke `completed`.
- `tasks/cancel` bersifat idempoten dalam implementasi ini.
- Pembangun HTTP menetapkan `Mcp-Name` ke `params.taskId` untuk `tasks/get`, `tasks/update`, dan `tasks/cancel`.
- Fungsi bantuan notifikasi menggunakan `notifications/subscriptions/acknowledged` dan `notifications/tasks`, keduanya diberi tag id permintaan listen.
- Notifikasi tanpa id tidak menghasilkan respons JSON-RPC.

Pekerja maju secara eksplisit alih-alih tidur di thread latar belakang. Hal itu membuat setiap transisi state deterministik dan menjaga contoh protokol tetap terpisah dari mekanika antrean.

## Gunakanlah

Dari root repositori:

```bash
cd phases/13-tools-and-protocols/13-mcp-async-tasks/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Urutan hasil yang diharapkan:

```text
id=0 resultType=complete status=ack
id=1 resultType=task status=working
id=2 resultType=complete status=working
id=3 resultType=complete status=input_required
id=4 resultType=complete status=ack
id=5 resultType=complete status=completed
```

Verifikasi juga bahwa `tasks/status`, `tasks/result`, dan `tasks/list` mengembalikan method-not-found pada layanan modern.
Verifikasi bahwa `tools/list` deterministik dan setiap metode tugas HTTP saat ini mencerminkan id tugasnya melalui `Mcp-Name`.

## Terapkanlah

`outputs/skill-task-store-designer.md` kini menghasilkan desain yang memahami ekstensi: negosiasi kapabilitas, penciptaan persisten sebelum pengembalian, metode terkini, alur pembaruan masukan, kepemilikan, kedaluwarsa, pembatalan, langganan, dan migrasi dari metode eksperimental yang dihapus.

## Latihan

1. Tambahkan kunci masukan tertunda kedua. Kirim `tasks/update` parsial dan buktikan tugas tetap `input_required` hingga kedua kunci dijawab.
2. Tambahkan kepemilikan tenant ke penyimpanan dan tolak id tugas yang valid yang disajikan oleh prinsipal terautentikasi yang salah.
3. Tambahkan lease pekerja dengan kedaluwarsa. Tunjukkan bahwa dua instance layanan tidak dapat menyelesaikan tugas yang sama secara bersamaan.
4. Implementasikan adapter SSE respons POST untuk `subscriptions/listen`. Jangan tambahkan GET, `Last-Event-ID`, atau header sesi.
5. Tambahkan pembersihan kedaluwarsa. Bedakan tugas yang kedaluwarsa dari id tugas yang malformed tanpa membocorkan keberadaan lintas tenant.

## Istilah Kunci

| Istilah | Makna dalam ekstensi saat ini |
|--------|----------------------------------|
| Ekstensi Tasks | Kapabilitas `io.modelcontextprotocol/tasks` opsional untuk pekerjaan asinkron persisten |
| `CreateTaskResult` | Respons `resultType: "task"` yang diarahkan server untuk permintaan yang memenuhi syarat |
| `tasks/get` | Poll cuplikan tugas lengkap terkini, termasuk hasil terminal atau masukan yang tertunda |
| `tasks/update` | Kirim respons ke `inputRequests` tertunda dari sebuah tugas |
| `tasks/cancel` | Mengakui niat pembatalan kooperatif |
| `input_required` | Status tugas yang menandakan masukan klien masih tertunda |
| `pollIntervalMs` | Jeda minimal yang disarankan server sebelum polling berikutnya |
| `ttlMs` | Durasi kedaluwarsa yang diukur sejak penciptaan tugas |
| Persisten sebelum pengembalian | Aturan bahwa id tugas harus dapat diselesaikan sebelum handle-nya dikirim |
| `notifications/tasks` | Cuplikan tugas lengkap opsional yang dikirim pada respons SSE yang dilanggani |

## Kompatibilitas Warisan

Permukaan eksperimental 2025-11-25 menggunakan augmentasi tugas yang diminta klien, `tasks/status`, `tasks/result`, dan `tasks/list` opsional. Simpan nama-nama itu hanya di dalam adapter warisan yang terkunci. Klien saat ini menggunakan kapabilitas ekstensi, menerima handle yang diarahkan server, melakukan polling `tasks/get`, memasok masukan dengan `tasks/update`, dan membaca hasil akhir dari cuplikan tugas.

## Bacaan Lanjutan

- [Ekstensi Tasks MCP resmi](https://tasks.extensions.modelcontextprotocol.io/specification/draft/tasks)
- [Permintaan Multi Perjalanan Bolak-Balik MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [Streamable HTTP MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
