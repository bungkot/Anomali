# Sumber Daya dan Prompt MCP: Konteks yang Dapat Dialamatkan untuk Server Stateless

> Alat melakukan operasi. Sumber daya mengekspos konten yang dapat dialamatkan. Prompt mengemas templat pesan yang dipilih pengguna. Server MCP yang baik menjaga kontrak-kontrak ini tetap terpisah dan dapat diprediksi.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 07 (Membangun Server MCP), Fase 13, Pelajaran 09 (Transpor MCP)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Memilih di antara alat, sumber daya, dan prompt berdasarkan maksud konsumen.
- Mengiklankan permukaan sumber daya dan prompt melalui `server/discover` yang wajib.
- Membangun hasil `resources/list` dan `prompts/list` yang deterministik.
- Menerapkan `ttlMs` dan `cacheScope` tanpa membocorkan data khusus pengguna.
- Mengembalikan kesalahan JSON-RPC `-32602` untuk URI sumber daya yang tidak valid atau tidak dikenal.
- Membuka aliran respons POST `subscriptions/listen` dan mengorelasikan setiap event berdasarkan id langganan.
- Memperlakukan konten sumber daya dan templat prompt sebagai keluaran server yang tidak tepercaya.

## Mulailah dari Konsumen

Cara paling mudah menyalahgunakan MCP adalah mulai dari kode implementasi. Query basis data menjadi alat karena fungsi sudah akrab. Alur kerja yang dapat digunakan ulang menjadi sumber daya karena disimpan dalam berkas. Prompt menjadi kebijakan tersembunyi karena host dapat menyuntikkannya.

Mulailah dari siapa yang memilih dan apa yang mereka harapkan.

| Primitif | Maksud utama | Pemilik pemilihan | Hasil khas |
|----------|--------------|-------------------|------------|
| Alat | Melakukan operasi | Model atau aplikasi | Hasil aksi terstruktur |
| Sumber daya | Membaca konten di sebuah URI | Host, aplikasi, atau pengguna | Konten teks atau biner |
| Prompt | Memulai alur kerja pesan yang dapat dipakai ulang | Pengguna melalui UI host | Satu atau beberapa pesan prompt |

Catatan di `notes://note-1` adalah sumber daya karena merupakan konten yang dapat dialamatkan. `delete_note` adalah alat karena mengubah keadaan. `review_note` adalah prompt karena pengguna memilih alur tinjauan yang sudah disiapkan.

Jangan mengekspos satu operasi sebagai ketiganya hanya agar tampak lengkap. Setiap permukaan tambahan memerlukan penemuan, otorisasi, penembolokan, penanganan kesalahan, pengujian, dan dokumentasi.

## Amplop Stateless 2026-07-28

Pelajaran ini menargetkan revisi protokol MCP `2026-07-28`. Dalam profil ini tidak ada jabat tangan inisialisasi maupun sesi protokol. Setiap permintaan membawa versi protokol dan kapabilitas klien di kunci `_meta` yang dikhususkan.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "resources/list",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": {
        "name": "course-client",
        "version": "1.0.0"
      },
      "io.modelcontextprotocol/clientCapabilities": {}
    }
  }
}
```

Server wajib mengimplementasikan `server/discover`. Hasilnya mengiklankan versi
yang didukung, kapabilitas sumber daya dan prompt, identitas implementasi, serta
petunjuk tembolok. Klien boleh memanggil metode lain secara langsung, tetapi
penemuan memberinya satu cuplikan stabil sebelum membangun UI.

```json
{
  "resultType": "complete",
  "supportedVersions": ["2026-07-28"],
  "capabilities": {
    "resources": {"listChanged": true, "subscribe": true},
    "prompts": {"listChanged": true}
  },
  "ttlMs": 3600000,
  "cacheScope": "public"
}
```

Hasil normal menyatakan `"resultType": "complete"`. Respons `_meta` mengidentifikasi implementasi yang melayani melalui `io.modelcontextprotocol/serverInfo`. Informasi ini berguna untuk diagnostik. Informasi itu bukan identitas autentikasi. Permintaan yang membawa revisi yang tidak didukung mengembalikan `-32022` disertai revisi yang diminta dan revisi yang didukung server.

Kontrak stateless mengubah insting desain Anda. Daftar tidak boleh bergantung pada panggilan sebelumnya di satu koneksi. Otorisasi boleh mengubah himpunan yang terlihat karena kredensial adalah masukan permintaan, tetapi riwayat koneksi tidak boleh.

## Sumber Daya adalah Kontrak URI yang Stabil

Sumber daya adalah konten yang diidentifikasi oleh URI. Rancang URI sebelum menulis handler.

Sifat URI yang baik:

- Cukup stabil untuk ditandai atau diteruskan antar permintaan.
- Berada dalam ruang nama domain server.
- Independen dari id proses maupun koneksi.
- Divalidasi sebelum akses penyimpanan.
- Diotorisasi pada setiap pembacaan.

`notes://note-1` lebih baik daripada `note-1` karena ruang nama eksplisit. Server berkas boleh memakai URI `file://`, tetapi tetap harus memeriksa batas direktori yang dikonfigurasi setelah menyelesaikan symlink dan segmen relatif.

