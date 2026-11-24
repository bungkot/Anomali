# Input Model MCP: Migrasi Sampling dan MRTR Stateless

> MCP 2026-07-28 tidak lagi menyarankan Sampling untuk desain baru dan menghapus kanal permintaan server-ke-klien. Jika alur kerja yang ada masih memerlukan model klien, server mengembalikan hasil `input_required` dan klien mencoba ulang permintaan awal dengan keluaran model itu. Loop penalaran menjadi eksplisit, terbatas, dan stateless di lapisan protokol.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 07 (server MCP), Fase 13 · 10 (sumber daya dan prompt)
**Waktu:** ~75 menit

## Tujuan Pembelajaran

- Menjelaskan mengapa Sampling tidak lagi disarankan dalam MCP 2026-07-28 dan memilih integrasi model langsung sebagai bawaan untuk server baru.
- Mengimplementasikan alur kerja kompatibilitas yang membawa `sampling/createMessage` melalui Permintaan Multi Pulang-Pergi (MRTR).
- Menempatkan revisi protokol dan kapabilitas klien di setiap objek `_meta` permintaan.
- Mengembalikan `resultType: "input_required"` dan mencoba ulang metode awal dengan id JSON-RPC baru.
- Melindungi integritas `requestState` dan mengikatnya pada prinsipal, metode, argumen, dan masa kedaluwarsa.
- Membatasi loop berbantuan model dengan pemeriksaan kapabilitas, persetujuan, validasi respons, dan batas putaran.

## Keputusan Sebelum Protokol

Alat seperti `summarize_repo` memerlukan dua jenis pekerjaan:

1. Pekerjaan deterministik: mencantumkan berkas, membaca berkas yang diizinkan, memvalidasi jalur, dan merakit konten.
2. Pekerjaan model: memilih berkas perwakilan dan menyusun ringkasan.

Kini Anda memiliki dua arsitektur yang sama-sama valid.

### Server baru: berintegrasi langsung dengan penyedia model

Ini adalah pilihan bawaan terkini. Server memiliki pemilihan model, kredensial, anggaran, percobaan ulang, dan observabilitas. Server mengembalikan satu hasil `tools/call` biasa kepada klien MCP.

Pilih ini ketika server sudah menjadi layanan terhosting atau ketika perilaku model yang dapat diprediksi lebih penting daripada memakai model host.

### Alur kerja Sampling yang ada: migrasikan ke MRTR

Sampling masih ada selama jendela penghentian penggunaannya. Server yang menargetkan 2026-07-28 tidak dapat mengirim permintaan `sampling/createMessage` langsung balik ke klien. Server menyematkan permintaan itu di dalam `InputRequiredResult`.

Pilih jalur kompatibilitas ini hanya ketika memakai model dan kredensial klien benar-benar merupakan kebutuhan produk. Catat rencana penghapusan karena implementasi baru sebaiknya tidak mengadopsi Sampling yang sudah tidak disarankan.

## Kontrak Stateless

Protokol Juli 2026 tidak memiliki pertukaran `initialize`, tidak ada `notifications/initialized`, dan tidak ada `Mcp-Session-Id`. Setiap permintaan membawa informasi yang dulu hidup di jabat tangan:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "summarize_repo",
    "arguments": {"audience": "developer"},
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {"sampling": {}},
      "io.modelcontextprotocol/clientInfo": {
        "name": "lesson-client",
        "version": "1.0.0"
      }
    }
  }
}
```

Server memvalidasi revisi pada setiap permintaan. Versi yang hilang atau bukan string adalah parameter tidak valid, `-32602`. String yang tidak didukung mengembalikan `-32022` dengan data persis `{"supported":["2026-07-28"],"requested":"<client version>"}`. Kapabilitas Sampling yang hilang mengembalikan `-32021` dengan `data.requiredCapabilities` ditetapkan ke `{"sampling":{}}`.

Amplop tanpa `id` JSON-RPC adalah notifikasi. Penerima boleh memprosesnya, tetapi tidak mengeluarkan respons berhasil maupun respons kesalahan. Adapter Streamable HTTP mengembalikan `202 Accepted` tanpa badan untuk notifikasi yang diterima.

Server juga mengimplementasikan `server/discover` dengan kunci `supportedVersions` yang persis, kapabilitas, `ttlMs`, dan `cacheScope` agar klien dapat mempelajari dan menembolokkan kontrak server sebelum memanggil alat. Karena penemuan mengiklankan `tools`, server juga mengimplementasikan `tools/list` yang wajib. Descriptor `summarize_repo` yang deterministik mencakup `inputSchema` objek yang valid, `resultType: "complete"`, metadata identitas server, dan petunjuk tembolok publik.

Setiap hasil modern yang berhasil memiliki diskriminator:

- `resultType: "complete"` berarti operasi selesai.
- `resultType: "input_required"` berarti klien harus memenuhi permintaan yang disematkan dan mencoba ulang.
- Ekstensi dapat menentukan jenis hasil tambahan. Ekstensi Tasks menambahkan `"task"` pada Pelajaran 13.

## Satu Putaran MRTR

Server tidak dapat memanggil klien selagi menangani permintaan. Server mengembalikan hasil ini sebagai gantinya:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "pick_files": {
        "method": "sampling/createMessage",
        "params": {
          "messages": [
            {
              "role": "user",
              "content": {
                "type": "text",
                "text": "Choose three representative files and return a JSON array."
              }
            }
          ],
          "systemPrompt": "Return only the requested value.",
          "modelPreferences": {
            "costPriority": 0.8,
            "intelligencePriority": 0.2
          },
          "maxTokens": 400
        }
      }
    },
    "requestState": "opaque-integrity-protected-value"
  }
}
```

