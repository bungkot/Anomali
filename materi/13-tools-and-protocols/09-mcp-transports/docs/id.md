# Transpor MCP: stdio dan Streamable HTTP Stateless

> Transpor membawa pesan MCP. Transpor tidak menyediakan keadaan protokol yang hilang. Dalam `2026-07-28`, stdio lokal maupun Streamable HTTP jarak jauh sama-sama membawa permintaan yang menjelaskan dirinya sendiri.

**Jenis:** Belajar
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 07 dan 08
**Waktu:** ~65 menit

## Tujuan Pembelajaran

- Pilih stdio untuk proses anak lokal dan Streamable HTTP untuk layanan jaringan.
- Terapkan kontrak Streamable HTTP modern satu endpoint, khusus POST.
- Cerminkan dan validasi header versi, metode, dan nama MCP terhadap badan JSON-RPC.
- Sampaikan SSE cakupan permintaan dan aliran `subscriptions/listen` berumur panjang dengan benar.
- Migrasikan deployment berbasis sesi dan HTTP+SSE lama tanpa menyajikan perilaku lama seolah-olah modern.

## Masalahnya

Revisi Streamable HTTP sebelumnya menggabungkan negosiasi protokol dengan perilaku koneksi dan sesi. Server dapat menerbitkan `Mcp-Session-Id`, mengekspos aliran GET mandiri, menerima DELETE untuk mengakhiri sesi, dan melanjutkan SSE dengan `Last-Event-ID`.

MCP `2026-07-28` menghapus mekanisme tersebut dari kabel modern. Setiap permintaan dapat mendarat di pekerja yang sehat mana pun karena versi protokol dan kapabilitas klien ikut dalam badan permintaan. Header HTTP mencerminkan beberapa kolom untuk perutean dan kebijakan, tetapi server memvalidasi header tersebut terhadap badan sebelum mengeksekusi.

Hasilnya lebih mudah diskalakan dan lebih mudah dipahami. Ini juga berarti bahwa server yang mengajarkan transpor 2025 seolah-olah sebagai transpor terkini sedang mengajarkan model kegagalan dan model keamanan yang salah.

## Konsepnya

### stdio

Binding stdio ditujukan untuk proses bawahan yang diluncurkan klien:

- Klien menulis satu pesan JSON-RPC UTF-8 per baris ke stdin.
- Server menulis satu pesan JSON-RPC UTF-8 per baris ke stdout.
- Server menulis diagnostik ke stderr.
- Server keluar dengan segera saat stdin mencapai EOF.
- Setiap permintaan modern membawa versi dan kapabilitas klien di `params._meta`.

Proses dapat bertahan untuk banyak panggilan, tetapi itu bukan sesi protokol modern. Jika proses keluar secara tak terduga, permintaan yang sedang berjalan hilang. Mulai ulang proses, temukan ulang, daftarkan ulang, buka kembali langganan, dan coba ulang operasi yang aman dengan id permintaan baru.

### Streamable HTTP dalam 2026-07-28

Server modern mengekspos satu endpoint MCP, misalnya `/mcp`, yang menerima POST.

Setiap permintaan atau notifikasi JSON-RPC adalah POST HTTP baru. Badan memuat satu pesan JSON-RPC. Klien tidak mengirim respons JSON-RPC ke server.

Untuk sebuah permintaan, server mengembalikan salah satu dari:

- `Content-Type: application/json` dengan satu respons JSON-RPC; atau
- `Content-Type: text/event-stream` dengan notifikasi yang terkait dengan permintaan itu, diikuti respons JSON-RPC akhir.

Untuk notifikasi yang diterima, server mengembalikan `202 Accepted` tanpa badan.

Klien mengiklankan kedua jenis respons:

```http
Accept: application/json, text/event-stream
```

### POST-only berarti POST-only

Streamable HTTP modern tidak memiliki aliran GET mandiri dan tidak memiliki endpoint DELETE sesi.

