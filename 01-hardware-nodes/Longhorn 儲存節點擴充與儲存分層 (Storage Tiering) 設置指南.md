# Longhorn 儲存節點擴充與儲存分層 (Storage Tiering) 設置指南

**標籤 (Tags):** `Kubernetes`, `Rancher`, `Longhorn`, `StorageClass`, `Storage-Tiering`, `Ubuntu`, `SOP`  
**適用環境:** 帶有異質儲存媒體 (SSD/HDD) 的 K8s/Longhorn 叢集 (例：單機測試節點 `delloptiplex5090`)

---

## 📄 [主文件] 系統架構與核心概念

### 1. 異質磁碟識別機制
在 Ubuntu 作業系統中，硬體層級的磁碟類型是透過系統區塊設備的 `ROTA` (Rotational) 屬性來判別：
* **ROTA = 1 (HDD)**：具備物理旋轉碟片的傳統硬碟（如 SATA HDD）。
* **ROTA = 0 (SSD/NVMe)**：無旋轉零件的固態硬碟。

### 2. Longhorn 儲存分層架構 (Storage Tiering)
Longhorn 預設會將資料庫複本隨機分配到所有可用磁碟。若節點同時混用高 IOPS (SSD) 與低 IOPS (HDD) 磁碟，隨機分配將導致「木桶效應」——寫入效能會被最慢的 HDD 拖垮。
**解決方案**：透過 Longhorn 的 **Disk Tag** 與 K8s 的 **StorageClass (`diskSelector`)**，實現資源隔離：
* **SSD Tier (`diskSelector: ssd`)**：分配給高 IOPS 需求的負載（如：資料庫、快取服務）。
* **HDD Tier (`diskSelector: hdd`)**：分配給大容量、低 IOPS 需求的負載（如：備份檔案、靜態資源儲存）。

---

## 🛠️ [SOP 操作] 標準作業流程

### 階段一：作業系統層級磁碟初始化 (Ubuntu)
**目標：將新裸磁碟格式化並設定開機自動掛載。**

1. **確認磁碟位置與類型**：
   執行 `lsblk -d -o NAME,ROTA,SIZE,TYPE,MODEL` 找出目標磁碟（以 `/dev/sda` 為例）。
2. **格式化磁碟**：
```bash
   # 警告：此步驟將清除 /dev/sda 上的所有資料
   sudo mkfs.ext4 /dev/sda

```

3. **建立掛載點**：

```bash
   sudo mkdir -p /var/lib/longhorn-hdd

```

4. **寫入 UUID 至 /etc/fstab 以確保重開機掛載持久化**：

```bash
   echo "UUID=$(sudo blkid -s UUID -o value /dev/sda) /var/lib/longhorn-hdd ext4 defaults 0 2" | sudo tee -a /etc/fstab
   sudo mount -a
   df -h | grep longhorn-hdd

```

### 階段二：Longhorn UI 磁碟註冊與標記 (Tagging)

**目標：將掛載的磁碟交由 Longhorn 管理並賦予識別標籤。**

1. 登入 Rancher / Longhorn UI，導航至 **Node** 頁面。
2. 找到目標節點（例：`delloptiplex5090`），點擊 **Edit Node and Disks**。
3. **標記既有 SSD**：找到預設磁碟路徑（如 `/var/lib/longhorn`），於 **Tags** 欄位加入 `ssd`。
4. **加入並標記新 HDD**：

* 點擊 **Add Disk**。
* Path: `/var/lib/longhorn-hdd`
* Scheduling: `Enabled`
* Tags: `hdd`

5. 點擊 **Save**。

### 階段三：Kubernetes StorageClass 建立與驗證

**目標：建立特定標籤的 StorageClass 並配置 PVC。**

1. **部署 StorageClass**：
建立並套用以下 YAML 配置（需注意 `numberOfReplicas` 的設定）。

```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: longhorn-hdd
   provisioner: driver.longhorn.io
   allowVolumeExpansion: true
   reclaimPolicy: Delete
   volumeBindingMode: Immediate
   parameters:
     numberOfReplicas: "1" # 單節點測試環境務必設為 1
     staleReplicaTimeout: "2880"
     fsType: "ext4"
     diskSelector: "hdd"   # 強制調度至 hdd 標籤磁碟

```

2. **建立 PVC 驗證配置**：

```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-hdd-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: longhorn-hdd
     resources:
       requests:
         storage: 50Gi

```

---

## ❓ [QA] 常見問題與故障排除

**Q1: 為什麼我透過 StorageClass (HDD) 建立的 PVC 一直卡在 `Pending` 狀態？**

* **A:** 最常見原因是 `numberOfReplicas` 設定與實際擁有該標籤的節點/磁碟數量不符。Longhorn 預設 Replica 為 3，且預設開啟節點反親和性（Anti-affinity）。如果你的叢集中只有一台節點擁有 `hdd` 標籤的磁碟，必須將 StorageClass 的 `numberOfReplicas` 參數改為 `"1"`，否則 Longhorn 無法找到足夠的分佈位置。

**Q2: 如何在不重開機的情況下，確認系統新安裝的磁碟是 SSD 還是 HDD？**

* **A:** 可以使用指令 `lsblk -d -o NAME,ROTA` 查詢。如果回傳的 `ROTA` 值為 `1` 代表是含有機械馬達的 HDD，若為 `0` 則是 SSD/NVMe。

**Q3: 如果我重開機後，磁碟代號從 `/dev/sda` 變成了 `/dev/sdb`，Longhorn 會遺失資料嗎？**

* **A:** 如果在 `/etc/fstab` 中是使用 UUID 掛載磁碟（如 SOP 階段一所述），則作業系統仍會將該磁碟正確掛載至 `/var/lib/longhorn-hdd`。因為 Longhorn 是認「資料夾路徑」而非底層區塊設備代號，所以資料不會遺失且服務會正常恢復。

**Q4: 我可以不設定 Disk Tag，讓系統自動混用 SSD 和 HDD 嗎？**

* **A:** 技術上可行，但極度不建議。如果不設定標籤隔離，Longhorn 的分散式寫入機制會要求所有複本都寫入完成才算成功。只要其中一個複本被隨機分配到 HDD，整體的 I/O 延遲（Latency）就會降級至 HDD 的水準，導致 SSD 的效能優勢完全喪失。