Klien memverifikasi bahwa ia mendukung Sampling, menerapkan kebijakan persetujuan dan modelnya, lalu memperoleh respons model. Klien kemudian mengirim permintaan baru dengan id JSON-RPC yang berbeda:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "summarize_repo",
    "arguments": {"audience": "developer"},
    "inputResponses": {
      "pick_files": {
        "role": "assistant",
        "content": {
          "type": "text",
          "text": "[\"README.md\", \"server.py\", \"docs/intro.md\"]"
        },
        "model": "host-model",
        "stopReason": "endTurn"
      }
    },
    "requestState": "opaque-integrity-protected-value",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {"sampling": {}}
    }
  }
}
```

Percobaan ulang bukan kelanjutan dari sesi protokol. Ini permintaan baru yang mengulang metode dan argumen awal, hanya menambahkan `inputResponses` putaran terkini, dan menggaungkan `requestState` byte demi byte.

MRTR hanya diizinkan pada `tools/call`, `prompts/get`, dan `resources/read`. Server tidak boleh mengembalikan `input_required` dari metode yang tidak terkait.

## Keadaan Multi-Putaran

Pelajaran ini memerlukan dua panggilan model:

1. `pick_files` mengembalikan larik JSON.
2. `summary` mengembalikan prosa akhir.

Setiap percobaan ulang hanya membawa respons untuk putaran itu. Karena itu server menempatkan fase dan data antara yang tervalidasi ke dalam `requestState` berikutnya.

Perlakukan nilai itu sebagai masukan yang dikendalikan penyerang. Menandatangani nama fase mentah saja tidak cukup. Ikat keadaan pada:

- prinsipal terautentikasi, bukan `clientInfo` yang dilaporkan sendiri;
- metode asal;
- digest argumen awal;
- masa kedaluwarsa singkat;
- fase terkini dan nilai antara yang tervalidasi.

Pakai HMAC ketika kerahasiaan tidak diperlukan. Pakai enkripsi terautentikasi ketika klien tidak boleh membaca keadaan itu. Tolak tanda tangan buruk, nilai kedaluwarsa, perubahan prinsipal, atau perubahan argumen dengan `-32602`.

Klien tidak boleh mengurai atau mengubah `requestState`. Satu-satunya tugasnya adalah menggaungkan string persis itu pada percobaan ulang.

## Preferensi Model Hanya Petunjuk

`costPriority`, `speedPriority`, dan `intelligencePriority` adalah preferensi yang berdiri sendiri. Ketiganya bukan distribusi probabilitas dan tidak perlu berjumlah satu. Klien boleh mengabaikannya karena klien memiliki kebijakan model.

Jaga `includeContext` tetap `"none"` jika Anda mempertahankan alur Sampling lama. Mode konteks lainnya meningkatkan risiko kebocoran dan sendiri juga tidak disarankan. Teruskan konteks eksplisit seminimal mungkin dalam permintaan.

## Invarians Keamanan

Klien adalah batas kepercayaan untuk permintaan Sampling yang disematkan.

- Tampilkan kepada pengguna apa yang diminta server kepada model ketika kebijakan mensyaratkan persetujuan.
- Batasi putaran MRTR. Tanpa itu, server jahat dapat membuat loop pengeluaran model.
- Validasi setiap respons sampling sebelum memakainya sebagai nama berkas, URL, atau masukan alat.
- Batasi byte dan token per putaran.
- Tolak permintaan masukan yang tidak dinyatakan dalam kapabilitas klien terkini.
- Jaga keluaran model tetap keluar dari keputusan otorisasi.
- Catat metode asal dan kunci permintaan masukan tanpa mencatat konten prompt sensitif.

`clientInfo` dan `serverInfo` adalah metadata tampilan dan diagnostik. Jangan pernah memakai salah satunya sebagai identitas terautentikasi.

```figure
t3-sampling-flip
```

## Bangun

`code/main.py` mengimplementasikan alur dua putaran penuh tanpa paket pihak ketiga:

- `server/discover` mengembalikan `supportedVersions`, mengiklankan dukungan alat, dan mengembalikan petunjuk tembolok.
- `tools/list` mengembalikan descriptor `summarize_repo` yang deterministik dan dapat ditembolokkan dengan skema masukan objek.
- `tools/call` memvalidasi metadata per permintaan.
- Hasil pertama menyematkan `sampling/createMessage` untuk pemilihan berkas.
- Percobaan ulang pertama memvalidasi hasil model dan menyematkan permintaan kedua.
- `requestState` yang dilindungi HMAC membawa fase antar permintaan independen.
- Hasil akhir memakai `resultType: "complete"`.

Model host palsu membuat contoh menjadi deterministik. Ganti hanya `fake_host_model` saat menghubungkan host sungguhan. Mesin keadaan sisi server sebaiknya tetap deterministik dan dapat diuji.

## Gunakan

Dari akar repositori:

```bash
cd phases/13-tools-and-protocols/11-mcp-sampling/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Titik pemeriksaan yang diharapkan:

- Penemuan mengembalikan hasil lengkap dengan `ttlMs` dan `cacheScope`.
- Penemuan alat mengembalikan descriptor terurut yang sama dengan `resultType`, identitas server, dan petunjuk tembolok.
- Kapabilitas yang hilang dan versi yang tidak didukung memakai data kesalahan `-32021` dan `-32022` yang persis.
- Notifikasi tanpa id tidak menghasilkan respons JSON-RPC.
- Id permintaan adalah `[1, 2, 3]`, membuktikan setiap putaran MRTR independen.
- Dua hasil pertama adalah `input_required`.
- Hasil akhir adalah `complete` dan memuat berkas terpilih serta ringkasan.
- Mengubah argumen awal pada percobaan ulang menggagalkan pemeriksaan keadaan permintaan.

## Kirim

`outputs/skill-sampling-loop-designer.md` kini menjadi perencana migrasi. Dokumen itu pertama-tama memutuskan apakah Sampling sebaiknya dihapus demi integrasi model langsung. Jika kompatibilitas diperlukan, dokumen itu menghasilkan putaran MRTR, pengikatan keadaan, gerbang kapabilitas, anggaran, validasi, dan rencana penghapusan.

## Latihan

1. Ubah respons pemilihan berkas menjadi JSON tidak valid. Konfirmasi server mengembalikan `-32602`, bukan mempercayai keluaran model.
2. Ubah `audience` antara panggilan pertama dan percobaan ulang. Jelaskan mengapa keadaan yang tersegel memblokir penggunaan ulang lintas permintaan.
3. Tambahkan putaran ketiga yang meminta host mengkritik ringkasan. Bawa ringkasan sebelumnya di dalam keadaan bertanda tangan dan batasi seluruh alur menjadi tiga putaran.
4. Hapus Sampling dengan mengganti panggilan balik host palsu dengan adapter model milik server. Cantumkan tanggung jawab persetujuan, penagihan, dan observabilitas mana yang beralih ke server.
5. Tambahkan uji kedaluwarsa memakai nilai keadaan yang sudah lewat satu detik dari batas waktunya.

## Istilah Kunci

| Istilah | Arti dalam 2026-07-28 |
|---------|----------------------|
| Sampling | Fitur yang tidak disarankan dan meminta model klien untuk melengkapi teks |
| MRTR | Pola percobaan ulang stateless untuk masukan klien yang diperlukan saat permintaan berjalan |
| `InputRequiredResult` | Hasil dengan `resultType: "input_required"` |
| `inputRequests` | Peta yang ditetapkan server berisi permintaan elicitation, sampling, atau roots yang disematkan |
| `inputResponses` | Hasil klien putaran terkini yang dikunci seperti `inputRequests` |
| `requestState` | Keadaan server opak yang digaungkan persis oleh klien dan diverifikasi server |
| `resultType` | Diskriminator wajib untuk hasil MCP modern |
| Integrasi model langsung | Pengganti yang disarankan untuk server baru yang memerlukan inferensi model |
| Gerbang kapabilitas | Aturan yang mencegah pengiriman permintaan tersemat yang tidak diiklankan klien |
| Anggaran loop | Putaran, token, byte, waktu, dan pengeluaran maksimal yang diizinkan untuk operasi |

## Kompatibilitas Lama

Klien yang dikunci pada 2025-11-25 masih boleh memakai alur `sampling/createMessage` lama yang dimulai server atas koneksi langsung. Simpan perilaku itu hanya di dalam adapter khusus versi. Jangan jadikan jalur berbasis sesi itu sebagai arsitektur server 2026-07-28.

SDK resmi dapat menerjemahkan handler `input_required` modern untuk rekanan lama. Shim itu adalah batas kompatibilitas, bukan izin untuk menambahkan logika baru yang bergantung pada sesi.

## Bacaan Lanjutan

- [Permintaan Multi Pulang-Pergi MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [Catatan perubahan MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [Penghentian Sampling MCP](https://modelcontextprotocol.io/seps/2577-deprecate-roots-sampling-and-logging)
- [Penemuan server MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
