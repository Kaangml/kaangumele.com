---
layout: ../../layouts/BlogPost.astro
title: "Feature Engineering: Aykırı Değerler, Eksik Veriler, Encoding"
date: "2024-11-17"
description: "Makine öğrenmesinde veri hazırlığının temelleri: aykırı değer tespiti, eksik veri stratejileri ve kategorik değişken dönüşümü."
tags: ["Machine Learning", "Feature Engineering", "Veri Bilimi", "Python"]
canonical: "https://medium.com/@gumelekaan/feature-engineering-26d63e0eec77"
---

Model başarısı çoğunlukla feature engineering'e bağlıdır. Doğru veri hazırlığı yapılmadan en güçlü model bile beklenen performansı vermez. Bu yazıda üç temel konuya odaklanıyoruz: aykırı değerler, eksik veriler ve kategorik encoding.

Örnekler boyunca Iris veri setini (150 örnek, 4 özellik) kullanacağız.

## Aykırı Değerler (Outliers)

Aykırı değerler modeli yanıltır, özellikle uzaklık tabanlı algoritmalarda (KNN, SVM, linear regression) ciddi sorun yaratır.

### Standart Sapma Yöntemi

Ortalamadan ±3 standart sapma dışındaki değerleri aykırı kabul et:

```python
mean = df['feature'].mean()
std  = df['feature'].std()

outliers = df[(df['feature'] < mean - 3*std) |
              (df['feature'] > mean + 3*std)]
```

### Z-Score

Her değerin ortalamadan kaç standart sapma uzakta olduğunu ölçer:

```python
from scipy import stats

z_scores = stats.zscore(df['feature'])
outliers  = df[abs(z_scores) > 3]
```

### IQR (Interquartile Range)

Normal dağılım varsayımı gerektirmez — çarpık verilerde Z-score'dan daha güvenilir:

```python
Q1  = df['feature'].quantile(0.25)
Q3  = df['feature'].quantile(0.75)
IQR = Q3 - Q1

lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR

outliers = df[(df['feature'] < lower) | (df['feature'] > upper)]
```

**Öneri:** Veri dağılımı bilinmiyorsa IQR'ı tercih et.

## Eksik Değerler (Missing Values)

Eksik veri her veri setinde karşına çıkar. Strateji seçimi veri miktarına, dağılımına ve kullanılacak modele göre değişir.

```python
# Eksik değer oranı
df.isnull().sum() / len(df) * 100
```

### Doldurma Stratejileri

| Strateji | Ne zaman? |
|---|---|
| Sabit değer (0, "bilinmiyor") | Eksiklik anlamlıysa |
| Ortalama (mean) | Normal dağılımlı sayısal |
| Medyan | Çarpık dağılımlı sayısal |
| Mod | Kategorik |
| Forward/backward fill | Zaman serisi |
| İnterpolasyon | Düzgün değişen seri |
| Gruba göre ortalama | Alt grup örüntüsü varsa |
| ML ile tahmin | Karmaşık örüntü varsa |

```python
# Ortalama ile doldurma
df['feature'].fillna(df['feature'].mean(), inplace=True)

# Gruba göre medyan
df['feature'] = df.groupby('group')['feature'] \
                  .transform(lambda x: x.fillna(x.median()))
```

**Not:** Tree-based modeller (Random Forest, XGBoost) eksik değeri kendi başına idare edebilir; bu modelleri kullanıyorsan doldurma zorunlu değil.

## Kategorik Encoding

Model sayısal girdi bekler — kategorik değişkenleri dönüştürmek şart.

### Label Encoding

Sıralı (ordinal) kategoriler için:

```python
from sklearn.preprocessing import LabelEncoder

le = LabelEncoder()
df['kategori_encoded'] = le.fit_transform(df['kategori'])
# küçük → 0, orta → 1, büyük → 2
```

### One-Hot Encoding

Nominal (sırasız) kategoriler için:

```python
df = pd.get_dummies(df, columns=['renk'], drop_first=True)
# mavi → [1, 0], kırmızı → [0, 1], yeşil → [0, 0]
```

`drop_first=True` ile çoklu doğrusallık (multicollinearity) önlenir.

### Target Encoding

Yüksek kardinaliteli kategoriler için (şehir, ürün ID):

```python
target_mean = df.groupby('şehir')['hedef'].mean()
df['şehir_encoded'] = df['şehir'].map(target_mean)
```

## Özet

Veri hazırlığı doğrudan model başarısını etkiler. Yöntem seçimi kural değil, veriye özgü karar:

- **Aykırı değer:** dağılıma göre IQR veya Z-score
- **Eksik veri:** örüntüye göre doldur ya da modele bırak
- **Encoding:** sıralıysa label, sırasızsa one-hot, yüksek kardinaliteyse target

---

*Bu yazının orijinali [Medium'da](https://medium.com/@gumelekaan/feature-engineering-26d63e0eec77) yayınlanmıştır.*
