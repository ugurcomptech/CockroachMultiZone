# CockroachDB 5-Node Secure Cluster (Multi-Zone)

Bu repository, **5 node’luk secure (TLS) CockroachDB cluster**’ının kurulum, konfigürasyon ve yönetim sürecini kapsamaktadır. İki farklı subnet (rack) üzerinde dağıtılmış, HAProxy ile load balanced bir test/production ortamı hazırlanmıştır.

## Proje Özeti

- **Versiyon**: CockroachDB v23.1.0
- **Node Sayısı**: 5
- **Güvenlik**: Tam Secure Mode (TLS + Client Certificate)
- **Mimari**: Multi-Zone (2 Rack / Zone)
  - Rack1: 2 Node (`192.168.137.x`)
  - Rack2: 3 Node (`192.168.244.x`)
- **Load Balancer**: HAProxy (ön tarafa koyuldu)
- **Amaç**: Dağıtık veritabanı mimarisi, high availability ve locality testleri

## Mimari Diyagram
HAProxy (Load Balancer)
↓ (26257)
├── Rack1 (zone=rack1)
│   ├── Node 1 → 192.168.137.128
│   └── Node 2 → 192.168.137.129
└── Rack2 (zone=rack2)
├── Node 3 → 192.168.244.129
├── Node 4 → 192.168.244.132
└── Node 5 → 192.168.244.133

## 1. CockroachDB İndirme ve Kurulum

# Tüm node'larda çalıştır
```
wget https://binaries.cockroachdb.com/cockroach-v23.1.0.linux-amd64.tgz
```
# Arşivi aç
```
tar -xvf cockroach-v23.1.0.linux-amd64.tgz
```
# Binary'yi PATH'e taşı
```
sudo cp cockroach-v23.1.0.linux-amd64/cockroach /usr/local/bin/
```
# GEOS kütüphanelerini kopyala (spatial fonksiyonlar için)
```
sudo cp -r cockroach-v23.1.0.linux-amd64/lib/* /usr/local/lib/
sudo ldconfig
```


# Adım 2: Klasör Oluşturma
```
mkdir -p ~/cockroach-data
```

## 2. Sertifika Oluşturma (Secure Mode)
# Tüm node'larda klasör oluştur
```
mkdir -p /home/ugur/certs
mkdir -p /home/ugur/my-safe-directory
chmod 700 /home/ugur/my-safe-directory
```
# CA Sertifikası (Sadece Node1'de)
```
cockroach cert create-ca \
  --certs-dir=/home/ugur/certs \
  --ca-key=/home/ugur/my-safe-directory/ca.key
```
# Node Sertifikası (Tüm IP'ler)
```
cockroach cert create-node \
  192.168.137.128 192.168.137.129 \
  192.168.244.129 192.168.244.132 192.168.244.133 \
  localhost 127.0.0.1 \
  --certs-dir=/home/ugur/certs \
  --ca-key=/home/ugur/my-safe-directory/ca.key
```
# Root Client Sertifikası
```
cockroach cert create-client root \
  --certs-dir=/home/ugur/certs \
  --ca-key=/home/ugur/my-safe-directory/ca.key
```

## 3. Node Başlatma Komutları (Locality Tanımlı)
