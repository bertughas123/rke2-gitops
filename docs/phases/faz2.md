# Faz 2 - Argo CD Bootstrap ve Kustomize Karari

> Durum: Uygulama oncesi ogrenme ve review dokumani.
>
> Bu dosya Faz 2'yi uygulamaz. `kubectl` calistirilmaz, Kubernetes'e kaynak
> uygulanmaz, Argo CD kurulmaz, commit veya push yapilmaz.
> Amac, Faz 2 mimarisini ve yazilacak dosyalari netlestirmektir.

## 1. Faz Amaci

Faz 2'nin amaci mevcut tek node RKE2 cluster uzerine Argo CD bootstrap yapisini
hazirlamaktir.

Faz sonunda hedef davranis:

```text
bootstrap/root-app.yaml
  |
  v
Argo CD Application
  |
  v
spec.source.path: argocd
  |
  v
argocd/kustomization.yaml algilanir
  |
  v
yalnizca kustomization.yaml resources listesi uygulanir
```

Bu fazda demo uygulama, Jenkins, OpenBao, External Secrets Operator veya GHCR
akisi yoktur.

## 2. Mimari Karar

Karar:

```text
argocd/ klasoru Kustomize ile yonetilecek.
```

Neden:

- Argo CD, bir klasorde `kustomization.yaml` gorurse o klasoru Kustomize
  uygulamasi olarak isler;
- hangi manifestlerin uygulanacagi acik bir `resources` listesiyle kontrol
  edilir;
- klasordeki her YAML dosyasini otomatik gezmek yerine bilincli secim yapilir;
- repository Secret ornegi gibi uygulanmamasi gereken dosyalar Argo CD kapsam
  disinda tutulur.

Bu kararla `spec.source.directory.recurse` kullanilmaz. Argo CD alt klasorleri
kendiliginden gezmeyecek; sadece `argocd/kustomization.yaml` icindeki
`resources` listesinde yazan manifestleri yonetecektir.

## 3. Hedef Repo Yapisi

Faz 2 icin hedef yapi:

```text
rke2-gitops/
├── bootstrap/
│   └── root-app.yaml
├── argocd/
│   ├── namespace.yaml
│   ├── kustomization.yaml
│   └── projects/
│       ├── dev.yaml
│       └── platform.yaml
└── docs/
    ├── phase-2-report.md
    ├── troubleshooting.md
    └── examples/
        └── repository-secret.example.yaml
```

## 4. Kustomize Kapsami

`argocd/kustomization.yaml` dosyasi yalnizca gercek Kubernetes manifestlerini
listeleyecek:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - projects/dev.yaml
  - projects/platform.yaml
```

Neden bu liste sinirli:

- `namespace.yaml` gercek cluster kaynagidir;
- `projects/dev.yaml` gercek Argo CD AppProject kaynagidir;
- `projects/platform.yaml` gercek Argo CD AppProject kaynagidir;
- dokuman, ornek secret veya lokal notlar Argo CD tarafindan uygulanmamalidir.

## 5. Yazilacak Dosyalar

### 5.1 `argocd/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
```

Neden:

- Argo CD component'leri ve AppProject kaynaklari `argocd` namespace'inde
  duracak;
- namespace manifesti Kustomize resources listesinde yer alacak.

### 5.2 `argocd/projects/dev.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev
  namespace: argocd
spec:
  description: Development applications
  sourceRepos:
    - "*"
  destinations:
    - namespace: dev
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Neden:

- `dev` projesi ileride demo ve is uygulamalarini tasiyacak;
- hedef namespace simdilik `dev` ile sinirli tutulur;
- cluster geneli kaynak yetkisi verilmez.

### 5.3 `argocd/projects/platform.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Platform services
  sourceRepos:
    - "*"
  destinations:
    - namespace: "*"
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Neden:

- platform bilesenleri CRD, ClusterRole veya cluster seviyesinde kaynaklar
  isteyebilir;
- bu genis yetki uygulama projesine degil sadece `platform` projesine verilir.

### 5.4 `bootstrap/root-app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<GITHUB_USERNAME>/rke2-gitops.git
    targetRevision: main
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Neden:

- root Application, Argo CD'ye repo icindeki `argocd` klasorunu kaynak olarak
  verir;
- Argo CD bu klasorde `kustomization.yaml` gordugu icin Kustomize kullanir;
- `directory.recurse` kullanilmaz;
- `prune: true`, Git/Kustomize listesinden cikan kaynaklarin cluster'dan
  temizlenmesini saglar;
- `selfHeal: true`, cluster'da elle bozulan kaynagi Git'teki hale dondurur.

Onemli:

- `<GITHUB_USERNAME>` placeholder olarak kalir;
- gercek private key, token, parola veya kubeconfig bu dosyada bulunmaz.

### 5.5 `docs/examples/repository-secret.example.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-rke2-gitops
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: git@github.com:<GITHUB_USERNAME>/rke2-gitops.git
  sshPrivateKey: |
    <ARGOCD_READ_ONLY_DEPLOY_KEY_PRIVATE_PART>
```

Neden `argocd/` altinda degil:

- bu dosya gercek Kubernetes Secret ornegidir ama uygulanacak gercek kaynak
  degildir;
- icinde sadece placeholder vardir;
- Argo CD'nin Kustomize kapsaminda olmamalidir;
- gercek repository Secret lokalden veya ileride OpenBao/ESO ile yonetilecektir.

## 6. Uygulama Sirasinda Calisacak Komutlar

Bu komutlar bu dokuman guncellenirken calistirilmayacak. Faz 2 uygulamasina
gecildiginde tek tek kullanilacak.

### 6.1 Repo dizinine gel

```bash
cd /Users/bertughas/Documents/rke2-gitops/rke2-gitops
```

Neden:

- manifestler bu repo icinde durur;
- Git durumunu dogru repoda kontrol ederiz.

### 6.2 Cluster erisimini dogrula

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl get nodes -o wide
```

