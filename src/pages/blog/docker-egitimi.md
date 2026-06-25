---
layout: ../../layouts/BlogPost.astro
title: "Docker ile Container Dünyasına Giriş"
date: "2026-06-24"
description: "3 günlük Docker eğitiminin özeti: konteynerler, image'lar, Dockerfile ve AWS'ye taşıma."
tags: ["Docker", "Container", "AWS", "DevOps"]
sunum: "/sunumlar/ders1_docker_giris.html"
---

> **Eğitim sunumuna** erişmek için → [Ders 1 Slaytları](/sunumlar/ders1_docker_giris.html)

## Neden Docker?

"Benim makinemde çalışıyordu" — yazılım geliştirmenin en bilinen cümlesi. Her geliştirici farklı Python versiyonu, farklı kütüphane, farklı işletim sistemi. Test ortamı production'dan farklı. Production sunucusu da birbirinden farklı.

Docker bu problemi **konteyner** kavramıyla çözüyor: uygulamayı, bağımlılıklarını ve çalışma ortamını tek bir pakete sarıyor. Nerede açarsan aç, aynı çalışır.

## VM vs Konteyner

Sanal makine işletim sisteminin tamamını kopyalar — gigabaytlarca yer, dakikalarca başlangıç süresi. Konteyner ise sadece uygulamayı izole eder, host kernel'i paylaşır.

**Otel odası (VM) vs Airbnb (konteyner):** VM'de tüm bina senin için ayrılıyor, konteynerde sadece bir oda kullanıyorsun.

## Temel Kavramlar

**Dockerfile** — tarif defteri. Hangi base image, hangi paketler, nasıl başlatılacak.

**Image** — o tarifin paketlenmiş hali. Marketteki hazır karışım kutusu gibi. Henüz çalışmıyor, ama her şey içinde.

**Container** — image'ı çalıştırınca ortaya çıkan gerçek uygulama. Aynı image'dan istediğin kadar container açabilirsin; hepsi birbirinden bağımsız.

## Temel Komutlar

```bash
# Build
docker build -t uygulama:v1 .

# Çalıştır
docker run -d -p 8000:8000 --name api uygulama:v1

# İzle
docker logs -f api
docker stats api

# İçine gir
docker exec -it api bash

# Temizlik
docker stop api && docker rm api
```

## docker run vs docker compose

Tek container için `docker run` yeterli. Birden fazla servis (API + veritabanı + nginx) olduğunda `docker compose` devreye giriyor.

Fark şu: `docker run` ile başlattığın container'ı `docker compose ps` görmez. Compose kendi başlattığı container'lara label yapıştırır; `docker run` ile başlatılanlar onun radar'ında değildir.

```yaml
# docker-compose.yml — version satırı artık yok (Compose v2)
services:
  api:
    build: .
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      retries: 3
```

## AWS'ye Taşımak

Local'de çalışan container'ı production'a taşımak üç adım:

1. **ECR** — AWS'nin image deposu. `docker push` ile image'ı gönder.
2. **ECS Fargate** — serverless container çalıştırma. Sunucu yönetimi yok, kapasite yönetimi yok.
3. **ALB** — Application Load Balancer. Trafiği container'lara dağıt, bir container çökerse diğerine yönlendir.

Bu üç servis birlikte "kod yaz → container'a koy → AWS'de çalıştır" pipeline'ını oluşturuyor. Menü tespit modelimizi bu şekilde production'a aldık.
