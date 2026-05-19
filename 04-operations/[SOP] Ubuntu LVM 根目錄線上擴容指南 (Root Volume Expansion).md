# [SOP] Ubuntu LVM 根目錄線上擴容指南 (Root Volume Expansion)

## 📌 1. 詮釋資料 (Metadata)

為了讓 RAG 系統能精準匹配使用者意圖，請參考以下文件屬性：

* **文件類型:** 標準作業程序 (SOP) / 知識庫 (Knowledge Base)
* **適用系統:** Ubuntu / Linux (採用 LVM 架構之系統)
* **觸發情境 (Intents):** 伺服器磁碟空間不足、Root 目錄滿載、LVM 擴容、清理空間、`lvextend` 指令使用方式。
* **核心關鍵字:** LVM, Logical Volume, `df -lh`, `vgdisplay`, `lvextend`, `resize2fs`, 無縫擴容 (On-line resizing).

---

## 🛠️ 2. 標準作業程序 (Standard Operating Procedure)

當系統監控發出根目錄 (`/`) 空間不足警報時，請依照以下步驟進行線上擴容。

### 步驟一：確認當前磁碟使用狀態

執行以下指令檢查檔案系統的容量與掛載點，確認系統瓶頸。

```bash
df -lh

```

**預期觀察:** 根目錄 (例如 `/dev/mapper/ubuntu--vg-ubuntu--lv`) 的 `Use%` 可能偏高（如 62% 或以上）。

### 步驟二：檢查 Volume Group (VG) 剩餘空間

確認底層的 Volume Group 是否還有未分配的可用空間 (Free Space)。

```bash
sudo vgdisplay

```

**預期觀察:** 尋找 `Free  PE / Size` 欄位。若顯示有剩餘空間（例如 `13918 / <54.37 GiB`），則代表可以進行擴容。

### 步驟三：擴展邏輯卷並同步調整檔案系統大小

將所有剩餘的 VG 空間分配給指定的 Logical Volume，並同時放大檔案系統。

```bash
sudo lvextend -l +100%FREE -r /dev/mapper/ubuntu--vg-ubuntu--lv

```

**參數說明:**

* `-l +100%FREE`: 將所有剩餘的可用空間 (Free Extents) 全數分配。
* `-r` (resizefs): **關鍵參數**。指示 LVM 在擴展邏輯卷後，自動調用對應的指令（如 `resize2fs`）來放大檔案系統。
* `/dev/mapper/...`: 目標邏輯卷的路徑。

**預期輸出:** 系統提示 Logical volume successfully resized，並顯示 `on-line resizing required` 及檔案系統區塊已更新的訊息。

### 步驟四：驗證擴容結果

再次檢查磁碟空間，確認擴容生效。

```bash
df -lh

```

**預期觀察:** 根目錄的 `Size` 應顯著增加，且 `Use%` 大幅下降，系統恢復健康狀態。

---

## ❓ 3. 問答知識庫 (Q&A)

提供給 RAG 系統作為使用者常見問題的解答素材。

**Q1: 在執行 LVM 擴容時，需要將伺服器停機或卸載 (umount) 磁碟嗎？**

> **A:** 不需要。透過 LVM 與 ext4/xfs 檔案系統的特性，支援線上擴容 (On-line resizing)。您可以在系統運行中直接擴展磁碟並即時生效，無需停機或重啟服務。

**Q2: 為什麼在 `lvextend` 指令中必須加上 `-r` 參數？**

> **A:** `-r` 代表 `--resizefs`。如果只執行 `lvextend`，系統只會擴展底層的邏輯卷 (Logical Volume) 大小，但上層的檔案系統（如 ext4）並不知道空間變大了。加上 `-r` 會自動觸發 `resize2fs`，一步到位完成「磁碟擴展」與「檔案系統擴展」。

**Q3: 如果忘記加上 `-r` 參數，發現 `df -lh` 空間沒有變大怎麼辦？**

> **A:** 無需擔心，只需手動針對該邏輯卷執行檔案系統重置大小的指令即可。針對 ext4 檔案系統，請執行：`sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv`。

**Q4: 當 `vgdisplay` 顯示 Free PE 為 0 時，還能直接使用此 SOP 擴容嗎？**

> **A:** 不能。若 Volume Group (VG) 沒有剩餘空間，您必須先擴增實體磁碟（例如在虛擬機增加硬碟配置），接著使用 `pvcreate` 建立新的 Physical Volume，再透過 `vgextend` 將新空間加入現有的 VG 中，最後才能執行此 SOP 的 `lvextend` 步驟。