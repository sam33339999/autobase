# Autobase Virtual IP (VIP) 配置詳細指南

## 📋 概述

Virtual IP (VIP) 是 PostgreSQL 高可用性叢集的關鍵組件，它提供了一個穩定的 IP 位址供應用程式連接，確保在故障轉移時應用程式無需修改連接設定即可自動切換到新的主節點。

Autobase 提供兩種 VIP 實現方案，本文檔將詳細說明配置方法、運作原理和最佳實務。

---

## 🎯 VIP 實現方案

### 方案對比

| 特性 | HAProxy + Keepalived | vip-manager |
|------|---------------------|-------------|
| **適用架構** | 負載平衡器架構 | 直連架構 |
| **執行條件** | `with_haproxy_load_balancing: true` | `with_haproxy_load_balancing: false` |
| **部署位置** | `balancers` 主機群組 | `postgres_cluster` 主機群組 |
| **連接方式** | 透過 HAProxy 代理 | 直接連接 PostgreSQL |
| **負載分散** | ✅ 支援讀寫分離 | ❌ 僅支援主節點 |
| **連接池** | ✅ 整合 pgBouncer | ❌ 需額外配置 |
| **監控統計** | ✅ HAProxy 統計頁面 | ❌ 依賴外部監控 |
| **雲端相容性** | ⚠️ 有限制 | ⚠️ 有限制 |

---

## 🔧 基本配置

### 必要變數設定

在 `inventory` 檔案的 `[all:vars]` 區段中設定：

```yaml
# VIP 基本設定
cluster_vip: "10.128.64.200"          # ⚠️ 必須設定的 Virtual IP 位址
vip_interface: "ens32"                 # 網路介面名稱，預設自動偵測

# 選擇 VIP 實現方案
with_haproxy_load_balancing: true      # true=Keepalived方案, false=vip-manager方案
```

### 進階變數設定

**檔案位置**: `automation/roles/common/defaults/main.yml`

```yaml
# Keepalived 設定 (HAProxy 方案)
keepalived_virtual_router_id: "{{ cluster_vip.split('.')[3] | int }}"  # 自動使用 VIP 最後八位元
# virtual_router_id 範圍: 0-255，在同一網路中必須唯一

# vip-manager 設定 (直連方案)
vip_manager_version: 3.0.0            # vip-manager 版本
vip_manager_conf: "/etc/patroni/vip-manager.yml"
vip_manager_interval: "1000"          # 檢查間隔 (毫秒)
vip_manager_mask: "24"                # 子網路遮罩 (CIDR)
vip_manager_dcs_type: "{{ dcs_type }}" # DCS 後端: etcd, consul, patroni
```

---

## 🚀 方案一：HAProxy + Keepalived (推薦)

### 架構圖

```
                 Virtual IP: 10.128.64.200
                         │
                    ┌────▼────┐
                    │ Client  │
                    └─────────┘
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
┌───▼───┐           ┌────▼────┐           ┌───▼───┐
│HAProxy│◄─────────►│HAProxy  │◄─────────►│HAProxy│
│Node1  │ Keepalived │Node2    │ Keepalived │Node3  │
└───┬───┘           └────┬────┘           └───┬───┘
    │                     │                    │
    └─────────────────────┼────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    ┌───▼───┐         ┌───▼───┐         ┌───▼───┐
    │ PG    │◄───────►│ PG    │◄───────►│ PG    │
    │Master │  Patroni │Replica│  Patroni │Replica│
    └───────┘         └───────┘         └───────┘
```

### 執行流程

**階段 1**: HAProxy 部署 (`automation/playbooks/balancers.yml`)
```yaml
- hosts: balancers
  roles:
    - vitabaks.autobase.haproxy      # HAProxy 負載平衡器
    - vitabaks.autobase.keepalived   # Keepalived VIP 管理
      when: cluster_vip is defined and cluster_vip | length > 0
```

**階段 2**: Keepalived 配置生成

