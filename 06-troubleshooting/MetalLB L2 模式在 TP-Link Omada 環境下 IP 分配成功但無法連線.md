# [Troubleshooting] MetalLB L2 模式在 TP-Link Omada 環境下 IP 分配成功但無法連線

## 🏷️ Metadata
- **Tags:** Kubernetes, MetalLB, TP-Link Omada, L2 Mode, ARP, FluxCD, 網路連線問題
- **Component:** MetalLB (v0.13 或以上版本)
- **Environment:** 實體機叢集 (Bare-metal), TP-Link Omada 網路架構

---

## 📄 主文件 (Main Document)

### 問題現象 (Symptom)
在 Kubernetes 叢集中部署 MetalLB 並設定 `IPAddressPool` 後，出現以下情況：
1. Kubernetes 內部的服務已經成功獲取 IP。透過查看 `IPAddressPool` 的狀態，確認 `assignedIPv4` 數值大於 0（例如 `assignedIPv4: 1`）。
2. 外部網路無法連線/Ping 到該分配的 IP 位址。
3. 登入 TP-Link Omada Controller 查看 Log/Alerts 區塊，**沒有任何錯誤訊息或日誌**。

### 根本原因 (Root Cause)
這個問題起因於 MetalLB 新版架構設計以及 Layer 2 網路協定的特性：
1. **MetalLB 架構變更：** 自 MetalLB v0.13 版本開始，IP 的「池化分配 (Allocation)」與「網路廣播 (Advertisement)」被拆分為兩個獨立的自訂資源 (CRD)。僅部署 `IPAddressPool` 只會讓 MetalLB 內部將 IP 指派給 Service，但它**完全不會對實體網路發送任何封包**。
2. **Omada 無日誌的原因：** MetalLB 的 Layer 2 模式依賴向區域網路發送 Gratuitous ARP (GARP) 廣播來宣告 IP 所有權。由於尚未設定廣播，沒有封包產生，Omada 自然無紀錄；此外，即使正常發布 ARP，對 Omada 路由器 (如 ER605) 或交換機而言，ARP 廣播屬於正常的 L2 網路底層通訊，不會被視為系統事件或錯誤紀錄在 Controller 日誌中。

---

## 🛠️ 疑難排解標準作業程序 (Troubleshooting SOP)

### 解決方案：補充 L2Advertisement 資源

要讓實體網路知道該 IP 的存在，必須建立與 `IPAddressPool` 對應的 `L2Advertisement` 物件。

**步驟 1：確認原有的 IPAddressPool 名稱**
確認你已經套用的 `IPAddressPool` 名稱（範例中為 `ipaddresspool-basic`）。

**步驟 2：建立並套用 L2Advertisement YAML**
建立以下的 YAML 文件，確保 `spec.ipAddressPools` 的陣列中包含你的 `IPAddressPool` 名稱。若使用 FluxCD 等 GitOps 工具，請將此清單加入你的部署目錄中。

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement-basic
  namespace: metallb-system
  labels:
    # 替換為符合您 GitOps/專案環境的 Label
    kustomize.toolkit.fluxcd.io/name: cluster-management-cluster-init-deploy
    kustomize.toolkit.fluxcd.io/namespace: flux-system
spec:
  ipAddressPools:
  - ipaddresspool-basic

```

**步驟 3：驗證連線**
套用上述 YAML 後，MetalLB 會立即啟動 Speaker 節點發出 GARP 廣播，該 IP 位址即可被區網內的設備解析與連線。

---

## ❓ 常見問答 (Q&A)

**Q1: 為什麼 kubectl get svc 顯示 LoadBalancer IP 已經配發，卻還是連不上？**
**A1:** 在 MetalLB v0.13+ 之後，IP 配發與網路宣告是分離的。IP 成功配發只代表 Kubernetes 內部作業完成，若沒有建立 `L2Advertisement` 物件，MetalLB 就不會向外發送 ARP 廣播，實體網路中的設備（如路由器、交換機）就不會知道如何將流量路由到該節點。

**Q2: 為什麼路由器 (TP-Link Omada 等) 的日誌裡面完全沒有 MetalLB 的相關訊息或被阻擋的紀錄？**
**A2:** MetalLB 在 L2 模式下是不透過 DHCP 取得 IP，也不會呼叫路由器的 API，而是單純透過發送 ARP 封包來宣告 IP。ARP 屬於網路層最基礎的協定，對於 Omada 設備來說，這只是日常的網路底層雜訊，並不屬於需要記錄的系統事件、DHCP 請求或資安威脅，因此 Log/Alerts 中不會有紀錄。

**Q3: 升級 MetalLB 之後突然發生此問題，該如何處理？**
**A3:** 舊版 (v0.12 及更早) 的 MetalLB 使用 ConfigMap 同時管理 Pool 和宣告。升級到 v0.13+ 後，必須將原本的設定拆分為 `IPAddressPool` 與 `L2Advertisement` 兩個 CRD 物件才能正常運作。請補充遺漏的 `L2Advertisement` 即可恢復連線。