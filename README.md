# Minikube One Node Cluster — CKAD Çalışma Laboratuvarı

CKAD (Certified Kubernetes Application Developer) sınavı ve genel Kubernetes pratiği için oluşturulmuş 1-node Minikube cluster'ı. Gerçek bir 3-tier **SynergyChat** uygulaması üzerinden Deployment, Service, ConfigMap, Multi-container Pod, Volume ve Gateway API kavramlarını öğrenmeyi hedefler.

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
  synergychat-web (x3)     synergychat-api (x1)
                                   │ (internal)
                                   ▼
                          crawler-service:80
                                   │
                                   ▼
                         synergychat-crawler (x1 pod)
                         ┌── crawler-1 :8080 ──┐
                         ├── crawler-2 :8081 ──┤ emptyDir volume
                         └── crawler-3 :8082 ──┘   (/cache)
```

**Trafik akışı:**

1. Client, `synchat.internal` veya `synchatapi.internal` domain'ine istek gönderir
2. Envoy Gateway (L7) hostname'e göre `HTTPRoute` ile doğru Service'e yönlendirir
3. Service, `selector` ile eşleşen Pod'lara trafiği iletir (`port:80 → targetPort:8080`)
4. Web frontend, `API_URL=http://synchatapi.internal` ile API'ye erişir
5. API Pod'u yerel JSON dosyasından veri okur/yazar
6. Crawler Pod'u 3 sidecar container içerir; `emptyDir` volume üzerinden `/cache` dizinini paylaşır

---

## CKAD Konu Haritası — Bu Projede Hangi Kavramlar Var?

| CKAD Konusu | Projektaki Karşılığı | İlgili Dosya(lar) |
|---|---|---|
| **Deployment & ReplicaSet** | web (3 replica), api (1 replica), rollingUpdate varsayılan | `*-deployment.yaml` |
| **ConfigMap — envFrom** | Web deployment tüm anahtarları tek seferde alır | `web-deployment.yaml` → `web-configmap.yaml` |
| **ConfigMap — configMapKeyRef** | API ve crawler her anahtarı tek tek referans eder | `api-deployment.yaml`, `crawler-deployment.yaml` |
| **Services (ClusterIP)** | Varsayılan type, port→targetPort mapping | `*-service.yaml` |
| **Multi-container Pod (Sidecar)** | 3 crawler container aynı Pod içinde çalışır | `crawler-deployment.yaml` |
| **Volumes — emptyDir** | Sidecar'lar arasında `/cache` paylaşımı | `crawler-deployment.yaml` |
| **Labels & Selectors** | Service→Pod eşleşmesi `app: synergychat-*` | tüm dosyalar |
| **Gateway API** | GatewayClass → Gateway → HTTPRoute zinciri, L7 routing | `app-gateway*.yaml`, `*-httproute.yaml` |
| **Container Image** | `image: latest` kullanımı, `docker.io` prefix farkı | `*-deployment.yaml` |

### CKAD Sınavında Dikkat Edilecek Detaylar

- **`envFrom` vs `configMapKeyRef`**: Web deployment `envFrom` ile ConfigMap'in tüm anahtarlarını ortam değişkeni olarak alır. API ve crawler ise `configMapKeyRef` ile spesifik anahtarları seçerek alır. Sınavda her iki yöntemi de bilmen gerekir.
- **`selector.matchLabels` vs `template.metadata.labels`**: Deployment'ta bu iki alanın eşleşmesi zorunludur. Aksi halde `kubectl apply` hata verir.
- **Service `targetPort`**: Pod'un dinlediği port (8080) ile Service'in sunduğu port (80) farklıdır. Sınavda bu mapping sıkça sorulur.
- **emptyDir lifecycle**: Pod silindiğinde veri kaybolur. emptyDir, Pod yeniden başlatıldığında bile veriyi korur ama Pod silinirse veri gider.

---

## Kaynak Tablosu

### Aktif Kaynaklar

