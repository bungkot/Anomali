# Vektor, Matriks, dan Operasi

> Setiap jaringan saraf hanyalah perkalian matriks dengan langkah tambahan.

**Jenis:** Bangun
**Bahasa:** Python, Julia
**Prasyarat:** Fase 1, Pelajaran 01 (Intuisi Aljabar Linear)
**Waktu:** ~60 menit

## Tujuan Pembelajaran

- Membangun kelas Matrix dengan operasi elemen-demi-elemen, perkalian matriks, transpose, determinan, dan invers
- Membedakan perkalian elemen-demi-elemen dari perkalian matriks serta menjelaskan kapan masing-masing digunakan
- Menerapkan satu lapisan jaringan saraf rapat (`relu(W @ x + b)`) hanya dengan kelas Matrix buatan sendiri
- Menjelaskan aturan broadcasting dan cara kerja penambahan bias pada framework jaringan saraf

## Masalah

Anda ingin membangun jaringan saraf. Anda membaca kodenya dan melihat ini:

```
output = activation(weights @ input + bias)
```

`@` itu adalah perkalian matriks. `weights` adalah matriks. `input` adalah vektor. Jika Anda tidak tahu apa yang dilakukan operasi-operasi itu, baris ini terasa seperti sihir. Jika Anda tahu, baris itu adalah seluruh jaluran maju (forward pass) sebuah lapisan dalam tiga operasi.

Setiap gambar yang diproses model Anda adalah matriks nilai piksel. Setiap embedding kata adalah vektor. Setiap lapisan dari setiap jaringan saraf adalah transformasi matriks. Anda tidak dapat membangun sistem AI tanpa lancar dalam operasi matriks, sama seperti Anda tidak dapat menulis kode tanpa memahami variabel.

Pelajaran ini membangun kelancaran itu dari awal.

## Konsep

### Vektor: daftar bilangan terurut

Vektor adalah daftar bilangan yang memiliki arah dan magnitudo. Dalam AI, vektor mewakili titik data, fitur, atau parameter.

```
v = [3, 4]        -- vektor 2D
w = [1, 0, -2]    -- vektor 3D
```

Vektor 2D `[3, 4]` menunjuk ke koordinat (3, 4) pada sebuah bidang. Panjangnya (magnitudo) adalah 5 (segitiga 3-4-5).

### Matriks: kisi-kisi bilangan

Matriks adalah kisi 2D: baris dan kolom. Matriks $m \times n$ memiliki $m$ baris dan $n$ kolom.

```math
A = \begin{bmatrix} 1 & 2 & 3 \\ 4 & 5 & 6 \end{bmatrix} \quad \text{matriks 2x3 (2 baris, 3 kolom)}
```

Pada jaringan saraf, matriks bobot mengubah vektor masukan menjadi vektor keluaran. Sebuah lapisan dengan 784 masukan dan 128 keluaran memakai matriks bobot 128x784.

### Mengapa bentuk (shape) penting

Perkalian matriks punya aturan ketat: `(m x n) @ (n x p) = (m x p)`. Dimensi dalam harus sama.

```
(128 x 784) @ (784 x 1) = (128 x 1)
  bobot         masukan     keluaran

Dimensi dalam: 784 = 784  -- valid
```

Jika Anda mendapat error ketidaksesuaian bentuk di PyTorch, inilah penyebabnya.

### Peta operasi

| Operasi | Yang dilakukan | Penggunaan pada jaringan saraf |
|---------|----------------|--------------------------------|
| Penjumlahan | Menggabungkan elemen-demi-elemen | Menambahkan bias ke keluaran |
| Perkalian skalar | Menskalakan setiap elemen | Laju pembelajaran * gradien |
| Perkalian matriks | Mengubah vektor | Jaluran maju lapisan |
| Transpose | Menukar baris dan kolom | Propagasi balik |
| Determinan | Ringkasan berupa satu bilangan | Memeriksa ke-inversibelan |
| Invers | Membatalkan transformasi | Menyelesaikan sistem linear |
| Identitas | Matriks yang tidak melakukan apa-apa | Inisialisasi, koneksi residual |

