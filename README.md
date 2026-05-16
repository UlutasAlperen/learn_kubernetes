# Minikube One Node Cluster — CKAD Çalışma Laboratuvarı

CKAD (Certified Kubernetes Application Developer) sınavı ve genel Kubernetes pratiği için oluşturulmuş 1-node Minikube cluster'ı. Gerçek bir 3-tier **SynergyChat** uygulaması üzerinden Deployment, Service, ConfigMap, Multi-container Pod, Volume, PVC, HPA ve Gateway API kavramlarını öğrenmeyi hedefler.

---

## Mimari Genel Bakış

```
Client (browser / curl)
  │
  ▼
┌──────────────────────────────────────────────────┐
│  app-gateway (Gateway API — Envoy)  :80          │
│  ├─ synchat.internal    → web-httproute          │
│  └─ synchatapi.internal → api-httproute          │
└─────────────┬───────────────────────┬────────────┘
              │                       │
              ▼                       ▼
      web-service:80          api-service:80
              │                       │
              ▼                       ▼
  synergychat-web (HPA:1-4)  synergychat-api (x1)
                                    │
                              ┌─────┴─────┐
                              │ PVC: 1Gi  │
                              │ /persist  │
                              └─────┬─────┘
                                    │ (cross-namespace)
                                    ▼
                           crawler-service:80
                           (namespace: crawler)
                                    │
                                    ▼
                          synergychat-crawler (x1 pod)
                          ┌── crawler-1 :8080 ──┐
                          ├── crawler-2 :8081 ──┤ emptyDir volume
                          └── crawler-3 :8082 ──┘   (/cache)

Test Workloads (default namespace):
  synergychat-testcpu (10m CPU limit) ← testcpu-hpa
  synergychat-testram (256Mi mem limit) ← testram-configmap (MEGABYTES)
```

**Trafik akışı:**

1. Client, `synchat.internal` veya `synchatapi.internal` domain'ine istek gönderir
2. Envoy Gateway (L7) hostname'e göre `HTTPRoute` ile doğru Service'e yönlendirir
3. Service, `selector` ile eşleşen Pod'lara trafiği iletir (`port:80 → targetPort:8080`)
4. Web frontend, `API_URL=http://synchatapi.internal` ile API'ye erişir
5. API Pod'u PVC ile mount edilen `/persist` dizininden veri okur/yazar (`/persist/db.json`)
6. API, `CRAWLER_BASE_URL=http://crawler-service.crawler.svc.cluster.local` ile crawler namespace'indeki servise erişir
7. Crawler Pod'u 3 sidecar container içerir; `emptyDir` volume üzerinden `/cache` dizinini paylaşır
8. Web deployment HPA tarafından yönetilir, CPU kullanımına göre 1-4 replica arasında ölçeklenir

---

## CKAD Konu Haritası — Bu Projede Hangi Kavramlar Var?

| CKAD Konusu | Projektaki Karşılığı | İlgili Dosya(lar) |
|---|---|---|
| **Deployment & ReplicaSet** | web (HPA yönetiyor, min:1 max:4), api (1 replica), crawler (1 replica, ns: crawler) | `*-deployment.yaml` |
| **ConfigMap — envFrom** | Web deployment tüm anahtarları tek seferde alır | `web-deployment.yaml` → `web-configmap.yaml` |
| **ConfigMap — configMapKeyRef** | API, crawler ve testram her anahtarı tek tek referans eder | `api-deployment.yaml`, `crawler-deployment.yaml`, `testram-deployment.yaml` |
| **Services (ClusterIP)** | Varsayılan type, port→targetPort mapping | `*-service.yaml` |
| **Multi-container Pod (Sidecar)** | 3 crawler container aynı Pod içinde çalışır | `crawler-deployment.yaml` |
| **Volumes — emptyDir** | Sidecar'lar arasında `/cache` paylaşımı | `crawler-deployment.yaml` |
| **PersistentVolumeClaim** | API Pod'u PVC ile `/persist` dizinine kalıcı depolama mount eder | `api-pvc.yaml`, `api-deployment.yaml` |
| **Horizontal Pod Autoscaler (HPA)** | Web deployment CPU-based auto-scaling, testcpu HPA | `web-hpa.yaml`, `testcpu-hpa.yaml` |
| **Resource Limits (CPU/Memory)** | testcpu CPU limit (10m), testram memory limit (256Mi) | `testcpu-deployment.yaml`, `testram-deployment.yaml` |
| **Labels & Selectors** | Service→Pod eşleşmesi `app: synergychat-*` | tüm dosyalar |
| **Namespaces** | Crawler kaynakları ayrı namespace'de, cross-namespace Service erişimi | `crawler-*.yaml` |
| **Gateway API** | GatewayClass → Gateway → HTTPRoute zinciri, L7 routing | `app-gateway*.yaml`, `*-httproute.yaml` |
| **Container Image** | `image: latest` kullanımı, `docker.io` prefix farkı | `*-deployment.yaml` |