| Dosya | Kind | İsim | Açıklama |
|---|---|---|---|
| `api-configmap.yaml` | ConfigMap | `synergychat-api-configmap` | API port ve DB yolunu tutar |
| `api-deployment.yaml` | Deployment | `synergychat-api` | API backend, 1 replica |
| `api-service.yaml` | Service | `api-service` | ClusterIP, 80→8080 |
| `api-httproute.yaml` | HTTPRoute | `api-httproute` | `synchatapi.internal` → `api-service:80` |
| `web-configmap.yaml` | ConfigMap | `synergychat-web-configmap` | Web port ve API URL |
| `web-deployment.yaml` | Deployment | `synergychat-web` | Web frontend, 3 replica |
| `web-service.yaml` | Service | `web-service` | ClusterIP, 80→8080 |
| `web-httproute.yaml` | HTTPRoute | `web-httproute` | `synchat.internal` → `web-service:80` |
| `crawler-configmap.yaml` | ConfigMap | `synergychat-crawler-configmap` | 3 port, keywords, DB path |
| `crawler-deployment.yaml` | Deployment | `synergychat-crawler` | 3 sidecar container, emptyDir |
| `crawler-service.yaml` | Service | `crawler-service` | ClusterIP, 80→8080 |
| `app-gatewayclass.yaml` | GatewayClass | `app-gatewayclass` | Envoy gateway controller tanımı |
| `app-gateway.yaml` | Gateway | `app-gateway` | HTTP listener :80 |

### Eski Kaynaklar (`old_configs/`)

| Dosya | Açıklama |
|---|---|
| `old-api-service.yaml` | API Service NodePort:30080 olarak tanımlı (Gateway API öncesi) |
| `old-emptydir-crawler-deployment.yaml` | Crawler deployment snapshot'u |
| `old-emptydir-crawler-configmap.yaml` | Crawler configmap snapshot'u |
| `web-deployment.yaml` | Canlı cluster dump'ından alınan web deployment (çöp veriler dahil) |

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

### 3. Kaynakları Uygula (Apply)

Kaynakları uygulama sırası önemlidir — ConfigMap'ler ve GatewayClass önce oluşturulmalıdır:

```bash
# 1. ConfigMaps (env referansları önce olmalı)
kubectl apply -f api-configmap.yaml
kubectl apply -f crawler-configmap.yaml
kubectl apply -f web-configmap.yaml

# 2. Altyapı — GatewayClass + Gateway
kubectl apply -f app-gatewayclass.yaml
kubectl apply -f app-gateway.yaml

# 3. Deployments
kubectl apply -f api-deployment.yaml
kubectl apply -f crawler-deployment.yaml
kubectl apply -f web-deployment.yaml

# 4. Services
kubectl apply -f api-service.yaml
kubectl apply -f crawler-service.yaml
kubectl apply -f web-service.yaml

# 5. HTTPRoutes (Gateway hazır olduktan sonra)
kubectl apply -f api-httproute.yaml
kubectl apply -f web-httproute.yaml
```

> **Tek seferde apply (hızlı yol):**
> ```bash
> kubectl apply -f .
> ```
> Kubernetes dependenci'leri kendi yönetir ancak ilk çalıştırmada bazı Pod'lar ConfigMap henüz oluşmadığı için `ContainerCreating` durumunda kalabilir. Bu durumda Pod'lar kısa sürede kendiliğinden düzelir.

### 4. Tunnel Başlat

Minikube üzerinde Gateway API'nin external IP'alabilmesi için:

```bash
# Ayrı bir terminalde çalıştır
minikube tunnel
```

Minikube üzerinde Gateway API'nin external IP'alabilmesi için eger calismazsa:
```bash
minikube tunnel --bind-address="127.0.0.1" -c
```

