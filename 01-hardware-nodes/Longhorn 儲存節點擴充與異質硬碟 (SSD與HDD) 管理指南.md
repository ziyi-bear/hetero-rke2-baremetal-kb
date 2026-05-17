# Longhorn 儲存節點擴充與異質硬碟 (SSD/HDD) 管理指南

## 📝 簡介 (Overview)

本文件說明在 Kubernetes 叢集中，當節點具備異質硬碟（例如：同時擁有用於 OS 的 SSD 與用於擴充容量的 HDD）時，如何透過 Longhorn 進行次要硬碟的初始化、掛載、配置與標籤管理，以確保儲存資源的高效分配。

---

## 🛠️ 第一部分：標準作業流程 (SOP)

### 階段一：作業系統層級磁碟準備 (OS-Level Disk Preparation)

當節點新增一顆獨立硬碟（例如 `/dev/sdb`）準備交由 Longhorn 進行全權管理時，請依照以下步驟執行：

1. **格式化硬碟為 XFS 檔案系統**
```bash
sudo mkfs.xfs -f /dev/sdb

```


2. **建立專屬掛載目錄**
```bash
sudo mkdir -p /mnt/longhorn-hdd

```


3. **取得硬碟 UUID**

```bash
   sudo blkid /dev/sdb

```

*(預期輸出：擷取 `/dev/sdb` 的 UUID 字串)*
4. **設定開機自動掛載 (`/etc/fstab`)**
使用編輯器打開 `/etc/fstab`，並加入以下設定（請替換為實際 UUID）：

```text
   UUID=<你的硬碟UUID> /mnt/longhorn-hdd xfs defaults 0 2

```

5. **執行掛載並驗證狀態**

```bash
   sudo mount -a
   df -h /mnt/longhorn-hdd

```

### 階段二：Longhorn UI 節點配置 (Longhorn UI Configuration)

1. 進入 Longhorn UI，導航至 **Node** 頁面。
2. 選擇目標節點並點擊 **Edit Node** -> **Add Disk**。
3. 填入以下配置：
* **Disk Type:** `Filesystem`
* **Path:** `/mnt/longhorn-hdd`
* **Storage Reserved:** 建議設定較低的值（例如 `10 Gi` 或 `5%`），因為此為專用資料碟，無須像 OS 碟保留大量系統空間。
* **Tags:** 輸入 `hdd`（極度重要，用於後續 StorageClass 路由）。


4. 點擊 Save。Longhorn 將立即識別新容量並可開始調度 Volume。

---

## 💡 第二部分：建議說明與最佳實踐 (Recommendations & Best Practices)

### 1. 檔案系統的選擇 (XFS vs. EXT4)

對於大容量（如 3.6TB 以上）且專用於 Longhorn 儲存的硬碟，**強烈建議使用 XFS**。

* **XFS 優勢：** 針對大檔案、平行 I/O 操作及大型儲存卷有高度最佳化。在 TB 級硬碟上，XFS 執行檔案系統檢查 (`fsck`) 的速度遠快於 EXT4，且處理 Metadata 的效率更高。
* **EXT4：** 雖然穩定可靠，但在處理海量連續 I/O 串流或超大硬碟的檢查時，效率略遜於 XFS。

### 2. 未來擴充策略 (Future Expansion)

若未來增加更多實體 HDD，**請勿在 OS 層級將其合併（例如使用 LVM 或 RAID0）**。

* **正確做法：** 將新硬碟獨立格式化（如 `/dev/sdc` 為 XFS），掛載到新路徑（如 `/mnt/longhorn-hdd-2`），並在 Longhorn UI 中新增為獨立的 Disk。
* **原因：** Longhorn 具備叢集層級的儲存池化 (Pooling) 能力，會自動將 Volume 副本調度到可用空間最多的實體硬碟上。保持硬碟獨立可降低單點故障的影響範圍。

### 3. 避免「混合硬碟陷阱」 (The Mixed Disk Trap)

若節點同時存在 SSD（作業系統碟，通常為預設的 `/var/lib/longhorn/`）與大容量 HDD，且未使用標籤與自訂 StorageClass 進行隔離，預設的 Longhorn 調度器會因為 HDD 剩餘空間最大，而將所有新建的 Pod（包含需要高效能的資料庫或 LLM 快取）優先分配到慢速 HDD 上。

**解決方案：**

* 必須在 Longhorn UI 中為 `/var/lib/longhorn/` 加上 `ssd` 標籤。
* 必須在 Longhorn UI 中為 `/mnt/longhorn-hdd/` 加上 `hdd` 標籤。
* 在 Kubernetes 中建立兩個獨立的 `StorageClass`，並使用 `diskSelector` 綁定對應標籤。

---

## ❓ 第三部分：常見問題與解答 (Q&A)

**Q1: 當我在新增的硬碟上設定了 Tags（例如 `hdd`）時，預設的 Longhorn StorageClass 還能使用它嗎？**
**A1: 可以的。** 預設的 StorageClass（若未設定 `diskSelector`）不會檢查任何標籤。它會將叢集內所有硬碟（無論有無標籤）視為一個大資源池，並傾向將資料寫入剩餘空間最多的硬碟。這就是為什麼在異質硬碟環境中，必須建立帶有 `diskSelector` 的自訂 StorageClass 來限制調度行為。

**Q2: 只有特定的 StorageClass 才能使用被標記的硬碟嗎？**
**A2: 否，如 Q1 所述，預設無標籤的 StorageClass 也能用。** 但如果建立了一個新的 StorageClass 並加入了 `diskSelector: "hdd"` 參數，那麼使用該 StorageClass 建立的 PVC，就 **只能** 被分配到擁有 `hdd` 標籤的實體硬碟上。

**Q3: 在 Longhorn 中新增硬碟時，`Disk Type` 應該選擇 `Filesystem` 還是 `Block`？**
**A3: 應選擇 `Filesystem`。** 因為硬碟已經在 OS 層級格式化為 XFS 並掛載為目錄。`Block` 類型專用於 Longhorn V2 Data Engine (基於 SPDK)，該引擎要求硬碟必須是未格式化的裸磁碟 (Raw Block Device)。

**Q4: OS 碟和掛載的資料碟在 `Storage Reserved` 設定上應該有什麼差異？**
**A4: 有顯著差異。** OS 碟（通常掛載於 `/`）建議保留至少 30% 的空間，以防止日誌或系統檔案佔滿空間導致 Node NotReady 或崩潰。而完全交由 Longhorn 專用的次要資料碟（如 `/mnt/longhorn-hdd`），可以將保留空間降至 5% 甚至更低（例如僅保留 10 GiB），以最大化儲存利用率。