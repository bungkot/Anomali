# Model keputusan primitif

Pilih berdasarkan tanggung jawab. Bisa jadi lebih dari satu primitif yang tepat.

| Kebutuhan | Primitif | Batas |
|---|---|---|
| Instruksi sekali jalan | Prompt | Hanya ada untuk interaksi saat ini |
| Default seluruh repositori | AGENTS.md | Berlaku selama bekerja dalam cakupan repositori itu |
| Metode tugas yang dapat dipakai ulang | Agent Skill | Memuat pengetahuan prosedural untuk sebuah tugas |
| Operasi atau data eksternal | Alat MCP | Menyediakan kapabilitas yang dapat dipanggil dengan kontrak masukan |
| Reaksi terhadap peristiwa runtime | Hook | Berjalan pada titik siklus hidup yang ditentukan host |
| Transformasi deterministik | Kode biasa | Menghasilkan keluaran yang dapat diulang tanpa penilaian model |
| Konteks terisolasi atau paralel | Subagen | Mendelegasikan tugas terbatas ke jendela konteks terpisah |

Metode tinjauan rilis yang mengueri layanan jarak jauh dapat memakai Agent Skill sekaligus alat MCP. Konvensi pengujian repositori dapat menambahkan AGENTS.md. Audit pasca-alat dapat menambahkan hook. Penguraian yang stabil sebaiknya berada di kode biasa. Riset mandiri yang mendapat manfaat dari isolasi konteks dapat memakai subagen.
