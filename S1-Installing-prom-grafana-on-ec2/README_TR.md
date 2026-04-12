# Hands-on Prometheus & Grafana-01: Temel Kurulum ve Kullanım

> **Amaç:** Bu uygulamalı eğitim, Prometheus ve Grafana'nın EC2 üzerinde kurulumunu, temel yapılandırmasını ve monitoring dashboard'u oluşturmayı kapsar.

---

## Öğrenme Hedefleri

Bu eğitimin sonunda öğrenciler:

- Prometheus ve Grafana'yı Linux üzerine kurabilecek
- Prometheus web arayüzü ile metrik sorgulayabilecek
- Node Exporter ile sistem metriklerini toplayabilecek
- Grafana'da veri kaynağı tanımlayıp özel dashboard oluşturabilecek

---

## İçerik

- [Bölüm 1 – Prometheus Kurulumu ve Yapılandırması](#bölüm-1--prometheus-kurulumu-ve-yapılandırması)
- [Bölüm 2 – Prometheus Web Arayüzü ile İzleme](#bölüm-2--prometheus-web-arayüzü-ile-i̇zleme)
- [Bölüm 3 – Grafana Kurulumu ve Yapılandırması](#bölüm-3--grafana-kurulumu-ve-yapılandırması)
- [Bölüm 4 – Grafana ile Dashboard Oluşturma](#bölüm-4--grafana-ile-dashboard-oluşturma)

---

## Bölüm 1 – Prometheus Kurulumu ve Yapılandırması

### EC2 Instance Hazırlama

Aşağıdaki ayarlarla bir Amazon EC2 instance'ı başlatın:

| Parametre | Değer |
|---|---|
| **AMI** | Amazon Linux 2023 |
| **Instance Type** | t2.micro |
| **Region** | N. Virginia (us-east-1) |
| **VPC** | Default VPC |
| **Security Group** | Port 22, 3000, 9090, 9100 açık olmalı |

> **Port açıklamaları:**
> - `22` → SSH erişimi
> - `9090` → Prometheus web arayüzü
> - `9100` → Node Exporter metrikleri
> - `3000` → Grafana web arayüzü

---

### Prometheus İndirme ve Kurma

```bash
# Prometheus'un son sürümünü GitHub'dan indirir
# wget: URL'den dosya indirme aracı
wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```

> Güncel sürümü kontrol etmek için: https://prometheus.io/download/  
> `*.linux-amd64.tar.gz` uzantılı dosyayı seçin.

```bash
# tar: arşivi açar
# x: extract (çıkart), v: verbose (detaylı çıktı), f: dosya adı, z: gzip sıkıştırması
tar xvfz prometheus-*.tar.gz

# Açılan klasöre gir (sürüm numarası wildcard ile eşleşir)
cd prometheus-*-amd64/
```

---

### Varsayılan Yapılandırma Dosyasını İnceleme

```bash
# prometheus.yml dosyasının içeriğini terminale yazdırır
cat prometheus.yml
```

Varsayılan yapılandırma çıktısı ve her alanın açıklaması:

```yaml
# Genel ayarlar — tüm job'lar için varsayılan değerler
global:
  scrape_interval: 15s      # Prometheus'un hedeflerden metrik topladığı sıklık (varsayılan: 1 dakika)
  evaluation_interval: 15s  # Alarm kurallarının değerlendirilme sıklığı (varsayılan: 1 dakika)
  # scrape_timeout: 10s     # Scrape zaman aşımı (varsayılan: 10 saniye)

# Alertmanager bağlantı yapılandırması
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093   # Alertmanager adresi (etkinleştirilmemiş)

# Kural dosyaları — alarm ve kayıt kuralları tanımlanır
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# Scrape hedefleri — Prometheus hangi adreslerden metrik toplayacak?
scrape_configs:
  - job_name: "prometheus"    # Job adı; metriklere "job=prometheus" etiketi olarak eklenir
    # metrics_path varsayılanı: /metrics
    # scheme varsayılanı: http
    static_configs:
      - targets: ["localhost:9090"]   # Prometheus kendi metriklerini de toplar
```

**Önemli kavramlar:**

| Kavram | Açıklama |
|---|---|
| `scrape_interval` | Prometheus'un `/metrics` endpoint'ini ne sıklıkla sorguladığı |
| `job_name` | Hedef grubuna verilen isim; metriklerde etiket olarak görünür |
| `targets` | Metrik toplanacak adres ve port listesi |

---

### Prometheus'u Başlatma (Test)

```bash
# Prometheus'u belirtilen yapılandırma dosyasıyla başlatır
# Ctrl+C ile durdurulabilir; arka planda çalışmaz
./prometheus --config.file=prometheus.yml
```

---

### Prometheus'u systemd Servisi Olarak Yapılandırma

Prometheus'u sistem yeniden başladığında otomatik çalışacak şekilde `systemd` servisine ekleyin:

```bash
# systemd servis tanım dosyasını oluşturur ve vim ile açar
sudo vim /etc/systemd/system/prometheus.service
```

Dosyaya aşağıdaki içeriği yapıştırın:

```ini
[Unit]
Description=Prometheus          # Servisin açıklama metni

[Service]
User=root                       # Hangi kullanıcı olarak çalışacağı
ExecStart=/home/ec2-user/prometheus-3.5.0.linux-amd64/prometheus \
  --config.file /home/ec2-user/prometheus-3.5.0.linux-amd64/prometheus.yml
                                # Çalıştırılacak komut ve yapılandırma dosyası yolu

[Install]
WantedBy=default.target         # Sistem başlarken hangi hedef altında başlatılacağı
```

```bash
# systemd'ye yeni servis dosyasını tanıtır (yeni dosya eklenince çalıştırılmalı)
sudo systemctl daemon-reload

# Prometheus servisini başlatır
sudo systemctl start prometheus.service

# Servisin çalışıp çalışmadığını kontrol eder
sudo systemctl status prometheus.service
```

Tarayıcıdan erişin:

```
http://<EC2-PUBLIC-IP>:9090
```

---

## Bölüm 2 – Prometheus Web Arayüzü ile İzleme

### Varsayılan Metrikleri İnceleme

```
http://<EC2-PUBLIC-IP>:9090/metrics
```

Bu sayfa Prometheus'un kendi hakkında dışa aktardığı ham metrikleri gösterir. Her satır bir zaman serisi ölçümüdür.

---

### PromQL ile Metrik Sorgulama

Prometheus, **PromQL** (Prometheus Query Language) adı verilen güçlü bir sorgu dilini kullanır. Web arayüzündeki **Expression Console**'a aşağıdaki sorguları girin ve **Execute** butonuna tıklayın.

#### Tüm değerleri getirme

```promql
prometheus_target_interval_length_seconds
```

Bu sorgu; Prometheus'un her scrape döngüsü arasındaki gerçek süreyi ölçen metriği döndürür. Farklı `quantile` ve `interval` etiketlerine sahip birden fazla zaman serisi görürsünüz.

#### Belirli bir yüzdelik dilimi filtreleme

```promql
prometheus_target_interval_length_seconds{quantile="0.99"}
```

Süslü parantez `{}` içindeki ifadeler **etiket filtresi (label matcher)** olarak çalışır. Bu sorgu yalnızca 99. yüzdelik dilimdeki (p99) gecikme değerlerini getirir.

#### Dönen zaman serisi sayısını sayma

```promql
count(prometheus_target_interval_length_seconds)
```

`count()` bir **aggregation (toplama) fonksiyonudur**; eşleşen zaman serisi sayısını tek bir değer olarak döndürür.

> Daha fazla PromQL öğrenmek için: https://prometheus.io/docs/prometheus/latest/querying/basics/

---

### Node Exporter ile Linux Sistem Metriklerini İzleme

#### Node Exporter Nedir?

**Node Exporter**, Linux sistem metriklerini (CPU, bellek, disk, ağ) Prometheus'un anlayacağı formata dönüştüren tek bir ikili dosyadan oluşan bir araçtır. Her izlenecek sunucuya kurulması gerekir.

#### Kurulum ve Çalıştırma

```bash
# Ana dizine dön
cd ~

# Node Exporter'ın son sürümünü indir
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz

# Arşivi aç
tar xvfz node_exporter-*.*-amd64.tar.gz

# Klasöre gir
cd node_exporter-*.*-amd64

# Node Exporter'ı başlat (varsayılan port: 9100)
./node_exporter
```

#### Metrikleri Doğrulama

```bash
# curl ile 9100 portundaki /metrics endpoint'ini sorgular
# Node Exporter çalışıyorsa onlarca sistem metriği listelenecektir
curl http://localhost:9100/metrics
```

Veya tarayıcıdan:

```
http://<EC2-PUBLIC-IP>:9100/metrics
```

---

### prometheus.yml'e Node Exporter Hedefini Ekleme

```bash
# Yapılandırma dosyasını düzenle
sudo vim /home/ec2-user/prometheus-3.5.0.linux-amd64/prometheus.yml
```

`scrape_configs` bölümüne yeni bir job ekleyin:

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Node Exporter için yeni scrape hedefi
  - job_name: node                        # Job adı; metriklerde "job=node" etiketi olarak görünür
    static_configs:
      - targets: ['localhost:9100']        # Node Exporter'ın çalıştığı adres ve port
```

```bash
# Yapılandırma değişikliğini uygulamak için Prometheus'u yeniden başlat
sudo systemctl restart prometheus
```

---

### Temel Node Metrikleri

Prometheus web arayüzünde aşağıdaki PromQL sorgularını deneyin:

| Metrik / Sorgu | Açıklama |
|---|---|
| `rate(node_cpu_seconds_total{mode="system"}[1m])` | Son 1 dakikada sistem modunda harcanan ortalama CPU süresi (saniye/saniye) |
| `node_filesystem_avail_bytes` | Root olmayan kullanıcılar için kullanılabilir dosya sistemi alanı (byte) |
| `rate(node_network_receive_bytes_total[1m])` | Son 1 dakikada alınan ortalama ağ trafiği (byte/saniye) |

> `rate()` fonksiyonu: Sayaç (counter) tipindeki metriklerin belirtilen zaman dilimindeki **ortalama artış hızını** hesaplar.

---

## Bölüm 3 – Grafana Kurulumu ve Yapılandırması

### Grafana Nedir?

**Grafana**, Prometheus, CloudWatch, Elasticsearch gibi çeşitli veri kaynaklarına bağlanabilen açık kaynaklı bir görselleştirme ve monitoring platformudur. Özelleştirilebilir panel ve dashboard'larıyla metrikleri anlamlı grafiklere dönüştürür.

### Grafana Kurulumu

```bash
# RPM paketini doğrudan URL'den yükler (Amazon Linux 2023 / RHEL uyumlu)
sudo yum install -y https://dl.grafana.com/grafana-enterprise/release/12.3.0/grafana-enterprise_12.3.0_19497075765_linux_amd64.rpm

# Grafana servisini başlatır
sudo systemctl start grafana-server.service

# Sistem yeniden başladığında otomatik çalışması için etkinleştirir
sudo systemctl enable grafana-server.service
```

> Güncel sürüm için: https://grafana.com/grafana/download  
> `Red Hat, CentOS, RHEL, and Fedora (64 Bit)` bölümünden komutu kopyalayın.

Tarayıcıdan erişin:

```
http://<EC2-PUBLIC-IP>:3000
```

İlk girişte kullanıcı adı ve şifre: `admin` / `admin`  
Giriş sonrasında yeni şifre belirlemeniz istenecektir.

---

### Prometheus'u Veri Kaynağı Olarak Ekleme

1. Sol menüden **Connections > Data Sources** yolunu izleyin
2. **Add new data source** butonuna tıklayın
3. Listeden **Prometheus** seçin
4. Aşağıdaki ayarı yapın:

   | Alan | Değer |
   |---|---|
   | **Prometheus server URL** | `http://localhost:9090` |

5. **Save & Test** butonuna tıklayın — yeşil onay mesajı görünmesi gerekir

---

## Bölüm 4 – Grafana ile Dashboard Oluşturma

### Prometheus Grafiği Oluşturma

1. Sol menüden **Dashboards > New > New Dashboard** yolunu izleyin
2. **Add new panel** butonuna tıklayın
3. **Data source** olarak `Prometheus` seçin
4. **Query** alanına istediğiniz PromQL sorgusunu yazın  
   Örnek: `rate(node_cpu_seconds_total{mode="system"}[1m])`
5. **Legend format** alanına etiket bazlı özel isim verin:  
   Örnek: `{{mode}} - {{instance}}` → zaman serisi adını mode ve instance etiketiyle formatlar
6. Panel ayarlarını (başlık, eksen, renk) düzenleyin ve **Apply** butonuna tıklayın

---

### Hazır Dashboard İçe Aktarma

Grafana.com'da topluluk tarafından paylaşılan binlerce hazır dashboard mevcuttur.

```
https://grafana.com/grafana/dashboards/
```

1. Arama kutusuna `node exporter` yazın
2. Beğendiğiniz bir dashboard'u seçin ve **Copy ID to Clipboard** butonuna tıklayın
3. Grafana web arayüzünde:
   - Sol menüden **Dashboards > New > Import** yolunu izleyin
   - **Import via grafana.com** alanına kopyaladığınız ID'yi yapıştırın (örn. `12486`)
   - **Load** butonuna tıklayın
   - Veri kaynağı olarak `Prometheus` seçin
   - **Import** butonuna tıklayın

Dashboard yüklendikten sonra CPU, bellek, disk ve ağ metriklerini görsel olarak izleyebilirsiniz.

---

### Genel Mimari Özeti

```
EC2 Instance
  ├── Prometheus         (port 9090)
  │     └── scrape eder
  │           ├── kendisini       → localhost:9090/metrics
  │           └── Node Exporter  → localhost:9100/metrics
  │
  ├── Node Exporter      (port 9100)
  │     └── Linux sistem metriklerini dışa aktarır
  │         (CPU, bellek, disk, ağ)
  │
  └── Grafana            (port 3000)
        └── Prometheus'a bağlanır
            └── Dashboard'larda metrikleri görselleştirir
```

---

## Kaynaklar

- [Prometheus Getting Started – Resmi Dokümantasyon](https://prometheus.io/docs/prometheus/latest/getting_started/)
- [Node Exporter Rehberi – Prometheus](https://prometheus.io/docs/guides/node-exporter/)
- [Grafana ile Prometheus Görselleştirme](https://prometheus.io/docs/visualization/grafana/)
- [Grafana Dashboard Kütüphanesi](https://grafana.com/grafana/dashboards/)
