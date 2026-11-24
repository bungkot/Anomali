---
name: skill-invocation-router
description: Rancang dan uji kebijakan aktivasi eksplisit manusia, implisit model atau agen, aplikasi programatik, komposisi skill yang terikat batas, dan aktivasi harness untuk katalog Agent Skill.
license: MIT
metadata:
  lesson: "25"
---

# Router pemanggilan skill

Gunakan skill ini ketika sebuah host memerlukan kebijakan aktivasi yang dapat diaudit, bukan satu flag `invocable` yang tidak membedakan.

1. Baca `references/invocation-model.md` dan klasifikasikan kanal yang diminta.
2. Tinjau `assets/host-policy.json` sebagai contoh konfigurasi adapter, bukan standar yang portabel.
3. Jalankan `python3 scripts/simulate_invocation.py --policy assets/host-policy.json --actor ACTOR --name NAME --description DESCRIPTION --query QUERY [--explicit-name NAME] [--caller-name NAME] [--depth N] [--user-invocable true|false] [--disable-model-invocation true|false]`.
4. Untuk permintaan dari manusia, aplikasi, skill, atau harness, wajibkan nama yang ditemukan persis dan daftar izin (allowlist) yang spesifik untuk kanalnya.
5. Untuk pemanggil skill, wajibkan pula identitas pemanggil, target yang tidak siklik, dan kedalaman komposisi yang terikat batas.
6. Untuk permintaan dari model atau agen otonom, singkirkan kandidat yang dinyatakan tidak memenuhi syarat oleh aktor atau oleh ekstensi host yang dikenali.
7. Beri skor hanya pada deskripsi yang tersisa. Pilih kecocokan yang memenuhi syarat dan terkuat, atau abstain jika tidak ada kandidat memenuhi syarat yang melampaui ambang batas.
8. Kembalikan keputusan JSON beserta adapter, kanal, skor, dan alasan kebijakan.

Aktivasi memuat instruksi. Aktivasi tidak menyetujui alat, perubahan sistem berkas, akses jaringan, penggunaan rahasia, maupun skrip yang dibundling.
