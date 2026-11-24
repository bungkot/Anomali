# Membangun Server MCP: Python dan TypeScript Stateless

> Server MCP modern tidak mengingat handshake. Server itu memvalidasi metadata pada setiap permintaan, menjalankan satu handler, dan mengembalikan satu hasil bertipe.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 06
**Waktu:** ~85 menit

## Tujuan Pembelajaran

- Mengimplementasikan `server/discover` wajib untuk MCP `2026-07-28`.
- Memvalidasi versi protokol dan kapabilitas klien pada setiap permintaan.
- Mengekspos tool, resource, dan prompt dengan urutan daftar deterministik.
- Mengembalikan `resultType`, identitas server, dan petunjuk cache pada hasil yang tepat.
- Menyajikan kontrak stateless yang sama lewat stdio yang dipisahkan newline di Python dan TypeScript.

## Masalah

Server yang menyimpan kapabilitas klien setelah pesan pertama mudah dibangun tetapi sulit dioperasikan. Proses yang sama dapat melayani klien berurutan. Permintaan jarak jauh dapat mendarat di worker yang berbeda. Deklarasi kapabilitas yang basi dapat membocorkan perilaku melintasi batas otorisasi.

MCP `2026-07-28` menyelesaikan bagian protokol dari masalah itu dengan membuat setiap permintaan mendeskripsikan dirinya sendiri. Aplikasi Anda tetap dapat menyimpan catatan, pekerjaan, atau handle status eksplisit yang awet. Yang tidak boleh disimpannya adalah status protokol tersembunyi yang mengubah cara permintaan berikutnya diterjemahkan.

Pelajaran ini membangun server catatan dua kali. Versi Python dan TypeScript hanya menggunakan pustaka standar mereka untuk inti protokol. Keduanya mengekspos metode yang sama dan menegakkan kontrak kabel yang sama.

## Konsep

### Loop dispatch modern

```text
baca satu baris JSON-RPC
urai amplop
jika notifikasi, jangan balas
validasi params._meta untuk permintaan ini
arahkan berdasarkan method
bungkus keberhasilan dengan resultType dan serverInfo
tulis satu baris respons JSON-RPC
lupakan metadata yang hanya berlaku untuk permintaan ini
```

Tiga aturan stdio tetap penting:

- Tulis hanya pesan JSON-RPC ke stdout. Kirim diagnostik ke stderr.
- Batasi pesan dengan newline dan flush setiap respons.
- Keluar segera ketika stdin mencapai EOF.

Masa hidup proses adalah masa hidup transpor. Itu bukan sesi MCP modern.

### Validasi permintaan

Setiap permintaan harus memiliki:

```json
{
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "notes-client",
        "version": "1.0.0"
      }
    }
  }
}
```

Dua field pertama wajib. `clientInfo` disarankan. Validasi bentuk identitas yang hadir, tetapi jangan memperlakukannya sebagai autentikasi.

Jika versi tidak didukung, kembalikan kode `-32022` dengan `requested` dan `supported`. Metadata permintaan yang hilang adalah params tidak valid, kode `-32602`. Jangan pernah mengisi field yang hilang dari panggilan sebelumnya.

### Penemuan wajib

Server modern harus mengimplementasikan `server/discover`. Hasil penemuan yang lengkap mencakup versi modern yang didukung, kapabilitas, instruksi opsional, petunjuk cache, dan identitas server di `_meta` hasil:

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": {
    "tools": {"listChanged": false},
    "resources": {"listChanged": false, "subscribe": false},
    "prompts": {"listChanged": false}
  },
  "ttlMs": 3600000,
  "cacheScope": "public",
  "_meta": {
    "io.modelcontextprotocol/serverInfo": {
      "name": "notes-server",
      "version": "2.0.0"
    }
  }
}
```

Penemuan tidak membuka kunci server. Klien dapat memanggil `tools/list` tanpa memanggil penemuan karena `tools/list` sudah membawa metadata permintaan yang sama.

### Tool

`tools/list` mengembalikan daftar deskriptor tool yang deterministik. Urutan yang stabil meningkatkan cache respons dan menjaga konteks model tetap stabil. Hasilnya juga memerlukan `ttlMs` dan `cacheScope`.

`tools/call` mengembalikan blok konten dan `isError`. Gunakan error JSON-RPC ketika amplop protokol atau parameter metode tidak valid. Gunakan `isError: true` ketika pemanggilan tool yang valid berjalan tetapi tool itu sendiri gagal.

Anotasi tool tetap berupa petunjuk, bukan penegakan:

- `readOnlyHint`
- `destructiveHint`
- `idempotentHint`
- `openWorldHint`

Host sebaiknya menggunakannya untuk konfirmasi dan penyajian. Server tetap harus menegakkan otorisasi sungguhan.

### Resource

`resources/list` mengembalikan deskriptor URI yang stabil. `resources/read` mengembalikan konten bertipe. Keduanya dapat ditembolok dalam `2026-07-28`, jadi keduanya menyertakan `ttlMs` dan `cacheScope`.

Gunakan `cacheScope: "private"` untuk data catatan khusus pengguna. Cache bersama tidak boleh menggunakan ulang respons privat melintasi konteks otorisasi.

Pengiriman perubahan modern tidak menggunakan `resources/subscribe`. Klien membuka `subscriptions/listen` dan meminta kategori `resourceSubscriptions` atau perubahan daftar. Pelajaran 10 membangun alur itu.

### Prompt

`prompts/list` dapat ditembolok dan deterministik. `prompts/get` merender prompt bernama dengan argumen. Hasil prompt yang dirender bersifat lengkap, tetapi bukan salah satu hasil daftar atau baca yang dapat ditembolok sehingga mewajibkan petunjuk cache.

### Setiap hasil yang berhasil bertipe

Contoh-contoh menggunakan satu pembungkus untuk setiap keberhasilan:

```python
def complete(payload):
    return {
        "resultType": "complete",
        **payload,
        "_meta": {SERVER_INFO_KEY: SERVER_INFO},
    }
