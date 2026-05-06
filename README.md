# algorycode-helm-charts-prod

Üretim (prod) için Helm chart’ları ve Argo CD `Application` örnekleri. Eureka kullanılmaz; **Kubernetes Cluster DNS** (`*.svc.cluster.local`) ile sabit URI’ler tanımlanır. k3s tarafında dış erişim için genelde **Traefik Ingress** veya **Service type: LoadBalancer** (ServiceLB / Klipper-lb) kullanılır.

## Dizin yapısı

```
algorycode-helm-charts-prod/
├── README.md
├── rent-service/
│   ├── chart/                 # Helm chart (Chart.yaml, values.yaml, templates/)
│   ├── env/
│   │   └── prod-values.yaml   # Prod override
│   └── argo/
│       └── application.yaml   # Argo CD Application (örnek)
└── gateway-service/
    ├── chart/
    ├── env/
    │   └── prod-values.yaml   # gateway.routes[].uri → rent Service DNS
    └── argo/
        └── application.yaml
```

## Önkoşullar

- Kubernetes kümesi (ör. k3s) ve `kubectl` yapılandırması
- Argo CD kurulu ve **bu Git reposuna erişim** (HTTPS veya SSH)
- Konteyner imajları (ör. `tarikhamarat/algorycode-rent-service`, `tarikhamarat/gatewayserver`) registry’de mevcut

## Rent servisi (`rent-service`)

- Chart `8090` portunda Pod açar; Service varsayılan olarak **ClusterIP** (sadece küme içi).
- **Veritabanı ve hassas değerler**: `env/prod-values.yaml` içindeki `secret.data` üretimde repoya yazılmamalıdır; Sealed Secrets, External Secrets veya Argo CD ile inject edilen Secret kullanın.
- İhtiyaç halinde `extraEnv` ile `SPRING_DATASOURCE_*` vb. ekleyebilirsiniz.

### Yerel Helm denemesi

```bash
cd rent-service/chart
helm dependency update   # yoksa atlanır
helm install prod-rent . -f ../env/prod-values.yaml --namespace algory-prod --create-namespace
kubectl get svc,pods -n algory-prod
```

Release adı Argo örneğiyle uyumlu olsun diye **`prod-rent`** kullanıldı; tam Service adı:

`prod-rent-algorycode-rent-service` (namespace `algory-prod`).

## Gateway (`gateway-service`)

- Spring Cloud Gateway için `ConfigMap` içinde **`application.yml`** üretilir: `spring.cloud.discovery.enabled: false`.
- Route hedefi `gateway.routes[].service` alanlarıyla tanımlanır:
  - `name` (Kubernetes Service adı)
  - `namespace` (boşsa release namespace)
  - `port`
- Chart bu alanlardan URI üretir: `http://<service>.<namespace>.svc.cluster.local:<port>`.
- İsterseniz `gateway.routes[].uri` vererek bu davranışı override edebilirsiniz.
- Gateway imajınızın (`spring-cloud-starter-gateway`, actuator) bu yapılandırmayı okuyabilmesi gerekir; **Eureka client** bağımlılığını uygulama kodundan kaldırın.

### Gateway uygulama kodunda yapılacaklar

- `spring-cloud-starter-netflix-eureka-client` kaldırılmalı.
- Route tanımı artık **`lb://...`** değil; sabit **`http://`** URI veya bu chart’ın ürettiği dış yapılandırma kullanılmalı.
- Çakışma olmaması için kod içi `RouteLocator` ile aynı path’leri iki kez tanımlamayın (veya yalnızca Helm tarafını kullanın).

### Yerel Helm denemesi

```bash
helm install prod-gateway ./gateway-service/chart \
  -f ./gateway-service/env/prod-values.yaml \
  --namespace algory-prod --create-namespace
```

## Argo CD ile dağıtım (özet)

### 1) Repoyu Argo’ya kaydet

Argo CD UI: **Settings → Repositories → Connect repo**  
veya CLI:

```bash
argocd repo add https://github.com/AlgoryCode/algorycode-helm-charts-prod.git \
  --username ... --password ...   # veya SSH
```

`repoURL` şu an **`https://github.com/AlgoryCode/algorycode-helm-charts-prod.git`** olarak ayarlıdır. **`targetRevision`** (branch/tag) ortamınıza göre güncelleyin.

### 2) Application YAML düzenleme

Her iki dosyada kontrol edin:

| Alan | Açıklama |
|------|----------|
| `spec.source.repoURL` | Bu monoreponun Git URL’si |
| `spec.source.targetRevision` | Örn. `main` |
| `spec.source.path` | `rent-service/chart` veya `gateway-service/chart` |
| `spec.source.helm.releaseName` | `prod-rent` / `prod-gateway` (Helm release adı; Service adının parçası) |
| `spec.source.helm.valueFiles` | Chart dizinine göre `../env/prod-values.yaml` |
| `spec.destination.namespace` | Örnek: `algory-prod` |
| `spec.destination.server` | Varsayılan küme: `https://kubernetes.default.svc` |

### 3) Uygulamayı oluştur

```bash
kubectl apply -n argocd -f rent-service/argo/application.yaml
kubectl apply -n argocd -f gateway-service/argo/application.yaml
```

Argo CD UI’da **Applications** listesinde görünür; **Sync** ile kükeye yayılır.

### 4) Sıra önerisi

1. Önce **rent-service** sync edilsin (Pod ayakta, Service oluşsun).
2. `kubectl get svc -n algory-prod` ile rent Service adını doğrulayın.
3. Gerekirse `gateway-service/env/prod-values.yaml` içindeki **`uri`** değerini güncelleyin; gateway Application’ı tekrar sync edin.

### 5) k3s ve dış erişim

- **Sadece gateway dışarı açılacaksa**: `gateway-service/env/prod-values.yaml` içinde `service.type: LoadBalancer` (varsayılan örnek) veya `ingress.enabled: true` + Traefik `ingressClassName` kullanın.
- Rent doğrudan internete açılmayacaksa **ClusterIP** yeterlidir (gateway üzerinden erişim).
- Rent’i doğrudan dışarı açmak isterseniz `rent-service/env/prod-values.yaml` içinde `service.type: LoadBalancer` yapın.

## algorycode-rent-service uygulama reposu

Kaynak kodda Eureka kaldırıldı (`pom.xml` + `application.yml`). Yeni imajı build/push ettikten sonra prod `image.tag` değerini güncelleyin.

## Sorun giderme

- **502 / bağlantı reddedildi (gateway → rent)**: `gateway.routes[].service.name` ve `gateway.routes[].service.namespace` değerleri rent Service adı ve namespace ile birebir uyumlu mu kontrol edin.
- **ImagePullBackOff**: `imagePullSecrets` veya registry erişimi.
- **Helm valueFiles Argo’da bulunamadı**: `path` ile `valueFiles` göreli yolu chart klasöründen doğru mu (`../env/prod-values.yaml`).