**檔案位置**: `automation/roles/keepalived/templates/keepalived.conf.j2`

生成的配置範例：
```
vrrp_script chk_haproxy {
    script "/usr/libexec/keepalived/haproxy_check.sh"
    interval 2
    weight 2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP                     # 所有節點都設為 BACKUP
    interface ens32                  # 網路介面
    virtual_router_id 200           # 來自 cluster_vip 最後八位元
    priority 100                     # 優先順序
    advert_int 2                     # 廣播間隔
    authentication {
        auth_type PASS
        auth_pass 1ce24b6e
    }
    virtual_ipaddresses {
        10.128.64.200               # Virtual IP
    }
    track_script {
        chk_haproxy
    }
}
```

### HAProxy 服務埠號配置

```yaml
haproxy_listen_port:
  master: 5000          # 主要寫入服務 (讀寫)
  replicas: 5001        # 讀取副本服務 (唯讀)
  replicas_sync: 5002   # 同步副本服務 (唯讀)
  replicas_async: 5003  # 非同步副本服務 (唯讀)
  stats: 7000           # HAProxy 統計監控頁面
```

### 應用程式連接方式

```bash
# 寫入操作 - 連接到 Master
psql -h 10.128.64.200 -p 5000 -U app_user -d app_database

# 讀取操作 - 連接到 Replica (負載分散)
psql -h 10.128.64.200 -p 5001 -U app_user -d app_database

# 監控頁面
http://10.128.64.200:7000/stats
```

### 核心參數調整

**檔案位置**: `automation/roles/keepalived/tasks/main.yml`

Keepalived 需要的核心參數：
```bash
net.ipv4.ip_nonlocal_bind = 1    # 允許綁定非本機 IP
net.ipv4.ip_forward = 1          # 啟用 IP 轉發
```

### 防火牆規則

```bash
# Keepalived VRRP 協定
iptables -p vrrp -A INPUT -j ACCEPT
iptables -p vrrp -A OUTPUT -j ACCEPT

# HAProxy 服務埠號
iptables -A INPUT -p tcp --dport 5000:5003 -j ACCEPT
iptables -A INPUT -p tcp --dport 7000 -j ACCEPT
```

---

## 🎮 方案二：vip-manager (直連)

### 架構圖

```
                 Virtual IP: 10.128.64.200
                         │
                    ┌────▼────┐
                    │ Client  │
                    └─────────┘
                         │
                         │ 直接連接 (無代理)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    ┌───▼───┐        ┌───▼───┐        ┌───▼───┐
    │ PG    │◄──────►│ PG    │◄──────►│ PG    │
    │Master │ Patroni │Replica│ Patroni │Replica│
    │+VIP   │        │       │        │       │
    └───────┘        └───────┘        └───────┘
        │                │                │
    ┌───▼───┐        ┌───▼───┐        ┌───▼───┐
    │vip-mgr│        │vip-mgr│        │vip-mgr│
    │Active │        │Standby│        │Standby│
    └───────┘        └───────┘        └───────┘
```

### 執行流程

**階段 1**: vip-manager 部署 (`automation/playbooks/deploy_pgcluster.yml`)
```yaml
- hosts: postgres_cluster
  roles:
    - role: vitabaks.autobase.vip_manager
      when: not with_haproxy_load_balancing | default(false) | bool and
            (cluster_vip is defined and cluster_vip | length > 0)
```

**階段 2**: vip-manager 安裝

**檔案位置**: `automation/roles/vip_manager/tasks/main.yml`

1. **下載套件**: 從 GitHub Releases 下載最新版本
```bash
wget https://github.com/cybertec-postgresql/vip-manager/releases/download/v3.0.0/vip-manager_3.0.0_Linux_x86_64.deb
```

2. **安裝套件**:
   - Debian/Ubuntu: `apt install vip-manager_3.0.0_Linux_x86_64.deb`
   - RedHat/CentOS: `yum install vip-manager_3.0.0_Linux_x86_64.rpm`

### vip-manager 配置

