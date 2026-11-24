# Keandalan, Pembatalan, dan Kontrol Aliran MCP

> Sebuah ID permintaan mengorelasikan sebuah pesan. Ia tidak membuat efek samping menjadi aman, menghentikan pekerja, atau melindungi aliran dari konsumen yang lambat.

**Jenis:** Bangun
**Bahasa:** Python, TypeScript
**Prasyarat:** Fase 13, Pelajaran 09 dan 13
**Waktu:** ~120 menit

## Tujuan Pembelajaran

- Mengimplementasikan sinyal pembatalan yang tepat untuk stdio dan Streamable HTTP.
- Menyelesaikan balapan antara penyelesaian dan pembatalan tanpa mengirim pesan setelah pembatalan.
- Memisahkan pembatalan permintaan dari semantik `tasks/cancel` yang tahan lama (durable).
- Menyusun keputusan percobaan ulang berdasarkan efek samping dan kunci idempotensi yang eksplisit.
- Membatasi antrean progres sambil tetap mempertahankan respons akhir.
- Memulihkan aliran melalui penyambungan ulang, pengambilan ulang, dan backoff dengan jitter.

## Masalahnya

Jalur normal menyembunyikan bug sistem terdistribusi yang paling mahal.

Sebuah klien memanggil sebuah alat. Server mulai bekerja. Progres datang. Sebuah proxy menahan aliran dalam buffer. Klien mencapai batas waktunya dan terputus. Server menyelesaikan pekerjaan satu milidetik kemudian. Klien mencoba ulang dengan ID JSON-RPC yang baru. Mutasi dijalankan dua kali.

Setiap komponen berperilaku benar secara lokal. Sistem gagal secara global.

MCP mendefinisikan perilaku pesan dan transpor, tetapi aplikasi Anda tetap bertanggung jawab atas:

- anggaran waktu;
- idempotensi bisnis;
- antrean yang terbatas;
- klasifikasi percobaan ulang;
- state tugas yang tahan lama;
- kebijakan penyambungan ulang dan pengambilan ulang.

Pelajaran ini membangun keputusan-keputusan tersebut ke dalam sebuah simulator deterministik. Tidak ada `sleep`, soket, atau kegagalan acak. Anda mengontrol urutan peristiwa pembatalan secara langsung. Satu uji utas tersinkronisasi memaksa dua klien buku besar bersaing untuk kunci idempotensi yang sama.

## Pembatalan Permintaan Bergantung pada Transpor

Niatnya sama di setiap transpor: klien tidak lagi membutuhkan hasil dari permintaan yang sedang berjalan. Sinyal pada kawatnya berbeda.

### stdio

