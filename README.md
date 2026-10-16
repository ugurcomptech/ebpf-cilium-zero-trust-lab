# Production-Grade eBPF & Cilium Infrastructure Lab (Multi-Tier Zero-Trust Architecture)

Bu repository; standart CNI eklentilerinden arındırılmış, doğrudan Linux çekirdeği seviyesinde eBPF (Extended Berkeley Packet Filter) ve Cilium CNI kullanan, Zero-Trust (Sıfır Güven) prensibiyle izole edilmiş kurumsal katmanlı bir Kubernetes (K3s) altyapı laboratuvarıdır.

Proje; klasik IP/Port (L4) engellemelerinin ötesine geçerek Envoy proxy entegrasyonu ile L7 (HTTP/API) seviyesinde trafik denetimi yapmayı, Hubble ile gerçek zamanlı ağ gözlemlenebilirliğini (observability) ve dağıtık sistem dayanıklılığını test etmek amacıyla tasarlanmıştır.

---

## Mimari ve Bileşenler

* **Orkestrasyon:** K3s (Lightweight Kubernetes v1.30+)
* **Ağ Altyapısı (CNI):** Cilium (Flannel devre dışı bırakılmış, eBPF tabanlı Native Routing / DSR modu)
* **Gözlemlenebilirlik (Observability):** Hubble UI & CLI (Çekirdek seviyesinde paket akışı ve drop analizleri)
* **Uygulama Katmanı (Enterprise 4-Tier Stack):**
  1. Ingress / L7 Katmanı: Envoy Proxy (Cilium L7 Policy Engine)
  2. Frontend Katmanı: Nginx (`enterprise-frontend`)
  3. Backend / API Katmanı: Nginx / API Service (`enterprise-backend`)
  4. Cache Katmanı: Redis (`enterprise-cache`)
  5. Database Katmanı: PostgreSQL (`enterprise-db`)

---

## Gereksinimler (Prerequisite)

* **2 Adet Sanal Sunucu (VDS / Cloud VM - Ubuntu 24.04 LTS önerilir):**
  * **Master Node:** Min. 2 vCPU, 4 GB RAM
  * **Worker Node:** Min. 2 vCPU, 4 GB RAM
* **Ağ Gereksinimleri:** Sunucular arasında port kısıtlamasız dahili/harici erişim ve root yetkisi.

---

## Adım Adım Kurulum Rehberi

#### Adım 1: K3s Master Kurulumu (Flannel Olmadan)
Master sunucusunda, standart CNI yerine Cilium entegrasyonu yapabilmek için Flannel'ı devre dışı bırakarak K3s başlatalım:

```bash
curl -sfL [https://get.k3s.io](https://get.k3s.io) | INSTALL_K3S_EXEC="server" sh -s - \
  --flannel-backend=none \
  --disable-network-policy \
  --node-ip=<MASTER_IP>
  ```

#### Adım 2: Worker Node'u Küme Dahil Etme
Master kurulumundan sonra ekrana gelen token ile worker sunucusunu cluster'a bağlayın:

```bash
curl -sfL [https://get.k3s.io](https://get.k3s.io) | K3S_URL="https://<MASTER_IP>:6443" K3S_TOKEN="<NODE_TOKEN>" INSTALL_K3S_EXEC="agent" sh -s - \
  --node-ip=<WORKER_IP>
```

## Adım 3: Cilium CNI ve Hubble Kurulumu
Master üzerinde Helm kullanarak Cilium'u ve eBPF gözlemlenebilirlik aracı Hubble'ı yükleyelim:

```bash
helm repo add cilium [https://helm.cilium.io/](https://helm.cilium.io/)
helm repo update

helm install cilium cilium/cilium --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set socketLB.enabled=true \
  --set bpf.masquerade=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```


## Test: Kurumsal Mimariyi Deploy Etme (4-Tier Stack)

Tüm katmanları (frontend, backend, cache, database) ve aralarındaki sıfır-güven eBPF kurallarını tek komutla ayağa kaldıralım.

#### 1. Stack Dosyası (enterprise-stack.yaml)