### CKAD Sınavında Dikkat Edilecek Detaylar

- **`envFrom` vs `configMapKeyRef`**: Web deployment `envFrom` ile ConfigMap'in tüm anahtarlarını ortam değişkeni olarak alır. API ve crawler ise `configMapKeyRef` ile spesifik anahtarları seçerek alır. Sınavda her iki yöntemi de bilmen gerekir.
- **`selector.matchLabels` vs `template.metadata.labels`**: Deployment'ta bu iki alanın eşleşmesi zorunludur. Aksi halde `kubectl apply` hata verir.
- **Service `targetPort`**: Pod'un dinlediği port (8080) ile Service'in sunduğu port (80) farklıdır. Sınavda bu mapping sıkça sorulur.
- **emptyDir lifecycle**: Pod silindiğinde veri kaybolur. emptyDir, Pod yeniden başlatıldığında bile veriyi korur ama Pod silinirse veri gider.
- **PVC vs emptyDir**: PVC, Pod silinse bile veriyi korur (kalıcı depolama). emptyDir ise Pod silindiğinde veriyi kaybeder (geçici depolama). Sınavda hangi durumda hangisini kullanacağını bilmen gerekir.
- **HPA ve Replica Yönetimi**: HPA aktifken deployment'ın replica sayısını `kubectl scale` ile değiştirmek çakışmaya yol açar. HPA yönetilen deployment'larda `scale` komutundan kaçın.
- **Memory Limits ve OOMKilled**: Container bellek limitini aştığında `OOMKilled` nedeniyle sonlanır. `kubectl describe pod` çıktısında `Last State: Terminated, Reason: OOMKilled` görürsen nedeni bellidir.
- **Cross-Namespace Service Erişimi**: `<service-name>.<namespace>.svc.cluster.local` formatıyla farklı namespace'deki servislere erişilebilir. API'nin crawler servisine erişimi bu şekildedir.

---

## Kaynak Tablosu

### Aktif Kaynaklar

| Dosya | Kind | İsim | Açıklama |
|---|---|---|---|
| `api-configmap.yaml` | ConfigMap | `synergychat-api-configmap` | API port, DB yolu ve crawler URL |
| `api-deployment.yaml` | Deployment | `synergychat-api` | API backend, 1 replica, PVC mount (/persist) |
| `api-service.yaml` | Service | `api-service` | ClusterIP, 80→8080 |
| `api-httproute.yaml` | HTTPRoute | `api-httproute` | `synchatapi.internal` → `api-service:80` |
| `api-pvc.yaml` | PersistentVolumeClaim | `synergychat-api-pvc` | API için 1Gi kalıcı depolama (ReadWriteOnce) |
| `web-configmap.yaml` | ConfigMap | `synergychat-web-configmap` | Web port ve API URL |
| `web-deployment.yaml` | Deployment | `synergychat-web` | Web frontend, HPA yönetiyor (min:1, max:4) |
| `web-service.yaml` | Service | `web-service` | ClusterIP, 80→8080 |
| `web-httproute.yaml` | HTTPRoute | `web-httproute` | `synchat.internal` → `web-service:80` |
| `web-hpa.yaml` | HorizontalPodAutoscaler | `web-hpa` | Web deployment CPU-based scaling (min:1, max:4, %50) |
| `crawler-configmap.yaml` | ConfigMap | `synergychat-crawler-configmap` | 3 port, keywords, DB path (ns: crawler) |
| `crawler-deployment.yaml` | Deployment | `synergychat-crawler` | 3 sidecar container, emptyDir (ns: crawler) |
| `crawler-service.yaml` | Service | `crawler-service` | ClusterIP, 80→8080 (ns: crawler) |
| `app-gatewayclass.yaml` | GatewayClass | `app-gatewayclass` | Envoy gateway controller tanımı |
| `app-gateway.yaml` | Gateway | `app-gateway` | HTTP listener :80 |
| `testcpu-deployment.yaml` | Deployment | `synergychat-testcpu` | CPU stress test workload (10m CPU limit) |
| `testcpu-hpa.yaml` | HorizontalPodAutoscaler | `testcpu-hpa` | CPU test HPA (min:1, max:1, %50) |
| `testram-configmap.yaml` | ConfigMap | `testram-configmap` | RAM test bellek miktarı (MEGABYTES: 10) |
| `testram-deployment.yaml` | Deployment | `synergychat-testram` | RAM stress test workload (256Mi memory limit) |

### Eski Kaynaklar (`old_configs/`)

| Dosya | Açıklama |
|---|---|
| `old-api-service.yaml` | API Service NodePort:30080 olarak tanımlı (Gateway API öncesi) |
| `old-emptydir-crawler-deployment.yaml` | Crawler deployment snapshot'u (namespace yok) |
| `old-emptydir-crawler-configmap.yaml` | Crawler configmap snapshot'u (namespace yok) |
| `web-deployment.yaml` | Canlı cluster dump'ından alınan web deployment (çöp veriler dahil) |
| `3-replicas-web-deployment.yaml` | Web deployment'ın 3 replikalı temiz versiyonu (HPA öncesi) |