- `GET /mcp` mengembalikan `405 Method Not Allowed`.
- `DELETE /mcp` mengembalikan `405 Method Not Allowed`.
- `Mcp-Session-Id` diabaikan dan tidak pernah diterbitkan maupun digaungkan kembali.
- `Last-Event-ID` diabaikan karena aliran modern tidak dapat dilanjutkan kembali.

Jika aliran cakupan permintaan putus sebelum respons akhirnya, klien kehilangan permintaan yang sedang berjalan itu. Klien dapat mengeluarkan permintaan baru dengan id JSON-RPC baru ketika mencoba ulang aman. Klien tidak boleh mencoba melanjutkan aliran kembali.

### Validasi Origin

Server memvalidasi `Origin` pada koneksi masuk untuk mencegah DNS rebinding (*pemindahan ulang DNS*). Jika header ada dan tidak secara eksplisit diizinkan, kembalikan `403 Forbidden`. Klien non-browser dapat mengabaikan `Origin`, yang diperbolehkan oleh aturan transpor resmi.

Server lokal sebaiknya terikat pada `127.0.0.1`, bukan semua antarmuka. Layanan jaringan tetap memerlukan autentikasi dan otorisasi pada setiap permintaan. Validasi Origin bukanlah autentikasi.

Gunakan pencocokan Origin persis setelah konfigurasi kanonik. Pemeriksaan awalan seperti `origin.startswith("https://trusted.example")` tidak aman karena dapat menerima akhiran yang dikendalikan penyerang.

### Header metadata HTTP yang wajib

Setiap permintaan POST modern mencakup:

```http
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: notes_search
```

Aturan header:

- `MCP-Protocol-Version` wajib dan harus sama dengan `params._meta.io.modelcontextprotocol/protocolVersion`.
- `Mcp-Method` wajib dan harus sama dengan `method` JSON-RPC.
- `Mcp-Name` wajib untuk `tools/call`, `resources/read`, dan `prompts/get`.
- `Mcp-Name` sama dengan `params.name`, atau `params.uri` untuk `resources/read`.
- Nilai header bersifat case-sensitive meskipun nama header bersifat case-insensitive.

Nilai `Mcp-Name` yang tidak aman atau bukan ASCII menggunakan penanda Base64 persis:

```text
=?base64?{Base64EncodedValue}?=
```

Server mendekode nilai itu sebelum membandingkannya dengan badan.

Header yang dicerminkan dan hilang, malformed, atau tidak cocok mengembalikan HTTP `400` dengan kode JSON-RPC `-32020`. Jika header dan badan sepakat pada versi yang tidak didukung server, kembalikan HTTP `400` dengan `-32022` dan data kesalahan persis seperti `{"supported":["2026-07-28"],"requested":"2027-01-01"}`.

Metode modern yang tidak dikenal mengembalikan HTTP `404` dengan JSON-RPC `-32601`. Badan JSON-RPC penting karena klien dua era menggunakannya untuk membedakan kesalahan modern dari endpoint lama yang tidak ditemukan.

### SSE cakupan permintaan

Server dapat memilih SSE untuk satu permintaan yang berjalan lama:

```text
POST tools/call id=41
  <- notifications/progress terkait id=41
  <- notifications/progress terkait id=41
  <- respons JSON-RPC id=41
aliran ditutup
```

Server tidak boleh mengirim permintaan JSON-RPC independen di aliran ini. Interaksi sampling, elicitation, dan roots menggunakan hasil Permintaan Multi Pulang-Pergi (MRTR). Menutup aliran respons membatalkan permintaan itu.

Jangan tambahkan id event SSE untuk pemutaran ulang. Pemulihan `Last-Event-ID` bukan bagian dari revisi modern.

### Perubahan berumur panjang menggunakan subscriptions/listen

Notifikasi perubahan menggunakan permintaan yang dibuka klien, bukan GET mandiri:

```json
{
  "jsonrpc": "2.0",
  "id": "listen-1",
  "method": "subscriptions/listen",
  "params": {
    "notifications": {
      "toolsListChanged": true,
      "resourceSubscriptions": ["notes://note-1"]
    },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "course-client",
        "version": "1.0.0"
      }
    }
  }
}
```

Respons POST adalah aliran SSE berumur panjang. Pesan protokol pertamanya adalah `notifications/subscriptions/acknowledged`. Pengakuan, setiap notifikasi perubahan, dan hasil akhir sama-sama membawa `io.modelcontextprotocol/subscriptionId` di `_meta`, yang sama dengan id permintaan listen. Server boleh memancarkan komentar SSE sebagai keepalive. Ketika aliran terputus, klien mengeluarkan ulang `subscriptions/listen` dengan id permintaan baru dan mengambil ulang data yang terdampak.

`resources/subscribe` dan `resources/unsubscribe` termasuk era lama. Jangan gunakan keduanya pada koneksi modern.

### Keadaan aplikasi yang eksplisit

Menghapus sesi protokol tidak melarang alur kerja berkeadaan. Server boleh menerbitkan pegangan (*handle*) keadaan yang opak dan mengembalikannya sebagai hasil alat biasa. Klien meneruskan pegangan itu sebagai argumen eksplisit pada panggilan berikutnya.

Ikat pegangan pada prinsipal terautentikasi, buat agar tidak dapat ditebak, beri masa kedaluwarsa, dan otorisasi setiap penggunaannya. Ini membuat keadaan terlihat di lapisan aplikasi, bukan tersembunyi dalam afinitas transpor.

Kegagalan akibat keadaan replika yang tersembunyi bersifat mekanis:

1. Permintaan A mencapai replika 1 dan membuat draf dalam memori proses itu.
2. Respons tidak mengembalikan pegangan draf karena implementasi menganggap koneksi sudah mengidentifikasi draf itu.
3. Permintaan B adalah POST baru dan mencapai replika 2.
4. Replika 2 memiliki metadata protokol yang valid tetapi tidak ada cara untuk menyebut atau memuat draf itu, sehingga alur kerja gagal atau membaca objek lokal yang salah.
5. Perutean sticky tampaknya memperbaiki gejalanya sampai restart, rollout, penjadwalan ulang, atau failover memindahkan permintaan berikutnya.

Batas yang benar memiliki dua bagian. Konteks protokol tetap berada di setiap permintaan. Keadaan aplikasi yang tahan lama hidup di penyimpanan bersama di bawah pegangan yang diterbitkan server dan dikembalikan ke klien. Panggilan berikutnya menyediakan pegangan itu, replika mana pun memuat catatan yang sama, dan otorisasi mengikat catatan pada prinsipal serta tenant terautentikasi. Memori replika boleh menembolok catatan, tetapi tidak boleh menjadi satu-satunya salinan yang diperlukan untuk kebenaran.

Pilih mekanisme keadaan berdasarkan umur hidupnya. Variabel lokal permintaan cukup untuk satu panggilan. Kelanjutan MRTR singkat dapat memakai `requestState` yang dilindungi integritas. Draf atau tugas tahan lama memerlukan pegangan eksplisit plus persistensi bersama, masa kedaluwarsa, kontrol konkurensi, dan idempotensi. Tidak ada satu pun objek itu yang merupakan sesi protokol MCP.

### Kompatibilitas dua era atas HTTP

Klien yang mendukung server modern dan lama mencoba POST modern terlebih dahulu. Jika menerima HTTP `400`, `404`, atau `405`, klien memeriksa badannya:

- Kesalahan JSON-RPC modern yang dikenali membuktikan server itu modern. Perbaiki permintaan atau coba ulang versi yang diiklankan. Jangan menurunkan versi.
- Badan kosong atau respons yang tidak dikenal mungkin menandakan server HTTP+SSE lama. Hanya saat itulah coba endpoint GET lama dan harapkan event `endpoint` lamanya.

