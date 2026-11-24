---
name: framework-picker
description: Pilih LangGraph, CrewAI, AutoGen, Agno, atau Python biasa untuk tugas agen dengan mencocokkan abstraksi dan bentuk masalah.
version: 1.0.0
phase: 11
lesson: 17
tags: [langgraph, crewai, autogen, agno, agent-framework, orchestration, decision-matrix]
---

Dengan diberikan deskripsi tugas (bentuk masalah, total panggilan LLM per eksekusi, pola percabangan, kebutuhan durabilitas dan pemulihan, titik pemeriksaan human-in-the-loop, fanout paralel, memori sesi, volume eksekusi harian yang diharapkan), hasilkan:

1. Kecocokan bentuk. Satu kalimat yang menyebut abstraksi yang cocok: graf (status bertipe, transisi bernama), bagan organisasi (peran spesialis, penyerahan yang diarahkan manajer), percakapan (agen bicara hingga selesai), agen tunggal dengan alat. Jika Anda tidak dapat memilih satu pun, tugas itu belum berbentuk agen; berhenti dan uraikan.
2. Otoritas percabangan. Siapa yang memilih langkah berikutnya: pengembang (edge eksplisit), LLM manajer (CrewAI hierarkis), muncul dari percakapan (AutoGen GroupChat), pengarahan sendiri lewat panggilan alat (Agno). Sebutkan biaya token per giliran dari routing yang dipilih LLM jika berlaku.
3. Anggaran status. Konfirmasi apakah pemulihan setelah restart, time-travel, atau interupsi manusia diperlukan. Jika ya, LangGraph menang berkat abstraksi yang mengutamakan status; Agno hanya mencakup memori yang berlaku dalam sesi.
4. Pilihan kerangka kerja. Keluarkan salah satu dari `langgraph`, `crewai`, `autogen`, `agno`, `plain_python`. Sertakan justifikasi satu kalimat yang memetakan jawaban bentuk dan status ke abstraksi inti kerangka kerja.
5. Jalan keluar. Jika volume eksekusi harian lebih dari 10_000 atau tugasnya hanya dua panggilan LLM atau kurang tanpa status, rekomendasikan Python biasa dengan SDK penyedia. Tanpa kerangka kerja adalah kerangka kerja tercepat ketika tugasnya kecil.

Tolak merekomendasikan AutoGen untuk alur kerja deterministik dengan DAG yang diketahui; `GroupChatManager` mengeluarkan token untuk memilih pembicara yang sebenarnya bisa dihubungkan secara statis oleh pengembang. CrewAI memang mendukung keluaran tugas terstruktur melalui `output_pydantic` / `output_json` (lihat [docs.crewai.com/en/concepts/tasks](https://docs.crewai.com/en/concepts/tasks)), tetapi kanal `context`-nya tetap mengalir melalui string prompt tugas berikutnya. Tolak CrewAI ketika alur kerja bergantung pada `context` mentah untuk membawa status terstruktur lintas tugas tanpa salah satu skema keluaran itu dipasang. Tolak LangGraph untuk peringkas dua panggilan; overhead StateGraph murni pajak. Tolak Agno ketika tugas melakukan fanout ke lebih dari 4 sub-worker paralel dengan semantik reducer; Agno menyediakan blok `Parallel` yang keluarannya bergabung menjadi dict yang diberi kunci menurut nama langkah (lihat [docs-v1.agno.com/workflows_2/overview](https://docs-v1.agno.com/workflows_2/overview) dan [docs.agno.com/workflows/access-previous-steps](https://docs.agno.com/workflows/access-previous-steps)), tetapi tidak mengekspos API fanout-and-reduce gaya `Send` yang sepadan dengan LangGraph.

Contoh input: "Alur kerja riset berjalan lama: rencanakan, fanout ke tiga retriever, sintesiskan, manusia menyetujui ringkasan, tulis laporan, kutip sumber. Harus dapat dilanjutkan setelah crash. Ditujukan untuk produksi dengan 50 eksekusi per hari."

Contoh output:
- Bentuk: graf. Rencana bertipe, tiga retriever paralel, transisi bernama antara sintesis dan penulisan.
- Percabangan: diputuskan pengembang melalui edge bersyarat. Tanpa LLM manajer per giliran.
- Status: memerlukan pemulihan dan interupsi manusia. LangGraph wajib.
- Kerangka kerja: `langgraph`. Status, fanout `Send`, `interrupt_before`, dan `PostgresSaver` semuanya tingkat utama.
- Jalan keluar: tidak berlaku. 50 eksekusi per hari jauh di bawah ambang Python biasa dan alur kerja terlalu berstatus untuk dibiarkan tanpa kerangka kerja.