### Perkalian elemen-demi-elemen vs perkalian matriks

Pembedaan ini terus menjerumuskan pemula.

Elemen-demi-elemen: kalikan posisi yang bersesuaian. Kedua matriks harus memiliki bentuk yang sama.

```math
\begin{bmatrix} 1 & 2 \\ 3 & 4 \end{bmatrix} * \begin{bmatrix} 5 & 6 \\ 7 & 8 \end{bmatrix} = \begin{bmatrix} 5 & 12 \\ 21 & 32 \end{bmatrix}
```

Perkalian matriks: produk titik antara baris dan kolom. Dimensi dalam harus sama.

```math
\begin{bmatrix} 1 & 2 \\ 3 & 4 \end{bmatrix} @ \begin{bmatrix} 5 & 6 \\ 7 & 8 \end{bmatrix} = \begin{bmatrix} 1 \cdot 5 + 2 \cdot 7 & 1 \cdot 6 + 2 \cdot 8 \\ 3 \cdot 5 + 4 \cdot 7 & 3 \cdot 6 + 4 \cdot 8 \end{bmatrix} = \begin{bmatrix} 19 & 22 \\ 43 & 50 \end{bmatrix}
```

Operasi berbeda, hasil berbeda, aturan berbeda.

### Broadcasting

Saat Anda menambahkan vektor bias ke matriks keluaran, bentuknya tidak sama. Broadcasting meregangkan larik yang lebih kecil agar sesuai.

```
| 1  2  3 |   +   [10, 20, 30]
| 4  5  6 |

Broadcasting meregangkan vektor melintasi baris:

| 1  2  3 |   | 10  20  30 |   | 11  22  33 |
| 4  5  6 | + | 10  20  30 | = | 14  25  36 |
```

Setiap framework modern melakukan ini secara otomatis. Memahaminya mencegah kebingungan saat bentuk tampak salah tetapi kode tetap berjalan.

```figure
vector-projection
```

## Bangun dari Awal

### Langkah 1: Kelas Vector

```python
class Vector:
    def __init__(self, data):
        self.data = list(data)
        self.size = len(self.data)

    def __repr__(self):
        return f"Vector({self.data})"

    def __add__(self, other):
        return Vector([a + b for a, b in zip(self.data, other.data)])

    def __sub__(self, other):
        return Vector([a - b for a, b in zip(self.data, other.data)])

    def __mul__(self, scalar):
        return Vector([x * scalar for x in self.data])

    def dot(self, other):
        return sum(a * b for a, b in zip(self.data, other.data))

    def magnitude(self):
        return sum(x ** 2 for x in self.data) ** 0.5
```

### Langkah 2: Kelas Matrix dengan operasi inti

```python
class Matrix:
    def __init__(self, data):
        self.data = [list(row) for row in data]
        self.rows = len(self.data)
        self.cols = len(self.data[0])
        self.shape = (self.rows, self.cols)

    def __repr__(self):
        rows_str = "\n  ".join(str(row) for row in self.data)
        return f"Matrix({self.shape}):\n  {rows_str}"

    def __add__(self, other):
        return Matrix([
            [self.data[i][j] + other.data[i][j] for j in range(self.cols)]
            for i in range(self.rows)
        ])

    def __sub__(self, other):
        return Matrix([
            [self.data[i][j] - other.data[i][j] for j in range(self.cols)]
            for i in range(self.rows)
        ])

    def scalar_multiply(self, scalar):
        return Matrix([
            [self.data[i][j] * scalar for j in range(self.cols)]
            for i in range(self.rows)
        ])

    def element_wise_multiply(self, other):
        return Matrix([
            [self.data[i][j] * other.data[i][j] for j in range(self.cols)]
            for i in range(self.rows)
        ])

    def matmul(self, other):
        return Matrix([
            [
                sum(self.data[i][k] * other.data[k][j] for k in range(self.cols))
                for j in range(other.cols)
            ]
            for i in range(self.rows)
        ])

    def transpose(self):
        return Matrix([
            [self.data[j][i] for j in range(self.rows)]
            for i in range(self.cols)
        ])

    def determinant(self):
        if self.shape == (1, 1):
            return self.data[0][0]
        if self.shape == (2, 2):
            return self.data[0][0] * self.data[1][1] - self.data[0][1] * self.data[1][0]
        det = 0
        for j in range(self.cols):
            minor = Matrix([
                [self.data[i][k] for k in range(self.cols) if k != j]
                for i in range(1, self.rows)
            ])
            det += ((-1) ** j) * self.data[0][j] * minor.determinant()
        return det

    def inverse_2x2(self):
        det = self.determinant()
        if det == 0:
            raise ValueError("Matrix is singular, no inverse exists")
        return Matrix([
            [self.data[1][1] / det, -self.data[0][1] / det],
            [-self.data[1][0] / det, self.data[0][0] / det]
        ])

    @staticmethod
    def identity(n):
        return Matrix([
            [1 if i == j else 0 for j in range(n)]
            for i in range(n)
        ])
```

