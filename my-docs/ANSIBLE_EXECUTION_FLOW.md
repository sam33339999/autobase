# Autobase Ansible 腳本執行流程詳細分析

## 📋 總覽

本文件詳細記錄 Autobase 專案中所有 Ansible Playbook 的執行順序、檔案位置、角色職責與依賴關係。

---

## 🚀 主要部署流程 (`deploy_pgcluster.yml`)

### 檔案位置
- **主要檔案**: `automation/playbooks/deploy_pgcluster.yml`
- **相關配置**: `automation/ansible.cfg`
- **清單檔案**: `automation/inventory.example`

### 執行命令
```bash
ansible-playbook -i inventory vitabaks.autobase.deploy_pgcluster
```

---

## 🔍 詳細執行階段分析

### 階段 1: 雲端資源準備
**執行位置**: `localhost` (控制節點)  
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L1-22)

```yaml
- name: Deploy PostgreSQL HA Cluster (based on "Patroni")
  hosts: localhost
  gather_facts: true
  any_errors_fatal: true
```

**執行角色**:
- `vitabaks.autobase.cloud_resources`
  - 檔案位置: `automation/roles/cloud_resources/`
  - 職責: 創建雲端資源 (EC2, VPC, 安全群組等)
  - 條件: `when: cloud_provider | default('') | length > 0`
  - 支援平台: AWS, GCP, Azure, DigitalOcean, Hetzner

**預設任務**:
- 設定 `pgbackrest_install` 變數
- 自動判斷是否啟用備份功能
- 僅適用於 AWS/GCP/Azure 平台

---

### 階段 2: 系統前置檢查與準備
**執行位置**: 所有目標節點  
**主機群組**: `etcd_cluster:consul_instances:balancers:postgres_cluster`  
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L24-181)

#### 2.1 系統資訊收集
```yaml
pre_tasks:
  - name: Gather package facts
    ansible.builtin.package_facts:
      manager: auto
```

**執行動作**:
- 收集已安裝套件資訊
- 定義 `bind_address` 變數
- 顯示系統資訊 (CPU, 記憶體, 磁碟等)
- 清理套件快取 (DNF/APT)

#### 2.2 套件管理準備
- **Red Hat 系列**: `dnf clean all`
- **Debian 系列**: `apt update` + 安裝 `gnupg`, `apt-transport-https`

#### 2.3 預部署指令執行
- 支援自訂 `pre_deploy_command`
- 非同步執行與結果監控
- 失敗時自動停止部署

#### 2.4 核心角色執行
```yaml
roles:
  - role: vitabaks.autobase.authorized_keys  # SSH 公鑰管理
  - role: vitabaks.autobase.pre_checks       # 系統相容性檢查
  - role: vitabaks.autobase.hostname         # 主機名稱設定
```

**角色詳細資訊**:

1. **authorized_keys**
   - 檔案位置: `automation/roles/authorized_keys/`
   - 職責: 配置 SSH 公鑰認證
   - 標籤: `ssh_public_keys`
   - 條件: `if 'ssh_public_keys' is defined`

2. **pre_checks**
   - 檔案位置: `automation/roles/pre_checks/`
   - 職責: 驗證 Ansible 版本、系統需求
   - 變數: `minimal_ansible_version: 2.17.0`
   - 特殊檢查: TimescaleDB 最低 PostgreSQL 版本要求

3. **hostname**
   - 檔案位置: `automation/roles/hostname/`
   - 職責: 設定系統主機名稱
   - 依據: `inventory` 中的 `hostname` 變數

---

### 階段 3: 分散式協調服務 (DCS) 部署
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L183-191)

#### 3.1 etcd 叢集部署
```yaml
- name: Deploy etcd cluster
  ansible.builtin.import_playbook: etcd_cluster.yml
  when: not dcs_exists | default(false) | bool and dcs_type | default('etcd') == "etcd"
  tags: etcd
```

**執行檔案**: `automation/playbooks/etcd_cluster.yml`  
**執行位置**: `etcd_cluster` 主機群組  
**執行條件**: 
- `dcs_exists: false` (不存在現有 DCS)
- `dcs_type: "etcd"` (選擇 etcd 作為 DCS)

