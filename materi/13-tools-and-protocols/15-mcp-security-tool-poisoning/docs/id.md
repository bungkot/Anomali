# Keamanan MCP: Metadata Beracun, Routing, dan Keadaan MRTR

> Tanpa keadaan tidak berarti tanpa kepercayaan. Artinya, setiap permintaan menyingkap bukti yang dibutuhkan server dan gerbang untuk memvalidasi panggilan secara mandiri.

**Jenis:** Pelajari
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13 · 07 (server MCP), Fase 13 · 08 (klien MCP)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Perlakukan deskripsi alat, anotasi, informasi klien, dan informasi server sebagai data yang tidak tepercaya.
- Deteksi peracunan metadata, perubahan deskriptor, dan tabrakan nama lintas server.
- Validasi metadata permintaan 2026-07-28 dan header routing Streamable HTTP.
- Lindungi `requestState` MRTR dari pemalsuan dan ikatkan konfirmasi ke argumen yang tepat.
- Terapkan otorisasi dan batas laju pada prinsipal, bukan pada sesi protokol yang dihapus.

## Masalahnya

Model membaca deskripsi alat untuk memutuskan apa yang akan dipanggil. Router membaca nama alat untuk memutuskan ke mana permintaan dikirim. Pengguna membaca label untuk memutuskan apa yang akan disetujui. Satu deskriptor berbahaya dapat menyasar ketiganya.

Panduan keamanan MCP resmi bersifat langsung: deskripsi dan anotasi harus diperlakukan sebagai tidak tepercaya kecuali berasal dari server tepercaya. Bahkan saat itu, kepercayaan deployment dapat berubah. Pembaruan server, paket yang dikompromi, kesalahan registri, atau penggabungan di gerbang dapat mengubah apa yang dilihat model.

Protokol terkini juga mengubah batas keamanan. Pada 2026-07-28 tidak ada jabat tangan inti dan tidak ada sesi transpor. Desain keamanan yang mengunci persetujuan, batas laju, atau riwayat audit hanya berdasarkan `Mcp-Session-Id` bukanlah desain terkini.

## Konsepnya

### Tujuh permukaan serangan yang layak diperiksa

Gunakan daftar konkret, bukan instruksi samar untuk "berhati-hati".

1. **Peracunan metadata.** Deskripsi memuat instruksi yang tidak berkaitan dengan perilaku alat yang dinyatakan.
2. **Rug pull deskriptor.** Nama, deskripsi, skema, atau anotasi yang sebelumnya disetujui berubah.
3. **Peniruan lintas server.** Dua backend mengekspos nama alat tanpa kualifikasi yang sama, dan routing memilih salah satu secara diam-diam.
4. **Kebingungan header dan badan.** `Mcp-Method` atau `Mcp-Name` tidak cocok dengan permintaan JSON-RPC.
5. **Peningkatan kapabilitas.** Sebuah pihak mengklaim ekstensi atau fitur klien, lalu server mengira deklarasi itu adalah otorisasi.
6. **Pemalsuan keadaan MRTR.** Klien mengubah `requestState`, menjawab pertanyaan lain, atau menggunakan ulang konfirmasi dengan argumen berbeda.
7. **Kebingungan identitas rantai pasok.** Nama tampilan yang akrab dianggap sebagai bukti identitas penerbit atau server.

Permukaan-permukaan ini tumpang tindih. Pin hash membantu mengatasi perubahan deskriptor, tetapi tidak membuktikan deskriptor pertama aman. Pemindaian statis menangkap frasa yang jelas, tetapi bukan instruksi halus. Pemberian namespace mencegah satu kelas tabrakan, tetapi bukan server ber-namespace yang berbahaya. Tumpukkan kontrolnya.

### Amplop permintaan terkini adalah bukti, bukan identitas

Setiap permintaan 2026-07-28 memuat:

```json
{
  "_meta": {
    "io.modelcontextprotocol/protocolVersion": "2026-07-28",
    "io.modelcontextprotocol/clientCapabilities": {
      "elicitation": {"form": {}}
    },
    "io.modelcontextprotocol/clientInfo": {
      "name": "security-lab",
      "version": "1.0.0"
    }
  }
}
```