Neden:

- Argo CD kurulmadan once RKE2 cluster'a Mac'ten erisebildigimizi goruruz;
- node `Ready` degilse kurulum yapmayiz.

### 6.3 Argo CD install manifest secimini yap

Faz 2 uygulamasindan once Argo CD install manifest'i sabit surumle secilecek.
`stable` URL'si bu asamada bilincli olarak kullanilmiyor.

Yer tutucu:

```text
<ARGO_CD_INSTALL_MANIFEST_URL_PINNED_VERSION>
```

Neden:

- `stable` zamanla degisebilir;
- sabit surum, ayni dokumani tekrar uyguladigimizda daha tahmin edilebilir
  sonuc verir.

### 6.4 Namespace'i uygula

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl apply -f argocd/namespace.yaml
```

Neden:

- Argo CD component'leri `argocd` namespace'ine kurulacak;
- namespace yoksa kurulum manifest'i basarisiz olabilir.

### 6.5 Argo CD'yi sabit surum manifestiyle kur

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl apply -n argocd \
  -f <ARGO_CD_INSTALL_MANIFEST_URL_PINNED_VERSION>
```

Neden:

- Argo CD ilk kez kurulurken henuz kendisini yonetecek Argo CD yoktur;
- bu tek seferlik bootstrap adimidir;
- URL bir sonraki adimda sabit surum olarak belirlenecek.

### 6.6 Argo CD pod'larini bekle

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl -n argocd get pods
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl -n argocd wait \
  --for=condition=Ready pods --all --timeout=300s
```

Neden:

- controller ve repo-server hazir olmadan root Application davranisini saglikli
  kontrol edemeyiz.

### 6.7 GitHub read-only Deploy Key hazirla

Bu adimda private key uretilecek ama Git'e yazilmayacak.

```bash
ssh-keygen -t ed25519 -C "argocd-readonly-rke2-gitops" \
  -f ~/.ssh/argocd_rke2_gitops_readonly
```

Neden:

- Argo CD icin kisinin kendi SSH key'i degil, repo seviyesinde read-only Deploy
  Key kullanilir;
- Jenkins write credential'i bundan ayri kalir.

GitHub'da manuel:

1. `bertughas123/rke2-gitops` repository sayfasina git.
2. `Settings` -> `Deploy keys` -> `Add deploy key`.
3. Title: `argocd-readonly-rke2-gitops`.
4. Public key olarak `~/.ssh/argocd_rke2_gitops_readonly.pub` icerigini ekle.
5. `Allow write access` kapali kalmali.

### 6.8 Repository Secret'i lokal dosyadan uygula

Ornek dosya:

```text
docs/examples/repository-secret.example.yaml
```

Gercek uygulama dosyasi repo disinda tutulacak. Ornek lokal yol:

```text
~/Desktop/argocd-rke2-gitops-repo-secret.yaml
```

Uygulama komutu:

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl apply \
  -f ~/Desktop/argocd-rke2-gitops-repo-secret.yaml
```

Neden:

- Argo CD private repo okuyacaksa repository credential gerekir;
- gercek private key Git'e yazilmaz;
- `docs/examples` altindaki dosya sadece placeholder'li egitim ornegidir.

### 6.9 Root Application'i uygula

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl apply -f bootstrap/root-app.yaml
```

Neden:

- Argo CD bundan sonra `argocd` path'ini takip eder;
- `argocd/kustomization.yaml` oldugu icin Kustomize motoru calisir;
- sadece `resources` listesinde yazan manifestler yonetilir.

Kontrol:

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl -n argocd get applications
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl -n argocd get appprojects
```

## 7. Basari Kapisi

Faz 2 basarili sayilmak icin:

- `argocd` namespace vardir;
- Argo CD pod'lari calisir;
- GitHub Deploy Key read-only olarak eklenmistir;
- repository Secret cluster'da vardir ama Git'te gercek secret yoktur;
- `bootstrap-root` Application `argocd` path'ini izler;
- Argo CD, `argocd/kustomization.yaml` uzerinden Kustomize kullanir;
- `dev` ve `platform` AppProject kaynaklari vardir;
- `directory.recurse` kullanilmaz.

Secret kontrolu:

```bash
rg -n "BEGIN (RSA|OPENSSH|EC|PRIVATE)|password|passwd|token|secret|kubeconfig|ghp_|github_pat_|AKIA|xoxb-" .
```

Her eslesme placeholder, dokuman uyarisi veya ornek olmalidir.

## 8. Durma Noktalari

Su durumlarda devam edilmeyecek:

- node `Ready` degilse;
- Argo CD install manifest'i sabit surumle secilmemisse;
- Argo CD pod'lari `CrashLoopBackOff` durumundaysa;
- GitHub Deploy Key write yetkisiyle eklenmisse;
- private key yanlislikla repo klasorune kaydedildiyse;
- `git status --short` secret, kubeconfig veya private key dosyasi gosteriyorsa;
- root Application repo'yu okuyamiyorsa.

## 9. Faz 2 Sonunda Yazilacak Rapor

Faz 2 bitince `docs/phase-2-report.md` dosyasina su bilgiler yazilacak:

- Argo CD kurulum tarihi;
- kullanilan sabit Argo CD manifest surumu;
- Argo CD namespace durumu;
- pod durumlari;
- Deploy Key read-only dogrulamasi;
- AppProject listesi;
- root Application sync/health durumu;
- Kustomize kaynak listesi;
- karsilasilan hata ve cozum notlari;
- secret commitlenmedigi kontrolu.
