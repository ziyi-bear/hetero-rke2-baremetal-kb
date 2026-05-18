# Longhorn 儲存引擎架構與磁碟配置指南 (v1 vs v2)

> **Document Metadata**
> * **Domain:** Cloud-Native Storage, Kubernetes, Longhorn
> * **Keywords:** Longhorn, Engine v1, Engine v2, SPDK, NVMe, HDD, Block, Filesystem, Storage Pool, CPU Polling
> * **Purpose:** 提供 Longhorn 儲存引擎版本差異、磁碟類型選型（HDD 場景），以及節點混合部署架構的知識參考。
> 
> 

---

## 📁 核心主文件 (Main Document)

### 1. Longhorn Engine 核心架構：v1 與 v2 的技術演進

Longhorn 提供了兩種不同的資料引擎（Data Engine），以應對不同世代的儲存硬體需求：

* **Engine v1 (傳統架構)**
* **底層技術：** 依賴 Linux 作業系統核心（Kernel space），透過標準檔案系統（VFS、Page Cache）以 Sparse files 形式管理資料區塊。
* **資源消耗：** 資源調度採用中斷機制（Interrupt-driven），閒置時 CPU 消耗極低，負載隨 I/O 動態變化。
* **適用場景：** 發展成熟、穩定度高，適合通用型應用場景，以及傳統 HDD 或一般 SATA SSD 儲存設備。


* **Engine v2 (新世代高效能架構)**
* **底層技術：** 基於 **SPDK (Storage Performance Development Kit)** 開發，資料路徑完全在使用者空間（User space）執行，徹底繞過 Linux 核心。
* **資源消耗：** 採用**輪詢模式 (Polling Mode)**，需要綁定專屬的 CPU 核心。即使在 I/O 閒置狀態下，被綁定的 CPU 核心也會保持極高負載（專核專用）。
* **適用場景：** 專為 NVMe 等超高速儲存設備設計，能提供極高 IOPS 與極低延遲，突破作業系統核心帶來的效能瓶頸。



### 2. Node 磁碟配置模式：Filesystem vs Block

在 Longhorn Node 加入新的底層磁碟時，可選擇兩種不同的管理層級：

* **Filesystem (檔案系統模式)**
* **機制：** 需預先在 Linux 格式化（如 ext4/xfs）並掛載路徑（mount path）。Longhorn 透過建立常規檔案來儲存卷資料。
* **特性：** 管理直覺，可透過標準 Linux 指令（`df`, `ls`）監控；享有作業系統 Page Cache（快取機制）的緩衝優勢。


* **Block (裸區塊設備模式)**
* **機制：** 直接將未格式化的原始磁碟路徑（如 `/dev/sdb`）交由 Longhorn 接管。
* **特性：** 繞過檔案系統層，I/O 路徑更短；但在作業系統層面視為黑盒子，無法直接查看內部檔案結構。



**📌 針對傳統 HDD 的最佳實踐：**
對於機械式硬碟（HDD），其效能瓶頸在於「物理磁頭尋道時間」而非 Linux 檔案系統層。因此，**強烈建議傳統 HDD 選擇 `Filesystem` 模式**。檔案系統層的 Page Cache 能有效緩衝 HDD 面對隨機讀寫時的劣勢，且後續維護與除錯遠比裸區塊設備容易。

### 3. 節點引擎混合佈署 (Hybrid Deployment)

Longhorn 支援在同一個 Kubernetes Cluster 甚至同一個 Node 上同時運行 v1 與 v2 引擎，但需遵守以下隔離原則：

* **儲存隔離：** v1 與 v2 必須使用物理或邏輯上分離的磁碟（或 Storage Pool）。同一個掛載路徑或同一個區塊設備不能同時指派給雙引擎。
* **資源隔離：** v2 引擎的 SPDK 輪詢機制會持續佔用 CPU 核心。在同節點混合部署時，必須妥善設定 CPU 隔離（如 `spdk-cpu-mask`），避免 v2 引擎擠壓到 v1 引擎或其他業務 Pod 的運算資源。
* **Volume 層級指定：** 使用者在建立 PVC/Volume 時，可依據需求宣告該 Volume 要綁定 v1 或 v2 引擎。

---

## 💬 Q&A 檢索問答 (Q&A Section)

**Q1: Longhorn engine v1 和 v2 的主要差異為何？**
**A1:** 主要差異在於**底層技術與資料路徑**。v1 依賴 Linux Kernel 與傳統檔案系統，適合一般 SSD/HDD，閒置時 CPU 佔用低；v2 則是基於 SPDK 技術，完全在 User space 運作並採用 CPU 輪詢模式（Polling），專門用來榨乾 NVMe 硬碟的極致效能，但代價是會強制佔用專屬的 CPU 核心。

**Q2: 如果要在 Node 新增一顆傳統 HDD，Disk Type 選擇 Block 或是 Filesystem 有什麼差異？**
**A2:** 選擇 `Filesystem` 需要先格式化並掛載，資料會變成常規檔案，優點是管理直覺且享有系統 Page Cache 緩衝；選擇 `Block` 則是直接使用未格式化的硬碟，路徑較短但管理上是黑盒子。

**Q3: 承上題，對於傳統 HDD，建議選擇哪一種 Disk Type？為什麼？**
**A3:** 強烈建議選擇 **Filesystem**。因為傳統 HDD 的瓶頸在於物理旋轉與尋道延遲，Block 模式減少的核心層延遲對 HDD 幫助微乎其微。相反地，Filesystem 提供的 Page Cache 快取能有效吸收部分隨機 I/O 寫入，且日常維護與監控更加方便。

**Q4: 在同一個 Longhorn Node 上可以混合使用 v1 與 v2 引擎嗎？**
**A4:** **可以，但必須進行資源隔離。** v1 和 v2 引擎必須配置在不同的磁碟或儲存池（Storage Pool）上，不能共用同一個掛載路徑。此外，因為 v2 引擎（SPDK）會霸佔 CPU 核心進行輪詢，混合使用時必須嚴格規劃 CPU 資源配置，以免 v2 影響到 v1 引擎或節點上其他應用的正常運作。