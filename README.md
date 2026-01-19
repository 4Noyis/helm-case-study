# C4N Vagrant & Helm Case Study

## Modül 1: Mevcut Paketleri Keşfetme ve Kurulum
**Görev:** ArtifactHub üzerinden resmi bir `nginx` veya `redis` chart'ı bulup Minikube/k3s üzerine kurun

**Ödev Detayı:**
  1. Bitnami reposunu Helm'e ekleyin. 
  2.  `helm install` ile bir uygulama ayağa kaldırın. 
  3.  `helm list` ile durumu kontrol edin.

- **Soru:**  `helm status` komutu çıktısındaki "Notes" bölümü ne işe yarar?

_Cevap:_
```
  Charts kurulumundan sonra gösterilen kullanım kılavuzu: uygulamaya nasıl erişileceği, DNS adresi, dışarıdan erişim komutları, yapılması gereken onraki adımlar
```

_Notlarım:_
```
Vagrant: Vm'lerin lifecycle'ını yönetmek için kullanılan bir CLI tool. Tek bir Vagrantfile dosyası ile bir veya birden fazla sanal makinenin tüm ayarlarını ve kurulum detayları tanımlanıp kullanılabilir.

Helm: Kubernetes paket yöneticisi. like apt/yum for Linux

Helm Chart: Kubernetes için kurulum paketi. Tıpki mac'deki .dmg, ubuntu .deb, node'da npm paketi gibi. k8s uygulamasının zip dosyası + akıllı kurum scripti gibi de düşünülebilir.

Bitnami: Popüler uygulamaları (Nginx, Redis, PostgreSQL..) Helm chart olarak paketler; test edilmiş, güvenli prod-ready konfigleri sunar. Kubernetes için Appstore.

helm repo add bitnami https://charts.bitnami.com/bitnami -> bitnami reposunu ekle
helm repo update                                         -> Repo listesini güncelle
helm install my-nginx bitnami/nginx                      -> nginx kurulumu
helm list                                                -> kurulu releaseleri listele
helm status my-nginx                                     -> detaylı durumu gör

Bu modülde manuel olarak yaptığımız helm komutlarını helmfile kullanarak chart otomasyonu da yapabiliriz.
```

## Modül 2: Yapılandırma ve "Values.yaml"
**Görev:** Kurduğunuz uygulamanın özelliklerini (örneğin replica sayısını veya servis tipini) dışarıdan müdahale ederek değiştirin.

**Ödev Detayı:**
1.  `values.yaml`  dosyasını export edin. 
2. Replica sayısını 1'den 3'e çıkarın. 
3. Uygulamanın 1 cpu, 2 gb memory ile çalışmasını sağlayın. 
4.  `helm upgrade`  komutuyla sistemi güncelleyin.

_Notlarım:_
```
helm show values bitnami/nginx > my-values.yaml       -> values.yaml dosyasını export

my-values.yaml dosyasını düzenle:
replicaCount: 3
resourcesPreset: "none"
resources:
  limits:
    cpu: "1"
    memory: "2Gi"
  requests:
    cpu: "500m"
    memory: "1Gi"
    
helm upgrade my-nginx bitnami/nginx -f my-values.yaml                      -> Helm'i güncelle
kubectl get pods                                                           -> Pod sayısı kontrolü (3)
kubectl describe deployment my-nginx | grep -A 10 "Replicas\|cpu\|memory"  -> Detaylı görünüm
```

## Modül 3: Kendi Chart'ını Oluşturma (Deep Dive)
İşin mutfağına girme zamanı. Bir uygulamanın nasıl paketlendiğini görmeleri gerekiyor.  

**Görev:** Çok basit bir "Hello World" (Flask veya Node.js olabilir) uygulamasını Helm Chart haline getirin.

**Ödev Detayı:**
1.  `helm create my-app`  komutuyla iskeleti oluşturun. 
2. Gereksiz dosyaları temizleyin. 
3.  `templates/`  klasörü altındaki YAML dosyalarında **Go Templating** kullanarak (örneğin: `{{ .Values.image.repository }}`) dinamik alanlar oluşturun. 
4.  `helm lint`  komutuyla yazdıkları chart'ın standartlara uygunluğunu test etsinler.