---

## Kurulum

### Ön Koşullar

| Araç | Kurulum |
|---|---|
| **Minikube** | `curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube` |
| **kubectl** | `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl` |
| **Envoy Gateway** | Aşağıda |

### 1. Minikube Başlat

```bash
minikube start --driver=docker --nodes=1
```

### 2. Envoy Gateway CRD'lerini ve Controller'ı Kur

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.2.0 -n envoy-gateway-system --create-namespace
```

> Envoy Gateway kurulumu tamamlandıktan sonra `GatewayClass` readiness beklemeniz gerekir. Kontrol için:
> ```bash
> kubectl get gatewayclass app-gatewayclass -o wide
> ```
> `ACCEPTED` durumunda olmalı.

### 3. Namespace Oluştur

```bash
kubectl create namespace crawler
```

> Crawler kaynakları (`crawler-deployment`, `crawler-service`, `crawler-configmap`) `crawler` namespace'inde çalışır. Apply etmeden önce bu namespace'i oluşturmanız gerekir.

### 4. Kaynakları Uygula (Apply)

Kaynakları uygulama sırası önemlidir — ConfigMap'ler, PVC ve GatewayClass önce oluşturulmalıdır:

```bash
# 1. ConfigMaps (env referansları önce olmalı)
kubectl apply -f api-configmap.yaml
kubectl apply -f crawler-configmap.yaml -n crawler
kubectl apply -f web-configmap.yaml
kubectl apply -f testram-configmap.yaml

# 2. PVC (Deployment'tan önce)
kubectl apply -f api-pvc.yaml

# 3. Altyapı — GatewayClass + Gateway
kubectl apply -f app-gatewayclass.yaml
kubectl apply -f app-gateway.yaml

# 4. Deployments
kubectl apply -f api-deployment.yaml
kubectl apply -f crawler-deployment.yaml -n crawler
kubectl apply -f web-deployment.yaml
kubectl apply -f testcpu-deployment.yaml
kubectl apply -f testram-deployment.yaml

# 5. Services
kubectl apply -f api-service.yaml
kubectl apply -f crawler-service.yaml -n crawler
kubectl apply -f web-service.yaml

# 6. HPA'lar (Deployment'lar hazır olduktan sonra)
kubectl apply -f web-hpa.yaml
kubectl apply -f testcpu-hpa.yaml

# 7. HTTPRoutes (Gateway hazır olduktan sonra)
kubectl apply -f api-httproute.yaml
kubectl apply -f web-httproute.yaml
```

> **Tek seferde apply (hızlı yol):**
> ```bash
> kubectl apply -f .
> kubectl apply -f crawler-configmap.yaml -n crawler
> kubectl apply -f crawler-deployment.yaml -n crawler
> kubectl apply -f crawler-service.yaml -n crawler
> ```
> Kubernetes dependenci'leri kendi yönetir ancak ilk çalıştırmada bazı Pod'lar ConfigMap henüz oluşmadığı için `ContainerCreating` durumunda kalabilir. Bu durumda Pod'lar kısa sürede kendiliğinden düzelir.

### 5. Tunnel Başlat

Minikube üzerinde Gateway API'nin external IP'alabilmesi için:

```bash
# Ayrı bir terminalde çalıştır
minikube tunnel
```

Minikube üzerinde Gateway API'nin external IP'alabilmesi için eger calismazsa:
```bash
minikube tunnel --bind-address="127.0.0.1" -c
```

### 6. `/etc/hosts` Güncelle

```bash
# Minikube IP'sini al
MINIKUBE_IP=$(minikube ip)

# /etc/hosts'a ekle
echo "$MINIKUBE_IP synchat.internal synchatapi.internal" | sudo tee -a /etc/hosts
```

> **Not:** `minikube tunnel` çalışıyorsa, IP `127.0.0.1` olabilir. Bu durumda:
> ```bash
> echo "127.0.0.1 synchat.internal synchatapi.internal" | sudo tee -a /etc/hosts
> ```

---

## Test & Doğrulama

### Tüm Kaynakları Kontrol Et

```bash
kubectl get pods,svc,gateway,httproute,hpa,pvc -o wide
kubectl get pods,svc,configmap -n crawler
```

### PVC Durumu

```bash
kubectl get pvc
kubectl describe pvc synergychat-api-pvc
```

### HPA Durumu

```bash
kubectl get hpa
kubectl describe hpa web-hpa
kubectl describe hpa testcpu-hpa
```

### Resource Usage

```bash
kubectl top pods
kubectl top nodes
```

### Gateway Durumu

```bash
kubectl get gateway app-gateway -o wide
kubectl get httproute
```

### Uygulamaya Erişim

```bash
# Web frontend
curl http://synchat.internal