### 5. `/etc/hosts` Güncelle

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
kubectl get pods,svc,gateway,httproute -o wide
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
kubectl logs -l app=synergychat-crawler -c synergychat-crawler-1
kubectl logs -l app=synergychat-crawler -c synergychat-crawler-2
kubectl logs -l app=synergychat-crawler -c synergychat-crawler-3
```

### Detaylı Pod Bilgisi

```bash
kubectl describe pod -l app=synergychat-api
kubectl describe pod -l app=synergychat-crawler
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
kubectl logs -l app=synergychat-crawler -c synergychat-crawler-2
```

Sidecar container'larından birini geçici olarak hata durumuna sok ve izle:

```bash
# Pod'un detaylı bilgisini al
kubectl describe pod -l app=synergychat-crawler

# Tüm container'ların durumunu gör
kubectl get pods -l app=synergychat-crawler -o jsonpath='{.items[*].status.containerStatuses[*].name}'
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

### Alştırma 8 — Pod Silme ve Self-Healing

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
synergychat-crawler Pod
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

**CKAD sınavında dikkat et:**
- `kubectl logs -c <container-adı>` ile spesifik container'ın loglarını alabilirsin
- `kubectl exec -c <container-adı>` ile spesifik container'da komut çalıştırabilirsin
- Sidecar container'lardan biri crash ederse Pod `CrashLoopBackOff` durumuna geçer

---

## Sorun Giderme (Troubleshooting)

### Pod Durumları

```bash
# Tüm Pod'ları listele
kubectl get pods -o wide

# Pod'un detaylı bilgisini al
kubectl describe pod <pod-adı>

# CrashLoopBackOff durumundaki Pod'un loglarını al
kubectl logs <pod-adı> --previous
```

### Service Endpoint Kontrolü

```bash
# Service'in hangi Pod'lara yönlendiğini gör
kubectl get endpoints api-service
kubectl get endpoints web-service
kubectl get endpoints crawler-service

# Endpoint boşsa → selector eşleşmesi yanlış
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

# Pod'daki ortam değişkenlerini listele
kubectl exec <pod-adı> -- env | sort
```

### Network Debug

```bash
# Cluster içi DNS çözümleme
kubectl run tmp --image=busybox --rm -it --restart=Never -- nslookup api-service.default.svc.cluster.local

# Service'e cluster içinden istek at
kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- http://api-service:80

# Dash temp pod ile çoklu test
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh
# Pod içinde:
# wget -qO- http://web-service:80
# wget -qO- http://crawler-service:80
# nslookup synchat.internal
```

---

## Dosya Yapısı

```
learn_kubernetes/
├── app-gatewayclass.yaml      # GatewayClass — Envoy controller tanımı
├── app-gateway.yaml           # Gateway — HTTP listener :80
├── api-configmap.yaml          # ConfigMap — API ortam değişkenleri
├── api-deployment.yaml         # Deployment — API backend (1 replica)
├── api-service.yaml            # Service — API ClusterIP 80→8080
├── api-httproute.yaml          # HTTPRoute — synchatapi.internal → API
├── crawler-configmap.yaml      # ConfigMap — Crawler ortam değişkenleri (3 port)
├── crawler-deployment.yaml     # Deployment — Crawler (3 sidecar + emptyDir)
├── crawler-service.yaml        # Service — Crawler ClusterIP 80→8080
├── web-configmap.yaml          # ConfigMap — Web ortam değişkenleri
├── web-deployment.yaml         # Deployment — Web frontend (3 replica)
├── web-service.yaml            # Service — Web ClusterIP 80→8080
├── web-httproute.yaml          # HTTPRoute — synchat.internal → Web
├── old_configs/                # Eski yapılandırmalar (arşiv)
│   ├── old-api-service.yaml           # NodePort:30080 (eski)
│   ├── old-emptydir-crawler-deployment.yaml
│   ├── old-emptydir-crawler-configmap.yaml
│   └── web-deployment.yaml            # Cluster dump (çöp veriler dahil)
└── README.md
```

---

## Kaynakça ve Devam Edelim

- [Kubernetes Gateway API Dokümantasyonu](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway Kurulum Rehberi](https://gateway.envoyproxy.io/)
- [CNCF CKAD Sınav Müfredatı](https://github.com/cncf/curriculum)
- [Kubernetes Resmi Dokümantasyon](https://kubernetes.io/docs/)