_Notlarım:_
```
main.go                                                   -> Basit bir GO App oluşturdum
Dockerfile                                                -> Docker image oluştur
helm create my-app                                        -> Chart template'i oluştur
rm -rf templates/tests templates/hpa.yaml ingress.yaml    -> Basit tutmak için bazı dosyaları sil
      serviceaccount.yaml NOTES.txt

values.yaml                                               -> Merkezi ayarlar, tek yerden tüm ayarları yönetebiliyoruz
helm upgrade -f prod-values.yaml                          -> değişiklik yapmak

deployment.yaml (Go Templating)
Go Template Variables:
  {{ .Release.Name }} | helm install hello ...   | hello
  {{ .Chart.Name }}   | Chart.yaml               | "my-app"
  {{ .Values.xxx }}   | values.yaml              | tanımlanmış değer
Özel Fonksiyonlar:
  toYaml     | Değeri YAML formatına çevirir
  nindent 12 | 12 boşluk girinti ekler
  quote      | Değeri tırnak içine alır
  default    | Varsayılan değeri verir
  
helm lint my-app                                          -> (YAML syntax hataları, Gerekli alanlar, 
                                                            Template'ler render edilebiliyor mu,Best practice uyarıları)
```
`Akış Şeması`:
```
values.yaml                    templates/deployment.yaml
┌─────────────────┐           ┌─────────────────────────────┐
│ replicaCount: 2 │ ────────► │ replicas: {{ .Values...}}   │
│ image:          │           │ image: {{ .Values...}}      │
│   repository:   │           └─────────────────────────────┘
│     hello-app   │                        │
│   tag: v1       │                        ▼
└─────────────────┘           ┌─────────────────────────────┐
                              │   helm template / install    │
                              └─────────────────────────────┘
                                           │
                                           ▼
                              ┌─────────────────────────────┐
                              │  Final Kubernetes YAML      │
                              │  replicas: 2                │
                              │  image: hello-app:v1        │
                              └─────────────────────────────┘
```

## Modül 4: Release Yönetimi ve Geri Dönüş (Rollback)**
**Görev:** 
Uygulamanın imaj versiyonunu bilerek "yanlış/bozuk" bir versiyonla güncelleyin ve sistemin çöküşünü izleyip geri dönün.

**Ödev Detayı:** 
1. Hatalı bir imaj etiketiyle  `upgrade`  yapın. 
2.  `kubectl get pods`  ile hatayı (ImagePullBackOff) görün. 
3.  `helm history`  ile geçmişi listeleyin. 
4.  `helm rollback`  komutuyla çalışan son versiyona saniyeler içinde geri dönün.

_Notlarım:_
```
helm upgrade hello my-app --set image.tag=v999-bozuk   -> Hatalı image ile upgrade

# Hata
Kubectl get pods -> ErrImageNeverPull

Kubernetes rolling update:
- yeni image versiyonu ile güncelleme yapıldığığında eski pod'lar çalışmaya devam eder
bu sayede yeni pod başarısız olsa bile uygulama erişilebilir kalır ama deployment tamamlanmaz

helm history hello                                     -> geçmişi listele
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete

helm rollback hello 1                                  -> rollback ile çalışan versiyona dönüş

kubectl get pods
curl $(minikube service hello-service --url)
```

## Modül 5: Final Projesi (Full Stack)  
**Proje:** 
Bir  **WordPress**  sitesini, veritabanı (MariaDB/MySQL) ile birlikte Helm kullanarak kurun.  

_Notlarım:_
```
wp-values.yaml                                      -> Bitnami WordPress kurulumu (MariaDB dahil)
wordpressUsername: admin
wordpressPassword: admin123
wordpressBlogName: "Helm Odev Blog"

service:
  type: NodePort
  nodePorts:
    http: 30080

mariadb:
  auth:
    rootPassword: rootpass123
    database: wordpress
    username: wordpress
    password: wppass123


helm install wp bitnami/wordpress -f wp-values.yaml -> bitnami wordpress indir

kubectl get pods -> kontrol
kubectl get svc  -> kontrol

# Secret Kontrol (base64 şifrelemesi)
kubectl get secrets
kubectl get secret wp-wordpress -o yaml
kubectl get secret wp-wordpress -o jsonpath='{.data.wordpress-password}' | base64 -d

curl http://192:168:49.2:30080 -> Erişim
kubectl port-forward svc/wp-wordpress 8080:80 --address 0.0.0.0     -> Tarayıcıdan giremedim port forwarding yaptım
http://192.168.56.20:8080/ &&  http://192.168.56.20:8080/wp-admin/
```
### **Ekstra Zorluk:** Veritabanı şifresini düz metin olarak değil, bir  `Secret`  objesi üzerinden Helm ile deploy edin. (secret.yaml).

### **2x Ekstra Zorluk:** Worldpress'e DNS tanımlayın ve HTTPS (SSL sertifikası ile) çalışmasını sağlayın.

_Notlarım:_
```
# Ekstra Zorluk 1
Bitnami chart'ı otomatik olarak şifreleri Secret içinde saklıyor, düz metin değil.
kubectl get secret wp-wordpress -o yaml

# Ekstra Zorluk 2

minikube addons enable ingress  -> Ingress Controller Kurulumu

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml -> SSL için cert manager

# cluster-issuer (self-signed issuer)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

kubectl apply -f cluster-issuer.yaml 

# /etc/hosts dosyasının içine 
192.168.56.20 wordpress.local

kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8443:443 --address 0.0.0.0 & -> port forward

https://wordpress.local:8443
```