# API backend
curl http://synchatapi.internal
```

### Pod Loglarını İncele

```bash
# Tek container'lı Pod
kubectl logs -l app=synergychat-api

# Multi-container Pod — spesifik container
kubectl logs -l app=synergychat-crawler -n crawler -c synergychat-crawler-1
kubectl logs -l app=synergychat-crawler -n crawler -c synergychat-crawler-2
kubectl logs -l app=synergychat-crawler -n crawler -c synergychat-crawler-3
```

### Detaylı Pod Bilgisi

```bash
kubectl describe pod -l app=synergychat-api
kubectl describe pod -l app=synergychat-crawler -n crawler
kubectl describe pod -l app=synergychat-testcpu
kubectl describe pod -l app=synergychat-testram
```

---

## CKAD Alıştırmaları

Her alıştırma, sınavda karşılaşabileceğin gerçek senaryoları temel alır. Kendi kendine çalış: önce dene, sonra çözüme bak.

### Alıştırma 1 — Replica Scaling

Web deployment'ı 5 replica'ya ölçeklendir:

```bash
kubectl scale deployment synergychat-web --replicas=5
kubectl get pods -l app=synergychat-web
```

Geri al:

```bash
kubectl scale deployment synergychat-web --replicas=3
```

**CKAD ipucu:** Sınavda `kubectl scale` yerine `kubectl edit` veya `kubectl patch` ile de yapılabilir. Hız için `scale` komutunu tercih et.

### Alıştırma 2 — ConfigMap Güncelleme ve Pod Restart

API'nin port'unu değiştirmek istiyorsun:

```bash
# ConfigMap'i güncelle
kubectl edit configmap synergychat-api-configmap
# API_PORT: "8080" → API_PORT: "9090"
```

> **Dikkat:** ConfigMap güncellendiğinde Pod otomatik olarak yeniden başlamaz. Değişikliğin etkili olması için:
> ```bash
> kubectl rollout restart deployment synergychat-api
> kubectl rollout status deployment synergychat-api
> ```

**CKAD ipucu:** ConfigMap değişikliklerinin Pod'lara yansıması için `rollout restart` gerekir. Bu, sınavın en çok bilinen tuzaklarından biridir.

### Alıştırma 3 — Sidecar Container Debug

Crawler Pod'undaki 2. sidecar'ın loglarını al:

```bash
kubectl logs -l app=synergychat-crawler -n crawler -c synergychat-crawler-2
```

Sidecar container'larından birini geçici olarak hata durumuna sok ve izle:

```bash
# Pod'un detaylı bilgisini al
kubectl describe pod -l app=synergychat-crawler -n crawler

# Tüm container'ların durumunu gör
kubectl get pods -l app=synergychat-crawler -n crawler -o jsonpath='{.items[*].status.containerStatuses[*].name}'
```

### Alıştırma 4 — Service Selector Kırma ve Onarma

`api-service.yaml`'daki selector'ü yanlış label ile değiştir ve trafiğin kesildiğini gözlemle:

```bash
# Service'i düzenle
kubectl edit svc api-service
# selector.app: synergychat-api → selector.app: wrong-label

# Endpoint'lerin boş olduğunu gör
kubectl get endpoints api-service

# Düzelt
kubectl edit svc api-service
# selector.app: wrong-label → selector.app: synergychat-api

# Endpoint'lerin geri geldiğini doğrula
kubectl get endpoints api-service
```

**CKAD ipucu:** Service trafiği ulaşmıyorsa ilk kontrol etmen gereken şey `selector` eşleşmesidir. `kubectl get endpoints <svc-name>` ile hızlıca kontrol edebilirsin.

### Alıştırma 5 — Rolling Update

Web deployment'ının image'ını güncelle:

```bash
kubectl set image deployment/synergychat-web synergychat-web=bootdotdev/synergychat-web:v2

# Rollout durumunu izle
kubectl rollout status deployment/synergychat-web

# Geri al
kubectl rollout undo deployment/synergychat-web

# Rollout geçmişini gör
kubectl rollout history deployment/synergychat-web
```

### Alıştırma 6 — Resource Limits Ekleme

Herhangi bir deployment'a resource requests ve limits ekle:

```bash
kubectl edit deployment synergychat-api
```

Aşağıdaki bloğu container spec'e ekle:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**CKAD ipucu:** Sınavda `kubectl edit` ile hızlıca eklenebilir. Alternatif olarak patch komutu:
```bash
kubectl patch deployment synergychat-api --type json -p '[{"op":"add","path":"/spec/template/spec/containers/0/resources","value":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"500m","memory":"256Mi"}}}]'
```

### Alıştırma 7 — Node Affinity

Minikube tek node olduğu için `nodeSelector` ve affinity kavramlarını anlamak için:

```bash
# Node label'larını gör
kubectl get nodes --show-labels

# Deployment'a nodeSelector ekle
kubectl edit deployment synergychat-web
```

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
```

