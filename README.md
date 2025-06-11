# Redis Sentinel 3-VM Docker Kurulum Rehberi

Bu rehber, 3 farklı VM üzerinde Docker Compose kullanarak HAProxy destekli Redis Sentinel kümesini kurmak isteyen kullanıcılar için hazırlanmıştır.

---

## 📌 Kullanılan Sunucular

- **VM1**: Master Redis + HAProxy
- **VM2**: Slave Redis + Sentinel
- **VM3**: Slave Redis + Sentinel

---

## 🔧 1. Sunucu Hazırlığı (Her 3 VM’de)

### Sistem Güncellemesi
```bash
sudo apt update && sudo apt upgrade -y
```

### Gerekli Paketlerin Kurulumu
```bash
sudo apt install -y curl wget net-tools ufw
```

### Firewall Ayarları
```bash
sudo ufw allow 6379/tcp
sudo ufw allow 26379/tcp
sudo ufw allow 22/tcp
sudo ufw --force enable
```

### Kernel Parametrelerini Ayarlama
```bash
echo 'vm.overcommit_memory = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.core.somaxconn = 65535' | sudo tee -a /etc/sysctl.conf
echo 'fs.file-max = 100000' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Transparent Huge Pages’i Devre Dışı Bırakma
```bash
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' | sudo tee -a /etc/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' | sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local
```

---

## 📁 2. Dizin Yapısı ve Dosyaların Yerleştirilmesi

Her VM’de:
```bash
mkdir -p ~/redis-sentinel
cd ~/redis-sentinel
mkdir -p redis-data
```

Gerekli `docker-compose.yml`, `redis.conf`, `sentinel.conf`, ve gerekiyorsa `haproxy.cfg` dosyalarını bu dizine kopyalayın. IP adreslerini kendi ortamınıza göre düzenlemeyi unutmayın.

```bash
chmod 644 redis.conf sentinel.conf
chmod 755 redis-data
```

---

## 🚀 3. Redis & Sentinel Başlatma

### VM1 (Master Redis + HAProxy)
```bash
cd ~/redis-sentinel
docker-compose up -d
```

### VM2 & VM3 (Slave Redis + Sentinel)
```bash
cd ~/redis-sentinel
docker-compose up -d
```

---

## ✅ 4. Durum Kontrolü

```bash
docker ps
docker exec redis-master redis-cli info replication
docker exec redis-sentinel redis-cli -p 26379 sentinel masters
```

HAProxy Arayüzü:  
`http://<VM1-IP>:8080`

---

## 🔁 5. Replikasyon ve Failover Testi

### Master’a Veri Yazma
```bash
docker exec redis-master redis-cli set test-key "test-value"
```

### Slave’den Veri Okuma
```bash
docker exec redis-slave redis-cli get test-key
```

### Master Düğümünü Kapatıp Failover Testi
```bash
docker stop redis-master
docker exec redis-sentinel redis-cli -p 26379 sentinel masters
```

---

## 📊 6. Monitoring ve Loglar

```bash
docker logs -f redis-master
docker logs -f redis-slave
docker logs -f redis-sentinel
```

```bash
docker exec redis-sentinel redis-cli -p 26379 sentinel masters
docker exec redis-sentinel redis-cli -p 26379 sentinel slaves mymaster
docker exec redis-master redis-cli info server
```

---

## 🔐 7. Güvenlik Notları

- Production ortamında Redis şifre koruması kullanın.
- UFW ile sadece ihtiyaç duyulan IP'lere bağlantı izni verin.
- Gerekirse SSL/TLS kullanın.
- Logları merkezi bir sistemde yönetin ve periyodik olarak temizleyin.

---

## ⚙️ 8. HAProxy Entegrasyonu (Önerilen)

### Avantajları:
- **Tek giriş noktası (Single Entry Point)**
- **Şeffaf Failover**
- **Read/Write trafiğinin ayrılması**
- **Bağlantı havuzu yönetimi (Connection Pooling)**

---

Bu adımları tamamlayan herkes, Redis Sentinel kümesini Docker üzerinden yüksek erişilebilirlikli bir yapıda ayağa kaldırabilir.