**檔案位置**: `/etc/patroni/vip-manager.yml`

```yaml
# vip-manager 配置範例
nodename: "pgnode01"                 # 節點名稱
ip: "10.128.64.200"                 # Virtual IP 位址
iface: "ens32"                      # 網路介面
mask: 24                            # 子網路遮罩
dcs_type: "etcd"                    # DCS 類型: etcd, consul, patroni
interval: 1000                      # 檢查間隔 (毫秒)

# etcd 設定 (如果使用 etcd 作為 DCS)
etcd_endpoints:
  - "http://10.128.64.140:2379"
  - "http://10.128.64.142:2379"  
  - "http://10.128.64.143:2379"
etcd_key: "/service/postgres-cluster/leader"

# Consul 設定 (如果使用 Consul 作為 DCS)
consul_endpoint: "http://127.0.0.1:8500"
consul_key: "service/postgres-cluster/leader"

# Patroni API 設定 (如果直接使用 Patroni API)
patroni_endpoints:
  - "http://10.128.64.140:8008"
  - "http://10.128.64.142:8008"
  - "http://10.128.64.143:8008"
```

### systemd 服務配置

**檔案位置**: `/etc/systemd/system/vip-manager.service`

```ini
[Unit]
Description=Manages a virtual IP based on the state of a PostgreSQL cluster
Wants=network-online.target
After=network-online.target
BindsTo=postgresql.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/vip-manager -config=/etc/patroni/vip-manager.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=0

[Install]
WantedBy=multi-user.target
```

### 應用程式連接方式

```bash
# 所有連接都透過 VIP (自動路由到當前 Master)
psql -h 10.128.64.200 -p 5432 -U app_user -d app_database

# 讀取副本需要直接連接到特定節點
psql -h 10.128.64.142 -p 5432 -U app_user -d app_database
```

---

## 📊 inventory 設定範例

### 完整 inventory 範例

```ini
# PostgreSQL 節點
[master]
10.128.64.140 hostname=pgnode01

[replica]
10.128.64.142 hostname=pgnode02
10.128.64.143 hostname=pgnode03

[postgres_cluster:children]
master
replica

# etcd 叢集 (DCS)
[etcd_cluster]
10.128.64.140
10.128.64.142
10.128.64.143

# HAProxy 負載平衡器 (方案一使用)
[balancers]
10.128.64.144 
10.128.64.145

[all:vars]
# ===== VIP 基本設定 =====
cluster_vip: "10.128.64.200"        # Virtual IP 位址
vip_interface: "ens32"               # 網路介面名稱

# ===== 選擇 VIP 方案 =====
with_haproxy_load_balancing: true    # true=Keepalived, false=vip-manager

# ===== HAProxy 埠號設定 (方案一) =====
haproxy_listen_port:
  master: 5000
  replicas: 5001
  replicas_sync: 5002
  replicas_async: 5003
  stats: 7000

# ===== PostgreSQL 設定 =====
postgresql_version: "16"
postgresql_data_dir: "/var/lib/postgresql/{{ postgresql_version }}/main"

# ===== Patroni 設定 =====
patroni_cluster_name: "postgres-cluster"
patroni_superuser_username: "postgres"
patroni_superuser_password: "SecurePassword123"
patroni_replication_username: "replicator"
patroni_replication_password: "ReplicatorPassword123"

# ===== DCS 設定 =====
dcs_type: "etcd"

# ===== 連接設定 =====
ansible_connection: "ssh"
ansible_ssh_port: "22"
ansible_user: "root"
```

---

## 🔍 故障排除指南

### 常見問題診斷

#### 1. VIP 無法啟動

**檢查項目**:
```bash
# 檢查網路介面是否存在
ip addr show ens32

# 檢查 VIP 是否已被其他服務使用
ping 10.128.64.200

# 檢查核心參數
sysctl net.ipv4.ip_nonlocal_bind
sysctl net.ipv4.ip_forward
```

#### 2. Keepalived 故障