Validasi versi dan bentuk kapabilitas pada setiap permintaan. Gunakan kapabilitas untuk memilih bentuk respons yang kompatibel. Jangan gunakan `clientInfo` sebagai prinsipal terautentikasi. Datanya dilaporkan sendiri.

Peringatan yang sama berlaku untuk `io.modelcontextprotocol/serverInfo` pada metadata hasil. Data itu berguna untuk log dan debugging. Data itu bukan sertifikat, bukti registri, maupun keputusan otorisasi.

### Validasi routing sebelum kebijakan

Untuk `tools/call`, Streamable HTTP memuat:

```text
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: notes.export
```

Metode pada header harus sama dengan metode pada badan. Nama pada header harus sama dengan `params.name`. Tolak ketidaksesuaian dengan `-32020` sebelum memilih backend, menerapkan RBAC, atau mengonsumsi token batas laju.

Urutan ini menutup ambiguitas umum: satu komponen mengotorisasi badan sementara komponen lain melakukan routing berdasarkan header.

Validasi kawat mengikuti satu urutan tepat. Validasi JSON-RPC dan tipe metadata, bandingkan nilai header dengan badan, lalu periksa apakah versi yang cocok didukung. Header yang tidak cocok mengembalikan HTTP 400 dengan `-32020`. Jika header dan badan sepakat pada versi yang tidak didukung, kembalikan HTTP 400 dengan `-32022` dan `data` tepat `{"supported":["2026-07-28"],"requested":"<actual>"}`. Metode yang tidak dikenal mengembalikan HTTP 404 dengan `-32601`.

Setiap objek galat menyertakan `data` opsional bila kontrak memerlukan informasi pemulihan terstruktur. Notifikasi tidak memiliki `id`, sehingga tidak pernah menerima respons JSON-RPC sukses maupun galat. Notifikasi HTTP yang diterima mengembalikan 202 dengan badan kosong.

### Kunci seluruh deskriptor

Hash deskripsi saja melewatkan perubahan skema dan anotasi. Kanonicalisasi dan hash field deskriptor yang disetujui pengguna:

```python
normalized = json.dumps(tool, sort_keys=True, separators=(",", ":"))
digest = hashlib.sha256(normalized.encode()).hexdigest()
```

Simpan intisari di bawah kunci berkualifikasi seperti `notes.export`, bersama bukti penerbit dan waktu persetujuan di luar contoh sederhana ini.

Pada setiap penyegaran:

- Kunci tidak dikenal: karantina sampai ditinjau.
- Kunci sama, intisari berbeda: karantina sebagai rug pull sampai disetujui ulang.
- Nama tanpa kualifikasi ganda: wajibkan pemberian namespace deterministik.
- Tembakan pemindai: blokir dan tinjau deskriptor lengkap.

Kesetaraan hash membuktikan kestabilan, bukan keamanan. Deskriptor yang beracun tetap beracun meskipun dikunci dengan sempurna.

### Pemindaian statis adalah kabel trip

Pola sederhana dapat menandai tag peran, penggantian instruksi, penyembunyian, akses rahasia, dan tujuan jaringan yang disamarkan. Pola itu cukup murah untuk dipakai saat instalasi dan di CI.

Pola itu bukan bukti semantik. Deskripsi yang aman dapat memuat frasa yang ditandai dalam peringatan yang sah. Deskripsi berbahaya dapat menghindari semua frasa itu. Perlakukan keluaran pemindai sebagai bukti untuk tinjauan, bukan nilai otomatis ketidakbersalahan.

### Beri namespace sebelum penggabungan

Misalkan dua server sama-sama mengekspos `search`. Jangan pernah biarkan urutan penemuan yang menentukan pemenangnya.

```text
notes.search
issues.search
```

Nama berkualifikasi adalah nama publik gerbang. Catat pemetaan backend secara terpisah. Nama yang stabil membuat persetujuan, audit, pin hash, dan routing `Mcp-Name` merujuk pada objek yang sama.

### Kapabilitas adalah deklarasi kompatibilitas