### Langkah 3: Lihat cara kerjanya

```python
A = Matrix([[1, 2], [3, 4]])
B = Matrix([[5, 6], [7, 8]])

print("A + B =", (A + B).data)
print("A @ B =", A.matmul(B).data)
print("A^T =", A.transpose().data)
print("det(A) =", A.determinant())
print("A^-1 =", A.inverse_2x2().data)

I = Matrix.identity(2)
print("A @ A^-1 =", A.matmul(A.inverse_2x2()).data)
```

### Langkah 4: Kaitkan dengan jaringan saraf

```python
import random

inputs = Matrix([[0.5], [0.8], [0.2]])
weights = Matrix([
    [random.uniform(-1, 1) for _ in range(3)]
    for _ in range(2)
])
bias = Matrix([[0.1], [0.1]])

def relu_matrix(m):
    return Matrix([[max(0, val) for val in row] for row in m.data])

pre_activation = weights.matmul(inputs) + bias
output = relu_matrix(pre_activation)

print(f"Input shape: {inputs.shape}")
print(f"Weight shape: {weights.shape}")
print(f"Output shape: {output.shape}")
print(f"Output: {output.data}")
```

Ini adalah satu lapisan rapat (dense): `output = relu(W @ x + b)`. Setiap lapisan rapat di setiap jaringan saraf melakukan persis ini.

## Gunakan

NumPy melakukan semua hal di atas dengan lebih sedikit baris dan berkali-kali lipat lebih cepat.

```python
import numpy as np

A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

print("A + B =\n", A + B)
print("A * B (element-wise) =\n", A * B)
print("A @ B (matrix multiply) =\n", A @ B)
print("A^T =\n", A.T)
print("det(A) =", np.linalg.det(A))
print("A^-1 =\n", np.linalg.inv(A))
print("I =\n", np.eye(2))

inputs = np.random.randn(3, 1)
weights = np.random.randn(2, 3)
bias = np.array([[0.1], [0.1]])
output = np.maximum(0, weights @ inputs + bias)

print(f"\nNeural network layer: {weights.shape} @ {inputs.shape} = {output.shape}")
print(f"Output:\n{output}")
```

Operator `@` di Python memanggil `__matmul__`. NumPy mengimplementasikannya dengan rutin BLAS teroptimasi yang ditulis dalam C dan Fortran. Matematikanya sama, 100 kali lebih cepat.

Broadcasting di NumPy:

```python
matrix = np.array([[1, 2, 3], [4, 5, 6]])
bias = np.array([10, 20, 30])
print(matrix + bias)
```

NumPy secara otomatis menyebarkan bias 1D ke kedua baris. Inilah cara kerja penambahan bias di setiap framework jaringan saraf.

## Kirimkan

Pelajaran ini menghasilkan sebuah prompt untuk mengajar operasi matriks melalui intuisi geometris. Lihat `outputs/prompt-matrix-operations.md`.

Kelas Matrix yang dibangun di sini menjadi fondasi untuk mini-framework jaringan saraf yang kita bangun di Phase 3, Lesson 10.

