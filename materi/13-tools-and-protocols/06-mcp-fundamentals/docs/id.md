# Dasar-Dasar MCP: Permintaan Stateless dan JSON-RPC

> MCP modern tidak memiliki handshake dan tidak memiliki sesi protokol. Setiap permintaan harus membawa metadata yang cukup untuk dipahami, diotorisasi, diarahkan, dan dicoba ulang secara mandiri.

**Jenis:** Pelajari
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 01 sampai 05
**Waktu:** ~55 menit

## Tujuan Pembelajaran

- Membedakan primitif sisi server MCP dari fitur sisi kliennya.
- Menyusun permintaan dan respons JSON-RPC 2.0 yang valid untuk MCP `2026-07-28`.
- Melampirkan versi protokol, kapabilitas klien, dan identitas klien pada setiap permintaan.
- Menggunakan `server/discover` dan menangani `UnsupportedProtocolVersionError` tanpa handshake.
- Menelusuri satu permintaan mandiri dari validasi hingga hasil lengkap.

## Masalah

Server MCP dapat menerima dua permintaan berurutan dari klien berbeda, dengan kapabilitas berbeda, pada proses atau worker HTTP yang sama. Jika server mengingat apa yang dinyatakan permintaan sebelumnya, server dapat menerapkan izin yang salah atau mengembalikan bentuk kabel (wire shape) yang salah.

MCP `2026-07-28` menghilangkan ambiguitas itu. Inti protokol bersifat stateless. Server harus memutuskan cara menangani permintaan saat ini dari permintaan itu sendiri, bukan dari riwayat koneksi.

Ini mengubah model mental. Urutan lama adalah koneksi terlebih dahulu, handshake kedua, operasi ketiga. Urutan modern lebih sederhana:

1. Klien mengirim permintaan yang mendeskripsikan dirinya sendiri.
2. Server memvalidasi versi dan kapabilitas permintaan itu.
3. Server menangani metode tersebut.
4. Server mengembalikan hasil bertipe atau error JSON-RPC.

Permintaan berikutnya mengulang proses yang sama dari awal.

## Konsep

### Primitif server

Server MCP mengekspos tiga primitif utama:

1. **Tool** adalah tindakan yang dikendalikan model, ditemukan dengan `tools/list` dan dipanggil dengan `tools/call`.
2. **Resource** adalah data yang dialamati URI, ditemukan dengan `resources/list` dan diambil dengan `resources/read`.
3. **Prompt** adalah templat yang dapat digunakan kembali, ditemukan dengan `prompts/list` dan dirender dengan `prompts/get`.

Roots, sampling, dan logging tetap ada dalam skema `2026-07-28` untuk kompatibilitas, tetapi sudah tidak disarankan (deprecated). Implementasi baru harus menggunakan input tool atau resource eksplisit untuk roots, API penyedia model langsung untuk sampling, serta stderr atau OpenTelemetry untuk logging. Elicitation tetap tersedia melalui Multi Round-Trip Request, di mana server mengembalikan permintaan input dan klien mencoba ulang operasi awal. Server modern tidak pernah memulai permintaan JSON-RPC mandiri.

### Amplop JSON-RPC

MCP menggunakan JSON-RPC 2.0:

- Permintaan: `{jsonrpc, id, method, params}`
- Respons: `{jsonrpc, id, result}` atau `{jsonrpc, id, error}`
- Notifikasi: `{jsonrpc, method, params}` tanpa `id`

`id` permintaan mengorelasikan satu respons. `id` itu tidak menciptakan sesi protokol.

### Metadata permintaan yang wajib