**etcd_cluster.yml 詳細流程**:
1. **套件事實收集**: `ansible.builtin.package_facts`
2. **網路位址定義**: `vitabaks.autobase.bind_address`
3. **系統更新**: APT/DNF 快取更新
4. **防火牆配置**: 動態埠號設定
5. **核心角色執行**:
   - `vitabaks.autobase.firewall` - 防火牆規則
   - `vitabaks.autobase.resolv_conf` - DNS 設定
   - `vitabaks.autobase.etc_hosts` - Hosts 檔案
   - `vitabaks.autobase.add_repository` - etcd 軟體庫
   - `vitabaks.autobase.packages` - 基礎套件
   - `vitabaks.autobase.sudo` - 權限設定
   - `vitabaks.autobase.etcd` - **etcd 服務配置**
   - `vitabaks.autobase.copy` - 設定檔複製

#### 3.2 Consul 叢集部署
```yaml
- name: Deploy Consul
  ansible.builtin.import_playbook: consul_cluster.yml
  when: dcs_type | default('etcd') == "consul"
  tags: consul
```

**執行檔案**: `automation/playbooks/consul_cluster.yml`  
**執行位置**: `consul_instances` 主機群組  
**執行條件**: `dcs_type: "consul"`

---

### 階段 4: PostgreSQL 叢集系統配置
**執行位置**: `postgres_cluster` 主機群組  
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L193-245)

#### 4.1 防火牆動態配置
```yaml
pre_tasks:
  - name: Build a firewall_ports_dynamic_var
    ansible.builtin.set_fact:
      firewall_ports_dynamic_var: "{{ firewall_ports_dynamic_var | default([]) + (firewall_allowed_tcp_ports_for[item] | default([])) }}"
    loop: "{{ hostvars[inventory_hostname].group_names }}"
```

**執行邏輯**:
- 根據主機群組動態建立防火牆埠號清單
- 根據主機群組動態建立防火牆規則清單
- 支援 Consul dnsmasq 的 nameserver 設定

#### 4.2 系統基礎配置角色 (按執行順序)

| 序號 | 角色名稱 | 檔案位置 | 主要職責 | 依賴關係 |
|------|----------|----------|----------|----------|
| 1 | `firewall` | `automation/roles/firewall/` | 配置 iptables/firewalld 規則 | - |
| 2 | `resolv_conf` | `automation/roles/resolv_conf/` | DNS 解析設定 | - |
| 3 | `etc_hosts` | `automation/roles/etc_hosts/` | /etc/hosts 檔案管理 | - |
| 4 | `add_repository` | `automation/roles/add_repository/` | 新增 PostgreSQL 官方軟體庫 | resolv_conf |
| 5 | `packages` | `automation/roles/packages/` | 安裝系統必要套件 | add_repository |
| 6 | `sudo` | `automation/roles/sudo/` | sudo 權限配置 | - |
| 7 | `mount` | `automation/roles/mount/` | 檔案系統掛載設定 | - |
| 8 | `swap` | `automation/roles/swap/` | Swap 記憶體設定 | - |
| 9 | `sysctl` | `automation/roles/sysctl/` | 核心參數調優 | - |
| 10 | `transparent_huge_pages` | `automation/roles/transparent_huge_pages/` | 禁用透明大頁 | - |
| 11 | `pam_limits` | `automation/roles/pam_limits/` | 系統資源限制設定 | - |
| 12 | `io_scheduler` | `automation/roles/io_scheduler/` | I/O 排程器設定 | - |
| 13 | `locales` | `automation/roles/locales/` | 語言環境設定 | - |
| 14 | `timezone` | `automation/roles/timezone/` | 時區設定 | - |
| 15 | `ntp` | `automation/roles/ntp/` | 時間同步服務 | timezone |
| 16 | `ssh_keys` | `automation/roles/ssh_keys/` | SSH 金鑰交換 | - |
| 17 | `copy` | `automation/roles/copy/` | 自訂檔案複製 | - |

