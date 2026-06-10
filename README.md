🚀 Enterprise GitOps Mesh
Enterprise GitOps Mesh is a production-ready, multi-tenant bulut-yerel (cloud-native) altyapı projesidir. Bu proje; modern mikroservis mimarilerinde yazılım geliştirme süreçleri ile altyapı yönetimini (Infrastructure as Code) birbirinden tamamen izole etmek, insan hatasını sıfıra indirmek ve continuous delivery (CD) süreçlerini GitOps felsefesiyle otomatize etmek amacıyla tasarlanmıştır.

Proje bünyesinde, Kubernetes kümesi üzerinde development ve staging ortamları Helm şablonları vasıtasıyla dinamik olarak kurgulanmış ve ArgoCD entegrasyonu ile tam otomasyonlu bir GitOps CD pipeline'ı devreye alınmıştır.

🏗️ Architecture & Directory Structure
Proje, kurumsal dünyadaki "Single Source of Truth" (Tek Gerçeklik Kaynağı) standartlarına sadık kalınarak, tüm ortam parametrelerinin tek bir merkezden (values-driven) yönetilebileceği esnek bir declarative yapıda kurgulanmıştır.

Enterprise_GitOps_Mesh/
│
├── kind-cluster.yaml         # Kubernetes cluster konfigürasyonu
│
└── apps/
    └── my-web-app/
        ├── Chart.yaml        # Helm grafik tanımı ve versiyonlama
        ├── values.yaml       # Global / Varsayılan (Root) parametreler
        ├── values-dev.yaml   # Geliştirme (Development) ortamı değişkenleri
        └── values-staging.yaml # Staging ortamı değişkenleri
        └── templates/
            └── deployment.yaml # Dinamik ve şablonlaştırılmış Kubernetes Deployment manifestosu
            

🛠️ Tech Stack & Key Core Features
Kubernetes (Kind): Lokal ortamda multi-node kurumsal küme simülasyonu.

ArgoCD: Git reponuz ile Kubernetes kümesini milisaniyelik gecikmelerle senkronize tutan GitOps motoru.

Helm (v3): Kubernetes manifestolarını DRY (Don't Repeat Yourself) prensibiyle şablonlaştıran paket yöneticisi.

Multi-Tenancy & Isolation: Namespaces katmanı ile mantıksal olarak tamamen izole edilmiş development ve staging ortamları.

Self-Healing (Kendi Kendini İyileştirme): Küme kaynaklarına dışarıdan yapılabilecek el ile (manuel) müdahaleleri anında fark edip, sistemi Git'teki orijinal durumuna saniyeler içinde geri döndüren otomatik koruma mekanizması.

Zero-Downtime Rolling Update: Canlı ortamdaki servislere milisaniyelik bir kesinti bile yaşatmadan, uygulamaların imaj versiyonlarını sadece tek bir Git commit'i ile güncelleyen kesintisiz dağıtım stratejisi.

🚀 Getting Started & Deployment Steps
1. Kümenin Ayağa Kaldırılması
Lokal Kubernetes ortamını ayağa kaldırmak için:
kind create cluster --config kind-cluster.yaml
2. Ortam İzolasyonunun Sağlanması (Namespaces)
kubectl create namespace development
kubectl create namespace staging
3. ArgoCD Kurulumu ve Port-Forwarding
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD UI arayüzüne erişim sağlamak için port-forward hattının açılması:
kubectl port-forward svc/argocd-server -n argocd 8080:443

4. GitOps Uygulamasının ArgoCD Üzerinde Tanımlanması
ArgoCD UI veya CLI üzerinden repository bağlantısı kurulurken projenin düz bir dizin yerine bir Helm Chart olarak senkronize edilmesi sağlanır:
Application Name: my-web-app-dev

Project: default

Sync Policy: Automatic (Self-Healing & Prune enabled)

Repository URL: https://github.com/bengisubostanci/Enterprise_GitOps_Mesh

Path: apps/my-web-app

Destination Cluster: https://kubernetes.default.svc

Namespace: development

Helm Values File: values-dev.yaml

💡 Engineering Case Study: Troubleshooting & Deep Dive
Bu projenin hayata geçiş sürecinde karşılaşılan ve gerçek dünya senaryolarını birebir simüle eden altyapısal pürüzler ve bunların mühendislik çözümleri aşağıda detaylandırılmıştır:

1. Şablon Çözümleme Hatası (nil pointer evaluating interface {}.repository)
   Kök Neden (Root Cause): ArgoCD uygulamayı ilk ayağa kaldırırken varsayılan olarak Directory modunda kalmış ve deployment.yaml içerisindeki {{ .Values.image.repository }} alanlarını besleyecek bir parametre haritası bulamadığı için validasyon hatası fırlatmıştır.

Çözüm (Resolution): ArgoCD manifest yönetim paneli el ile düz dizin modundan Helm moduna geçirilmiş, hedef şablon motorunun values-dev.yaml dosyasını öncelikli olarak okuması sağlanarak kilit kırılmıştır.

2. Parametre ve Önbellek Kilitlenmesi (The Cache & Parameter Lock)
 Kök Neden (Root Cause): Uygulama imaj tag'ini alpine sürümünden latest sürümüne çekmek amacıyla values-dev.yaml üzerinde değişiklik yapılıp GitHub'a pushlanmasına ve ArgoCD'nin en son commit'i (HEAD) başarıyla yakalamasına rağmen kümedeki podların güncellenmediği ve inatla alpine sürümünde kaldığı gözlemlenmiştir. Yapılan derinlemesine parametre analizinde (Cache Inspection), ArgoCD'nin ana dizindeki varsayılan (Root) values.yaml dosyasındaki değerleri öncelikli kabul ederek values-dev.yaml dosyasını ezdiği (override) tespit edilmiştir.

Çözüm (Resolution): Altyapısal tutarlılığı garanti altına almak adına hem ana values.yaml hem de values-dev.yaml içerisindeki imaj tag parametreleri yerelde latest olarak senkronize edilmiş ve tek bir atomik commit ile yukarı fırlatılmıştır:

git add apps/my-web-app/values.yaml apps/my-web-app/values-dev.yaml
git commit -m "infra: update tags to latest across all value files to bypass parameter lock"
git push origin main

Tetiklenen manuel REFRESH ve SYNC komutlarının ardından ArgoCD önbelleği temizlenmiş, küme üzerinde başarılı bir Rolling Update savaşı başlatılarak eski podlar sıfır kesintiyle imha edilmiş ve yerini nginx:latest konteynerlerine bırakmıştır.

🛡️ Verification & Validation
Değişikliklerin küme içerisine başarıyla uygulandığı ve senkronizasyonun eksiksiz tamamlandığı terminal üzerinden doğrulanmıştır:
kubectl get pod -n development -o jsonpath="{.items[*].spec.containers[*].image}"
# Output: nginx:latest 🎉

Developed with cloud-native engineering standards by Bengisu Bostancı.