`resources/list` mengembalikan sumber daya yang saat ini terlihat oleh pemanggil. Urutkan berdasarkan kunci stabil seperti URI. Urutan deterministik mencegah cache miss yang berisik, cuplikan yang berubah-ubah, dan UI host yang melompat antar penyegaran.

```json
{
  "resultType": "complete",
  "resources": [
    {
      "uri": "notes://note-1",
      "name": "Architecture decision",
      "description": "Why the service uses a stateless boundary",
      "mimeType": "text/markdown"
    }
  ],
  "ttlMs": 300000,
  "cacheScope": "public",
  "_meta": {
    "io.modelcontextprotocol/serverInfo": {
      "name": "notes-server",
      "version": "2.0.0"
    }
  }
}
```

`resources/read` mengembalikan satu atau beberapa butir konten. URI yang tidak dikenal bukanlah pembacaan kosong yang berhasil. Spesifikasi Resources saat ini menetapkan URI sumber daya yang tidak valid atau tidak dikenal sebagai parameter tidak valid JSON-RPC, kode `-32602`.

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "error": {
    "code": -32602,
    "message": "Unknown or invalid resource URI",
    "data": {
      "uri": "notes://missing"
    }
  }
}
```

Pembedaan itu memungkinkan klien memisahkan ketiadaan dari dokumen kosong yang valid. Pembedaan itu juga mencegah jatuhnya ke pencarian yang lebih luas secara tidak sengaja.

### Templat sumber daya

Templat sumber daya menjelaskan keluarga URI berparameter. Gunakan templat ketika mendaftarkan setiap butir konkret akan mahal atau tidak terbatas. Misalnya, `notes://projects/{project}/decisions/{decision}` memberi tahu klien cara membentuk alamat yang valid tanpa mengembalikan setiap keputusan.

Templat tidak melemahkan validasi. Uraikan variabel, terapkan otorisasi, tegaskan batas panjang dan karakter, serta bangun query penyimpanan dengan parameter bertipe. Jangan pernah menggabungkan ekor URI sembarangan ke jalur sistem berkas atau pernyataan basis data.

### Konten bukan instruksi yang tepercaya

Teks sumber daya dapat berisi prompt injection, rahasia, perintah yang menyesatkan, atau markup yang malformed. Host sebaiknya menjaga provenansi dan memperlakukan konten sumber daya sebagai data. Server sebaiknya membatasi ukuran konten, mengembalikan tipe MIME yang akurat, menyunting kolom yang tidak dapat diakses pemanggil, dan menghindari pengembalian catatan yang tidak terkait.

## Prompt adalah Templat yang Dikendalikan Pengguna

Prompt MCP dirancang untuk pemilihan eksplisit oleh pengguna. Host dapat menampilkannya sebagai perintah slash, item menu, atau tombol alur kerja. Protokol tidak mewajibkan satu UI tertentu.

`prompts/list` sebaiknya deterministik untuk otorisasi permintaan yang sama. Setiap prompt memerlukan nama yang stabil, deskripsi yang berguna, dan deklarasi argumen yang memungkinkan host mengumpulkan masukan sebelum `prompts/get`.

```json
{
  "resultType": "complete",
  "prompts": [
    {
      "name": "review_note",
      "title": "Review a note",
      "description": "Review one note for a named concern",
      "arguments": [
        {
          "name": "uri",
          "description": "The note resource URI",
          "required": true
        }
      ]
    }
  ],
  "ttlMs": 600000,
  "cacheScope": "public"
}
```

`prompts/get` menyelesaikan argumen menjadi pesan. Prompt tidak menggantikan instruksi sistem host. Host yang menentukan bagaimana pesan yang dikembalikan masuk ke konteks model dan menjaga kebijakan tepercaya sendiri pada prioritas lebih tinggi.