Aşağıdaki içeriğe sahip `enterprise-stack.yaml` dosyasını oluşturun:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: enterprise-app
---
# Database Katmanı (PostgreSQL)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise-db
  namespace: enterprise-app
  labels:
    app: enterprise-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: enterprise-db
  template:
    metadata:
      labels:
        app: enterprise-db
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: enterpriseSecret
---
apiVersion: v1
kind: Service
metadata:
  name: db-svc
  namespace: enterprise-app
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: enterprise-db
---
# Cache Katmanı (Redis)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise-cache
  namespace: enterprise-app
  labels:
    app: enterprise-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: enterprise-cache
  template:
    metadata:
      labels:
        app: enterprise-cache
    spec:
      containers:
      - name: redis
        image: redis:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: cache-svc
  namespace: enterprise-app
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: enterprise-cache
---
# Backend API Katmanı (Nginx / HTTP Server)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise-backend
  namespace: enterprise-app
  labels:
    app: enterprise-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: enterprise-backend
  template:
    metadata:
      labels:
        app: enterprise-backend
    spec:
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: enterprise-app
spec:
  ports:
  - port: 8080
    targetPort: 80
  selector:
    app: enterprise-backend
---
# Frontend Katmanı (Nginx)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enterprise-frontend
  namespace: enterprise-app
  labels:
    app: enterprise-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: enterprise-frontend
  template:
    metadata:
      labels:
        app: enterprise-frontend
    spec:
      containers:
      - name: web
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: enterprise-app
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: frontend
```


#### 2. eBPF Sıfır-Güven Güvenlik Politikaları

Aşağıdaki `enterprise-security.yaml` dosyası ile katmanlar arası yetkisiz erişimleri çekirdek seviyesinde bloklayalım:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: protect-database-enterprise
  namespace: enterprise-app
spec:
  endpointSelector:
    matchLabels:
      app: enterprise-db
  ingress:
  - toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
    fromEndpoints:
    - matchLabels:
        app: enterprise-backend
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: protect-cache-enterprise
  namespace: enterprise-app
spec:
  endpointSelector:
    matchLabels:
      app: enterprise-cache
  ingress:
  - toPorts:
    - ports:
      - port: "6379"
        protocol: TCP
    fromEndpoints:
    - matchLabels:
        app: enterprise-backend
```

Uygulamayı cluster'a basalım:
```bash
kubectl apply -f enterprise-stack.yaml
kubectl apply -f enterprise-security.yaml
```

## Test Senaryoları ve Doğrulama

#### 1. L4 Güvenlik Duvarı Testi (Frontend -> DB Bloklanmalı)
Frontend pod'undan veritabanına erişmeyi deneyelim:

```bash
FRONTEND_POD=$(kubectl get pods -n enterprise-app -l app=enterprise-frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n enterprise-app $FRONTEND_POD -- nc -zv db-svc 5432
# Beklenen Çıktı: "Host is unreachable" (eBPF tarafından anında engellendi)
```
#### 2. L7 (HTTP / API) Seviyesi Testi

Cilium Envoy proxy entegrasyonu ile HTTP metod ve path filtrelemesi:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-api-filter
  namespace: enterprise-app
spec:
  endpointSelector:
    matchLabels:
      app: enterprise-backend
  ingress:
  - toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/public"
    fromEndpoints:
    - matchLabels:
        app: enterprise-frontend
```
Cluster'a basalım:

```
kubectl apply -f l7-policy.yaml
```


## Hubble UI ile Görsel Gözlem

Hubble arayüzünü dışarıya açarak canlı akışları (forwarded / dropped paketler) izleyin:

```
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 --address 0.0.0.0
```
- Tarayıcınızdan http://<MASTER_IP>:12000 adresine gidin ve enterprise-app namespace'ini seçin.

## Karşılaşılan Yaygın Sorunlar ve Çözümleri (Troubleshooting)

1- 503 Service Unavailable / Connection Refused (L7 Testlerinde):

- Neden: İstek atılan hedef pod'un içinde portu dinleyen gerçek bir HTTP sunucusu (Nginx vb.) yoktur.

- Çözüm: Deployment imajının bir web sunucusu çalıştırdığından emin olun.

2- Hubble UI Arayüzünde Verilerin Görünmemesi:

- Neden: Yanlış namespace seçilmiş olabilir veya ağda aktif trafik (transit packet) yoktur.

- Çözüm: Sol üst menüden doğru namespace'i seçin ve test komutlarıyla arka planda trafik üretin.

3- Broken Pipe Hataları (Port-Forward sırasında):

- Neden: Tarayıcının canlı polling (WebSocket) istekleri sırasında tünelin yenilenmesi.

- Çözüm: Tamamen kozmetiktir, sistemi etkilemez. Arka planda çalıştırmak için nohup kubectl port-forward ... & komutunu kullanabilirsiniz.