```

Handler daftar, baca, dan penemuan menambahkan `ttlMs` plus `cacheScope`. Memusatkan pembungkus ini mencegah satu handler diam-diam menghilangkan field hasil modern.

### Tanpa permintaan yang dimulai server

Server modern boleh mengirim notifikasi yang terkait permintaan klien, atau notifikasi pada aliran `subscriptions/listen` yang dibuka klien. Server tidak boleh mengirim permintaan JSON-RPC sendiri.

Ketika sebuah handler memerlukan sampling, elicitation, atau masukan roots, handler mengembalikan hasil `input_required`. Klien memenuhi permintaan input yang tertanam lalu mencoba ulang metode awal dengan id permintaan baru. Pelajaran 11 membahas pola Multi Round-Trip Request itu.

### Kompatibilitas legacy yang eksplisit

Server dua era juga dapat mengimplementasikan handshake `2025-11-25` pada cabang legacy yang jelas terpisah. Server memilih perilaku modern ketika field `_meta` modern yang wajib hadir dan perilaku legacy ketika menerima `initialize`.

Jangan memasukkan permintaan `2026-07-28` ke jalur handshake legacy. Jangan menempelkan field `resultType` modern pada hasil inisialisasi legacy. Kode dalam pelajaran ini sengaja hanya modern agar invarian tetap terlihat.

```figure
t3-dispatch-loop
```

## Gunakan

Jalankan demo terbatas dan pengujian server Python:

```bash
cd code
python3 main.py --demo
python3 -m unittest discover tests -v
```

Jalankan port TypeScript dengan runner TypeScript:

```bash
npx tsx main.ts --demo
```

Demo mengirim `server/discover`, mencantumkan setiap primitif, memanggil tool, dan menampilkan error versi tidak didukung. Setiap permintaan modern mengulang metadata. Setiap keberhasilan menyertakan identitas server.

## Rilis

Pelajaran ini merilis `outputs/skill-mcp-server-scaffolder.md`. Artefak itu menghasilkan rencana server modern dengan kontrak penemuan, validasi per permintaan, daftar dapat ditembolok yang deterministik, dan adapter legacy opsional yang terisolasi.

## Latihan

1. Hapus kapabilitas dari satu permintaan dan buktikan server tidak menggunakan ulang deklarasi permintaan sebelumnya.
2. Balik urutan penyisipan `TOOLS`, `PROMPTS`, dan catatan. Konfirmasi semua hasil daftar tetap stabil.
3. Tambahkan tool `notes_delete` yang destruktif dan wajibkan pemeriksaan otorisasi di dalam executor. Pertahankan `destructiveHint` hanya sebagai petunjuk UX.
4. Tambahkan `resources/templates/list` dengan `ttlMs`, `cacheScope`, dan urutan deterministik.
5. Bangun adapter legacy terpisah untuk `2025-11-25`. Tambahkan pengujian yang membuktikan permintaan modern tidak pernah masuk ke sana.

## Istilah Kunci

| Istilah | Arti |
|---------|------|
| Server stateless | Menangani setiap permintaan dari metadatanya sendiri tanpa memori sesi protokol |
| `server/discover` | Metode modern wajib yang mengiklankan versi dan kapabilitas |
| Hasil lengkap | Hasil modern yang berhasil dengan `resultType: "complete"` |
| Hasil dapat ditembolok | Hasil penemuan, daftar, atau baca resource dengan `ttlMs` dan `cacheScope` |
| Daftar deterministik | Registri logis yang sama menghasilkan urutan item yang sama |
| Identitas server | `io.modelcontextprotocol/serverInfo` yang disarankan di `_meta` hasil |
| Error tool | Panggilan tool valid yang mengembalikan konten dengan `isError: true` |
| Error protokol | Permintaan JSON-RPC atau MCP tidak valid yang dikembalikan lewat `error` |

## Bacaan Lanjutan

- [Spesifikasi MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/)
- [Penemuan Server MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [Tool MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
- [Resource MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/resources)
- [Prompt MCP](https://modelcontextprotocol.io/specification/2026-07-28/server/prompts)
- [Transpor stdio MCP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
