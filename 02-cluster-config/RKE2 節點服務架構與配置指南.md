# RKE2 節點服務架構與配置指南

## 1. 主文件 (Core Knowledge Document)

本文件說明 Rancher Kubernetes Engine 2 (RKE2) 在不同節點角色上的服務配置標準，以及控制平面與工作節點的架構差異。

### 1.1 RKE2 核心服務總覽

RKE2 叢集主要由兩種系統服務構成，分別對應不同的節點角色：

* **`rke2-server.service`**：專用於 Master 節點（控制平面 Control Plane）。
* **`rke2-agent.service`**：專用於 Worker 節點（純工作節點）。

### 1.2 Master 節點 (Server) 架構特性

* **運行服務**：Master 節點上**僅需且只能**啟動 `rke2-server.service`。
* **組件包含**：該服務負責啟動所有控制平面組件（如 `kube-apiserver`、`kube-scheduler`、`kube-controller-manager`、`etcd`）。
* **內建 Agent 功能**：`rke2-server` 已在底層內建並包含了 Agent 的功能（包含 `kubelet` 與 `kube-proxy`）。因此，預設情況下，Master 節點具備工作節點的特性，能夠被調度並運行一般的工作負載 (Workloads)。

### 1.3 Worker 節點 (Agent) 架構特性

* **運行服務**：純工作節點上僅啟動 `rke2-agent.service`。
* **組件包含**：不包含任何控制平面組件，僅負責啟動與運行容器相關的必要服務（`kubelet` 與 `kube-proxy`），並向 Server 進行註冊。

### 1.4 衝突警告與架構禁忌

* **嚴禁雙開服務**：絕對不可在同一個節點上同時啟動 `rke2-server.service` 與 `rke2-agent.service`。這會導致嚴重的資源與連接埠（Port）衝突，造成節點服務啟動失敗或叢集狀態異常。

### 1.5 節點調度管理 (Taint)

若架構需求為「Master 節點專職管理，不運行一般 Pod」，**不應**嘗試關閉內建的 Agent 功能，而是應該使用 Kubernetes 的 **Taint (污點)** 機制來限制調度。

* **配置指令範例**：
```bash
# 針對舊版或自定義標籤
kubectl taint nodes <node-name> node-role.kubernetes.io/master=true:NoSchedule

# 針對新版 Kubernetes 標準標籤
kubectl taint nodes <node-name> node-role.kubernetes.io/control-plane:NoSchedule

```



---

## 2. QA 類型 (Q&A Knowledge Base)

此區塊提供常見問題的標準解答，適合 RAG 系統直接檢索以回答使用者的自然語言提問。

**Q1: 在 RKE2 的叢集中，扮演 Master 的節點是否會同時啟動 rke2-agent.service 與 rke2-server.service？**
**A1:** 不會。Master 節點只會啟動 `rke2-server.service`，不會也不應該同時啟動 `rke2-agent.service`。

**Q2: 為什麼 RKE2 的 Master 節點預設可以運行一般的工作負載 (Pod)？**
**A2:** 因為 `rke2-server` 服務在架構上已經內建了 Agent 的功能。當啟動 server 服務時，它會在底層同時啟動 `kubelet` 和 `kube-proxy`，這使得 Master 節點預設也具備 Worker 節點的運算能力。

**Q3: 如果在同一個 RKE2 節點上同時啟動 rke2-server 和 rke2-agent 會發生什麼事？**
**A3:** 會發生嚴重的資源與連接埠（Port）衝突。例如，兩個程序會同時嘗試綁定 `kubelet` 的專用 Port，最終導致節點服務啟動崩潰或叢集狀態異常。

**Q4: 我希望我的 RKE2 Master 節點只做叢集管理，不要運行一般的 Pod，我應該去關閉它的 agent 服務嗎？**
**A4:** 不應該。因為 Master 節點的 agent 功能是包含在 `rke2-server` 服務內的，無法單獨關閉。正確的做法是透過 Kubernetes 給該 Master 節點打上 Taint（污點），例如執行 `kubectl taint nodes <node-name> node-role.kubernetes.io/control-plane:NoSchedule`，藉此阻止一般的 Pod 被調度到該節點上。

**Q5: rke2-agent.service 的主要作用是什麼？**
**A5:** `rke2-agent.service` 是專門為純工作節點（Worker Node）設計的服務。它負責啟動 `kubelet` 和 `kube-proxy`，讓節點具備運行容器的能力，並負責將該節點註冊到 RKE2 的控制平面中。