---
name: skill-pytorch-patterns
description: Pola referensi untuk pelatihan, evaluasi, dan deployment PyTorch
version: 1.0.0
phase: 03
lesson: 11
tags: [pytorch, training, deep-learning, gpu, patterns]
---

## Loop Pelatihan Kanonik

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = Model().to(device)
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

for epoch in range(num_epochs):
    model.train()
    for inputs, targets in train_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()

    model.eval()
    with torch.no_grad():
        for inputs, targets in val_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
```

## Pelatihan Presisi Campuran

```python
from torch.amp import autocast, GradScaler

scaler = GradScaler()
for inputs, targets in train_loader:
    inputs, targets = inputs.to(device), targets.to(device)
    optimizer.zero_grad()
    with autocast(device_type="cuda"):
        outputs = model(inputs)
        loss = criterion(outputs, targets)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

Gunakan saat: melatih di GPU dengan perangkat keras yang mendukung float16 (V100, A100, H100, RTX 3090+). Harapkan peningkatan kecepatan ~1,5-2x dan penghematan memori ~50%.

## Akumulasi Gradien

```python
accumulation_steps = 4
optimizer.zero_grad()
for i, (inputs, targets) in enumerate(train_loader):
    inputs, targets = inputs.to(device), targets.to(device)
    outputs = model(inputs)
    loss = criterion(outputs, targets) / accumulation_steps
    loss.backward()
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

Gunakan saat: ukuran batch efektif perlu lebih besar daripada yang memungkinkan memori GPU. Membagi loss dengan accumulation_steps menjaga skala gradien tetap konsisten.

## Menyimpan dan Memuat

```python
torch.save({
    "epoch": epoch,
    "model_state_dict": model.state_dict(),
    "optimizer_state_dict": optimizer.state_dict(),
    "loss": loss.item(),
}, "checkpoint.pt")

checkpoint = torch.load("checkpoint.pt", weights_only=True)
model.load_state_dict(checkpoint["model_state_dict"])
optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
```

Selalu simpan state pengoptimal untuk melanjutkan pelatihan. Untuk inferensi saja, cukup simpan `model.state_dict()`.

## Dataset Kustom

```python
class CustomDataset(torch.utils.data.Dataset):
    def __init__(self, data_dir, transform=None):
        self.samples = self._load_samples(data_dir)
        self.transform = transform

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        x, y = self.samples[idx]
        if self.transform:
            x = self.transform(x)
        return x, y

    def _load_samples(self, data_dir):
        ...
```

## Konfigurasi DataLoader

```python
train_loader = torch.utils.data.DataLoader(
    dataset,
    batch_size=64,
    shuffle=True,
    num_workers=4,
    pin_memory=True,
    drop_last=True,
    persistent_workers=True,
)
```

| Parameter | Fungsinya | Kapan digunakan |
|-----------|-----------|-----------------|
| num_workers=4 | Pemuatan data paralel | Selalu aktifkan pada mesin multicore |
| pin_memory=True | Memori CPU terkunci halaman (page-locked) | Saat melatih di GPU |
| drop_last=True | Buang batch terakhir yang tidak lengkap | Saat menggunakan BatchNorm |
| persistent_workers=True | Pertahankan worker tetap aktif lintas epoch | Saat num_workers > 0 |

## Jadwal Laju Pembelajaran

```python
scheduler = torch.optim.lr_scheduler.OneCycleLR(
    optimizer,
    max_lr=1e-3,
    total_steps=num_epochs * len(train_loader),
    pct_start=0.1,
)

for epoch in range(num_epochs):
    for inputs, targets in train_loader:
        ...
        optimizer.step()
        scheduler.step()
```

OneCycleLR: pilihan default terbaik untuk sebagian besar tugas. Fungsi ini memanaskan laju hingga max_lr, lalu melakukan peluruhan kosinus. Panggil `scheduler.step()` setelah setiap batch, bukan setiap epoch.

## Inisialisasi Bobot

```python
def init_weights(module):
    if isinstance(module, nn.Linear):
        nn.init.kaiming_normal_(module.weight, nonlinearity="relu")
        if module.bias is not None:
            nn.init.zeros_(module.bias)
    elif isinstance(module, nn.Conv2d):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")

model.apply(init_weights)
```

## Mode Inferensi

```python
model.eval()

with torch.inference_mode():
    outputs = model(inputs)
```

`torch.inference_mode()` lebih cepat daripada `torch.no_grad()` karena menonaktifkan autograd sepenuhnya, bukan hanya menekan komputasi gradien.

## Daftar Periksa Kesalahan Umum

1. Menerapkan softmax sebelum CrossEntropyLoss (fungsi ini sudah menyertakan log_softmax secara internal)
2. Lupa memanggil model.eval() saat validasi
3. Lupa memindahkan tensor ke perangkat yang sama dengan model
4. Tidak memanggil optimizer.zero_grad() (gradien terakumulasi secara default)
5. Menggunakan torch.no_grad() saat pelatihan (menonaktifkan komputasi gradien)
6. Menetapkan num_workers terlalu tinggi (terlalu banyak proses yang dibentuk, memori menjadi thrashing)
7. Tidak menggunakan pin_memory=True saat melatih di GPU
8. Menyimpan seluruh objek model alih-alih state_dict (rusak saat refactoring)