Validasi argumen prompt di batas server. URI prompt sebaiknya melalui pemeriksaan otorisasi yang sama dengan pembacaan sumber daya langsung. Jangan jadikan prompt sebagai kanal samping di luar akses sumber daya.

## Petunjuk Tembolok adalah Bagian dari Kebenaran

`ttlMs` memberi tahu klien berapa lama hasil boleh digunakan kembali. `cacheScope` menjelaskan siapa yang boleh berbagi nilai tembolok itu.

| Cakupan | Arti | Penggunaan khas |
|---------|------|-----------------|
| `public` | Boleh digunakan ulang lintas pengguna ketika otorisasi mengizinkan | Katalog prompt publik |
| `private` | Terikat pada pengguna atau konteks kredensial peminta | Konten catatan milik pengguna |

Pilih TTL berdasarkan laju perubahan data dan kerugian akibat kebasian. Lima menit mungkin cocok untuk katalog prompt publik. Pembacaan catatan privat dapat memakai satu menit.

MCP hanya menetapkan `public` dan `private` sebagai nilai `cacheScope`. Untuk hasil yang memuat rahasia atau berubah cepat, kembalikan `cacheScope: "private"` dengan `ttlMs: 0`, lalu terapkan aturan no-store yang lebih ketat dalam kebijakan tembolok host. `no-store` sendiri bukan nilai `cacheScope` MCP.

Petunjuk tembolok tidak pernah menggantikan otorisasi. Kunci tembolok harus memasukkan setiap dimensi permintaan yang mengubah keterlihatan, termasuk tenant, pengguna, cakupan, lokal, dan kursor paginasi. Jika tembolok bersama tidak dapat menyatakan dimensi itu dengan aman, gunakan `private` dengan TTL nol dan kebijakan no-store di tingkat host.

## Langganan Memakai Aliran Respons yang Dibuka Klien

Pola langganan modern menggantikan RPC `resources/subscribe` sebelumnya dan endpoint event GET lama.

Klien mengirim `subscriptions/listen` sebagai permintaan JSON-RPC biasa. Atas Streamable HTTP, ini adalah POST yang responsnya tetap terbuka sebagai aliran SSE. Objek `notifications` adalah daftar izin. Server tidak boleh mengirim jenis notifikasi yang tidak diminta.

```json
{
  "jsonrpc": "2.0",
  "id": 17,
  "method": "subscriptions/listen",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "course-client",
        "version": "1.0.0"
      }
    },
    "notifications": {
      "resourcesListChanged": true,
      "promptsListChanged": true,
      "resourceSubscriptions": [
        "notes://note-1"
      ]
    }
  }
}
```

Id permintaan adalah id langganan. Sebelum event yang diminta apa pun, server mengirim `notifications/subscriptions/acknowledged`. Filter-nya hanya memuat himpunan bagian yang diterima server.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/subscriptions/acknowledged",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/subscriptionId": 17
    },
    "notifications": {
      "resourcesListChanged": true,
      "resourceSubscriptions": [
        "notes://note-1"
      ]
    }
  }
}
```

Setiap event berikutnya di aliran itu membawa metadata yang sama.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/subscriptionId": 17
    },
    "uri": "notes://note-1"
  }
}
```

Notifikasi itu menyatakan sumber daya telah berubah. Klien membacanya lagi melalui `resources/read`, tunduk pada otorisasi terkini. Klien tidak menganggap event memuat dokumen baru.

Beberapa langganan dapat berbagi satu kanal stdio. Id langganan memungkinkan klien memisahkannya. Atas HTTP, menutup aliran respons membatalkan langganan. Server yang mengakhiri aliran dengan anggun mengembalikan respons `resultType: "complete"` akhir yang berkorelasi dengan permintaan awal.

Jangan pakai aliran langganan sebagai sesi protokol. Pembacaan berikutnya tetap permintaan lengkap yang dapat mencapai instance server sehat mana pun.

```figure
t3-primitive-sort
```

## Laboratorium Interaktif

Gunakan gambar untuk mengklasifikasikan lima kemampuan dari pelacak proyek: detail isu, buat isu, templat tinjauan sprint, kebijakan proyek, dan tutup isu. Lalu putuskan daftar mana yang boleh ditembolokkan secara publik, pembacaan mana yang harus tetap privat, dan sumber daya mana yang layak mendapat notifikasi pembaruan.

Untuk setiap klasifikasi, sebutkan siapa yang memilih. Jika model melakukan aksi, gunakan alat. Jika host membaca konten yang dialamatkan URI, gunakan sumber daya. Jika pengguna memulai alur kerja pesan yang sudah disiapkan, gunakan prompt.

