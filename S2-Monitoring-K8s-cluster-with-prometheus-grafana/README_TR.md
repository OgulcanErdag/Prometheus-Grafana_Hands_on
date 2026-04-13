# Kubernetes Hands-on: Prometheus & Grafana ile Monitoring

> **Amaç:** Bu uygulamalı eğitim, Kubernetes cluster'ını Prometheus ile izlemeyi ve Grafana ile görselleştirme dashboard'ları oluşturmayı kapsar.

---

## Öğrenme Hedefleri

Bu eğitimin sonunda öğrenciler:

- Prometheus'u Kubernetes cluster'ına Helm ile deploy edebilecek
- Prometheus üzerinden cluster metriklerini sorgulayabilecek
- Grafana ile görsel monitoring dashboard'ları oluşturabilecek
- CloudWatch'u Grafana veri kaynağı olarak ekleyip AWS metriklerini izleyebilecek

---

## İçerik

- [Bölüm 1 – Kubernetes Cluster Kurulumu](#bölüm-1--kubernetes-cluster-kurulumu)
- [Bölüm 2 – Prometheus Deploy Edilmesi](#bölüm-2--prometheus-deploy-edilmesi)
- [Bölüm 3 – Grafana ile Dashboard Oluşturma](#bölüm-3--grafana-ile-dashboard-oluşturma)

---

## Bölüm 1 – Kubernetes Cluster Kurulumu

### Cluster Hazırlama

Ubuntu 22.04 üzerinde iki node'lu (1 master, 1 worker) bir Kubernetes cluster'ı hazırlayın. CloudFormation şablonu kullanılarak master node ayağa kalktıktan sonra worker node otomatik olarak cluster'a katılır.

### Cluster Durumunu Doğrulama

```bash
# Cluster'ın erişilebilir olduğunu, API server ve CoreDNS adreslerini gösterir
kubectl cluster-info
```

---

## Bölüm 2 – Prometheus Deploy Edilmesi

### Prometheus Nedir?

**Prometheus**, açık kaynaklı bir sistem izleme ve uyarı (alerting) aracıdır. Kubernetes ortamında yaygın kullanım senaryoları şunlardır:

- Node ve Pod kaynak kullanımı (CPU, bellek, disk)
- Uygulama metriklerinin toplanması (scraping)
- Alert Manager ile uyarı tetikleme

### Helm Nedir?

**Helm**, Kubernetes için bir paket yöneticisidir. Karmaşık Kubernetes uygulamalarını tek komutla kurmak, güncellemek ve kaldırmak için kullanılır. Helm paketlerine **chart** denir.

### Helm Kurulumu

```bash
# Helm'in resmi kurulum betiğini indirir ve çalıştırır
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Kurulumun başarılı olduğunu ve sürümü doğrular
helm version
```

| Komut | Açıklama |
|---|---|
| `curl ... \| bash` | Betiği indirip doğrudan çalıştırır |
| `helm version` | Yüklü Helm sürümünü gösterir |

### Prometheus Helm Repository Ekleme

Helm chart'larını kullanabilmek için ilgili repository'nin önce yerel listeye eklenmesi gerekir.

```bash
# Prometheus community chart repository'sini yerel Helm listesine ekler
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Tüm eklenmiş repository'lerin en güncel chart bilgilerini çeker
helm repo update
```

| Komut | Açıklama |
|---|---|
| `helm repo add` | Yeni bir chart repository ekler |
| `helm repo update` | Yerel cache'i günceller, en yeni chart sürümlerini getirir |

> Daha fazla bilgi: https://artifacthub.io/packages/helm/prometheus-community/prometheus

### Prometheus Kurulumu

```bash
# "prometheus" adıyla Prometheus chart'ını cluster'a yükler
# Format: helm install <release-adı> <repo>/<chart-adı>
helm install prometheus prometheus-community/prometheus
```

### Oluşturulan Kubernetes Nesnelerini İnceleme

```bash
# Prometheus server ve alertmanager Deployment'larını listeler
kubectl get deployments

# Node-exporter gibi her node'da çalışan bileşenleri (DaemonSet) listeler
kubectl get daemonsets

# Tüm çalışan Pod'ları gösterir; prometheus-server, alertmanager, node-exporter vs.
kubectl get pods

# Prometheus bileşenlerine ait Service'leri listeler
kubectl get services
```

| Nesne Türü | Prometheus'ta Karşılık Gelen Bileşen |
|---|---|
| Deployment | prometheus-server, alertmanager, kube-state-metrics |
| DaemonSet | prometheus-node-exporter (her node'da çalışır) |
| Service | Her bileşenin ağ erişim noktası |

### Prometheus'u Dışarıdan Erişilebilir Yapma

Varsayılan olarak `prometheus-server` Service'i `ClusterIP` tipindedir; yani yalnızca cluster içinden erişilebilir. Dışarıdan erişim için `NodePort`'a çevrilmesi gerekir.

```bash
# prometheus-server Service'ini canlı olarak düzenlemeye açar (varsayılan editör: vi)
kubectl edit svc prometheus-server
```

Açılan YAML dosyasında aşağıdaki değişiklikleri yapın:

```yaml
spec:
  type: NodePort          # "ClusterIP" yerine "NodePort" yazın
  ports:
    - port: 80
      targetPort: 9090
      nodePort: 30001     # Dışarıdan erişilecek port (30000-32767 arasında olmalı)
```

Kaydedip çıktıktan sonra tarayıcıdan erişin:

```
http://<MASTER-NODE-PUBLIC-IP>:30001
```

> **Not:** `<MASTER-NODE-PUBLIC-IP>` kısmını EC2 master node'unuzun genel IP adresiyle değiştirin.  
> Örnek: `http://ec2-54-89-159-197.compute-1.amazonaws.com:30001`

---

## Bölüm 3 – Grafana ile Dashboard Oluşturma

### Grafana Nedir?

**Grafana**, çeşitli veri kaynaklarından (Prometheus, CloudWatch, Elasticsearch vb.) gelen metrikleri görsel olarak sunan açık kaynaklı bir analitik ve monitoring platformudur. Özelleştirilebilir dashboard'lar ve panel türleri sunar.

### Grafana Helm Repository Ekleme

```bash
# Grafana resmi chart repository'sini Helm listesine ekler
helm repo add grafana https://grafana.github.io/helm-charts

# Repository listesini günceller
helm repo update
```

> Daha fazla bilgi: https://artifacthub.io/packages/helm/grafana/grafana

### Grafana Kurulumu

```bash
# "grafana" adıyla Grafana chart'ını cluster'a yükler
helm install grafana grafana/grafana
```

### Oluşturulan Nesneleri İnceleme

```bash
# Grafana Deployment'ının durumunu gösterir
kubectl get deployment grafana

# "grafana" içeren Pod'ları filtreler ve listeler
kubectl get pods | grep grafana

# Grafana Service'ini ve port bilgilerini gösterir
kubectl get service grafana
```

### Grafana'yı Dışarıdan Erişilebilir Yapma

```bash
# Grafana Service'ini düzenleme modunda açar
kubectl edit svc grafana
```

Aşağıdaki değişiklikleri yapın:

```yaml
spec:
  type: NodePort          # "ClusterIP" → "NodePort"
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30002     # Grafana için ayrı bir port
```

Tarayıcıdan erişin:

```
http://<MASTER-NODE-PUBLIC-IP>:30002
```

---

### İlk Giriş – Admin Şifresi Alma

Grafana, giriş bilgilerini Kubernetes Secret olarak saklar. Base64 ile şifrelenmiş bu değerleri çözmek gerekir.

```bash
# Grafana Secret'ının tüm içeriğini YAML formatında gösterir
# data alanı altında admin-user ve admin-password base64 ile kodlanmış şekilde bulunur
kubectl get secret grafana -o yaml
```

Çıktıdaki `admin-user` ve `admin-password` değerlerini kopyalayıp decode edin:

```bash
# Base64 decode işlemi: "|" pipe ile "base64 -d" komutuna aktarır
# Sondaki "echo" komutları yeni satır ekler, okunabilirliği artırır
echo "YWRtaW4=" | base64 --decode ; echo
echo "dFozeEV6bGFxUWUyODFoeDhSamlRQmdqM2l5eVFJTmFybURub0tBdQ==" | base64 --decode ; echo
```

| Komut | Açıklama |
|---|---|
| `kubectl get secret grafana -o yaml` | Secret içeriğini YAML olarak getirir |
| `base64 --decode` | Base64 kodlu metni çözer |

> **Not:** Çıktıdaki değerleri Grafana giriş ekranında **Username** ve **Password** olarak kullanın.

---

### Prometheus'u Veri Kaynağı Olarak Ekleme

1. Sol menüden **Connections > Data Sources** sayfasına gidin
2. **Add new data source** butonuna tıklayın
3. Listeden **Prometheus** seçin
4. **URL** alanına şunu yazın:

   ```
   http://prometheus-server:80
   ```

   > Kubernetes içinde Service adını (`prometheus-server`) DNS olarak kullanabilirsiniz.  
   > Port 80 varsayılan HTTP portu olduğundan belirtilmese de çalışır.

5. **Save & Test** butonuna tıklayın — yeşil onay mesajı görünmesi gerekir

---

### Hazır Dashboard İçe Aktarma

Grafana, binlerce topluluk dashboard'unu barındıran bir kütüphaneye sahiptir.

1. Sol menüden **Dashboards** > **New** > **Import** yolunu izleyin
2. https://grafana.com/grafana/dashboards/ adresinden bir dashboard ID seçin  
   Örnek: **`6417`** — Kubernetes cluster genel görünüm dashboard'u
3. ID'yi **Import via grafana.com** alanına girin ve **Load** butonuna tıklayın
4. **Prometheus** veri kaynağını seçin
5. **Import** butonuna tıklayın

---

### CloudWatch'u Veri Kaynağı Olarak Ekleme

AWS CloudWatch entegrasyonu sayesinde EC2, RDS, Lambda gibi AWS servislerinin metriklerini de Grafana üzerinde izleyebilirsiniz.

1. Sol menüden **Connections > Data Sources** > **Add data source** yolunu izleyin
2. Listeden **CloudWatch** seçin
3. Aşağıdaki alanları doldurun:

   | Alan | Değer |
   |---|---|
   | **Auth Provider** | Access & Secret Key |
   | **Access Key ID** | AWS IAM Access Key'iniz |
   | **Secret Access Key** | AWS IAM Secret Key'iniz |
   | **Default Region** | Örn: `us-east-1` |

4. **Save & Test** butonuna tıklayın

5. **Dashboards** sekmesine geçin ve aşağıdakileri içe aktarın:
   - **Amazon EC2**
   - **Amazon CloudWatch Logs**

6. **Home > Amazon EC2** dashboard'una gidin
7. **Network Detail** paneline tıklayarak ağ trafiğini izleyin

---

### Özel Dashboard ve Panel Oluşturma

Kendi metriklerinize göre sıfırdan panel oluşturmak için:

1. Sol menüdeki **+** (Create) simgesine tıklayın > **Dashboard** seçin
2. **Add new panel** butonuna tıklayın
3. Sol taraftan panel görselleştirme türünü seçin: **Gauge**
4. Sorgu parametrelerini aşağıdaki gibi ayarlayın:

   | Parametre | Değer |
   |---|---|
   | **Query Mode** | CloudWatch Metrics |
   | **Region** | default |
   | **Namespace** | AWS/EC2 |
   | **Metric Name** | CPUUtilization |
   | **Statistic** | Average |
   | **Dimensions** | InstanceId = `<Instance ID>` |

5. Sağ üstteki **Apply** butonuna tıklayın
6. Dashboard'u kaydetmek için **Save dashboard** simgesini kullanın (Ctrl+S)

---

### Monitoring Mimarisinin Özeti

```
Kubernetes Cluster
  ├── Node Exporter (DaemonSet)   → Her node'un sistem metriklerini toplar
  ├── kube-state-metrics          → Kubernetes nesne durumlarını (Pod, Deploy vb.) toplar
  └── Prometheus Server           → Metrikleri düzenli aralıklarla "scrape" eder
                                    PromQL ile sorgulanabilir
                                    NodePort: 30001
          │
          │ (veri kaynağı olarak bağlanır)
          ▼
      Grafana                     → Metrikleri görselleştirir, dashboard yönetir
                                    NodePort: 30002
          │
          │ (ek veri kaynağı)
          ▼
      AWS CloudWatch              → EC2, RDS, Lambda gibi AWS servis metrikleri
```

---

## Kaynaklar

- [Prometheus Community Helm Charts – ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus)
- [Grafana Helm Charts – ArtifactHub](https://artifacthub.io/packages/helm/grafana/grafana)
- [Grafana Dashboard Kütüphanesi](https://grafana.com/grafana/dashboards/)
- [Prometheus Resmi Dokümantasyonu](https://prometheus.io/docs/)
- [Grafana Resmi Dokümantasyonu](https://grafana.com/docs/)
