---
layout: ../../layouts/BlogPost.astro
title: "Uygulamayı Konteynere Taşımak: Dockerfile'dan Compose'a"
date: "2026-06-25"
description: "Gerçek bir FastAPI uygulamasını sıfırdan konteynerize etmek: Dockerfile yazımı, build/run akışı, docker run vs docker compose farkı ve servis ölçekleme."
tags: ["Docker", "FastAPI", "Docker Compose", "Deployment", "DevOps"]
sunum: "/sunumlar/ders2_deployment.html"
---

> **Eğitim sunumuna** erişmek için → [Ders 2 Slaytları](/sunumlar/ders2_deployment.html)

## Ne İnşa Ediyoruz?

Teorinin yerini pratik aldığı ders bu. Hedef: gerçek bir Python uygulamasını — görüntü sınıflandıran bir FastAPI servisi — sıfırdan konteynerize etmek. Tek komutla ayağa kalkacak, logları takip edilebilecek, gerektiğinde ölçeklenebilecek.

Uygulama yapısı şu:

```
image-classifier-api/
├── main.py          # FastAPI endpoint'leri
├── model.py         # PyTorch model yükleme + inference
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

## Dockerfile Anatomisi

Dockerfile bir tarif; sırayla okunur ve her satır bir katman oluşturur. Sıralamanın önemi var: en az değişen şeyleri üste, en çok değişenleri alta yaz ki cache verimli çalışsın.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Önce bağımlılıklar — kod değişse bile bu katman cache'den gelir
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Sonra kod
COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`slim` variant tercih ediliyor çünkü tam Python image'ı ~1 GB'ın üzerinde; slim ~150 MB. Production'da image boyutu her yerde önemli: registry transfer süresi, disk alanı, saldırı yüzeyi.

`EXPOSE` sadece belge; gerçek port mapping `docker run -p` veya compose'daki `ports:` ile yapılır.

## Build ve Çalıştırma

```bash
# Image oluştur
docker build -t classifier:v1 .

# Container başlat
docker run -d -p 8000:8000 --name classifier classifier:v1

# Logları takip et
docker logs -f classifier

# Test et
curl http://localhost:8000/health
curl -X POST http://localhost:8000/predict \
     -F "file=@test.jpg"
```

Build sırasında her adım "Step X/Y" olarak çıktı verir. Değişmeyen adımlar `---> Using cache` der; bu cache mekanizması büyük projelerde build süresini dramatik düşürür.

## Container İçini Anlamak

Çalışan container'ı incelemenin en hızlı yolu:

```bash
# İçine gir
docker exec -it classifier bash

# Kaynak kullanımı
docker stats classifier

# Dosya sistemi
docker exec classifier ls /app
```

`docker exec` mevcut container'a yeni bir process başlatır — container'ı durdurmaz, yeniden başlatmaz. Debug için vazgeçilmez.

## Kriz Senaryosu

Container çöktüğünde ne olur? İki yaklaşım:

**Kötü:** `docker start classifier` — aynı container'ı diriltiyorsun, problem hâlâ içinde.

**İyi:** Container'ı at, yenisini başlat. Container stateless olmalı; durum dışarıda (volume, veritabanı) tutulmalı. Böylece yeni bir container başlatmak her zaman temiz bir başlangıç demek.

Bunun için restart policy:

```bash
docker run -d --restart=unless-stopped -p 8000:8000 classifier:v1
```

## docker run vs docker compose

Tek container için `docker run` yeterli. Ama gerçek dünyada servisler birden fazla: API + veritabanı + cache + reverse proxy.

| | `docker run` | `docker compose` |
|---|---|---|
| Yönetim | Her container tek tek | Tüm stack tek komut |
| Networking | Manuel `--network` | Otomatik internal DNS |
| Başlatma sırası | Siz ayarlarsınız | `depends_on` ile tanımlı |
| `docker compose ps` | Görmez | Gösterir |
| Tekrar üretilebilirlik | Komut ezberinde | `docker-compose.yml`'de |

`docker compose ps`'nin `docker run` ile başlatılan container'ları görmemesi bir bug değil, tasarım kararı. Compose kendi container'larına label yapıştırır; bu label'lara göre filtreler. `docker run` ile başlatılanlar o label'lara sahip olmadığı için radar dışında kalır.

```yaml
# docker-compose.yml — version satırı yok (Compose v2)
services:
  api:
    build: .
    ports:
      - "8000:8000"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      retries: 3
```

## Ölçekleme

Compose ile yatay ölçekleme tek satır:

```bash
docker compose up -d --scale api=3
```

Bu komut 3 ayrı container başlatır; hepsi aynı image'dan, hepsi aynı ağda. Önüne bir load balancer (nginx veya Traefik) koyduğunda trafik üçe bölünür. Klasik sunucu mimarisinde bu kurulumu yapmak günler sürerdi.

## Sonraki Adım

Local'de çalışan bu stack'i AWS'ye taşımak için üç servis gerekiyor: ECR (image deposu), ECS Fargate (çalıştırma), ALB (trafik dağıtımı). Bir sonraki derste bunların tamamını elle kuracağız.
