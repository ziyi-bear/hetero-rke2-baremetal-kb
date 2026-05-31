# [RAG 知識庫] 單節點 Kubernetes 叢集：Service 內部網路連線限制與架構

**文件屬性 (Metadata):**

* **類別:** 系統架構、網路 (Networking)
* **關鍵字:** Kubernetes, K8s, Single-node (單節點), Service, 網路頻寬 (Network bandwidth), 網卡限制 (NIC), CNI, kube-proxy, veth
* **適用情境:** 協助解答關於 K8s 單節點叢集內部 Pod 與 Service 互相通訊時的網路效能與實體硬體限制之關聯。

---

## 📄 主文件 (Main Document)

### 1. 核心觀念：實體網卡不限制內部流量

在單節點（Single-node）Kubernetes 叢集中，當兩個 Service（及其後端 Pod）透過 K8s Service Name 互相連線時，其網路傳輸速度 **不受限於機器的實體網卡頻寬（例如 1 Gbps 的實體網卡限制）**。

這是因為這兩個 Service 都運行在同一台實體機（或虛擬機）上，它們之間的通訊完全在作業系統的記憶體與核心（Kernel）網路堆疊中進行，封包永遠不會離開伺服器去觸碰外部的實體網路交換機。

### 2. 單節點 Service 通訊的網路流量路徑

當 Service A 的 Pod 嘗試透過 Service Name 與 Service B 的 Pod 通訊時，底層的網路封包流向如下：

1. **封包離開 Pod A:** 封包透過 Pod A 的虛擬乙太網路介面（`veth` pair）送出。
2. **進入 Node 的網路命名空間:** 封包到達 Node 的內部網路橋接器（通常是 `cni0` 或類似的虛擬 bridge）。
3. **kube-proxy 攔截與位址轉換 (NAT):** Kubernetes 的 `kube-proxy`（底層通常使用 `iptables` 或 `IPVS` 技術）會攔截該封包，解析出 Service IP，並執行網路位址轉換（NAT），將目的地改寫為 Service B 背後實際的 Pod IP。
4. **封包抵達 Pod B:** 封包沿著目標 Pod B 的 `veth` pair 進入其內部。
*結論：整個過程皆由 Linux Kernel 的虛擬網路處理，完全繞過實體網卡 (NIC)。*

### 3. 真正的效能瓶頸與決定因素

既然不受限於實體網卡，單節點 K8s 內部網路連線的極限實際上取決於 **CPU 運算能力與記憶體頻寬**。處理每個虛擬封包都需要消耗 CPU 資源（包含封裝、路由、iptables 規則比對、Context Switch 等）。

影響單節點內部傳輸極限的主要因素包含：

* **CPU 處理能力 (CPU Power):** 現代多核心伺服器 CPU 通常可輕鬆處理 **10 Gbps 到 40+ Gbps** 的節點內虛擬網路流量。
* **CNI 的選擇與架構 (Container Network Interface):**
* 傳統基於 `iptables` 的 CNI（如 Flannel 或基礎設定的 Calico）會有較高的 CPU 負載。
* 基於 eBPF 技術的 CNI（如 Cilium）能繞過繁瑣的 iptables 路由，以更低的 CPU 消耗達到更高的傳輸速度。


* **最大傳輸單元 (MTU) 與封包大小:** 傳送大封包（Payload）的整體頻寬吞吐量會高於傳送數百萬個微小封包，因為大封包能減少每位元組的 CPU 處理負擔。

---

## ❓ QA (問答集)

**Q1: 我的 Kubernetes 單節點叢集實體網卡只有 1 Gbps，兩個內部 Service 互相連線的速度最高只能達到 1 Gbps 嗎？**
**A1:** 不是。內部 Service 之間的通訊速度會遠大於 1 Gbps。因為在單一節點上，Pod 對 Pod 的連線完全透過 Linux Kernel 的虛擬網路堆疊（如 `veth` pair 與 iptables/IPVS）在記憶體中進行，並不會經過實體網卡，因此不受 1 Gbps 實體硬體的限制。

**Q2: 為什麼單節點的 Kubernetes Service 互通不需要經過實體網卡？**
**A2:** 因為發送端與接收端的 Pod 都位於同一個作業系統（Node）內。當發送端將封包送出後，系統核心內的 `kube-proxy` 會直接將目標 Service IP 轉換為本地端的目標 Pod IP，然後透過內部的虛擬橋接器（Bridge）直接將封包送達目的地，這個過程不涉及任何外部的物理網路設備。

**Q3: 如果不是實體網卡，那麼單節點 K8s 內部網路連線的極限實際上受什麼限制？**
**A3:** 實際的網路極限受限於「CPU 處理能力」與「記憶體頻寬」。處理虛擬網路封包（如封裝、解封裝、路由判斷、NAT 轉換）需要消耗 CPU 週期。現代伺服器通常能達到 10 Gbps 到 40 Gbps 以上的內部吞吐量。

**Q4: 我可以如何提升單節點內 K8s Service 的連線頻寬或降低網路延遲？**
**A4:** 可以透過以下方式優化：

1. 確保伺服器有足夠的 CPU 資源不被應用程式完全佔滿，保留餘裕給網路堆疊處理封包。
2. 更換高效能的 CNI 外掛，例如從傳統 `iptables` 驅動的 CNI 升級為使用 eBPF 技術的 CNI（如 Cilium），可大幅降低 CPU 處理網路的額外開銷 (Overhead)。
3. 在應用程式層面，盡量增加單次傳輸的 Payload 大小，減少細碎的微小封包，以降低封包處理的頻率。