`clientCapabilities` per permintaan memberi tahu server fitur protokol apa yang dapat diproses klien. Data itu tidak memberikan klien akses ke alat, data, atau aksi.

Otorisasi tetap berasal dari prinsipal terautentikasi dan kebijakan sumber daya. Urutannya adalah:

1. Autentikasi kredensial transpor.
2. Validasi versi, header, dan bentuk permintaan.
3. Periksa kompatibilitas kapabilitas.
4. Otorisasi prinsipal, alat, sumber daya, dan argumen.
5. Jalankan atau minta masukan pengguna.

### Lindungi konfirmasi MRTR tanpa keadaan

Alat berdampak mungkin memerlukan konfirmasi pengguna. MCP terkini menggunakan Permintaan Multi Putaran (MRTR), bukan callback server-ke-klien.

Respons pertama:

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "confirm": {
      "method": "elicitation/create",
      "params": {
        "mode": "form",
        "message": "Export notes to archive?",
        "requestedSchema": {
          "type": "object",
          "properties": {
            "confirm": {"type": "boolean"}
          },
          "required": ["confirm"]
        }
      }
    }
  },
  "requestState": "opaque-integrity-protected-value"
}
```

Klien memperoleh masukan dan mencoba lagi metode asli dengan id JSON-RPC baru:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "notes.export",
    "arguments": {"query": "private", "destination": "archive"},
    "requestState": "opaque-integrity-protected-value",
    "inputResponses": {
      "confirm": {
        "action": "accept",
        "content": {"confirm": true}
      }
    },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {
        "elicitation": {"form": {}}
      }
    }
  }
}
```

Setiap nilai `inputRequests` adalah permintaan tersemat lengkap dengan `method` dan `params`. Kuncinya harus cocok dengan entri yang bersangkutan di `inputResponses`. Elicitation formulir menggunakan `requestedSchema` berakar objek, dan klien harus menyatakan kapabilitas elicitation formulir sebelum server memintanya.

Kapabilitas terkini memiliki dua deklarasi formulir yang valid. `{"elicitation":{}}` mendukung elicitation formulir secara implisit, sedangkan `{"elicitation":{"form":{}}}` menyatakannya secara eksplisit. Deklarasi hanya-URL seperti `{"elicitation":{"url":{}}}` tidak mendukung permintaan formulir. Server mengembalikan HTTP 400 dengan `-32021` dan `data.requiredCapabilities` sebesar `{"elicitation":{"form":{}}}`.

Perlakukan `requestState` sebagai masukan bermusuhan. Tanda tangani atau enkripsi, validasi, dan ikatkan ke metode, alat, argumen yang tepat, tujuan, kedaluwarsa, prinsipal, serta nonce sekali pakai bila replay menjadi perhatian.

Buku besar nonce tidak boleh berada di dalam satu objek gerbang. Model yang dapat dijalankan menyuntikkan penyimpanan replay terbatas yang dipangkas berdasarkan TTL dan dapat dibagikan oleh beberapa instance gerbang. Klaim atomiknya adalah batas eksekusi: hanya penerimaan yang tervalidasi atau penolakan akhir eksplisit yang mengonsumsi keadaan. Respons yang malformed atau `cancel` tidak menjalankan apa pun dan tetap dapat dicoba ulang sampai kedaluwarsa. Armada produksi memerlukan klaim bersyarat yang sama di penyimpanan tahan lama bersama.

Jangan simpan konteks konfirmasi tersembunyi dalam sesi protokol. Semua instance server harus dapat memvalidasi percobaan ulang.

### Aturan dua untuk panggilan berisiko tinggi

Klasifikasikan panggilan berdasarkan tiga sumbu:

- Panggilan itu mengonsumsi masukan yang tidak tepercaya.
- Panggilan itu dapat mengakses data sensitif.
- Panggilan itu menyebabkan aksi eksternal berdampak.

Satu langkah otomatis tidak boleh menggabungkan ketiganya. Pisahkan, kurangi hak, atau minta masukan pengguna eksplisit melalui MRTR. Ini adalah heuristik desain, bukan kapabilitas protokol.

### Kurangi otoritas sebelum eksekusi