**診斷命令**:
```bash
# 檢查服務狀態
systemctl status keepalived

# 檢查日誌
journalctl -u keepalived -f

# 檢查配置檔案
cat /etc/keepalived/keepalived.conf

# 檢查 HAProxy 健康檢查
/usr/libexec/keepalived/haproxy_check.sh
echo $?  # 應該回傳 0
```

#### 3. vip-manager 故障

**診斷命令**:
```bash
# 檢查服務狀態
systemctl status vip-manager

# 檢查日誌
journalctl -u vip-manager -f

# 檢查配置檔案
cat /etc/patroni/vip-manager.yml

# 手動測試
sudo -u postgres vip-manager -config=/etc/patroni/vip-manager.yml
```

#### 4. DCS 連接問題

**etcd 連接測試**:
```bash
# 檢查 etcd 叢集狀態
etcdctl --endpoints=http://10.128.64.140:2379 cluster-health

# 檢查 Patroni leader 資訊
etcdctl --endpoints=http://10.128.64.140:2379 get /service/postgres-cluster/leader
```

**Consul 連接測試**:
```bash
# 檢查 Consul 叢集狀態
consul members

# 檢查 Patroni leader 資訊
consul kv get service/postgres-cluster/leader
```

### 網路排除

```bash
# 檢查 VRRP 多播流量 (Keepalived)
tcpdump -i ens32 vrrp

# 檢查 VIP 綁定狀態
ip addr show ens32 | grep 10.128.64.200

# 檢查路由表
ip route show table local | grep 10.128.64.200

# 檢查 ARP 表
arp -a | grep 10.128.64.200
```

---

## ⚠️ 重要注意事項

### 雲端環境限制

**問題**: VIP 解決方案在雲端環境中可能無法正常運作

**原因**:
- 雲端平台禁止任意 MAC 位址變更
- 不允許自訂 ARP 響應
- 安全群組和網路 ACL 限制

**雲端替代方案**:

#### AWS
```yaml
# 使用 AWS Elastic IP 和 Route 53
aws_use_elastic_ip: true
aws_route53_zone: "example.com"
aws_route53_record: "postgres.example.com"
```

#### Google Cloud Platform
```yaml
# 使用 GCP Static IP 和 Cloud DNS
gcp_use_static_ip: true
gcp_dns_zone: "example-zone"
gcp_dns_record: "postgres.example.com"
```

#### Azure
```yaml
# 使用 Azure Public IP 和 Traffic Manager
azure_use_public_ip: true
azure_traffic_manager_profile: "postgres-tm"
```

### 安全性考量

#### 防火牆規則
```bash
# Keepalived VRRP 協定 (組播 224.0.0.18)
iptables -A INPUT -d 224.0.0.18/32 -j ACCEPT
iptables -A INPUT -p vrrp -j ACCEPT
iptables -A OUTPUT -p vrrp -j ACCEPT

# HAProxy 服務埠號
iptables -A INPUT -p tcp -m multiport --dports 5000:5003,7000 -j ACCEPT

# PostgreSQL 直連埠號 (vip-manager 方案)
iptables -A INPUT -p tcp --dport 5432 -j ACCEPT
```

#### SELinux 設定
```bash
# Keepalived SELinux 策略
setsebool -P keepalived_connect_any on

# 檢查 SELinux 狀態
getsebool keepalived_connect_any
```

### 效能調優

#### Keepalived 參數調整
```
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    advert_int 1           # 減少廣播間隔 (預設 2 秒)
    preempt_delay 60       # 延遲搶佔時間
    nopreempt             # 禁用自動搶佔
}

# 健康檢查調整
vrrp_script chk_haproxy {
    interval 1            # 減少檢查間隔
    timeout 3             # 設定逾時時間
    fall 2                # 減少失敗閾值
    rise 1                # 減少恢復閾值
}
```

#### vip-manager 參數調整
```yaml
# /etc/patroni/vip-manager.yml
interval: 500             # 減少檢查間隔到 500ms
retry_num: 3              # 增加重試次數
retry_after: 1000         # 重試間隔
```