stdio menggunakan satu kanal bersama dua arah. Klien mengirim notifikasi:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/cancelled",
  "params": {
    "requestId": 41,
    "reason": "User closed the operation"
  }
}
```

Notifikasi ini bersifat fire-and-forget. Server tidak mengirim respons JSON-RPC untuk notifikasi tersebut.

Server sebaiknya menghentikan pekerjaan, membebaskan sumber daya, dan menghindari pengiriman respons untuk permintaan yang dibatalkan. Server boleh mengabaikan pembatalan apabila permintaannya tidak dikenal, sudah selesai, atau tidak dapat dihentikan dengan aman.

Notifikasi pembatalan yang rusak, tidak dikenal, atau menyangkut permintaan yang sudah selesai diabaikan. Mengubah balapan tersebut menjadi error baru justru akan menciptakan lebih banyak balapan.

### Streamable HTTP

Streamable HTTP modern memberi setiap permintaan respons HTTP atau aliran respons SSE-nya sendiri. Klien membatalkan dengan menutup aliran respons permintaan tersebut.

Jangan mengirim POST `notifications/cancelled` untuk permintaan HTTP biasa. Penutupan aliran adalah sinyal pembatalannya.

Begitu server mendeteksi pemutusan, server sebaiknya menghentikan pekerjaan dan **tidak boleh** mengirim pesan lagi untuk permintaan itu.

### Pembatalan yang dikirim server bersifat sempit

Server tidak menggunakan `notifications/cancelled` untuk membatalkan panggilan klien sembarangan. Pada stdio, pembatalan yang dikirim server dikhususkan untuk mengakhiri permintaan `subscriptions/listen`. Jaga jalur ini tetap terpisah dari pembatalan permintaan klien biasa.

## Pembatalan Adalah Balapan

Kedua urutan peristiwa berikut sama-sama valid.

### Pembatalan menang

```text
permintaan dimulai
klien mengirim sinyal pembatalan
server menandai permintaan sebagai dibatalkan
pekerja mencapai penyelesaian
server menekan respons
```

### Penyelesaian menang

```text
permintaan dimulai
pekerja menerapkan hasil
server mengirim respons
pembatalan datang terlambat
server mengabaikan notifikasi yang terlambat
```

Klien juga harus mengabaikan respons terlambat untuk permintaan yang sudah ia tinggalkan. Latensi jaringan berarti tidak ada pihak yang dapat membuktikan peristiwa mana yang lebih dilihat oleh pihak lainnya.

```figure
mcp-reliability-race
```

`RequestCoordinator` dalam pelajaran ini menyimpan satu state terminal. `complete()` tidak mengembalikan respons setelah pembatalan. Pembatalan yang terlambat tidak dapat mengubah catatan yang sudah selesai.

## Batas Waktu Memerlukan Dua Jam

Satu penghitung waktu diam (inactivity timer) saja tidak cukup.

Gunakan dua batas:

1. **Batas waktu diam (idle timeout).** Berapa lama permintaan boleh tidak menghasilkan aktivitas berguna.
2. **Batas waktu maksimum.** Anggaran waktu absolut sejak permintaan dimulai.

Progres boleh mengatur ulang jam diam. Progres tidak boleh pernah menghapus batas waktu maksimum.

```text
mulai: 0 ms
progres: 400 ms
progres: 800 ms
progres: 1200 ms
batas waktu diam: 500 ms
batas waktu maksimum: 2000 ms
```

Pada 1500 ms, permintaan masih aktif karena progres terakhir baru berusia 300 ms. Pada 2000 ms, batas maksimum membatalkannya meskipun peristiwa progres lain tiba pada 1999 ms.

Progres bersifat opsional. Server boleh menerima token progres tanpa mengirim pembaruan apa pun. Jangan pernah mengubah keberadaan token menjadi batas waktu tak hingga.

Nilai progres MCP harus meningkat. Notifikasi berhenti setelah penyelesaian atau pembatalan. Beri batas laju progres agar pekerja yang cepat tidak membanjiri transpor.

## Pembatalan Permintaan Bukan `tasks/cancel`

Kedua mekanisme ini menyelesaikan masa hidup yang berbeda.

| Mekanisme | Target | Sinyal | Arti keberhasilan |
|-----------|--------|--------|-------------------|
| Pembatalan permintaan pada stdio | Satu RPC yang sedang berjalan | `notifications/cancelled` | Klien meninggalkan permintaan; server sebaiknya berhenti jika memungkinkan |
| Pembatalan permintaan pada HTTP | Satu aliran respons yang sedang berjalan | Tutup aliran | Klien meninggalkan permintaan; server sebaiknya berhenti jika memungkinkan |
| `tasks/cancel` | Satu Task yang tahan lama | Permintaan MCP biasa | Server mengakui niat pembatalan |

Hasil `tasks/cancel` yang berhasil tidak membuktikan bahwa pekerja telah berhenti. Task mungkin tetap berstatus `working` hingga sebuah checkpoint pekerja melihat flag tersebut. Pekerjaan mungkin selesai sebelum checkpoint itu.

Jangan menghapus state task yang tahan lama ketika koneksi HTTP tertutup. Alasan dibuatnya Task adalah agar masa hidupnya melampaui satu permintaan dan satu koneksi.

## ID JSON-RPC Baru Bukan Idempotensi

ID JSON-RPC mengorelasikan permintaan dan respons. ID tersebut tidak mengidentifikasi operasi bisnis.

Misalkan seorang klien mengirim transaksi pembayaran dengan id `41`, kehilangan responsnya, lalu mencoba ulang dengan id `42`. Server melihat dua pesan yang berbeda. Tanpa kunci aplikasi, server tidak dapat mengetahui bahwa keduanya mewakili satu proses checkout.

Kunci idempotensi mengidentifikasi niat bisnis:

```json
{
  "name": "charge_account",
  "arguments": {
    "account": "acct-7",
    "cents": 1200,
    "idempotencyKey": "checkout-7"
  }
}
```

Server menyimpan:

- kunci tersebut;
- sidik jari (fingerprint) dari argumen operasi;
- hasil yang sudah di-commit.

Kunci yang sama dengan argumen yang sama mengembalikan hasil yang tersimpan. Kunci yang sama dengan argumen yang berbeda ditolak. Ini mencegah penggunaan ulang kunci yang tidak sengaja mengubah operasi bisnis lain.

### Batas buku besar harus atomik dan tahan lama

Urutan berikut tidak aman:

```text
periksa kunci
jalankan mutasi
simpan hasil
```

Dua pekerja dapat sama-sama melihat kunci yang belum ada dan sama-sama menjalankan mutasi. Kecelakaan setelah efek tetapi sebelum penyimpanan menciptakan ambiguitas yang sama saat percobaan ulang.

Pelajaran ini menggunakan buku besar SQLite berbasis berkas. `BEGIN IMMEDIATE` menyerialisasi pemeriksaan kunci, efek bisnis simulasi, penghitung eksekusi, dan hasil yang disimpan ke dalam satu transaksi. Dua koneksi buku besar independen yang berbalapan dengan kunci yang sama oleh karena itu melihat satu hasil yang di-commit dan satu eksekusi. Menutup dan membuka kembali buku besar mempertahankan catatan tersebut.

Setiap nilai kembalian direkonstruksi dari JSON yang tersimpan. Pemanggil tidak pernah menerima objek mutable yang dipegang oleh buku besar, sehingga mengubah dictionary yang dikembalikan tidak dapat merusak hasil replay berikutnya.

Efek bisnis simulator adalah kwitansi dan penghitung eksekusi di dalam transaksi SQLite yang sama. Pembayaran, deployment, atau panggilan API eksternal yang nyata tidak menjadi atomik hanya karena Anda menulis tabel lokal. Produksi memerlukan transaksi basis data bersama yang tahan lama, transactional outbox, atau penyedia upstream yang memberlakukan kunci idempotensi yang sama. Kunci proses (process lock) saja tidak melindungi beberapa replika atau bertahan setelah restart.

### Matriks percobaan ulang

Klasifikasikan percobaan ulang sebelum mengimplementasikannya.

| Kelas | Contoh | Aturan percobaan ulang |
|------|---------|------------------------|
| Aman | Pembacaan deterministik tanpa efek samping | Coba ulang dengan ID JSON-RPC baru setelah batas kegagalan dipahami |
| Bersyarat | Mutasi dengan kunci idempotensi tahan lama | Coba ulang dengan kunci yang sama dan argumen yang identik |
| Tidak aman | Mutasi tanpa deduplikasi bisnis | Jangan mencoba ulang otomatis; rekonsiliasi terlebih dahulu |

Anotasi alat seperti `readOnlyHint` dan `idempotentHint` tetap merupakan petunjuk yang tidak dipercaya begitu saja. Kontrak aplikasi dan implementasi server yang menentukan keamanan percobaan ulang.

## Backpressure Bagian dari Keperluan (Correctness)

Produsen SSE dapat menghasilkan progres lebih cepat daripada yang dapat dikonsumsi klien, proxy, atau jaringan. Antrean tanpa batas mengubah kelambatan menjadi kehabisan memori.

Gunakan antrean yang dibatasi dan tentukan apa yang boleh hilang.

Progres dapat digantikan. Nilai progres yang lebih baru menggantikan nilai progres sebelumnya untuk token yang sama. Respons JSON-RPC akhir tidak dapat digantikan.

Buffer dalam pelajaran ini menerapkan kebijakan berikut:

1. Gabungkan progres berdekatan untuk token yang sama.
2. Buang progres tertua ketika kapasitas penuh.
3. Tandai aliran sebagai memerlukan pengambilan ulang otoritatif (authoritative refetch).
4. Pertahankan respons akhir.
5. Tolak state apa pun yang mempertahankan respons akhir dengan cara membuang respons akhir lainnya.

Ini adalah kehilangan yang terbatas dengan pemulihan yang eksplisit. Kehilangan diam-diam bukanlah strategi.

### Penyanggaan proxy

Server dapat melakukan streaming dengan benar sementara proxy terbalik menahan peristiwa di dalam buffer.

Untuk respons SSE, kirim:

```http
Content-Type: text/event-stream
Cache-Control: no-cache
X-Accel-Buffering: no
```

Spesifikasi Streamable HTTP tahun 2026 merekomendasikan `X-Accel-Buffering: no` agar proxy yang kompatibel mengirimkan peristiwa segera.

Untuk aliran yang sepi dan berumur panjang, kirimkan secara berkala komentar SSE:

```text
:
```

Klien mengabaikan baris komentar. Perantara melihat lalu lintas sehingga lebih kecil kemungkinannya menutup koneksi yang menganggur.

Keepalive bukanlah progres. Jangan mengatur ulang batas waktu diam semantik suatu operasi hanya karena komentar transpor tiba.

## Menyambung Kembali Berarti Mengambil Ulang

Streamable HTTP modern tidak mendukung SSE yang dapat dilanjutkan melalui `Last-Event-ID`.

Setelah aliran `subscriptions/listen` terputus:

1. Buka permintaan listen baru dengan ID JSON-RPC yang baru.
2. Pulihkan filter langganan yang diinginkan.
3. Ambil ulang alat, sumber daya, prompt, atau Task yang terdampak dari metode otoritatif.
4. Hilangkan duplikasi state aplikasi berdasarkan pengenal yang stabil.
5. Jangan memutar ulang mutasi yang tidak aman hanya karena responsnya hilang.

Rencana pemulihan contoh secara eksplisit menetapkan `sendLastEventId` ke false dan mencantumkan sumber daya yang akan diambil ulang.

### Cegah kawanan penyambungan ulang

Jika 10.000 klien menyambung ulang tepat pada satu detik yang sama, server yang sedang pulih akan gagal lagi.

Gunakan exponential backoff dengan jitter dan batas atas. Pelajaran ini menghitung jitter deterministik dari ID klien dan nomor percobaan agar uji tetap dapat direproduksi:

```text
percobaan 0: hingga 250 ms
percobaan 1: hingga 500 ms
percobaan 2: hingga 1000 ms
...
batas atas: 8000 ms
```

Produksi dapat menggunakan randomness yang aman secara kriptografis atau randomness runtime. Invariannya adalah distribusi, bukan rumus tertentu.

## Bangunlah

`code/main.py` membangun lima komponen keandalan kecil.

### `RequestCoordinator`

- memulai permintaan yang sedang berjalan dengan batas diam dan batas maksimum;
- menghasilkan notifikasi progres yang monoton;
- menghasilkan sinyal pembatalan stdio atau HTTP yang tepat;
- mengabaikan notifikasi pembatalan yang tidak valid;
- membuat balapan terminal antara pembatalan dan penyelesaian menjadi eksplisit;
- menghususkan pembatalan yang dikirim server untuk langganan stdio.

### `MutationLedger`

- membuktikan bahwa dua ID JSON-RPC menjalankan mutasi dua kali tanpa kunci bisnis;
- menggunakan transaksi SQLite berbasis berkas untuk pemeriksaan kunci, efek simulasi, penghitung eksekusi, dan commit hasil;
- menghilangkan duplikasi argumen yang cocok di bawah satu kunci idempotensi lintas koneksi buku besar yang independen;
- menolak satu kunci yang dipakai ulang dengan argumen berbeda;
- mengembalikan salinan defensif dan mempertahankan catatan yang di-commit setelah dibuka kembali.

### `DurableTaskService`

- mengakui permintaan pembatalan;
- mempertahankan task tetap `working` hingga checkpoint pekerja;
- menunjukkan mengapa pengakuan bukanlah status akhir.

### `BoundedSseBuffer`

- menggabungkan atau membuang progres di bawah tekanan;
- mencatat bahwa pengambilan ulang otoritatif diperlukan;
- tidak pernah membuang respons akhir.

### Helper pemulihan

- mengembalikan header SSE yang aman terhadap proxy dan komentar keepalive;
- membuat rencana penyambungan ulang dan pengambilan ulang;
- menyebarkan percobaan ulang dengan exponential backoff dan jitter deterministik.

## Gunakanlah

Dari akar repositori:

```bash
cd phases/13-tools-and-protocols/29-mcp-reliability-cancellation-and-flow-control/code
python3 main.py
npx tsx main.ts
python3 -m unittest discover tests -v
```

Demo ini menjalankan kedua sisi dari balapan utama, mengeksekusi mutasi yang dideduplikasi secara transaksional dalam buku besar sementara berbasis berkas, membebani buffer progres yang dibatasi, dan menunjukkan Task tahan lama yang bertransisi dari pengakuan pembatalan menjadi pembatalan yang teramati pekerja.

## Laboratorium Interaktif

Jalankan empat urutan peristiwa tanpa menambahkan `sleep`.

1. Mulai permintaan `A`, batalkan, lalu panggil `complete()`.
2. Mulai permintaan `B`, selesaikan, lalu kirim pembatalan.
3. Mulai permintaan `C`, kirim progres sebelum setiap batas diam, lalu lewati batas maksimum.
4. Mulai permintaan `D` melalui Streamable HTTP dan tutup aliran responsnya.

Catat untuk setiap skenario:

- state terminal permintaan;
- apakah respons akhir ada;
- sinyal pembatalan yang ditempatkan pada kawat;
- peristiwa mana yang sebaiknya diabaikan klien.

Kemudian ubah `D` menjadi stdio. Operasinya identik, tetapi sinyal pembatalannya harus berubah.

## Laboratorium Praktik

Tambahkan mutasi `reserve_inventory` ke `MutationLedger`.

Persyaratan:

1. Kunci mengikat SKU, kuantitas, tenant, dan nama operasi.
2. Percobaan ulang dengan kunci dan argumen yang sama mengembalikan reservasi pertama.
3. Percobaan ulang dengan kuantitas yang diubah gagal tanpa reservasi tambahan.
4. Eksekusi yang sudah di-commit tetapi kehilangan respons dapat direkonsiliasi berdasarkan kunci.
5. Hasil tidak mencatat data rahasia atau data pembayaran.
6. Percobaan ulang otomatis dinonaktifkan ketika klien tidak memberikan kunci.
7. Tambahkan simulasi pemutusan langganan dan ambil ulang catatan inventaris sebelum memutuskan langkah berikutnya.
8. Mulai dua koneksi buku besar pada sebuah barrier dan kirim kunci yang sama secara konkuren. Tegaskan bahwa satu reservasi di-commit.
9. Ubah objek reservasi pertama yang dikembalikan. Putar ulang kuncinya dan buktikan bahwa hasil tersimpan tidak berubah.
10. Tutup dan buka kembali berkas buku besar, lalu rekonsiliasi reservasi berdasarkan kunci.

Jaga agar laboratorium tetap jujur: jika inventaris berada di layanan lain, jelaskan apakah layanan tersebut menerima kunci idempotensi yang sama atau apakah transactional outbox menjembatani commit lokal dengan efek jarak jauh.

## Artefak yang Dihasilkan

`outputs/skill-mcp-reliability-reviewer.md` adalah skill tinjauan keandalan yang sederhana. Berikan sebuah operasi MCP, transpor, kebijakan batas waktu, perilaku percobaan ulang, kebijakan antrean, dan rencana pemulihan. Skill ini mengembalikan tabel balapan, klasifikasi percobaan ulang, batas idempotensi, pemeriksaan kontrol aliran, dan fixture kegagalan.

## Verifikasikanlah

Pelajaran selesai ketika pernyataan-pernyataan berikut benar:

- Pembatalan stdio mengirim `notifications/cancelled` dan tidak menerima respons.
- Pembatalan Streamable HTTP menutup aliran permintaan dan tidak mengirim POST pembatalan.
- Pembatalan sebelum penyelesaian menekan respons akhir.
- Penyelesaian sebelum pembatalan mempertahankan respons dan mengabaikan pembatalan yang terlambat.
- Progres dapat mengatur ulang batas waktu diam tetapi tidak pernah batas waktu maksimum.
- ID JSON-RPC yang baru saja menyebabkan mutasi dijalankan kembali.
- Satu kunci idempotensi dan argumen identik dijalankan sekali dalam balapan konkuren dua koneksi.
- Catatan yang di-commit bertahan setelah dibuka kembali dan replay mengembalikan salinan defensif.
- Mengubah salah satu hasil yang dikembalikan tidak dapat mengubah hasil yang tersimpan.
- Buffer yang dibatasi tetap berada dalam kapasitas dan mempertahankan respons akhir.
- Penyambungan ulang menggunakan permintaan baru, tidak mengirim `Last-Event-ID`, dan mengambil ulang state yang terdampak.
- Pengakuan `tasks/cancel` membuat task tetap tidak terminal hingga pekerja melihatnya.

## Mode Kegagalan Produksi

| Kegagalan | Gejala yang teramati | Respons yang tepat |
|-----------|----------------------|--------------------|
| Klien HTTP mengirim POST notifikasi pembatalan | Server dan klien tidak sepakat tentang masa hidup permintaan | Tutup aliran respons SSE permintaan tersebut |
| Server merespons setelah pembatalan diterima | Klien menerima hasil terlambat yang tidak dapat dipakai | Hentikan pekerjaan dan tekan pesan selanjutnya ketika pembatalan menang |
| Progres mengatur ulang setiap batas waktu | Pekerjaan yang macet bertahan selamanya | Pertahankan batas waktu maksimum absolut yang terpisah |
| ID RPC baru dianggap sebagai deduplikasi | Pembayaran, deployment, atau penghapusan dijalankan dua kali | Tambahkan kunci idempotensi aplikasi yang tahan lama |
| Pemeriksaan kunci dan efek terpisah | Dua pekerja konkuren sama-sama melihat kunci yang belum ada | Commit klaim kunci, catatan efek, dan hasil secara atomik |
| Buku besar dalam memori dipakai lintas replika | Restart atau pekerja lain melupakan commit sebelumnya | Gunakan penyimpanan bersama yang tahan lama atau idempotensi upstream |
| Hasil mutable tersimpan dikembalikan langsung | Mutasi oleh pemanggil merusak replay berikutnya | Serialisasi hasil yang di-commit dan kembalikan salinan defensif |
| Kunci dipakai ulang dengan argumen berbeda | Satu kunci mewakili dua niat bisnis | Simpan dan bandingkan sidik jari argumen |
| Antrean progres tanpa batas | Memori meningkat bersama konsumen yang lambat | Gabungkan dan buang progres yang dapat digantikan dalam batas tertentu |
| Respons akhir dibuang di bawah tekanan | Klien tidak dapat mengetahui hasil permintaan | Sisihkan kapasitas atau buang progres, jangan pernah respons akhir |
| Proxy menyangga SSE | Progres tiba dalam kelompok atau setelah batas waktu | Nonaktifkan penyanggaan dan konfigurasikan batas waktu proxy yang sesuai |
| `Last-Event-ID` dianggap didukung | Klien melanjutkan dari state yang tidak didukung server | Sambung ulang dengan permintaan baru dan ambil ulang |
| Setiap klien menyambung ulang segera | Pemulihan menciptakan pemadaman baru | Gunakan exponential backoff dengan batas atas dan jitter |
| Ack task dianggap sebagai pembatalan akhir | Pekerja terus berjalan setelah UI menyatakan berhenti | Poll Task hingga mencapai status terminal |

## Kaitan dengan Proyek Akhir (Capstone)

Proyek akhir ekosistem alat sebaiknya memperlakukan keandalan sebagai bukti yang dapat dijalankan, bukan sekadar paragraf dalam diagram arsitektur.

Wajibkan artefak-artefak berikut:

- satu transkrip balapan pembatalan untuk setiap transpor;
- tabel percobaan ulang untuk setiap mutasi yang diekspos;
- catatan kunci idempotensi dan fixture ketidaksesuaian;
- transkrip konkuren kunci yang sama, pemeriksaan pembukaan ulang, dan pemeriksaan alias mutasi;
- hasil kelebihan beban buffer yang dibatasi;
- header SSE proxy terbalik dan kebijakan diam;
- rencana penyambungan ulang yang menyebut metode pengambilan ulang otoritatif;
- jejak pembatalan Task tahan lama apabila proyek akhir menggunakan Task.

Permintaan yang berhasil dalam proses lokal hanya membuktikan jalur normal. Proyek akhir siap produksi ketika respons yang hilang, pembatalan terlambat, konsumen yang lambat, dan kawanan penyambungan ulang memiliki hasil yang deterministik.

## Istilah Kunci

| Istilah | Arti |
|---------|------|
| Pembatalan permintaan | Penyetelan (abandonment) satu permintaan MCP yang sedang berjalan |
| Balapan pembatalan | Persaingan antara peristiwa penyelesaian terminal dan pembatalan |
| Batas waktu diam | Batas sejak aktivitas berguna terakhir pada permintaan |
| Batas waktu maksimum | Batas absolut sejak permintaan dimulai, tidak dipengaruhi progres |
| Kunci idempotensi | Pengenal aplikasi yang menghilangkan duplikasi satu niat bisnis |
| Buku besar atomik | Batas tahan lama yang meng-commit klaim kunci, catatan efek, dan hasil sebagai satu kesatuan |
| Backpressure | Kontrol yang diterapkan ketika produsen melampaui konsumen |
| Penggabungan progres | Penggantian progres lama dengan nilai otoritatif yang lebih baru |
| Pengambilan ulang (refetch) | Pembacaan state terkini lagi setelah jeda aliran |
| Jitter | Variasi yang disengaja untuk menyebarkan percobaan ulang sepanjang waktu |

## Bacaan Lanjutan

- [MCP Cancellation](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/cancellation)
- [MCP Progress](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/progress)
- [MCP Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [MCP Tasks Extension](https://tasks.extensions.modelcontextprotocol.io/specification/draft/tasks)
