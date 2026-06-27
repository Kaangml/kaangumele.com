---
layout: ../../layouts/BlogPost.astro
title: "Bir Nöron Ne Yapabilir?"
date: "2024-12-10"
description: "Tek bir yapay nöronun gradient descent ile binary classification yapabileceğini sıfırdan PyTorch ile gösteriyoruz."
tags: ["Deep Learning", "Gradient Descent", "PyTorch", "Classification"]
canonical: "https://medium.com/@gumelekaan/bir-n%C3%B6ron-ne-yapabilir-815179c62d63"
---

Sinir ağlarının temelinde nöronlar yer alır. Peki tek bir nöron ne yapabilir? Basit bir veri seti üzerinden sıfırdan gidip gösterelim.

## Veri Setinin Oluşturulması

İki özelliği olan, iki kümeden oluşan yapay bir veri seti oluşturuyoruz:

**Küme 1:**
- x₁: Ortalama = 5, Standart Sapma = 2
- y₁: Ortalama = 10, Standart Sapma = 3

**Küme 2:**
- x₂: Ortalama = 15, Standart Sapma = 1
- y₂: Ortalama = 20, Standart Sapma = 5

Her kümeden 300 nokta, Küme 1 → etiket 0, Küme 2 → etiket 1. Veriler karıştırılıp PyTorch tensörüne dönüştürülüyor.

## Tek Nöron Modeli

### Lineer Çıktı

```
z = W^T · X + b
```

- W: ağırlık vektörü
- X: giriş özellikleri
- b: bias

### Sigmoid Aktivasyon

Lineer çıktıyı [0, 1] aralığına sıkıştırıyoruz:

```
σ(z) = 1 / (1 + e^(−z))
```

Başlangıçta ağırlıklar sıfır. İlk tahminlere bakarsak nöron henüz hiçbir şey öğrenmemiş:

```
z = 6.09  →  σ(z) = 0.9977  →  gerçek etiket: 0  ← yanlış
z = 16.68 →  σ(z) = 0.9999  →  gerçek etiket: 1  ← doğru
```

## Loss Fonksiyonu: Binary Cross Entropy

```
L = −(1/N) · Σ [y · log(ŷ) + (1 − y) · log(1 − ŷ)]
```

## Gradient Descent ile Ağırlık Güncellemesi

Ağırlıkları loss'un gradyanı yönünde güncelliyoruz:

```
W := W − μ · (∂L/∂W)
```

Sigmoid + binary cross-entropy kombinasyonunda zincir kuralı uygulanınca türev basitleşir:

```
∂L/∂W = (ŷ − y) · X
```

## Kodun Tamamı

```python
import torch
import matplotlib.pyplot as plt
import numpy as np

def sigmoid(z):
    return 1 / (1 + torch.exp(-z))

def plot_decision_boundary(X, y, W, b, epoch):
    w1, w2 = W[0].item(), W[1].item()
    bias = b.item()

    plt.scatter(X[:, 0], X[:, 1], c=y, cmap=plt.cm.Spectral)
    plt.xlabel('Feature 1')
    plt.ylabel('Feature 2')

    x_vals = np.linspace(X[:, 0].min(), X[:, 0].max(), 100)
    y_vals = -(w1 * x_vals + bias) / w2
    plt.plot(x_vals, y_vals, label='Decision Boundary', color='red')

    plt.title(f'Epoch {epoch + 1}')
    plt.legend()
    plt.show()

def calculate_accuracy_by_boundary(X, y, W, b):
    with torch.no_grad():
        z_test = torch.matmul(X, W) + b
        y_pred_test = sigmoid(z_test)

        correct_preds = 0
        for i in range(len(X)):
            if (y_pred_test[i] >= 0.5 and y[i] == 1) or \
               (y_pred_test[i] < 0.5 and y[i] == 0):
                correct_preds += 1
        return correct_preds / len(X)

def train_model_with_boundary(X, y, lr=0.01, epochs=100):
    W = torch.zeros((2, 1), dtype=torch.float32)
    b = torch.zeros(1, dtype=torch.float32)

    for epoch in range(epochs):
        total_loss = 0
        for i in range(len(X)):
            x_sample, y_sample = X[i], y[i]

            z = torch.matmul(x_sample, W) + b
            y_pred = sigmoid(z)

            loss = -(y_sample * torch.log(y_pred) +
                     (1 - y_sample) * torch.log(1 - y_pred))
            total_loss += loss.item()

            e = y_pred - y_sample
            grad_W = e * x_sample.view(-1, 1)
            grad_b = e

            W -= lr * grad_W
            b -= lr * grad_b

        if epoch % 10 == 0:
            accuracy = calculate_accuracy_by_boundary(X, y, W, b)
            print(f"Epoch {epoch+1}, Loss: {total_loss/len(X):.4f}, "
                  f"Accuracy: {accuracy*100:.2f}%")
            plot_decision_boundary(X, y, W, b, epoch)

    return W, b

W, b = train_model_with_boundary(X, y, lr=0.01, epochs=100)
```

## Sonuçlar

Evet — tek bir nöron, doğru ağırlıklandırıldığında **doğrusal olarak ayrılabilir** veri kümelerini birbirinden ayırabilir.

| Uzay | Karar sınırı |
|---|---|
| 2D | Doğru |
| 3D | Düzlem |
| nD | Hiperuzay |

### Sınırlama

Tek nöron **doğrusal olmayan** ilişkileri öğrenemez. Kümeler doğrusal olarak ayrılamıyorsa — birden fazla nörondan oluşan bir ağ gerekir.

---

*Bu yazının orijinali [Medium'da](https://medium.com/@gumelekaan/bir-n%C3%B6ron-ne-yapabilir-815179c62d63) yayınlanmıştır.*