---

## 📈 監控與維護

### Keepalived 監控

```bash
# 檢查 VIP 狀態
ip addr show ens32 | grep "10.128.64.200"

# 檢查 Keepalived 狀態
systemctl is-active keepalived

# 監控 VRRP 狀態變化
journalctl -u keepalived -f | grep "VRRP_Instance"

# 檢查優先順序
grep "priority" /etc/keepalived/keepalived.conf
```

### vip-manager 監控

```bash
# 檢查 VIP 管理狀態
systemctl is-active vip-manager

# 監控 VIP 切換日誌
journalctl -u vip-manager -f

# 檢查 DCS 連接狀態
sudo -u postgres vip-manager -config=/etc/patroni/vip-manager.yml -check-dcs
```

### 自動化監控腳本

```bash
#!/bin/bash
# vip-monitor.sh - VIP 狀態監控腳本

VIP="10.128.64.200"
INTERFACE="ens32"
LOG_FILE="/var/log/vip-monitor.log"

check_vip_status() {
    if ip addr show $INTERFACE | grep -q $VIP; then
        echo "$(date): VIP $VIP is active on $(hostname)" >> $LOG_FILE
        return 0
    else
        echo "$(date): VIP $VIP is not active on $(hostname)" >> $LOG_FILE
        return 1
    fi
}

# 每分鐘執行一次
while true; do
    check_vip_status
    sleep 60
done
```

### Prometheus 監控指標

```yaml
# prometheus.yml
- job_name: 'keepalived'
  static_configs:
    - targets: ['10.128.64.144:9165', '10.128.64.145:9165']
  
- job_name: 'vip-manager'
  static_configs:
    - targets: ['10.128.64.140:9166', '10.128.64.142:9166', '10.128.64.143:9166']
```

---

## 🎯 最佳實務建議

### 部署策略

1. **測試環境驗證**:
   - 先在測試環境完整驗證 VIP 功能
   - 測試故障轉移場景
   - 驗證應用程式連接穩定性

2. **段階式部署**:
   - 先部署 PostgreSQL 叢集
   - 確認叢集穩定後再啟用 VIP
   - 逐步導入生產流量

3. **監控告警**:
   - 設定 VIP 狀態監控告警
   - 監控故障轉移事件
   - 設定效能基準告警

### 維護作業

1. **定期檢查**:
   - 每週檢查 VIP 服務狀態
   - 驗證故障轉移功能
   - 檢查日誌是否有異常

2. **更新策略**:
   - 更新 vip-manager/keepalived 版本前先測試
   - 保持配置檔案備份
   - 規劃維護時間窗口

3. **災難恢復**:
   - 準備 VIP 服務快速恢復程序  
   - 備份所有相關配置檔案
   - 測試完整的災難恢復流程

---

## 📚 相關資源

### 官方文件
- [vip-manager GitHub](https://github.com/cybertec-postgresql/vip-manager)
- [Keepalived 官方文檔](https://keepalived.readthedocs.io/)
- [HAProxy 配置指南](http://www.haproxy.org/download/2.8/doc/configuration.txt)

### 相關檔案位置
- **VIP 配置變數**: `automation/roles/common/defaults/main.yml`
- **Keepalived 角色**: `automation/roles/keepalived/`
- **vip-manager 角色**: `automation/roles/vip_manager/`
- **HAProxy 角色**: `automation/roles/haproxy/`
- **防火牆規則**: `automation/roles/firewall/`

### 故障排除資源
- **Ansible 執行日誌**: `/var/log/ansible.log`
- **Keepalived 日誌**: `journalctl -u keepalived`
- **vip-manager 日誌**: `journalctl -u vip-manager`
- **HAProxy 日誌**: `/var/log/haproxy.log`

---

**最後更新**: 2025年11月
**版本**: v1.0
**適用範圍**: Autobase v2.4.1+
