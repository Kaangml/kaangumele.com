---
layout: ../../layouts/BlogPost.astro
title: "Medikal Görüntü İşlemede Transfer Learning"
date: "2024-10-29"
description: "ResNet50 ile akciğer röntgeninden zatürre tespiti: veri dengesizliği, augmentation, fine-tuning ve %95.92 doğruluk."
tags: ["Deep Learning", "Transfer Learning", "Computer Vision", "TensorFlow", "Medikal AI"]
canonical: "https://medium.com/@gumelekaan/medikal-g%C3%B6r%C3%BCnt%C3%BC-%C4%B0%C5%9Flemede-derin-%C3%B6%C4%9Frenme-teknikleri-transfer-learning-ile-g%C3%BC%C3%A7lendirilmi%C5%9F-%C3%A7%C3%B6z%C3%BCmler-163687d5d3ab"
---

X-ray, MR, CT görüntülerinin analizinde derin öğrenme modelleri, uzman hekimlere yakın ya da bazen üstün doğruluk sergileyebiliyor. Ancak medikal veri etiketlemek pahalı ve yavaş — bu noktada transfer learning devreye giriyor.

Bu yazıda Kaggle'daki "Chest X-Ray Images (Pneumonia)" veri setiyle ResNet50 kullanarak zatürre tespiti yapıyoruz.

## Veri Seti

Guangzhou Kadın ve Çocuk Tıp Merkezi'nden 1–5 yaş arası çocuklara ait **5.863 göğüs röntgeni**. İki sınıf: NORMAL ve PNEUMONIA.

```python
import os
import numpy as np
import tensorflow as tf
from tensorflow.keras.applications import ResNet50
from tensorflow.keras import layers, Model
from sklearn.model_selection import train_test_split

IMG_SIZE = (224, 224)
BATCH_SIZE = 32
```

## Veri Dengesizliği

Veri seti dengesiz: PNEUMONIA sınıfı belirgin biçimde fazla. İki yöntemle çözüyoruz:

**Undersampling** — fazla sınıfı azalt:

```python
pneumonia_files = pneumonia_files[:target_count]
```

**Data Augmentation** — az sınıfı artır:

```python
from tensorflow.keras.preprocessing.image import ImageDataGenerator

aug = ImageDataGenerator(
    rotation_range=20,
    width_shift_range=0.2,
    height_shift_range=0.2,
    shear_range=0.2,
    zoom_range=0.2,
    horizontal_flip=True,
)
```

## Model Mimarisi

ResNet50'yi ImageNet ağırlıklarıyla yüklüyoruz, katmanlarını donduruyoruz, üstüne kendi sınıflandırma katmanlarımızı ekliyoruz:

```python
base_model = ResNet50(
    weights='imagenet',
    include_top=False,
    input_shape=(224, 224, 3)
)
base_model.trainable = False  # freeze

x = base_model.output
x = layers.Flatten()(x)
x = layers.Dense(256, activation='relu')(x)
x = layers.Dropout(0.5)(x)
output = layers.Dense(1, activation='sigmoid')(x)

model = Model(inputs=base_model.input, outputs=output)

model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=0.001),
    loss='binary_crossentropy',
    metrics=['accuracy']
)
```

## Eğitim

Callback'lerle eğitimi kontrol altında tutuyoruz:

```python
callbacks = [
    tf.keras.callbacks.ReduceLROnPlateau(
        monitor='val_loss', factor=0.5, patience=3
    ),
    tf.keras.callbacks.EarlyStopping(
        monitor='val_loss', patience=5, restore_best_weights=True
    ),
    tf.keras.callbacks.ModelCheckpoint(
        'best_model.h5', save_best_only=True
    ),
]

history = model.fit(
    train_dataset,
    validation_data=val_dataset,
    epochs=30,
    callbacks=callbacks,
)
```

Eğitim verisi %80, test verisi %20 — stratified sampling ile sınıf dağılımı korundu.

## Sonuçlar

| Metrik | Değer |
|---|---|
| Hassasiyet (Recall) | %97.46 |
| Özgüllük (Specificity) | %93.99 |
| Kesinlik (Precision) | %95.30 |
| Doğruluk (Accuracy) | %95.92 |
| F1 Skoru | 0.9637 |
| AUC | 0.99 |

Model PNEUMONIA vakalarının %97.46'sını yakaladı. Medikal bağlamda recall kritik — bir vakayı kaçırmak, yanlış alarm vermekten daha tehlikeli.

## Öğrenilen Dersler

**Veri ön işleme başarının anahtarı.** Dengesizliği çözmeden modeli eğitmiş olsaydım, model her şeyi PNEUMONIA tahmin ederek yüksek accuracy görünür, ama işe yaramaz olurdu.

Transfer learning büyük avantaj sağladı: ResNet50'nin ImageNet'te öğrendiği kenar, doku, şekil özellikleri tıbbi görüntülere de aktarıldı. Sıfırdan eğitmek hem daha fazla veri ister hem de daha uzun sürer.

---

Kaggle notebook: [chest-xray](https://www.kaggle.com/code/kaangmele/chest-xray)

*Bu yazının orijinali [Medium'da](https://medium.com/@gumelekaan/medikal-g%C3%B6r%C3%BCnt%C3%BC-%C4%B0%C5%9Flemede-derin-%C3%B6%C4%9Frenme-teknikleri-transfer-learning-ile-g%C3%BC%C3%A7lendirilmi%C5%9F-%C3%B7%C3%B6z%C3%BCmler-163687d5d3ab) yayınlanmıştır.*