## Latihan

1. **Verifikasi invers.** Kalikan `A @ A.inverse_2x2()` dan pastikan Anda mendapat matriks identitas. Coba dengan tiga matriks 2x2 yang berbeda. Apa yang terjadi ketika determinannya nol?

2. **Terapkan invers 3x3.** Perluas kelas Matrix untuk menghitung invers matriks 3x3 menggunakan metode adjugat. Uji dengan `np.linalg.inv` dari NumPy.

3. **Bangun jaringan dua lapis.** Hanya dengan kelas Matrix Anda (tanpa NumPy), buat jaringan saraf dua lapis: masukan (3) -> tersembunyi (4) -> keluaran (2). Inisialisasi bobot acak, jalankan jaluran maju, dan verifikasi semua bentuk sudah benar.

### Petunjuk

**Latihan 1.** Kalikan `A @ A.inverse_2x2()` untuk tiga matriks dan bandingkan setiap sel dengan matriks identitas dalam toleransi `1e-9`. Untuk matriks dengan determinan 0, seperti [[1, 2], [2, 4]], `inverse_2x2` menaikkan `ValueError`, dan itulah hasil yang diharapkan.

**Latihan 2.** Hitung matriks kofaktor 3x3, transpose untuk mendapat adjugat, lalu bagi setiap sel dengan `determinant()`. Periksa bahwa `A @ A_inv` menghasilkan identitas dan setiap sel cocok dengan `np.linalg.inv(A)` dalam toleransi `1e-9`.

**Latihan 3.** Tulis bentuk setiap matriks sebelum mengalikan: `W1` berukuran 4x3 dan `W2` berukuran 2x4, seperti pada `demo_neural_network_layer`. Masukan 3x1 harus menghasilkan vektor tersembunyi 4x1 dan keluaran 2x1.

## Istilah Kunci

| Istilah | Kata orang | Makna sebenarnya |
|---------|------------|------------------|
| Vektor | "Sebuah panah" | Daftar bilangan terurut. Dalam AI: sebuah titik dalam ruang berdimensi tinggi. |
| Matriks | "Tabel bilangan" | Transformasi linear. Ia memetakan vektor dari satu ruang ke ruang lain. |
| Perkalian matriks | "Cukup kalikan bilangannya" | Produk titik antara setiap baris matriks pertama dan setiap kolom matriks kedua. Urutan berpengaruh. |
| Transpose | "Balik saja" | Menukar baris dan kolom. Mengubah matriks $m \times n$ menjadi $n \times m$. Sangat penting dalam propagasi balik. |
| Determinan | "Angka dari matriks" | Mengukur seberapa besar matriks menskalakan luas (2D) atau volume (3D). Nol berarti transformasi menghancurkan satu dimensi. |
| Invers | "Batalkan matriks" | Matriks yang membalikkan transformasi. Hanya ada jika determinan tidak nol. |
| Matriks identitas | "Matriks yang membosankan" | Padanan matriks dari perkalian dengan 1. Dipakai pada koneksi residual (ResNet). |
| Broadcasting | "Perbaikan bentuk ajaib" | Meregangkan larik yang lebih kecil agar cocok dengan larik lebih besar dengan mengulanginya pada dimensi yang hilang. |
| Elemen-demi-elemen | "Perkalian biasa" | Mengalikan posisi yang bersesuaian. Kedua larik harus memiliki bentuk sama (atau dapat di-broadcast). |

## Bacaan Lanjutan

- [3Blue1Brown: Essence of Linear Algebra](https://www.3blue1brown.com/topics/linear-algebra) — intuisi visual untuk setiap operasi yang dibahas di sini
- [Dokumentasi NumPy tentang broadcasting](https://numpy.org/doc/stable/user/basics.broadcasting.html) — aturan persis yang diikuti NumPy
- [Tinjauan Aljabar Linear CS229 Stanford](http://cs229.stanford.edu/section/cs229-linalg.pdf) — referensi ringkas aljabar linear khusus ML