---

### 階段 5: 負載平衡器部署
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L247-250)

```yaml
- name: Deploy balancers
  ansible.builtin.import_playbook: balancers.yml
  when: with_haproxy_load_balancing | default(false) | bool
  tags: load_balancing, haproxy
```

**執行檔案**: `automation/playbooks/balancers.yml`  
**執行位置**: `balancers` 主機群組  
**執行條件**: `with_haproxy_load_balancing: true`

#### balancers.yml 詳細流程:
1. **系統準備** (與前面階段類似)
2. **角色執行**:
   - `vitabaks.autobase.firewall` - HAProxy 專用防火牆規則
   - `vitabaks.autobase.resolv_conf` - DNS 設定
   - `vitabaks.autobase.etc_hosts` - Hosts 檔案
   - `vitabaks.autobase.add_repository` - HAProxy 軟體庫
   - `vitabaks.autobase.packages` - HAProxy 相關套件
   - `vitabaks.autobase.sudo` - 權限設定
   - `vitabaks.autobase.haproxy` - **HAProxy 配置**
   - `vitabaks.autobase.keepalived` - 高可用性設定 (可選)
   - `vitabaks.autobase.copy` - 設定檔複製

**HAProxy 配置詳情**:
- 檔案位置: `automation/roles/haproxy/`
- 監聽埠號:
  - Master: 5000
  - Replicas: 5001
  - Sync Replicas: 5002
  - Async Replicas: 5003
  - Stats: 7000

---

### 階段 6: 備份系統安裝
**執行位置**: `pgbackrest:postgres_cluster` 主機群組  
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L252-272)

```yaml
- name: Install and configure pgBackRest
  hosts: pgbackrest:postgres_cluster
  roles:
    - role: vitabaks.autobase.pgbackrest
      when: pgbackrest_install | default(false) | bool
```

**pgBackRest 角色詳情**:
- 檔案位置: `automation/roles/pgbackrest/`
- 支援儲存: 本地、S3、Azure Blob、GCS
- 執行條件: `pgbackrest_install: true`
- 特殊設定: 雲端平台自動啟用備份

---

### 階段 7: PostgreSQL 核心部署
**執行位置**: `postgres_cluster` 主機群組  
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L274-343)

#### 7.1 SSL/TLS 憑證準備
```yaml
pre_tasks:
  - name: Generate Postgres TLS certificate
    ansible.builtin.include_role:
      name: vitabaks.autobase.tls_certificate
    vars:
      tls_group_name: "postgres_cluster"
      tls_cert_regenerate: "{{ postgres_tls_cert_regenerate | default(false) }}"
    when: tls_cert_generate | default(true) | bool
```

**TLS 憑證管理**:
- 檔案位置: `automation/roles/tls_certificate/`
- 憑證生成: 自簽名 CA 和節點憑證
- 憑證分發: 複製到所有叢集節點
- 安全性: 支援 SSL/TLS 加密連接

#### 7.2 核心服務角色部署 (按執行順序)

| 序號 | 角色名稱 | 檔案位置 | 主要職責 | 執行條件 |
|------|----------|----------|----------|----------|
| 1 | `wal_g` | `automation/roles/wal_g/` | WAL-G 備份工具安裝 | `wal_g_install: true` |
| 2 | `pg_probackup` | `automation/roles/pg_probackup/` | PostgreSQL 專業備份工具 | `pg_probackup_install: true` |
| 3 | `cron` | `automation/roles/cron/` | 定時任務設定 | 總是執行 |
| 4 | `pgbouncer` | `automation/roles/pgbouncer/` | 連接池安裝與基礎配置 | `pgbouncer_install: true` |
| 5 | `pgpass` | `automation/roles/pgpass/` | PostgreSQL 密碼檔案 | 總是執行 |
| 6 | **`patroni`** | `automation/roles/patroni/` | **Patroni 高可用性核心** | **總是執行** |
| 7 | `vip_manager` | `automation/roles/vip_manager/` | 虛擬 IP 管理 | 無 HAProxy 且有 cluster_vip |
| 8 | `postgresql_users` | `automation/roles/postgresql_users/` | 資料庫使用者創建 | 僅 master 節點 |
| 9 | `postgresql_databases` | `automation/roles/postgresql_databases/` | 資料庫創建 | 僅 master 節點 |
| 10 | `postgresql_schemas` | `automation/roles/postgresql_schemas/` | Schema 管理 | 僅 master 節點 |
| 11 | `postgresql_privs` | `automation/roles/postgresql_privs/` | 權限設定 | 僅 master 節點 |
| 12 | `postgresql_extensions` | `automation/roles/postgresql_extensions/` | 擴充功能安裝 | 僅 master 節點 |
| 13 | `netdata` | `automation/roles/netdata/` | 監控系統安裝 | 總是執行 |

