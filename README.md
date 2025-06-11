# Redis Sentinel 3-VM Docker Deployment Guide

This repository provides a high-availability Redis Sentinel cluster setup using **Docker Compose** across **three virtual machines (VMs)** with **HAProxy** as the load balancer.

---

## 📌 Server Roles

- **VM1**: Redis Master + HAProxy
- **VM2**: Redis Slave + Sentinel
- **VM3**: Redis Slave + Sentinel

---

## 🔧 1. Initial Server Setup (All VMs)

### System Update
```bash
sudo apt update && sudo apt upgrade -y
```

### Required Package Installation
```bash
sudo apt install -y curl wget net-tools ufw
```

### Firewall Configuration
```bash
sudo ufw allow 6379/tcp
sudo ufw allow 26379/tcp
sudo ufw allow 22/tcp
sudo ufw --force enable
```

### Kernel Parameters for Redis
```bash
echo 'vm.overcommit_memory = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.core.somaxconn = 65535' | sudo tee -a /etc/sysctl.conf
echo 'fs.file-max = 100000' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Disable Transparent Huge Pages
```bash
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' | sudo tee -a /etc/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' | sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local
```

---

## 📁 2. Directory Structure and Configuration Files

On each VM:
```bash
mkdir -p ~/redis-sentinel
cd ~/redis-sentinel
mkdir -p redis-data
```

Place your `docker-compose.yml`, `redis.conf`, `sentinel.conf`, and optionally `haproxy.cfg` files into this directory. Adjust IP addresses and node names according to your environment.

```bash
chmod 644 redis.conf sentinel.conf
chmod 755 redis-data
```

---

## 🚀 3. Starting the Services

### VM1 (Redis Master + HAProxy)
```bash
cd ~/redis-sentinel
docker-compose up -d
```

### VM2 and VM3 (Redis Slave + Sentinel)
```bash
cd ~/redis-sentinel
docker-compose up -d
```

---

## ✅ 4. Health Checks and Cluster Status

```bash
docker ps
docker exec redis-master redis-cli info replication
docker exec redis-sentinel redis-cli -p 26379 sentinel masters
```

HAProxy stats UI:  
`http://<VM1-IP>:8080`

---

## 🔁 5. Replication and Failover Test

### Write to Master
```bash
docker exec redis-master redis-cli set test-key "test-value"
```

### Read from Slaves
```bash
docker exec redis-slave redis-cli get test-key
```

### Simulate Failover by Stopping Master
```bash
docker stop redis-master
docker exec redis-sentinel redis-cli -p 26379 sentinel masters
```

---

## 📊 6. Logs and Monitoring

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

## 🔐 7. Security Recommendations

- Always enable Redis authentication in production.
- Restrict firewall rules to only trusted sources.
- Consider enabling SSL/TLS encryption.
- Centralize and rotate logs periodically.

---

## ⚙️ 8. HAProxy Integration (Recommended)

### Benefits:
- **Single entry point** for Redis connections
- **Transparent failover** for applications
- **Read/write separation** (direct reads to slaves)
- **Efficient connection pooling**

---

By following this guide, you can deploy a robust and highly available Redis Sentinel cluster with Docker Compose and HAProxy across three nodes.
