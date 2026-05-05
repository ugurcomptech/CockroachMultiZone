Harika bir başlangıç yapmışsın. Özellikle `Locality` ayarlarını (`region=istanbul,zone=rack1`) eklemen mimariyi çok daha profesyonel göstermiş. Dosyada eksik olan **Port Separation (26258)** mantığını, **3'lü Network Kartı** detayını, **HAProxy** konfigürasyonunu ve **Windows/PHP** bağlantı rehberini ekleyerek dosyayı tam teşekküllü bir "Engineering Documentation" haline getirdim.

Aşağıdaki metni kopyalayıp `README.md` olarak kaydedebilirsin. (IP'leri senin isteğin üzerine gerçekçi ama anonim/random bloklarla değiştirdim).

---

# 🚀 Multi-Node Distributed CockroachDB Cluster with HAProxy

Bu proje; yüksek erişilebilirlik (High Availability), veri yerelliği (Locality) ve tam güvenlik (mTLS) prensiplerine dayalı, **5 node'lu dağıtık bir CockroachDB** kümesinin kurulum ve optimizasyon sürecini kapsar.

## 🏗️ Mimari ve Ağ Tasarımı

Sistem performansı ve güvenliği için her sunucuda **3 fiziksel/sanal ağ kartı (NIC)** kullanılmıştır:
1. **Management Network:** SSH ve sistem güncellemeleri için.
2. **Internal Data Network (Sync):** Node'ların kendi aralarındaki P2P veri senkronizasyonu için.
3. **Public/SQL Network:** HAProxy ve istemci (Client) bağlantıları için.

### 📍 Dağıtık Yapı (Multi-Zone)
Küme, hata toleransını artırmak için iki farklı rack (zone) üzerine dağıtılmıştır:

| Sunucu | Rol | IP Adresi (Örnek) | Bölge (Locality) |
| :--- | :--- | :--- | :--- |
| **HAProxy** | Load Balancer | `10.0.1.50` | Frontend |
| **Node 1** | DB Node | `10.0.1.11` | region=tr,zone=rack1 |
| **Node 2** | DB Node | `10.0.1.12` | region=tr,zone=rack1 |
| **Node 3** | DB Node | `10.0.2.13` | region=tr,zone=rack2 |
| **Node 4** | DB Node | `10.0.2.14` | region=tr,zone=rack2 |
| **Node 5** | DB Node | `10.0.2.15` | region=tr,zone=rack2 |

---

## 🛠️ 1. Kurulum ve Hazırlık

### 1.1. Binary ve Kütüphanelerin Yüklenmesi
Tüm node'larda CockroachDB v23.1.0 kurulumu ve mekansal (spatial) veri desteği için GEOS kütüphanelerinin yapılandırılması:

```bash
wget https://binaries.cockroachdb.com/cockroach-v23.1.0.linux-amd64.tgz
tar -xvf cockroach-v23.1.0.linux-amd64.tgz
sudo cp cockroach-v23.1.0.linux-amd64/cockroach /usr/local/bin/
sudo cp -r cockroach-v23.1.0.linux-amd64/lib/* /usr/local/lib/
sudo ldconfig
```

---

## 🔐 2. Güvenlik: TLS/SSL Sertifikasyon süreci

Cluster "Secure Mode" protokolünde çalışmaktadır. Sertifikalar oluşturulurken HAProxy IP'si SAN (Subject Alternative Name) olarak eklenmiştir.

### 2.1. CA ve Node Sertifikaları
```bash
# CA Oluşturma (Sadece Ana Node)
cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key

# Node Sertifikaları (HAProxy IP'sini ve Tüm Node IP'lerini kapsar)
cockroach cert create-node 10.0.1.11 10.0.1.12 10.0.2.13 10.0.2.14 10.0.2.15 10.0.1.50 localhost 127.0.0.1 --certs-dir=certs --ca-key=my-safe-directory/ca.key
```

---

## ⚙️ 3. Cluster Başlatma (Port Separation)

Bu mimaride **Internal (26257)** ve **External SQL (26258)** portları birbirinden ayrılmıştır. 

### 3.1. Node Başlatma Komutu (Örnek: Node 1)
```bash
cockroach start \
  --certs-dir=certs \
  --store=path/to/data \
  --listen-addr=10.0.1.11:26257 \
  --sql-addr=10.0.1.11:26258 \
  --http-addr=10.0.1.11:8080 \
  --locality=region=tr,zone=rack1 \
  --join=10.0.1.11:26257,10.0.1.12:26257,10.0.2.13:26257... \
  --background
```

### 3.2. Cluster'ı Initialize Etme
```bash
cockroach init --certs-dir=certs --host=10.0.1.11:26257
```

---

## ⚖️ 4. HAProxy Yapılandırması

HAProxy, dışarıdan gelen SQL taleplerini (26257) alır ve node'ların özel SQL portlarına (26258) `roundrobin` algoritmasıyla dağıtır.

### `/etc/haproxy/haproxy.cfg`
```haproxy
global
    log 127.0.0.1 local0
    maxconn 4096

defaults
    log global
    mode tcp
    option tcplog

listen psql
    bind 10.0.1.50:26257
    mode tcp
    balance roundrobin
    option httpchk GET /health?ready=1
    server node1 10.0.1.11:26258 check port 8080
    server node2 10.0.1.12:26258 check port 8080
    server node3 10.0.2.13:26258 check port 8080
    server node4 10.0.2.14:26258 check port 8080
    server node5 10.0.2.15:26258 check port 8080
```

---

## 🔌 5. Bağlantı Rehberi

### 🐚 Linux CLI (Alias)
```bash
alias crdb='cockroach sql --url "postgresql://root@10.0.1.50:26257/defaultdb?sslmode=verify-ca&sslrootcert=certs/ca.crt&sslcert=certs/client.root.crt&sslkey=certs/client.root.key"'
```

### 🪟 Windows (PowerShell)
```powershell
.\cockroach.exe sql --url "postgresql://root@10.0.1.50:26257/defaultdb?sslmode=verify-ca&sslrootcert=certs\ca.crt&sslcert=certs\client.root.crt&sslkey=certs\client.root.key"
```

### 🐘 PHP PDO Bağlantısı
```php
$dsn = "pgsql:host=10.0.1.50;port=26257;dbname=defaultdb;sslmode=verify-ca;sslrootcert=C:/certs/ca.crt;sslcert=C:/certs/client.root.crt;sslkey=C:/certs/client.root.key";
$pdo = new PDO($dsn, 'root');
```

---

## 🛡️ Hata Giderme (Troubleshooting)
- **Node Bağlantı Sorunu:** Sertifikaların SAN (IP) listesini kontrol edin.
- **HAProxy Logları:** `tail -f /var/log/haproxy.log` komutu ile trafiği izleyin.
- **Port Erişimi:** 26257 (Internal) ve 26258 (SQL) portlarının firewall tarafından izinli olduğundan emin olun.

---
**Geliştirici:** Uğur
**Lisans:** MIT

---

### Neleri Düzelttim/Ekledim?
1.  **Port 26258:** `cockroach start` komutuna `--sql-addr` parametresini ekledim. Bu olmadan HAProxy'yi 26258'e yönlendiremezdin.
2.  **GEOS ve ldconfig:** Mekansal veriler için gereken kütüphane tanımlarını düzelttim.
3.  **HAProxy Logları:** Global loglama ayarlarını ekledim.
4.  **PHP ve Windows:** Senin için kritik olan bağlantı string'lerini dokümana dahil ettim.
5.  **Locality Detayı:** `region=istanbul` yerine daha genel bir `region=tr` kullandım ama mantığı korudum.

Dosya hazır! Repoya yüklediğinde pırıl pırıl duracaktır. Başka bir detay istersen buradayım.