#### 7.3 Patroni 角色深度分析
**檔案位置**: `automation/roles/patroni/`  
**這是整個部署過程的核心角色**

**主要任務**:
1. **PostgreSQL 安裝**: 安裝指定版本的 PostgreSQL
2. **Patroni 安裝**: 安裝 Patroni 套件
3. **配置生成**: 生成 Patroni YAML 配置檔案
4. **叢集初始化**: 初始化 PostgreSQL 叢集
5. **高可用性設定**: 設定故障轉移規則
6. **服務啟動**: 啟動 Patroni 服務

**關鍵配置**:
- DCS 連接設定 (etcd/Consul)
- PostgreSQL 參數調優
- 複製設定
- 故障轉移政策
- SSL/TLS 配置

---

### 階段 8: 後續配置與完成
**檔案位置**: `automation/playbooks/deploy_pgcluster.yml` (L345-421)

#### 8.1 備份 Stanza 創建
```yaml
tasks:
  - name: Create pgbackrest stanza
    ansible.builtin.include_role:
      name: vitabaks.autobase.pgbackrest
      tasks_from: stanza_create
    when: pgbackrest_install | default(false) | bool
```

#### 8.2 pgBouncer 最終配置
```yaml
  - name: Install and configure pgbouncer
    ansible.builtin.include_role:
      name: vitabaks.autobase.pgbouncer
      tasks_from: config
    when: pgbouncer_install | default(true) | bool
```

#### 8.3 後部署自訂指令
- 支援 `post_deploy_command` 變數
- 非同步執行與監控
- 錯誤處理與日誌收集

#### 8.4 部署完成確認
```yaml
  - name: Cluster deployment completed
    ansible.builtin.include_role:
      name: vitabaks.autobase.deploy_finish
```

**deploy_finish 角色**:
- 檔案位置: `automation/roles/deploy_finish/`
- 職責: 顯示部署完成資訊、連接資訊等

---

## 🔄 其他重要 Playbook 分析

### 配置管理 (`config_pgcluster.yml`)
**檔案位置**: `automation/playbooks/config_pgcluster.yml`  
**用途**: 重新配置現有叢集設定

#### 執行階段:
1. **叢集狀態檢查** (`postgres_cluster:etcd_cluster`)
2. **配置更新** (`postgres_cluster`)
3. **服務重載** (Patroni reload)

#### 主要角色:
- `vitabaks.autobase.patroni` (配置更新)
- `vitabaks.autobase.postgresql_users`
- `vitabaks.autobase.postgresql_databases`
- `vitabaks.autobase.postgresql_extensions`

---

### 滾動更新 (`update_pgcluster.yml`)
**檔案位置**: `automation/playbooks/update_pgcluster.yml`  
**用途**: 系統套件和 PostgreSQL 小版本更新

#### 執行階段:
1. **更新前檢查**
2. **Master 節點更新**
3. **Replica 節點逐一更新**
4. **服務驗證**

#### 關鍵角色:
- `vitabaks.autobase.update` - 系統更新核心
- `vitabaks.autobase.patroni` - 服務重啟管理

---

### 版本升級 (`pg_upgrade.yml`)
**檔案位置**: `automation/playbooks/pg_upgrade.yml`  
**用途**: PostgreSQL 主要版本升級 (如 13 -> 14)

