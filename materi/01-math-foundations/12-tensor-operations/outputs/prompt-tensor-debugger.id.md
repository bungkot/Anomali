---
name: prompt-tensor-debugger
description: Prompt debugging langkah demi langkah untuk galat bentuk tensor dalam kode pembelajaran mendalam
phase: 1
lesson: 12
---

Saya mengalami galat bentuk tensor dalam kode pembelajaran mendalam saya. Bantu saya memperbaikinya.

**Pesan galat:** [tempel pesan galat di sini]

**Bentuk tensor saya:**
- [nama]: [bentuk]
- [nama]: [bentuk]

**Operasi yang ingin saya lakukan:** [jelaskan]

---

Saat melakukan debugging, ikuti proses ini persis:

**Langkah 1: Identifikasi jenis operasinya.**
Operasi apa yang menghasilkan galat tersebut? Petakan ke salah satu dari yang berikut:
- Perkalian matriks / lapisan Linear (dimensi dalam harus cocok)
- Broadcasting (ratakan dari kanan, setiap dimensi harus sama atau 1)
- Concatenation (semua dimensi cocok kecuali dimensi `cat`)
- Konvolusi (mengharapkan rank dan posisi kanal tertentu)
- Reshape (jumlah total elemen harus tetap)

**Langkah 2: Tulis kontrak bentuknya.**
Untuk operasi yang teridentifikasi, tulis bentuk yang diharapkan secara eksplisit:
```
matmul(A, B): A adalah (..., m, k), B adalah (..., k, n) -> (..., m, n)
broadcast(A, B): ratakan dari kanan, setiap pasangan harus (sama) atau (salah satunya 1)
cat([A, B], dim=d): semua dimensi cocok kecuali dimensi d
Linear(in_f, out_f): dimensi terakhir input harus sama dengan in_f
Conv2d(in_c, out_c, k): input harus (B, in_c, H, W)
```

**Langkah 3: Temukan ketidaksesuaiannya.**
Bandingkan bentuk aktual dengan kontrak. Identifikasi dimensi tepat yang melanggar aturan.

**Langkah 4: Pilih perbaikan minimal.**
Pilih dari tabel ini:

| Gejala | Perbaikan |
|---|---|
| Dimensi batch hilang | `.unsqueeze(0)` |
| Dimensi kanal hilang | `.unsqueeze(1)` |
| Dimensi berukuran 1 berlebih | `.squeeze(dim)` |
| Dimensi dalam salah untuk matmul | `.transpose(-1, -2)` atau periksa bentuk bobot |
| Perlu NCHW dari NHWC | `.permute(0, 3, 1, 2)` |
| Perlu NHWC dari NCHW | `.permute(0, 2, 3, 1)` |
| Ratakan dimensi spasial untuk linear | `.flatten(1)` atau `.reshape(B, -1)` |
| Pecah head: (B,T,D) ke (B,H,T,D/H) | `.reshape(B, T, H, D//H).transpose(1, 2)` |
| Gabungkan head: (B,H,T,D/H) ke (B,T,D) | `.transpose(1, 2).reshape(B, T, H*(D//H))` |
| Tensor non-kontigu dengan `.view()` | `.contiguous().view(...)` atau gunakan `.reshape(...)` |

**Langkah 5: Verifikasi perbaikannya.**
Tunjukkan bentuk yang dihasilkan pada setiap langkah. Pastikan jumlah total elemen tetap terjaga pada setiap reshape. Pastikan kontrak bentuk operasi kini terpenuhi.

**Langkah 6: Periksa bug senyap.**
Bahkan jika bentuk sudah cocok, verifikasi:
- Broadcasting terjadi pada sumbu yang dimaksud (tidak secara tidak sengaja)
- Reduksi menjumlahkan dimensi yang benar
- Dimensi batch (dim 0) tetap bertahan sepanjang seluruh forward pass
- Transpose + reshape digunakan (bukan hanya reshape) saat urutan dimensi penting

Format jawaban Anda sebagai:
```
OPERATION: [operasi apa yang gagal]
EXPECTED: [kontrak bentuk]
ACTUAL: [bentuk apa yang diberikan]
MISMATCH: [dimensi mana, mengapa]
FIX: [kode tepat]
RESULT: [bentuk setelah perbaikan]
```
