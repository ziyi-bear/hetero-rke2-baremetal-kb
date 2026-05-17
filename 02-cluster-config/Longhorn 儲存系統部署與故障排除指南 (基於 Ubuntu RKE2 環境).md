# 📄 Longhorn 儲存系統部署與故障排除指南 (基於 Ubuntu RKE2 環境)

## 📁 第一部分：主文件 (Main Documentation)

本文件概述在 Ubuntu 作業系統的 RKE2 叢集上，部署與維護 Longhorn 分散式區塊儲存系統的必備環境配置與系統調優指南。

### 1.1 必備系統套件 (APT Packages)

在部署 Longhorn 之前，必須在**所有 Ubuntu 工作節點**上安裝以下系統層級工具，以支援儲存、加密與備份掛載功能：

* **`open-iscsi`** (必要)：Longhorn 依賴 iSCSI 將 Volume 複本暴露為 Pod 的區塊設備。
* **`nfs-common`** (選用，強烈建議)：若需使用 ReadWriteMany (RWX) 儲存卷，或配置 NFS 作為 Longhorn 備份目標 (Backup Target) 時必須安裝。
* **`cryptsetup`** 與 **`dmsetup`** (選用)：若需啟用 Longhorn 的原生儲存卷加密功能時必須安裝。

**安裝指令：**

```bash
sudo apt-get update
sudo apt-get install -y open-iscsi nfs-common cryptsetup dmsetup
sudo systemctl enable --now iscsid

```

### 1.2 系統檔案監控限制調優 (inotify Limits)

Kubernetes 與 Longhorn 高度依賴 `inotify` 子系統來監控檔案與目錄的變更。Ubuntu 預設的限制過低，容易導致 `ENOSPC: System limit for number of file watchers reached` 或 `too many open files` 錯誤。

**建議配置值：**

* `fs.inotify.max_user_watches` = 524288 (高密度節點可設為 1048576)
* `fs.inotify.max_user_instances` = 8192

**配置指令：**

```bash
# 建立專屬的 sysctl 設定檔
sudo tee /etc/sysctl.d/99-longhorn.conf << EOF
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=8192
EOF

# 套用設定
sudo sysctl -p /etc/sysctl.d/99-longhorn.conf

```

### 1.3 核心模組載入 (Kernel Modules)

即使安裝了 `open-iscsi`，系統核心也必須載入對應的驅動程式模組才能正常運作。

**配置指令：**

```bash
# 即時載入模組
sudo modprobe iscsi_tcp
sudo modprobe dm_crypt

# 設定開機自動載入
echo -e "iscsi_tcp\ndm_crypt" | sudo tee /etc/modules-load.d/longhorn.conf

```

### 1.4 多路徑守護進程配置 (Multipathd)

Ubuntu 預設執行的 `multipathd` 服務會嘗試管理所有區塊設備，這會干擾 Longhorn 建立的 iSCSI 設備，導致掛載失敗。必須設定黑名單讓 `multipathd` 略過 Longhorn 設備。

**配置指令：**

```bash
# 將標準區塊設備加入黑名單
sudo tee /etc/multipath.conf << EOF
blacklist {
    devnode "^sd[a-z0-9]+"
}
EOF

# 重啟服務以套用規則
sudo systemctl restart multipathd

```

---

## ❓ 第二部分：常見問答 (Q&A)

*此區塊專為 RAG 系統的語意問答與問題排解 (Troubleshooting) 檢索所設計。*

### Q1: 在 Longhorn UI 的節點狀態中，出現 `Multipathd` 條件檢查失敗 (紅色驚嘆號)，該如何解決？

**A:** 這是因為 Ubuntu 預設的 `multipathd` 服務接管了 Longhorn 的 iSCSI 設備。解決方法是修改 `/etc/multipath.conf`，加入黑名單規則 `blacklist { devnode "^sd[a-z0-9]+" }`，接著執行 `sudo systemctl restart multipathd` 重啟服務。Longhorn 稍後會自動重新檢查，該狀態即會轉為綠色。

### Q2: 在 Longhorn UI 中，`RequiredPackages` 顯示正常，但 `KernelModulesLoaded` 卻顯示檢查失敗，為什麼？

**A:** `RequiredPackages` 顯示綠色代表已經成功安裝了 `open-iscsi` 等套件，但這不代表對應的 Linux 核心模組已經被載入到當前的 Kernel 中。請在該節點執行 `sudo modprobe iscsi_tcp` 與 `sudo modprobe dm_crypt` 進行手動載入，並將這兩個模組名稱寫入 `/etc/modules-load.d/longhorn.conf` 以確保重啟後自動生效。

### Q3: 為什麼我的 Longhorn 系統或是 Kubernetes Pod 一直出現 `ENOSPC: System limit for number of file watchers reached` 錯誤？

**A:** 這是因為 Ubuntu 作業系統預設的 `inotify` 資源限制不足以應付容器化環境的需求。請透過修改 `/etc/sysctl.d/99-longhorn.conf` 檔案，將 `fs.inotify.max_user_watches` 提高至 524288，並將 `fs.inotify.max_user_instances` 提高至 8192，隨後執行 `sudo sysctl -p /etc/sysctl.d/99-longhorn.conf` 使其生效。

### Q4: 如果我要在 Longhorn 使用 NFS 備份，或建立 ReadWriteMany (RWX) 類型的 PVC，Ubuntu 節點需要事前安裝什麼？

**A:** 所有的 Ubuntu 節點都必須事先安裝 `nfs-common` 套件 (指令為 `sudo apt-get install nfs-common`)。若未安裝此套件，節點將無法掛載 NFS 備份目標，也無法處理 RWX 存取模式的儲存卷。