### Alıştırma 8 — Pod Silme ve Self-Healing

Bir Pod'u sil ve ReplicaSet'in onu otomatik olarak yeniden oluşturmasını gözlemle:

```bash
# Pod adını al
kubectl get pods -l app=synergychat-web

# Sil
kubectl delete pod <pod-adı>

# Yeni Pod'un oluştuğunu gör
kubectl get pods -l app=synergychat-web
```

**CKAD ipucu:** Sınavda Pod silme, deployment'ın self-healing özelliğini test etmek için kullanılır. Pod adının değiştiğine dikkat et — yeni Pod yeni bir ad alır.

### Alıştırma 9 — HPA ile Otomatik Ölçeklendirme

Web deployment'ın HPA davranışını gözlemle:

```bash
# HPA durumunu kontrol et
kubectl get hpa web-hpa

# Web Pod'larını izle (ayrı terminal)
kubectl get pods -l app=synergychat-web -w

# Yük oluşturmak için (birden fazla istek)
kubectl run load-gen --image=busybox --rm -it --restart=Never -- /bin/sh -c "while true; do wget -qO- http://web-service:80; done"

# HPA'nın replica sayısını artırdığını gözlemle
kubectl get hpa web-hpa -w

# Yükü durdur, replica sayısının azaldığını gözlemle
```

**CKAD ipucu:** HPA aktifken `kubectl scale` ile replica değiştirmek çakışmaya yol açar. HPA kendi min/max aralığında yönetim sağlar.

### Alıştırma 10 — PVC ve Veri Kalıcılığı

API Pod'unda PVC mount edilen `/persist` dizininin Pod silindikten sonra veriyi koruduğunu doğrula:

```bash
# PVC durumunu kontrol et
kubectl get pvc synergychat-api-pvc

# API Pod'una bağlan ve veri yaz
kubectl exec <api-pod> -- ls /persist/

# Pod'u sil ve yeni Pod oluşmasını bekle
kubectl delete pod <api-pod>
kubectl get pods -l app=synergychat-api -w

# Yeni Pod oluştuğunda verinin hala mevcut olduğunu kontrol et
kubectl exec <yeni-api-pod> -- ls /persist/
```

**CKAD ipucu:** PVC, Pod silinse bile veriyi korur. emptyDir ise Pod silindiğinde veriyi kaybeder. Sınavda hangi volume tipinin kullanılacağı sorusu sıkça gelir.

### Alıştırma 11 — Memory Limits ve OOMKilled

testram deployment'ında bellek limitini aşarak `OOMKilled` durumunu gözlemle:

```bash
# testram ConfigMap'ini güncelle — 500MB bellek kullanımı
kubectl edit configmap testram-configmap
# MEGABYTES: "10" → MEGABYTES: "500"

# Deployment'ı yeniden başlat
kubectl rollout restart deployment synergychat-testram

# Pod'un OOMKilled durumunu gözlemle
kubectl get pods -l app=synergychat-testram -w

# Detaylı bilgi al
kubectl describe pod -l app=synergychat-testram
# Last State: Terminated, Reason: OOMKilled

# Geri al
kubectl edit configmap testram-configmap
# MEGABYTES: "500" → MEGABYTES: "10"
kubectl rollout restart deployment synergychat-testram
```

**CKAD ipucu:** Container bellek limitini aştığında kernel tarafından `OOMKilled` edilir. `kubectl describe pod` çıktısında bu nedeni görebilirsin. CPU limiti aşımında ise container throttle edilir, öldürülmez.

---

## Gateway API vs NodePort — Mimari Geçiş

### Eski Yaklaşım: NodePort

```yaml
# old_configs/old-api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort          # ← Dışarıya açık port
  selector:
    app: synergychat-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080     # ← NodePort ataması
```

**NodePort sorunları:**
- Port aralığı sınırlı (30000-32767)
- Domain-based routing yok — her servis için ayrı port gerekir
- Production'da güvenlik riski (tüm node'lar üzerinde açık)
- L7 özellikleri (path-based routing, header matching) yok

### Yeni Yaklaşım: Gateway API

```yaml
# app-gatewayclass.yaml — Gateway controller tanımı
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: app-gatewayclass
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller

---
# app-gateway.yaml — Gateway instance
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
spec:
  gatewayClassName: app-gatewayclass
  listeners:
    - name: http
      protocol: HTTP
      port: 80

---
# api-httproute.yaml — Domain-based routing
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-httproute
spec:
  parentRefs:
    - name: app-gateway
  hostnames:
    - "synchatapi.internal"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: api-service
          port: 80
```

**Gateway API avantajları:**
- Domain-based routing — tek IP üzerinden birden fazla servis
- Path-based routing desteği (`PathPrefix`, `Exact`, `RegularExpression`)
- Header ve query param matching
- L7 load balancing
- ClusterRole/RoleBinding ile yetki ayrımı (infra team vs app team)
- CKAD v1.31+ müfredatında yer alıyor