Setiap permintaan modern membawa objek `_meta` di dalam `params`:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/list",
  "params": {
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

Versi protokol dan kapabilitas klien wajib. Identitas klien disarankan. Itu adalah data tampilan dan debugging yang dilaporkan sendiri, bukan kredensial keamanan.

Server tidak boleh menyimpulkan nilai-nilai itu dari permintaan sebelumnya, proses stdio, koneksi HTTP, atau header transpor saja.

### Hasil lengkap dan identitas server

Setiap hasil modern yang berhasil menyertakan `resultType`. Hasil akhir normal menggunakan `"complete"`. Server sebaiknya juga mengidentifikasi dirinya dalam metadata hasil:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "resultType": "complete",
    "tools": [],
    "ttlMs": 30000,
    "cacheScope": "public",
    "_meta": {
      "io.modelcontextprotocol/serverInfo": {
        "name": "notes-server",
        "version": "1.0.0"
      }
    }
  }
}
```

`tools/list`, `resources/list`, `prompts/list`, `resources/templates/list`, `resources/read`, dan `server/discover` adalah hasil yang dapat ditembolok (cacheable). Hasil-hasil itu menyertakan `ttlMs` dan `cacheScope`. Default yang aman adalah `ttlMs: 0` dan `cacheScope: "private"`. Item daftar sebaiknya memiliki urutan deterministik agar respons yang setara menghasilkan kunci cache yang stabil dan konteks model yang stabil.

### Penemuan tanpa handshake

Setiap server modern harus mengimplementasikan `server/discover`. Klien dapat memanggilnya sebelum metode lain untuk mengambil:

- `supportedVersions`
- `capabilities` server
- `instructions` penggunaan opsional
- identitas server di `_meta` hasil
- petunjuk cache

Penemuan itu berguna, tetapi bukan gerbang. Klien dapat mengirim `tools/list` lebih dulu karena permintaan itu sudah membawa versi protokol dan kapabilitasnya.

Jika versi yang diminta tidak didukung, server mengembalikan kode JSON-RPC `-32022` dengan:

```json
{
  "requested": "2027-01-01",
  "supported": ["2026-07-28"]
}
```

Klien memilih versi modern yang didukung bersama lalu mencoba ulang dengan id permintaan JSON-RPC yang baru.

### Siklus hidup satu permintaan

Telusuri permintaan modern dalam urutan ini:

1. Urai satu amplop JSON-RPC.
2. Pastikan `jsonrpc` adalah `"2.0"`, `id` ada, `method` adalah string, dan `params` adalah objek.
3. Wajibkan string versi dan objek kapabilitas di `params._meta`; metadata yang cacat atau hilang adalah `-32602`.
4. Pada batas HTTP, bandingkan versi, metode, dan header nama yang berlaku dengan isi pesan. Ketidaksesuaian adalah `-32020` meskipun salah satu dari kedua nilai versi itu tidak didukung.
5. Setelah kesetaraan terbukti, tolak versi yang cocok tetapi tidak didukung dengan `-32022`.
6. Periksa kapabilitas yang wajib, lalu arahkan berdasarkan `method` dan validasi argumen khusus metode.
7. Autentikasi dan otorisasi operasi konkret sebelum handler-nya berjalan.
8. Kembalikan hasil lengkap dengan identitas server.
9. Lupakan metadata protokol yang hanya berlaku untuk permintaan itu.

Urutan ini mencegah dua komponen menafsirkan panggilan yang berbeda. Sebuah gateway tidak boleh mengotorisasi `Mcp-Name: notes.read` sementara asal mengeksekusi `params.name: notes.delete`. Urutan ini juga menjaga agar masukan cacat, kebingungan header, negosiasi versi, kegagalan kapabilitas, otorisasi, dan kegagalan handler tetap menjadi bukti yang terpisah.

Menutup stdin atau respons HTTP mengakhiri aktivitas transpor. Hal itu tidak mengakhiri sesi protokol karena MCP modern tidak memiliki sesi protokol.

### Kompatibilitas legacy yang eksplisit

Versi hingga `2025-11-25` menggunakan `initialize`, `notifications/initialized`, kapabilitas yang berlaku pada koneksi, dan, pada Streamable HTTP yang lebih awal, sesi protokol opsional. Perilaku itu masih relevan ketika klien dari dua era berbicara ke server lama.

Jaga agar kedua era tetap terpisah. Permintaan modern dikenali dari metadata wajib per permintaan. Koneksi legacy hanya dipilih melalui jalur fallback yang terdokumentasi. Jangan mengirim `initialize` sebagai default untuk server `2026-07-28`.

Dengan demikian, "stateless" memiliki arti yang spesifik per era. Dalam `2026-07-28`, itu adalah invarian protokol: setiap permintaan biasa dapat ditafsirkan secara mandiri dan tidak ada sesi MCP. Pada versi hingga `2025-11-25`, inisialisasi dan kapabilitas yang dinegosiasi melekat pada koneksi, sehingga adapter kompatibilitas boleh menyimpan status koneksi legacy itu. Implementasi dua era bukanlah satu mesin status yang permisif. Implementasi itu adalah inti modern yang stateless di samping adapter legacy yang terisolasi, dengan keputusan pemilihan yang eksplisit sebelum parser mana pun berjalan.

Kedua makna itu tidak melarang status aplikasi yang awet. Alur kerja, tugas, atau draf dapat hidup di balik handle opak dalam penyimpanan bersama. Klien mengirim handle itu sebagai masukan biasa, dan setiap replika mengautentikasi serta mengotorisasi penggunaannya. Konteks protokol tidak boleh bocor ke penyimpanan itu sebagai pengganti sesi yang telah dihapus.

```figure
mcp-tool-call
```

## Gunakan

`code/main.py` menyusun, memvalidasi, menelusuri, dan mendispatches pesan MCP modern tanpa kerangka kerja. Jalankan:

```bash
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover code/tests -v
```

Perhatikan tiga invarian pada keluaran:

- Setiap permintaan mengulang field `_meta`-nya.
- Setiap hasil yang berhasil adalah `resultType: "complete"` dan menyertakan identitas server.
- Hasil daftar diurutkan secara deterministik dan memiliki petunjuk cache yang eksplisit.

## Rilis

Pelajaran ini merilis `outputs/skill-mcp-handshake-tracer.md`. Nama file historis tetap stabil, tetapi artefaknya sekarang adalah pelacak permintaan stateless. Artefak itu mengaudit setiap pesan secara mandiri dan hanya memberi label lalu lintas handshake legacy ketika memang benar-benar ada.

## Latihan

1. Ubah versi protokol satu permintaan menjadi `2027-01-01`. Konfirmasi kode error-nya `-32022` dan datanya mengiklankan versi yang didukung.
2. Hapus `io.modelcontextprotocol/clientCapabilities` dari permintaan kedua. Konfirmasi server tidak menggunakan ulang kapabilitas dari permintaan pertama.
3. Balik registri alat dalam memori. Konfirmasi `tools/list` tetap mengembalikan urutan deterministik yang sama.
4. Ubah `cacheScope` dari `public` menjadi `private`. Jelaskan konteks otorisasi mana yang boleh menggunakan ulang respons pada masing-masing kasus.
5. Tambahkan pengujian opsional penghilangan `clientInfo`. Permintaan harus tetap valid karena identitas klien disarankan, bukan wajib.

## Istilah Kunci

| Istilah | Arti |
|---------|------|
| Protokol stateless | Setiap permintaan menyediakan metadata yang diperlukan untuk menafsirkannya |
| Metadata permintaan | Versi, kapabilitas klien, dan identitas klien yang disarankan di `params._meta` |
| `server/discover` | Metode server wajib untuk versi, kapabilitas, instruksi, dan identitas |
| `resultType` | Diskriminator pada setiap hasil modern yang berhasil |
| Hasil yang dapat ditembolok | Hasil yang menyertakan petunjuk wajib `ttlMs` dan `cacheScope` |
| Era protokol | Metadata modern per permintaan atau inisialisasi legacy yang berlaku pada koneksi |
| Masa hidup transpor | Masa hidup proses, koneksi, atau aliran respons, bukan status sesi protokol |
| `-32022` | Error versi protokol tidak didukung dengan versi yang diminta dan yang didukung |

## Bacaan Lanjutan

- [Arsitektur MCP](https://modelcontextprotocol.io/specification/2026-07-28/architecture)
- [Protokol Dasar MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic)
- [Penemuan Server MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [Catatan Perubahan MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
