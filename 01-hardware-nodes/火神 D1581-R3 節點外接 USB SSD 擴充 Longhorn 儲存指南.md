# 火神 D1581-R3 節點外接 USB SSD 擴充 Longhorn 儲存指南

## 1. 專案概述 (Overview)

本文件記錄在基於火神 (HSGM) D1581-R3 主機板的 Kubernetes 節點上，透過 USB 介面擴充 Longhorn 分散式儲存的硬體相容性分析與標準掛載流程。

* **節點主機：** `d1581with6600xt` (處理器：Intel Xeon D-1581)
* **儲存系統：** Kubernetes Longhorn
* **外接設備：** Samsung T7 Portable SSD 1TB
* **掛載路徑：** `/mnt/longhorn-t7`

---

## 2. 標準作業程序 (SOP：掛載外接硬碟至 Longhorn)

當接入新的 USB SSD（如 `/dev/sdg`）並確認設備代號後，請依序執行以下指令進行初始化與掛載。

### Step 2.1: 清除既有資料與分割表

確保硬碟未被佔用，並徹底清除舊有資料與分割表。

```bash
sudo wipefs -a /dev/sdg

```

### Step 2.2: 建立 GPT 分割區與格式化

為 Longhorn 建立單一 `ext4` 分割區。

```bash
# 建立 GPT 分割表
sudo parted -s /dev/sdg mklabel gpt

# 建立主分割區 (使用 100% 空間)
sudo parted -s -a opt /dev/sdg mkpart primary ext4 0% 100%

# 將分割區格式化為 ext4
sudo mkfs.ext4 /dev/sdg1

```

### Step 2.3: 設定開機自動掛載 (fstab)

使用 UUID 掛載以避免 USB 重新插拔導致裝置代號 (sdg) 漂移。

```bash
# 建立掛載點目錄
sudo mkdir -p /mnt/longhorn-t7

# 寫入 UUID 至 /etc/fstab
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdg1) /mnt/longhorn-t7 ext4 defaults 0 2" | sudo tee -a /etc/fstab

```

### Step 2.4: 重新載入守護行程與驗證

```bash
# 重新載入 systemd 配置 (消除 fstab 修改警告)
sudo systemctl daemon-reload

# 執行全域掛載測試
sudo mount -a

# 驗證掛載結果與容量
df -h | grep t7

```

### Step 2.5: 於 Longhorn UI 加入節點磁碟

1. 登入 Longhorn Dashboard，進入 **Node** 頁面。
2. 編輯目標節點 (`d1581with6600xt`)，點擊 **+ Add Disk**。
3. **Path:** 輸入 `/mnt/longhorn-t7`。
4. **Scheduling:** `Enable`。
5. **Tags:** 輸入 `ssd` 或 `fast` (用於後續 StorageClass 節點親和性調度)。
6. 儲存設定，等待狀態轉為 `Schedulable`。

---

## 3. 常見問題與硬體知識庫 (Q&A)

### Q1: 火神 D1581-R3 主機板的後方面板 USB 速度極限在哪？

**A:** 該主機板後方的藍色 USB 接口為 **USB 3.0 (USB 3.2 Gen 1)**，理論頻寬為 5 Gbps，實際極限傳輸速度約落在 **450~500 MB/s**。

* 該面板**不具備** 10Gbps 或 Type-C 接口。
* 外接高速 NVMe M.2 隨身盒在此接口上無法發揮 1000MB/s 以上的效能，速度會被限制在與傳統 SATA III SSD (500 MB/s) 相當的水準。

### Q2: 如何判斷 USB 隨身碟是否適合用於 Longhorn Disk？

**A:** 關鍵在於是否支援 **UASP (USB Attached SCSI Protocol)**。可透過指令 `lsusb -t` 確認 `Driver` 類型：

* **BOT 協議 (`Driver=usb-storage`)**：如 SanDisk SDCZ430。單工處理，不支援 NCQ，4K 隨機讀寫極差。**強烈不建議**用於 Longhorn，會導致 I/O 卡死與 Volume 重建 (Degraded)。
* **UASP 協議 (`Driver=uas`)**：如 Samsung T7, Kingston DTMAX。支援全雙工與 NCQ，大幅降低 CPU 佔用率並提升 4K IOPS。可作為 Longhorn 輕中度負載的儲存節點。

### Q3: 效能排序比較 (SATA vs. UASP vs. BOT)

**A:** 在 D1581 平台上，作為系統儲存空間的穩定性與效能排序為：

1. **原生 SATA III SSD**：穩跑 500~550 MB/s，無 USB 橋接晶片造成的延遲，穩定性最高。
2. **USB 3.0 UASP (如 Samsung T7)**：極限約 400~450 MB/s，適合一般 Container 運行，但需承擔 Linux 內核偶發的 USB 重置 (USB reset) 風險。
3. **一般 USB 隨身碟 (如 SDCZ430)**：連續讀取約 100 MB/s，但遇到密集的碎檔寫入時效能會遠低於傳統 HDD。

### Q4: 叢集中同時有 HDD 與外接 SSD，Longhorn 該如何分配？

**A:** 務必在 Longhorn Node Disk 設定中使用 **Disk Tags** 功能。例如將 HDD 標記為 `hdd`，將 Samsung T7 標記為 `ssd`。在建立 PVC 時，透過配置不同的 StorageClass 參數，強制將具備狀態的資料庫 Pods 長在 `ssd` 上，以確保服務穩定性。