---

## Crawler Sidecar Pattern — Detaylı İnceleme

`crawler-deployment.yaml`, Kubernetes'te **sidecar (multi-container)** pattern'inin güzel bir örneğidir:

```
synergychat-crawler Pod (namespace: crawler)
┌────────────────────────────────────────────┐
│  shared emptyDir volume: /cache            │
│  ┌──────────────┐ ┌──────────────┐        │
│  │ crawler-1    │ │ crawler-2    │        │
│  │ port: 8080   │ │ port: 8081   │        │
│  │ env: PORT    │ │ env: PORT_2  │        │
│  └──────┬───────┘ └──────┬───────┘        │
│         │                 │                │
│         └─────┬───────────┘                │
│               ▼                            │
│  ┌──────────────────┐                     │
│  │ crawler-3        │                     │
│  │ port: 8082       │                     │
│  │ env: PORT_3      │                     │
│  └──────────────────┘                     │
└────────────────────────────────────────────┘
```

**Önemli noktalar:**

1. **3 container, 1 Pod:** Hepsi aynı network namespace'ini paylaşır — `localhost` üzerinden birbirlerine erişebilirler
2. **emptyDir volume:** Tüm container'lar `/cache` dizinini paylaşır. Pod silindiğinde veri kaybolur
3. **ConfigMap ile port yönetimi:** Her container farklı bir configMap anahtarı kullanır (`CRAWLER_PORT`, `CRAWLER_PORT_2`, `CRAWLER_PORT_3`)
4. **Service yalnızca 1. container'ı hedefler:** `crawler-service` yalnızca `targetPort: 8080`'ı (crawler-1) expose eder. Diğer container'lara doğrudan dışarıdan erişilemez
5. **Cross-namespace erişim:** API, `http://crawler-service.crawler.svc.cluster.local` ile farklı namespace'deki crawler servisine erişir

**CKAD sınavında dikkat et:**
- `kubectl logs -c <container-adı>` ile spesifik container'ın loglarını alabilirsin
- `kubectl exec -c <container-adı>` ile spesifik container'da komut çalıştırabilirsin
- Sidecar container'lardan biri crash ederse Pod `CrashLoopBackOff` durumuna geçer
- Cross-namespace Service erişimi: `<svc>.<ns>.svc.cluster.local`

---

## Persistent Volume Pattern — Detaylı İnceleme

`api-pvc.yaml` ve `api-deployment.yaml`, Kubernetes'te **PersistentVolumeClaim** pattern'inin uygulamasıdır:

```
synergychat-api Pod
┌──────────────────────────────────────┐
│  container: synergychat-api         │
│  ┌────────────────────────────────┐ │
│  │ volumeMount:                   │ │
│  │   name: synergychat-api-volume │ │
│  │   mountPath: /persist          │ │
│  └─────────────┬──────────────────┘ │
│                │                    │
│  ┌─────────────▼──────────────────┐ │
│  │ volume:                        │ │
│  │   persistentVolumeClaim:       │ │
│  │     claimName:                 │ │
│  │       synergychat-api-pvc      │ │
│  └────────────────────────────────┘ │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│  PersistentVolumeClaim               │
│  name: synergychat-api-pvc          │
│  accessModes: ReadWriteOnce          │
│  storage: 1Gi                       │
│  (Minikube host-path PV              │
│   dinamik olarak sağlanır)           │
└──────────────────────────────────────┘
```

**PVC vs emptyDir karşılaştırması:**

| Özellik | PVC | emptyDir |
|---|---|---|
| Veri kalıcılığı | Pod silinse bile veri korunur | Pod silindiğinde veri kaybolur |
| Erişim modu | `ReadWriteOnce`, `ReadWriteMany` | Pod içi erişim |
| Dinamik provisioning | StorageClass ile otomatik PV oluşturma | Yok |
| Kullanım senaryosu | Veritabanı, dosya depolama | Geçici cache, shared buffer |
| Bağımsız yaşam döngüsü | Evet — Pod'dan bağımsız | Hayır — Pod'a bağlı |

**CKAD sınavında dikkat et:**
- PVC kullanmadan önce `PersistentVolume` veya `StorageClass` olmalıdır
- Minikube'de `standard` StorageClass varsayılan olarak dinamik provisioning sağlar
- `ReadWriteOnce` sadece tek node tarafından mount edilebilir (Minikube tek node — sorun değil)
- `kubectl get pvc` ile PVC durumunu, `kubectl describe pvc` ile detayları görebilirsin

---

## Advanced Scaling & Resource Management

### Horizontal Pod Autoscaler (HPA)

HPA, CPU/memory kullanımına göre deployment'ın replica sayısını otomatik olarak ölçeklendirir.

**web-hpa.yaml — Web frontend auto-scaling:**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: synergychat-web
  minReplicas: 1
  maxReplicas: 4
  targetCPUUtilizationPercentage: 50