Tanpa keadaan saja bukan berarti aman. Pendekatan itu menghilangkan riwayat protokol yang tersembunyi, tetapi permintaan mandiri tetap dapat meminta handler yang terlalu berkuasa untuk membocorkan data atau melakukan perubahan yang tidak dapat dibatalkan. Keamanan muncul dari pengurangan otoritas di setiap batas:

1. **Kata kerja bertipe.** Ekspos satu operasi terbatas seperti `archive_note`, bukan alat `run` atau `request` umum yang dapat mengekspresikan kekuatan yang tidak terkait.
2. **Argumen tervalidasi.** Gunakan skema tertutup bila memungkinkan, tolak field yang tidak dikenal, normalisasi identifier sekali, batasi ukuran, dan validasi tujuan, tenant, serta kepemilikan sumber daya sebelum evaluasi kebijakan.
3. **Otorisasi terkini.** Ikat prinsipal terautentikasi ke kata kerja, sumber daya, lingkungan, dan argumen ternormalisasi yang tepat. Anotasi alat dan kapabilitas klien tidak memberikan otoritas ini.
4. **Persetujuan terikat aksi.** Untuk panggilan berdampak, ikatkan persetujuan ke intisari kata kerja bertipe dan argumen ternormalisasi, plus prinsipal, kedaluwarsa, dan kebijakan sekali pakai. Setiap field yang berubah memerlukan keputusan baru.
5. **Penolakan sebagai warga kelas satu.** Modelkan penolakan, persetujuan kedaluwarsa, penolakan pengguna, dan tujuan tidak aman sebagai hasil biasa yang tidak menjalankan efek samping. Jangan menerjemahkan penolakan menjadi alat fallback yang lebih lemah.
6. **Bukti audit teredaksi.** Catat siapa yang meminta, deskriptor dan versi kebijakan yang diterima mana yang dipakai, target ternormalisasi apa yang diotorisasi, mengapa keputusan mengizinkan atau menolak, serta apakah eksekusi dimulai. Simpan intisari atau nilai teredaksi, bukan rahasia.

Setiap langkah mempersempit apa yang boleh dilakukan komponen berikutnya. Handler akhir seharusnya menerima perintah domain yang telah tervalidasi, bukan teks model mentah plus kredensial yang terlalu luas. Ulangi seluruh rantai pada percobaan ulang MRTR, pembaruan task, atau panggilan yang diteruskan gerbang. Persetujuan sebelumnya tidak mengubah permintaan berikutnya menjadi lalu lintas sesi tepercaya.

### Jalur interaksi terkini dan lawas

Roots, Sampling, dan Logging tidak digunakan lagi untuk implementasi baru 2026-07-28. Gerbang boleh mempertahankan kode kanal permintaan lama hanya sebagai jalur kompatibilitas yang dijaga oleh versi.

Jangan membangun pertahanan baru di sekitar pembatas sampling per sesi. Terapkan kuota pada prinsipal terautentikasi, penerbit, sumber daya, alat, dan jendela waktu. Untuk kerja interaktif terkini, periksa permintaan dan respons masukan MRTR.

### Pemeriksaan transpor tanpa keadaan

- Terima pesan MCP modern di endpoint POST tunggal.
- Kembalikan 405 untuk GET dan DELETE modern.
- Jangan menerbitkan atau bergantung pada `Mcp-Session-Id`.
- Abaikan header sesi dan replay lawas sebagai masukan otoritas.
- Kembalikan JSON atau SSE cakupan permintaan untuk POST itu.
- Gunakan `subscriptions/listen` hanya untuk notifikasi perubahan berumur panjang yang memilih ikut serta.

```figure
tp-tool-poisoning
```

## Bangunlah

`code/main.py` mengimplementasikan model gerbang keamanan kecil dalam proses. Model ini mengkanonicalisasi dan mengunci deskriptor alat lengkap, melaporkan peracunan metadata dan peniruan, memvalidasi amplop serta nilai routing permintaan modern, dan melakukan ekspor terkonfirmasi dua putaran dengan `requestState` bertanda tangan serta penyimpanan replay bersama yang disuntikkan.