#### 執行階段:
1. **升級前備份**
2. **服務停止與升級**
3. **資料遷移**
4. **服務驗證**

#### 相關檔案:
- `pg_upgrade_rollback.yml` - 升級回滾
- `automation/roles/upgrade/` - 升級核心邏輯

---

### 節點管理

#### 新增節點 (`add_node.yml`)
**檔案位置**: `automation/playbooks/add_node.yml`  
**用途**: 向現有叢集新增 PostgreSQL 或 etcd 節點

#### 移除節點 (`remove_node.yml`)
**檔案位置**: `automation/playbooks/remove_node.yml`  
**用途**: 從叢集中安全移除節點

#### 叢集清理 (`remove_cluster.yml`)
**檔案位置**: `automation/playbooks/remove_cluster.yml`  
**用途**: 完全移除叢集 (⚠️ 危險操作)

**支援變數**:
- `remove_postgres: true` - 移除 PostgreSQL 資料
- `remove_etcd: true` - 移除 etcd 資料  
- `remove_consul: true` - 移除 Consul 資料

---

## 📂 重要檔案位置摘要

### 配置檔案
- **Ansible 主配置**: `automation/ansible.cfg`
- **Galaxy 定義**: `automation/galaxy.yml`
- **依賴需求**: `automation/requirements.yml`
- **清單範例**: `automation/inventory.example`
- **容器配置**: `automation/Dockerfile`

### Playbook 檔案
- **主要部署**: `automation/playbooks/deploy_pgcluster.yml`
- **配置管理**: `automation/playbooks/config_pgcluster.yml`
- **滾動更新**: `automation/playbooks/update_pgcluster.yml`
- **版本升級**: `automation/playbooks/pg_upgrade.yml`
- **節點管理**: `automation/playbooks/add_node.yml`, `remove_node.yml`
- **叢集清理**: `automation/playbooks/remove_cluster.yml`

### 角色檔案 (40+ roles)
- **核心角色**: `automation/roles/patroni/`, `automation/roles/etcd/`
- **系統角色**: `automation/roles/firewall/`, `automation/roles/packages/`
- **資料庫角色**: `automation/roles/postgresql_*/`
- **備份角色**: `automation/roles/pgbackrest/`, `automation/roles/wal_g/`
- **監控角色**: `automation/roles/netdata/`

---

## ⚡ 執行流程關鍵點

### 並行執行
- 不同主機群組可同時執行 (etcd_cluster, postgres_cluster, balancers)
- 同一群組內的主機並行處理 (受 `forks = 10` 限制)

### 錯誤處理
- `any_errors_fatal: true` - 任何錯誤立即停止
- 重試機制 - 網路操作自動重試
- 條件執行 - 失敗時跳過非必要步驟

### 依賴管理
- DCS 必須在 PostgreSQL 之前部署
- 備份系統在 PostgreSQL 安裝後配置
- 負載平衡器在 PostgreSQL 叢集建立後配置

### 標籤系統
- `tags: always` - 總是執行的任務
- `tags: etcd, consul` - 條件性標籤
- `tags: firewall, ssh_public_keys` - 功能性標籤

---

## 🎯 最佳實務建議

### 部署順序
1. **首次部署**: 使用 `deploy_pgcluster.yml`
2. **配置變更**: 使用 `config_pgcluster.yml`  
3. **系統更新**: 使用 `update_pgcluster.yml`
4. **版本升級**: 使用 `pg_upgrade.yml`

### 監控要點
- 檢查 DCS 叢集健康狀態
- 監控 Patroni 叢集狀態
- 驗證備份功能運作
- 確認負載平衡器運作

### 故障排除
- 查看 Ansible 執行日誌
- 檢查各服務狀態 (patroni, etcd/consul, haproxy)
- 驗證網路連通性和防火牆設定
- 確認 SSL/TLS 憑證有效性

這份文件提供了 Autobase Ansible 腳本的完整執行地圖，幫助您深入理解每個階段的運作機制與檔案位置。