```

**testcpu-hpa.yaml — CPU test HPA:**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: testcpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: synergychat-testcpu
  minReplicas: 1
  maxReplicas: 1
  targetCPUUtilizationPercentage: 50
```

**HPA çalışma prensibi:**

1. HPA, `scaleTargetRef` ile hedef deployment'ı belirler
2. `targetCPUUtilizationPercentage` hedef CPU yüzdesini tanımlar
3. Pod'ların ortalama CPU kullanımı hedefi aştığında HPA replica sayısını artırır
4. CPU kullanımı altına düştüğünde HPA replica sayısını azaltır (minReplicas'a kadar)
5. replica sayısı `maxReplicas`'ı geçemez

```bash
# HPA durumunu izle
kubectl get hpa -w

# Detaylı HPA bilgisi
kubectl describe hpa web-hpa
```

### CPU Limits — testcpu

```yaml
# testcpu-deployment.yaml
containers:
  - image: bootdotdev/synergychat-testcpu:latest
    name: synergychat-testcpu
    resources:
      limits:
        cpu: 10m    # 10 millicores = %1 CPU
```

`10m` CPU limiti, container'ın en fazla %1 CPU kullanabileceği anlamına gelir. Bu extremely düşük bir limit ve HPA davranışını gözlemlemek için tasarlanmıştır.

- **CPU throttle:** CPU limiti aşıldığında container öldürülmez, yavaşlatılır (throttle)
- **CPU vs Memory farkı:** CPU limiti aşımında throttle, memory limiti aşımında `OOMKilled`

### Memory Limits & OOMKilled — testram

```yaml
# testram-deployment.yaml
containers:
  - image: bootdotdev/synergychat-testram:latest
    name: synergychat-testram
    resources:
      limits:
        memory: 256Mi
    env:
      - name: MEGABYTES
        valueFrom:
          configMapKeyRef:
            name: testram-configmap
            key: MEGABYTES
```

```yaml
# testram-configmap.yaml
data:
  MEGABYTES: "10"
  # MEGABYTES: "500"  ← OOMKilled tetiklemek için
```

**OOMKilled senaryosu:**

1. Varsayılan: `MEGABYTES: "10"` → 10MB bellek kullanımı, 256Mi limit dahilinde, normal çalışır
2. Tetikleme: `MEGABYTES: "500"` → 500MB bellek kullanımı, 256Mi limiti aşar → `OOMKilled`
3. ConfigMap güncellenip rollout restart yapıldığında Pod yeniden başlar ve bellek limitini aşarsa kernel tarafından öldürülür

```bash
# OOMKilled Pod'ları bul
kubectl get pods -l app=synergychat-testram -o jsonpath='{.items[*].status.containerStatuses[0].lastState.reason}'

# Detaylı neden gör
kubectl describe pod -l app=synergychat-testram | grep -A5 "Last State"
```

---

## Bu Projede Olmayan CKAD Konuları

Aşağıdaki konular CKAD müfredatında yer alır ancak bu projede uygulamalı olarak yer almaz. Sınava hazırlanırken bu konuları da çalışmanı öneririz:

| CKAD Konusu | Kısa Açıklama |
|---|---|
| **Secrets** | Hassas verileri base64 ile saklama, env/volume olarak mount etme |
| **Init Containers** | Ana container'dan önce çalışan hazırlık container'ları |
| **Probes (Liveness/Readiness/Startup)** | Pod sağlık kontrolü, `exec`/`httpGet`/`tcpSocket` tipleri |
| **NetworkPolicies** | Ingress/Egress kısıtlama, pod selector ile trafik izolasyonu |
| **Jobs & CronJobs** | Tek seferlik ve zamanlanmış batch işleri, `completions`, `parallelism` |
| **RBAC** | ServiceAccount, Role, RoleBinding, ClusterRole, ClusterRoleBinding |
| **Taints & Tolerations** | Node scheduling kısıtlama, `NoSchedule`/`NoExecute` efektleri |
| **Pod Affinity/Anti-Affinity** | Pod'ları aynı/farklı node'a yerleştirme stratejileri |
| **Security Context** | `runAsUser`, `fsGroup`, `capabilities`, `readOnlyRootFilesystem` |
| **ServiceAccount** | Pod'lar için kimlik, token mount, ImagePullSecrets |
| **StorageClass & PV** | Dinamik provisioning, accessModes, reclaimPolicy |
| **StatefulSet** | Stable network identity, ordered deploy/scale, volumeClaimTemplates |
| **DaemonSet** | Her node'da bir Pod çalıştırma (log agent, monitoring vb.) |
| **Ingress** | Gateway API dışındaki geleneksel L7 routing yöntemi |

---

## Sorun Giderme (Troubleshooting)

### Pod Durumları

```bash
# Tüm Pod'ları listele
kubectl get pods -o wide
kubectl get pods -n crawler -o wide

# Pod'un detaylı bilgisini al
kubectl describe pod <pod-adı>

# CrashLoopBackOff durumundaki Pod'un loglarını al
kubectl logs <pod-adı> --previous

# OOMKilled Pod'larını bul
kubectl describe pod <pod-adı> | grep -A5 "Last State"
```

### Service Endpoint Kontrolü

```bash
# Service'in hangi Pod'lara yönlendiğini gör
kubectl get endpoints api-service
kubectl get endpoints web-service
kubectl get endpoints crawler-service -n crawler

# Endpoint boşsa → selector eşleşmesi yanlış
```

### PVC Debug

```bash
# PVC durumunu kontrol et
kubectl get pvc
kubectl describe pvc synergychat-api-pvc

# PVC'nin bağlı olduğu PV'yi gör
kubectl get pv
```

### HPA Debug

```bash
# HPA durumunu kontrol et
kubectl get hpa
kubectl describe hpa web-hpa

# HPA olaylarını izle
kubectl get events --field-selector reason=SuccessfulRescale
```

### Gateway ve HTTPRoute Debug

```bash
# Gateway durumunu kontrol et
kubectl describe gateway app-gateway

# HTTPRoute'ları listele
kubectl get httproute -o wide

# Gateway'in aldığı IP'yi gör
kubectl get gateway app-gateway -o jsonpath='{.status.addresses[0].value}'
```

### ConfigMap ve Ortam Değişkenleri

```bash
# ConfigMap içeriğini gör
kubectl get configmap synergychat-api-configmap -o yaml
kubectl get configmap testram-configmap -o yaml

# Pod'daki ortam değişkenlerini listele
kubectl exec <pod-adı> -- env | sort
```

### Network Debug

```bash
# Cluster içi DNS çözümleme
kubectl run tmp --image=busybox --rm -it --restart=Never -- nslookup api-service.default.svc.cluster.local

# Cross-namespace DNS çözümleme
kubectl run tmp --image=busybox --rm -it --restart=Never -- nslookup crawler-service.crawler.svc.cluster.local

# Service'e cluster içinden istek at
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- http://api-service:80

# Dash temp pod ile çoklu test
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh
# Pod içinde:
# wget -qO- http://web-service:80
# wget -qO- http://crawler-service.crawler.svc.cluster.local:80
# nslookup synchat.internal
```

---

## Dosya Yapısı

```
learn_kubernetes/
├── app-gatewayclass.yaml        # GatewayClass — Envoy controller tanımı
├── app-gateway.yaml             # Gateway — HTTP listener :80
├── api-configmap.yaml           # ConfigMap — API ortam değişkenleri
├── api-deployment.yaml          # Deployment — API backend (1 replica, PVC mount)
├── api-service.yaml             # Service — API ClusterIP 80→8080
├── api-httproute.yaml           # HTTPRoute — synchatapi.internal → API
├── api-pvc.yaml                 # PersistentVolumeClaim — API 1Gi kalıcı depolama
├── crawler-configmap.yaml       # ConfigMap — Crawler ortam değişkenleri (ns: crawler)
├── crawler-deployment.yaml      # Deployment — Crawler (3 sidecar + emptyDir, ns: crawler)
├── crawler-service.yaml         # Service — Crawler ClusterIP 80→8080 (ns: crawler)
├── web-configmap.yaml           # ConfigMap — Web ortam değişkenleri
├── web-deployment.yaml          # Deployment — Web frontend (HPA: min:1, max:4)
├── web-service.yaml             # Service — Web ClusterIP 80→8080
├── web-httproute.yaml           # HTTPRoute — synchat.internal → Web
├── web-hpa.yaml                 # HorizontalPodAutoscaler — Web CPU-based scaling
├── testcpu-deployment.yaml      # Deployment — CPU stress test (10m CPU limit)
├── testcpu-hpa.yaml             # HorizontalPodAutoscaler — CPU test auto-scaling
├── testram-configmap.yaml       # ConfigMap — RAM test bellek miktarı
├── testram-deployment.yaml      # Deployment — RAM stress test (256Mi memory limit)
├── old_configs/                 # Eski yapılandırmalar (arşiv)
│   ├── old-api-service.yaml           # NodePort:30080 (eski)
│   ├── old-emptydir-crawler-deployment.yaml
│   ├── old-emptydir-crawler-configmap.yaml
│   ├── web-deployment.yaml            # Cluster dump (çöp veriler dahil)
│   └── 3-replicas-web-deployment.yaml # Web deployment 3 replikalı temiz versiyon (HPA öncesi)
└── README.md
```

---

## Kaynakça ve Devam Edelim

- [Kubernetes Gateway API Dokümantasyonu](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway Kurulum Rehberi](https://gateway.envoyproxy.io/)
- [CNCF CKAD Sınav Müfredatı](https://github.com/cncf/curriculum)
- [Kubernetes Resmi Dokümantasyon](https://kubernetes.io/docs/)
- [Kubernetes HPA Dokümantasyonu](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)