Model dimulai setelah adapter HTTP mengurai badan JSON dan header routing. Model ini tidak memvalidasi `Content-Type` maupun `Accept`. Hubungkan dispatcher yang sama ke adapter Streamable HTTP lengkap pada Pelajaran 09, yang mewajibkan `Content-Type: application/json` dan nilai `Accept` yang memuat baik `application/json` maupun `text/event-stream`.

Jalankan:

```bash
cd phases/13-tools-and-protocols/15-mcp-security-tool-poisoning
python3 code/main.py
npx tsx code/main.ts
python3 -m unittest discover code/tests -v
```

Contoh sengaja mengubah sebuah deskriptor. Pemindai dan perbandingan intisari menghasilkan temuan yang independen. Ekspor kemudian mendemonstrasikan respons `input_required` dan percobaan ulang tanpa keadaan.

## Gunakanlah

Ganti `SAFE_TOOLS` dengan cuplikan ternormalisasi dari server Anda sendiri yang telah disetujui. Jauhkan kredensial dan rahasia dari cuplikan itu. Tinjau setiap deskriptor baru atau yang berubah sebelum memperbarui intisarinya.

Di gerbang, jalankan pemeriksaan yang sama saat penemuan dan sekali lagi sebelum dispatch. Tembolok dapat mengurangi kerja penemuan, tetapi persetujuan yang ditembolok harus kedaluwarsa atau dibatalkan ketika deskriptor berubah.

## Kirimkan

Pelajaran ini mengirimkan `outputs/skill-mcp-threat-model.md`. Berkas itu menghasilkan model ancaman protokol terkini melintasi batas metadata, routing, kapabilitas, otorisasi, MRTR, penembolokan, registri, dan kompatibilitas.

## Latihan

1. Ikatkan prinsipal terautentikasi dan keputusan otorisasi terkini ke keadaan MRTR yang disegel, lalu tolak percobaan ulang di bawah prinsipal yang berbeda.
2. Ganti penyimpanan replay dalam memori dengan penyisipan bersyarat persisten dan buktikan dua proses tidak dapat mengklaim nonce yang sama.
3. Suntikkan kegagalan setelah klaim replay tetapi sebelum ekspor simulasi. Tentukan dan uji aturan transaksi atau idempotensi yang membuat pemulihan aman.
4. Ubah `inputSchema` sebuah alat tanpa mengubah deskripsinya. Pastikan penguncian seluruh deskriptor menangkapnya.
5. Tambahkan kebijakan yang menolak penembolokan publik ketika `tools/list` berbeda menurut prinsipal.
6. Modelkan server lawas di belakang gerbang. Letakkan semua perilaku jabat tangan dan sesi di balik cabang kompatibilitas `2025-11-25` yang eksplisit.

## Istilah Kunci

| Istilah | Makna |
|---------|-------|
| Peracunan metadata | Instruksi atau klaim menyesatkan yang disematkan dalam deskriptor alat |
| Rug pull | Perubahan pada deskriptor yang sebelumnya disetujui |
| Peniruan alat | Routing ambigu yang disebabkan oleh nama tanpa kualifikasi yang ganda |
| Ketidaksesuaian header | Header routing dan badan JSON-RPC tidak cocok, galat `-32020` |
| Pin hash | Intisari dari deskriptor lengkap yang disetujui |
| MRTR | Pola respons dan percobaan ulang tanpa keadaan untuk masukan yang diminta server |
| `requestState` | Nilai putaran-pergi yang buram dan harus diperlakukan sebagai masukan tidak tepercaya |
| Deklarasi kapabilitas | Pernyataan kompatibilitas protokol, bukan otorisasi |
| Dukungan formulir implisit | Objek kapabilitas `elicitation` kosong, setara dengan dukungan formulir |
| Nama alat berkualifikasi | Nama gerbang yang stabil seperti `notes.search` |

## Bacaan Lanjutan

- [Panduan keamanan dan kepercayaan MCP](https://modelcontextprotocol.io/specification/2026-07-28#security-and-trust--safety)
- [Permintaan Multi Putaran](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [Transpor Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [Fitur yang tidak digunakan lagi](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)