## Laboratorium Praktik

Jalankan simulator dari akar repositori:

```bash
cd phases/13-tools-and-protocols/10-mcp-resources-and-prompts/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Periksa transkrip dalam urutan ini:

1. Konfirmasi `server/discover` mengiklankan revisi terkini dan kedua kapabilitas.
2. Konfirmasi kedua hasil daftar diurutkan dan memakai `resultType: "complete"`.
3. Konfirmasi hasil daftar dan pembacaan membawa petunjuk tembolok yang disengaja.
4. Ubah URI pembacaan menjadi `notes://missing` dan amati `-32602`.
5. Konfirmasi pengakuan langganan mendahului event sumber daya.
6. Konfirmasi event dan penutupan anggun sama-sama membawa id langganan `5`.

Model Python tidak membuka koneksi HTTP sungguhan. Model ini menyajikan pesan yang harus ditempatkan SDK pada aliran respons cakupan permintaan. Gunakan SDK resmi untuk framing dan transpor dalam produksi.

## Artefak yang Dikirimkan

`outputs/skill-primitive-splitter.md` adalah tinjauan desain yang dapat dipakai ulang untuk pemilihan primitif MCP. Dokumen itu kini memeriksa penemuan deterministik, cakupan tembolok, perilaku URI tidak valid, dan filter langganan modern.

Pelajaran ini juga mengirimkan `assets/primitive-split.svg`, versi statis dari batas primitif dan langganan untuk belajar luring.

## Verifikasi

```bash
cd phases/13-tools-and-protocols/10-mcp-resources-and-prompts/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Hasil yang diharapkan: program utama mencetak transkrip JSON dan perintah uji melaporkan minimal dua belas uji yang lulus.

## Kaitan Proyek Akhir

Gunakan kontrak ini ketika server proyek akhir Anda mengekspos pengetahuan yang dapat dialamatkan di samping aksi. Sertakan satu cuplikan katalog deterministik, satu pembacaan sumber daya terotorisasi, satu penyelesaian prompt, satu kasus URI tidak valid, dan satu transkrip langganan.

Bukti Anda harus menunjukkan bahwa tidak ada daftar yang bergantung pada riwayat koneksi dan bahwa event langganan tidak pernah memberikan akses ke sumber daya yang mendasarinya.

## Latihan

1. Tambahkan templat sumber daya `notes://projects/{project}/notes/{id}` dan validasi kedua variabel.
2. Tambahkan paginasi pada `resources/list` sambil menjaga urutan deterministik.
3. Ubah satu sumber daya menjadi `cacheScope: "private"` dengan `ttlMs: 0`, tambahkan kebijakan no-store di tingkat host, dan jelaskan ancaman yang membenarkan kedua kontrol itu.
4. Tambahkan langganan perubahan daftar prompt dan buktikan tidak ada event yang dikirim ketika filter mengabaikan `promptsListChanged`.
5. Buat dua langganan bersamaan dan buktikan setiap event membawa id permintaan yang benar.
6. Tambahkan subjek otorisasi pada handler pembacaan dan buktikan entri tembolok tidak dapat melintasi subjek.

## Istilah Kunci

- **Sumber daya:** Konten yang dialamatkan URI dan diekspos oleh server MCP.
- **Prompt:** Templat pesan yang dikendalikan pengguna dan diekspos oleh server MCP.
- **Daftar deterministik:** Hasil penemuan dengan keanggotaan dan urutan stabil untuk masukan permintaan yang sama.
- **`ttlMs`:** Lama kesegaran tembolok dalam milidetik.
- **`cacheScope`:** Batas pembagian untuk hasil yang ditembolokkan.
- **`subscriptions/listen`:** Permintaan berumur panjang yang aliran responsnya menyampaikan notifikasi yang difilter secara eksplisit.
- **Id langganan:** Id permintaan listen awal, yang diulang dalam metadata notifikasi.
- **Parameter tidak valid:** Kesalahan JSON-RPC `-32602`, dipakai untuk URI sumber daya yang tidak valid atau tidak dikenal.
- **Versi protokol yang tidak didukung:** Kesalahan JSON-RPC `-32022`, disertai revisi `supported` dan `requested`.
- **`server/discover`:** Metode server wajib yang mengembalikan revisi yang didukung, kapabilitas, identitas, dan petunjuk tembolok opsional.

## Bacaan Lanjutan

- [Sumber Daya MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/resources)
- [Prompt MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/prompts)
- [Langganan MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions)
- [Penembolokan MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/caching)
