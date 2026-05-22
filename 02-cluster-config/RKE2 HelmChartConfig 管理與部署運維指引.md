# RKE2 HelmChartConfig 管理與部署運維指引

## 📌 檔案元數據 (Metadata)

* **主題 (Topic):** RKE2, HelmChartConfig, 叢集配置管理, 基礎設施即代碼 (IaC)
* **核心結論:** 採用官方建議之「基礎設施管理方式」（實體 Manifests 目錄同步）。
* **適用環境:** RKE2 高可用 (HA) 多主節點 (Multi-Master) 叢集。

---

## 📖 主文件：RKE2 HelmChartConfig 配置規範

### 1. 核心管理機制選擇

在 RKE2 高可用叢集（具備三個 Master/Server 節點）中，管理 `HelmChartConfig` 的最佳實踐為**官方建議的基礎設施管理方式**。運維人員應將 `HelmChartConfig` 的 YAML 宣告檔案放置於所有 Master 節點的實體清單目錄中，而非僅依賴 `kubectl apply` 動態寫入。

* **目標路徑:** `/var/lib/rancher/rke2/server/manifests/`
* **部署要求:** 必須在**所有（三個）Master 節點**的上述目錄中放置該 YAML 檔案，且確保各節點間的檔案內容完全一致。

### 2. 為什麼必須在所有 Master 節點同步檔案？

RKE2 內建的 `Manifest Controller` 會持續監控各節點本地的 `/var/lib/rancher/rke2/server/manifests/` 目錄。

* **配置衝突風險:** 若僅將檔案放置於單一 Master 節點（例如 Master A），當該節點發生故障、重啟或叢集進行版本升級時，其他沒有該檔案的 Master 節點（例如 Master B/C）在接管控制權時，可能會將該 Helm Chart 的設定還原為系統預設值，進而導致叢集配置來回覆蓋（打架）或非預期遺失。
* **高可用保障:** 三節點同步能確保不論哪一個 Master 節點當機，叢集的宣告式配置（Declarative Configuration）皆能保持一致。

### 3. 自動化運維建議 (GitOps / IaC)

為了落實「基礎設施即代碼 (IaC)」，強烈建議不要手動複製檔案，應採用以下自動化流程：

1. **版本控制:** 將所有的 `HelmChartConfig` YAML 檔案提交至 Git 儲存庫。
2. **組態管理工具:** 使用 Ansible、SaltStack 或 Puppet 等工具，將 Git 上的最新設定自動派發（Sync）至 3 台 Master 節點的 `/var/lib/rancher/rke2/server/manifests/` 目錄下。

---

## ❓ 常見問題與解答 (QA 集)

### Q1: 在 RKE2 三主節點(Master)環境中，HelmChartConfig YAML 檔案只需要放一台，還是三台都要放？

**A1:** **必須三台 Master 節點都要放置該檔案，且內容必須完全一致。** 因為 RKE2 的 Manifest Controller 是各節點獨立運作並監控本地 `/var/lib/rancher/rke2/server/manifests/` 目錄的。如果只放一台，當該節點重啟或故障時，其他沒有檔案的節點可能會將設定覆蓋回預設值，導致叢集狀態不穩定。

---

### Q2: 能否不將檔案放到 Master 節點的目錄下，直接在現有 RKE2 群集上執行 `kubectl apply -f config.yaml`？

**A2:** **可以，但此方式僅建議用於「運行時的臨時微調」，不建議作為生產環境的長期管理手段。**

* **運作原理:** 直接執行 `kubectl apply` 會將 `HelmChartConfig` 作為自訂資源 (CRD) 直接寫入 `etcd` 資料庫。RKE2 的 `helm-controller` 偵測到後一樣會觸發 Helm 升級更新。
* **缺點與風險:** 這種做法會脫離實體檔案的管理。未來如果發生災難復原、etcd 損壞重建，或是需要透過腳本自動化擴展新叢集時，這些直接 apply 的設定將會遺失。因此長期維護仍建議使用實體目錄派發法。

---

### Q3: 哪些 RKE2 配置修改「必須」使用目錄放置法，而不能只靠 `kubectl apply`？

**A3:** **叢集初始化階段（Bootstrap）就需要生效的設定，必須使用目錄放置法。**
例如：在叢集建立初期就要修改或停用預設的 CNI 網路套件（如 Cilium、Canal）或預設的 Nginx Ingress Controller。此時叢集尚未完全就緒，無法執行 `kubectl apply`，必須在啟動 RKE2 服務前，將 `HelmChartConfig` 預先放入 `manifests` 目錄中。

---

### Q4: 如果我先用 `kubectl apply` 修改了設定，後來又想改用官方建議的「目錄放置法」，需要注意什麼？

**A4:** **必須確保寫入實體目錄的 YAML 檔案內容，與目前 `etcd` 中的設定（即現行的配置）完全一致。**
當您將實體檔案放入 `/var/lib/rancher/rke2/server/manifests/` 後，RKE2 的 Manifest Controller 會以該「實體檔案」的內容為最終依歸。如果檔案內容與先前 `kubectl apply` 的內容不同，實體檔案的內容將會強行覆蓋掉 `etcd` 中的現狀。