Server dapat mendukung kedua era selama migrasi dengan merutekan metadata modern ke implementasi POST-only modern dan mempertahankan endpoint lama terpisah untuk klien lama. Jangan pernah menggambarkan GET, DELETE, id sesi, atau perilaku replay lama sebagai bagian dari `2026-07-28`.

```figure
tp-transport-handshake
```

## Gunakan

`code/main.py` mengimplementasikan server Streamable HTTP modern yang terbatas dengan pustaka standar Python. Server itu memvalidasi Origin dan header yang dicerminkan, mengabaikan header sesi yang dihapus, mengembalikan JSON untuk panggilan biasa, dan mendemonstrasikan aliran SSE `subscriptions/listen` yang terbatas.

```bash
cd code
python3 main.py --probe
npx tsx main.ts --probe
python3 -m unittest discover tests -v
```

Probe memeriksa:

- Origin tidak valid ditolak;
- penemuan berhasil tanpa id sesi;
- `Mcp-Session-Id` dan `Last-Event-ID` diabaikan;
- ketidaksesuaian header mengembalikan `-32020`;
- versi yang tidak didukung mengembalikan `-32022` dengan data `supported` dan `requested` yang persis;
- notifikasi tanpa id yang diterima mengembalikan HTTP `202` tanpa badan;
- GET dan DELETE mengembalikan `405`;
- `subscriptions/listen` adalah aliran respons POST yang pengakuan, notifikasi, dan hasil akhirnya membawa id langganannya.

## Kirim

Pelajaran ini mengirimkan `outputs/skill-mcp-transport-migrator.md`. Dokumen itu menghapus sesi protokol modern, menambahkan validasi header-badan, mengganti GET mandiri dengan `subscriptions/listen`, dan menjaga jembatan lama apa pun tetap terlihat terpisah.

## Latihan

1. Hapus `Mcp-Method` dari sebuah POST. Konfirmasi HTTP `400` dan kesalahan `-32020`.
2. Kirim versi header dan badan yang cocok, `2027-01-01`. Konfirmasi HTTP `400`, kesalahan `-32022`, dan data persis `{"supported":["2026-07-28"],"requested":"2027-01-01"}`.
3. Kirim penanda `Mcp-Name` Base64 untuk URI sumber daya bukan ASCII. Konfirmasi nilai hasil dekode dibandingkan dengan `params.uri`.
4. Putuskan aliran listen terbatas sebelum respons akhirnya. Keluarkan ulang dengan id JSON-RPC baru dan ambil ulang daftar alat.
5. Tambahkan pegangan alur kerja eksplisit pada alat ping. Ikat pegangan itu pada subjek otorisasi tanpa memakai afinitas koneksi.

## Istilah Kunci

| Istilah | Arti |
|---------|------|
| stdio | JSON-RPC dipisah baris baru atas proses bawahan yang diluncurkan klien |
| Streamable HTTP | Satu endpoint tempat setiap pesan modern adalah POST baru |
| SSE cakupan permintaan | Aliran respons POST yang memuat notifikasi terkait dan respons akhir |
| `subscriptions/listen` | Permintaan POST berumur panjang untuk notifikasi perubahan yang diikutsertakan |
| Ketidaksesuaian header | HTTP `400` dan JSON-RPC `-32020` ketika header yang dicerminkan tidak cocok dengan badan |
| Validasi Origin | Pertahanan terhadap DNS rebinding untuk koneksi masuk, bukan autentikasi |
| Pegangan keadaan eksplisit | Token aplikasi yang diteruskan sebagai argumen biasa, bukan keadaan sesi tersembunyi |
| Jembatan lama | Perilaku era sebelumnya yang terpisah dan hanya disimpan untuk kompatibilitas |

## Bacaan Lanjutan

- [Ringkasan Transpor MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)
- [Transpor stdio MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [Streamable HTTP MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [Langganan MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions)
- [Catatan Perubahan MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
