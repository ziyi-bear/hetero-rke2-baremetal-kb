# Kubernetes GPU 配置指南：Intel GPU 與 NVIDIA GPU 的架構差異與調度實踐

本文件旨在說明在 Kubernetes (K8s) 環境中，Intel GPU 與 NVIDIA GPU 在硬體調度、驅動架構以及 Pod 配置上的本質差異。文件結構採結構化設計，適合做為 RAG (檢索增強生成) 系統的知識庫文本。

---

## 核心概念與架構對比 (Main Guide)

### 1. 驅動程式與 Linux 核心的關係
GPU 在 K8s 中的配置方式，取決於其驅動程式在 Linux 核心中的定位：

* **Intel GPU (與 AMD GPU)：** 採用 **In-Tree（內建）** 核心驅動（如 `i915` 或較新的 `xe`）。由於 Linux 核心原生支援，系統啟動時會直接在宿主機生成標準的 Linux 字元設備節點 `/dev/dri/`（例如 `card0`、`renderD128`）。標準容器運行時（如 `runc`）原生理解此類設備，因此可直接透過掛載設備的方式讓容器存取。
* **NVIDIA GPU：** 採用 **Out-of-Tree（外掛/專有）** 驅動。Linux 核心預設不包含 NVIDIA 的專有技術。容器若要使用 NVIDIA GPU，除了需要設備節點（`/dev/nvidia0`）外，還極度依賴一整套龐大的使用者空間專有函式庫（如 `libcuda.so`、`libnvidia-ml.so` 等）。

### 2. 容器運行時工具箱的「自動化魔術」
在 K8s 中，使用 `nvidia.com/gpu: 1` 看似不需要手動掛載設備，這並非因為 NVIDIA 不需要設備路徑，而是因為 **NVIDIA Container Toolkit** 在背後完成了自動化隱藏操作：
1. 當 Pod 宣告 `resources.limits.nvidia.com/gpu` 時，K8s 設備插件會攔截此請求。
2. 修改後的容器運行時（如調用過 NVIDIA 鉤子的 `containerd` 或 `cri-o`）會在容器啟動前，**自動將宿主機的設備節點與幾十個驅動函式庫（.so 檔案）綁定掛載（Bind-Mount）** 進入容器內部。

### 3. Intel GPU 的兩種部署路徑
因為 Intel 驅動的原生性，開發者在 K8s 中有兩種完全不同的配置選擇：

#### 模式 A：手動設備掛載 (Manual Device Mount)
* **作法：** 直接在 Pod YAML 中使用 `hostPath` 或 `securityContext` 將 `/dev/dri` 對應進容器。
* **優點：** 極度簡單，不需要在叢集中安裝任何額外的 DaemonSet 或插件，適用於單節點、測試環境或自建機房（Homelab）。
* **缺點：** K8s 調度器（Scheduler）無法感知 GPU 的使用狀況。多個 Pod 可以同時掛載同一個設備，可能導致資源搶奪與效能衝突，無法做到叢集整體的資源管理。

#### 模式 B：聲明式資源限制 (Resource Limits)
* **作法：** 安裝 **Intel Device Plugins for Kubernetes**，並在 Pod 中宣告 `gpu.intel.com/i915: 1`。
* **優點：** 納入 K8s 原生調度體系。調度器能精確掌握哪台節點還有剩餘的 GPU，實現真正的多租戶隔離與排隊機制。
* **缺點：** 需要維護額外的叢集插件（Operator / Helm Chart）。

---

## 常見問答集 (Q&A Section)

### Q1：為什麼大家常說 Intel GPU 需要手動掛載設備，而 NVIDIA 只需要寫 resource limit？
**A1：** 這是因為**維護成本與架構便利性**的差異。
* NVIDIA 的專有驅動結構複雜，不透過 Toolkit 自動掛載根本無法在容器內跑起 CUDA，因此使用者「被迫」必須安裝插件並使用 resource limit。
* Intel GPU 因為驅動內建於 Linux 核心，直接掛載 `/dev/dri` 就能直接動。許多開發者為了省去安裝與維護 Intel Device Plugin 的麻煩，選擇了最簡單的手動掛載路徑，因而造成了「Intel 只能/必須手動掛載」的刻板印象。

### Q2：如果我不安裝 Intel Device Plugin，直接在 YAML 中寫 `gpu.intel.com/i915: 1` 會怎樣？
**A2：** Pod 將會**永遠處於 Pending（暫停）狀態**。
因為原生 K8s 調度器只認識 `cpu` 與 `memory` 等標準資源。如果沒有安裝 Intel 設備插件向 Kubelet 註冊該自定義資源（Extended Resource），調度器會判定叢集中「沒有任何節點滿足此硬體需求」，並回報 `FailedScheduling`（Insufficient gpu.intel.com/i915）錯誤。

### Q3：當我成功使用 resource limit 掛載 Intel GPU 後，跟 NVIDIA 相比還有什麼潛在的「坑」？
**A3：** 核心區別在於**使用者空間函式庫（User-Space Libraries）的打包責任**：
* **NVIDIA 模式：** 插件會自動把宿主機的專有驅動函式庫（如 `libcuda.so`）注入到容器內。容器內的應用程式不需要自備驅動，只要有 CUDA Toolkit 即可。
* **Intel 模式：** Intel Device Plugin **只負責自動掛載設備節點 (`/dev/dri`)**，它**不會**幫你從宿主機複製驅動函式庫。因此，你的**容器映像檔（Container Image）內部必須事先安裝好對應的使用者空間運算函式庫**（例如：`intel-opencl-icd`、`mesa-va-drivers` 或 `intel-media-va-driver`），否則程式會因為找不到硬體加速驅動而報錯。

### Q4：在生產環境（Production）中，推薦使用哪種方式配置 Intel GPU？
**A4：** 強烈推薦使用 **模式 B：安裝 Intel Device Plugin 並使用 Resource Limits**。
原因如下：
1. **避免資源衝突：** 防止多個業務 Pod 在不知情的狀況下擠在同一個 GPU 上。
2. **叢集彈性擴展：** 配合 Cluster Autoscaler 時，當 GPU 資源不足，K8s 可以自動開起帶有 Intel GPU 的新節點。
3. **安全性：** 手動掛載設備通常需要給予容器較高的權限（如 `privileged: true` 或特定的 Linux Capabilities），這會帶來資安漏洞；使用設備插件則能維持標準的最小權限原則。
