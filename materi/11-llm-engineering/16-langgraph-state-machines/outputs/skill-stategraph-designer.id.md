---
name: stategraph-designer
description: Ubah tugas agen menjadi StateGraph LangGraph dengan node bernama, status bertipe, reducer, checkpointer, dan interupsi manusia.
version: 1.0.0
phase: 11
lesson: 16
tags: [langgraph, stategraph, checkpointer, interrupt, time-travel, react-agent, human-in-the-loop]
---

Dengan diberikan tugas agen (tujuan yang menghadap pengguna, alat yang tersedia, perkiraan jumlah giliran, efek samping dengan radius dampak keamanan, persyaratan durabilitas, anggaran latensi target), hasilkan:

1. Daftar node. Beri nama setiap langkah diskrit: pemikir LLM, setiap pelaksana alat, setiap langkah peninjauan manusia, peringkas atau kritikus apa pun, retriever apa pun. Tolak desain jika ada node yang menyentuh lebih dari satu perhatian; pisahkan.
2. Skema status. Field TypedDict (atau Pydantic) dengan reducer untuk setiap daftar. Selalu gunakan `Annotated[list, add_messages]` pada log pesan. Angkat daftar khusus tugas apa pun keluar dari `messages` (sebuah `plan`, penghitung `budget`, daftar `retrieved_docs`) agar reducer tetap benar di bawah pembaruan paralel.
3. Peta edge. Edge statis di mana langkah berikutnya deterministik. Edge bersyarat dengan fungsi router bernama hanya di mana model memilih langkah berikutnya. Tolak graf mana pun yang fungsi routernya bergantung pada panggilan LLM baru yang belum Anda lakukan di node sebelumnya.
4. Penempatan interupsi. `interrupt_before` pada setiap node dengan efek samping yang tidak dapat dibatalkan (tulis, hapus, pembayaran, panggilan API eksternal yang berbiaya). `interrupt_after` pada node model ketika validasi keluaran berjalan di proses terpisah. Tolak `interrupt_after` pada node yang ber-efek samping; pada saat itu efek samping sudah terjadi.
5. Checkpointer. `MemorySaver` hanya untuk pengujian. Pilih dari `PostgresSaver`, `SQLiteSaver`, `RedisSaver` untuk lingkungan apa pun yang harus bertahan setelah restart. Konfirmasi strategi `thread_id` (per pengguna, per sesi, per percakapan) dan TTL checkpoint.

Tolak merilis LangGraph tanpa checkpointer. Tanpa checkpointer berarti tanpa lanjut, tanpa time-travel, tanpa pemutaran ulang human-in-the-loop. Tolak merilis field `messages` tanpa `add_messages`; tulisan kedua menimpa yang pertama secara diam-diam dan separuh percakapan hilang. Tolak graf yang setiap transisinya adalah edge bersyarat yang diarahkan oleh LLM planner; itu AutoGen dengan langkah ekstra dan membakar token per giliran.

Contoh input: "Agen penanganan pengembalian dana di atas Anthropic Claude dengan tiga alat (`lookup_order`, `issue_refund`, `send_email`), harus berhenti untuk manusia sebelum pengembalian dana apa pun di atas 100 dolar, harus dapat dilanjutkan setelah restart server, anggaran latensi p95 8 detik."

Contoh output:
- Node: `agent` (panggilan LLM), `lookup_tool`, `refund_tool`, `email_tool`, `human_review`.
- Status: `messages` dengan `add_messages`, `order_context` (overwrite), `refund_amount` (overwrite), `reviewer_decision` (overwrite).
- Edge: `agent` ke router `should_continue` dengan cabang `lookup_tool`, `refund_tool`, `email_tool`, `human_review`, `END`. Node alat kembali ke `agent`.
- Interupsi: `interrupt_before` pada `refund_tool` ketika `refund_amount > 100`. Tanpa interupsi pada `lookup_tool` atau `email_tool`.
- Checkpointer: `PostgresSaver` dengan `thread_id` "user:{user_id}:case:{case_id}" dan TTL 30 hari.
