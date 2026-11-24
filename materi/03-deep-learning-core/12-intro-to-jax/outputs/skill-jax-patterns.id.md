---
name: skill-jax-patterns
description: Pola pemrograman fungsional di JAX -- kapan dan bagaimana menggunakan grad, jit, vmap, dan pmap
version: 1.0.0
phase: 3
lesson: 12
tags: [jax, functional-programming, autodiff, compilation, vectorization]
---

# Pola Fungsional JAX

JAX mentransformasi fungsi murni. Setiap pola di bawah mengikuti satu aturan: tulis fungsi yang menerima masukan dan mengembalikan keluaran, tanpa efek samping. Lalu transformasikan.

## Empat Transformasi

### grad -- Turunkan sebuah fungsi

```python
grads = jax.grad(loss_fn)(params, x, y)
loss, grads = jax.value_and_grad(loss_fn)(params, x, y)
```

Gunakan saat: Anda membutuhkan gradien untuk optimasi.
Batasan: fungsi harus mengembalikan skalar. Untuk keluaran non-skalar, gunakan `jax.jacobian`.

### jit -- Kompilasi sebuah fungsi

```python
fast_fn = jax.jit(f)
```

Gunakan saat: fungsi akan dipanggil lebih dari sekali dengan masukan berbentuk sama.
Batasan: tidak boleh ada aliran kontrol Python yang bergantung pada nilai yang dilacak. Gunakan `jax.lax.cond` untuk kondisional dan `jax.lax.scan` untuk loop.

### vmap -- Vektorkan sebuah fungsi

```python
batch_fn = jax.vmap(f, in_axes=(None, 0))
```

Gunakan saat: Anda menulis fungsi untuk satu contoh dan ingin fungsi itu bekerja pada batch.
`in_axes` menentukan sumbu argumen mana yang dijadikan batch. `None` berarti tidak dibuat batch (disiarkan).

### pmap -- Paralelkan lintas perangkat

```python
parallel_fn = jax.pmap(f, axis_name='devices')
```

Gunakan saat: Anda memiliki beberapa GPU/TPU dan ingin paralelisme data.
Di dalam fungsi, `jax.lax.pmean(x, 'devices')` merata-ratakan nilai lintas perangkat.

## Aturan Komposisi

Transformasi dapat disusun. Urutannya penting:

```python
per_example_grads = jax.jit(jax.vmap(jax.grad(loss_fn), in_axes=(None, 0, 0)))
```

Dibaca dari kanan ke kiri: turunkan gradien loss_fn, vektorkan lintas contoh, lalu kompilasi hasilnya.

Komposisi yang valid:
- `jit(grad(f))` -- komputasi gradien terkompilasi
- `jit(vmap(f))` -- komputasi batch terkompilasi
- `vmap(grad(f))` -- gradien per contoh
- `pmap(jit(f))` -- komputasi terkompilasi paralel
- `grad(jit(f))` -- gradien fungsi terkompilasi (sama seperti jit(grad(f)))

## Pola Manajemen Parameter

Parameter JAX adalah pytree (dict array bersarang):

```python
params = {
    'layer1': {'w': jnp.zeros((784, 256)), 'b': jnp.zeros(256)},
    'layer2': {'w': jnp.zeros((256, 10)),  'b': jnp.zeros(10)},
}
```

Perbarui semua parameter sekaligus:
```python
params = jax.tree.map(lambda p, g: p - lr * g, params, grads)
```

Hitung jumlah parameter:
```python
n_params = sum(p.size for p in jax.tree.leaves(params))
```

## Manajemen Kunci PRNG

JAX memerlukan kunci acak eksplisit:

```python
key = jax.random.PRNGKey(0)
key, subkey = jax.random.split(key)
noise = jax.random.normal(subkey, shape)
```

Untuk beberapa operasi acak, lakukan split sekali:
```python
keys = jax.random.split(key, n)
```

Jangan pernah gunakan ulang kunci. Selalu lakukan split sebelum menggunakannya.

## Kesalahan Umum

1. **Mengubah array di dalam jit**: array JAX bersifat immutable. Gunakan `x.at[i].set(v)` alih-alih `x[i] = v`.

2. **Menggunakan print Python di dalam jit**: `print` berjalan saat tracing, bukan saat eksekusi. Gunakan `jax.debug.print("{}", x)`.

3. **if/for Python di dalam jit pada nilai yang dilacak**: Gunakan `jax.lax.cond`, `jax.lax.switch`, `jax.lax.scan`, `jax.lax.fori_loop`.

4. **Lupa memanggil `.block_until_ready()`**: JAX menggunakan dispatch asinkron. Untuk benchmarking, panggil `.block_until_ready()` agar menunggu penyelesaian aktual.

5. **Menggunakan ulang kunci PRNG**: Dua operasi dengan kunci yang sama menghasilkan nilai "acak" yang sama. Selalu lakukan split.

6. **State global dalam fungsi ter-jit**: Variabel global ditangkap saat tracing. Perubahan setelah tracing tidak terlihat. Teruskan semuanya sebagai argumen.

## Daftar Periksa Keputusan

1. Apakah fungsi dipanggil lebih dari sekali? Tambahkan `@jax.jit`.
2. Apakah fungsi membutuhkan gradien? Bungkus dengan `jax.grad` atau `jax.value_and_grad`.
3. Apakah fungsi memproses satu contoh tetapi Anda punya batch? Bungkus dengan `jax.vmap`.
4. Apakah Anda memiliki beberapa perangkat? Bungkus dengan `jax.pmap`.
5. Apakah fungsi menggunakan keacakan? Teruskan kunci PRNG secara eksplisit.
6. Apakah fungsi memiliki aliran kontrol Python pada nilai array? Ganti dengan primitif `jax.lax`.

## Kapan Menggunakan JAX

Gunakan JAX saat:
- Anda membutuhkan gradien per contoh (privasi diferensial, informasi Fisher)
- Anda melatih di TPU (JAX adalah framework native-nya)
- Anda membutuhkan turunan orde lebih tinggi (Hessian, Jacobian)
- Anda ingin mengompilasi seluruh langkah pelatihan menjadi satu kernel
- Tim Anda berada di Google DeepMind atau Anthropic

Gunakan PyTorch saat:
- Anda menginginkan ekosistem terbesar (HuggingFace, torchvision, Lightning)
- Anda mengutamakan kemudahan debugging dibanding kecepatan mentah
- Anda melakukan deploy ke GPU NVIDIA dengan TorchServe/Triton
- Anda sedang merekrut (lebih banyak pengembang PyTorch)
- Anda ingin bereksperimen cepat dengan arsitektur baru
