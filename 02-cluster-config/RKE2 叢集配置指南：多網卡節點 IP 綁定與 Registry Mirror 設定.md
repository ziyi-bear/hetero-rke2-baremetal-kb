# RKE2 叢集配置指南：多網卡節點 IP 綁定與 Registry Mirror 設定

**文件屬性 (Metadata)**

* **標籤 (Tags)**: RKE2, Kubernetes, Network Interface, Node-IP, Containerd, Registry Mirror, Agent Node, Master Node, Configuration
* **適用架構**: RKE2 多節點叢集 (Control-plane + Agent)
* **摘要**: 本文件說明如何解決 RKE2 節點多網卡導致的 IP 選擇錯誤問題，以及如何在多節點架構下正確佈署 Container Registry Mirror 配置。

---

## 一、 主文件 (Main Document)

### 1. 指定 RKE2 Agent 節點專屬網路介面 (Node-IP)

**【情境與問題】**
當 RKE2 節點具備多張實體或虛擬網卡（例如同時存在有線網卡 `enp8s0` 與無線網卡 `wlp9s0`）時，RKE2 初始化時可能會自動選擇錯誤的網路介面，導致 kubelet 路由、CNI 或叢集內部通訊異常。

**【解決方案】**
必須透過 `node-ip` 參數，將 RKE2 服務強制綁定在特定的網路介面 IP 上。

**【操作步驟】**

1. **編輯設定檔**：在該 Agent 節點上，編輯或建立 `/etc/rancher/rke2/config.yaml`。
2. **加入參數**：寫入 `server`、`token` 以及明確的 `node-ip`（以下以綁定 `enp8s0` 的 IP `192.168.80.27` 為例）：
```yaml
server: https://<MASTER_SERVER_IP>:9345
token: <YOUR_CLUSTER_TOKEN>
node-ip: 192.168.80.27

```


3. **重啟服務**：執行以下指令套用設定：

```bash
   sudo systemctl restart rke2-agent

```

4. **驗證結果**：回到 Master (Control-plane) 節點，執行 `kubectl get nodes -o wide`。檢查該節點的 `INTERNAL-IP` 欄位是否已正確顯示為 `192.168.80.27`。

> **注意**：若該網路介面使用 DHCP 動態分配 IP，當 IP 發生變更時，必須同步更新 `config.yaml` 內的 `node-ip` 並重啟服務。建議該網卡設定為靜態 IP 或在 Router 端綁定 MAC 地址。

---

### 2. 多節點叢集的 Registry Mirror 佈署範圍

**【情境與問題】**
為了加速映像檔拉取或符合內網安全規範，叢集需要配置私有倉庫或映像檔鏡像 (`registries.yaml`)。使用者常誤以為只需在 Master 節點設定即可。

**【核心架構原則】**
`registries.yaml` 必須存在於叢集內的**所有節點**（包含 1 個 Master 與 N 個 Agent 節點），且內容必須保持一致。

**【運作原理】**
RKE2 使用 `containerd` 作為各個節點底層的容器執行環境 (Container Runtime)。當 Kubernetes 排程器將 Pod 分派到特定的 Agent 節點時，是由**該節點本地端的 `containerd` 服務**負責對外拉取映像檔。
Agent 節點無法跨網路讀取或繼承 Master 節點上的 Registry 設定。若 Agent 節點本機缺少 `/etc/rancher/rke2/registries.yaml`，它將會繞過鏡像站，直接向預設的上游來源（如 Docker Hub）請求拉取。

**【操作步驟】**

1. 將正確的 `/etc/rancher/rke2/registries.yaml` 檔案複製並派發到所有 Master 與 Agent 節點。
2. **重啟 Master 節點服務**：
```bash
sudo systemctl restart rke2-server

```


3. **重啟 Agent 節點服務**：
```bash
sudo systemctl restart rke2-agent

```



---

## 二、 QA 問答 (Q&A)

**Q1: RKE2 節點有多張網卡（例如同時有 Wi-Fi 和實體線路）時，如何防止 RKE2 使用錯誤的網卡進行通訊？**
**A1**: 需要在節點的 `/etc/rancher/rke2/config.yaml` 檔案中，新增 `node-ip: <指定的網卡IP>` 參數。設定完成後重啟 `rke2-agent` (或 `rke2-server`) 服務，即可強制 RKE2 只透過該 IP 進行叢集通訊。

**Q2: 更改 RKE2 Agent 的 node-ip 後，如何確認設定是否成功生效？**
**A2**: 在 Master 節點上執行 `kubectl get nodes -o wide` 命令。觀察目標節點的 `INTERNAL-IP` 欄位，若顯示的 IP 與您在 `config.yaml` 中設定的 `node-ip` 相符，即代表設定成功。

**Q3: 在 RKE2 叢集中設定 Registry Mirror 時，`registries.yaml` 檔案只需要放在 Master 節點上嗎？**
**A3**: 錯誤。`registries.yaml` 檔案必須放置在叢集中的**每一個節點**上，包含所有的 Master 節點與 Agent 節點。

**Q4: 為什麼 RKE2 的 Agent 節點也需要自己獨立的 registries.yaml 設定檔？**
**A4**: 因為 Kubernetes 的映像檔拉取 (Image Pull) 動作是由實際運行 Pod 的該節點上的 `containerd` 獨立執行的。Agent 節點不會自動讀取 Master 節點的配置。如果 Agent 節點沒有自己的 `registries.yaml`，它會忽略鏡像設定並直接連線到預設的外部倉庫拉取映像檔。

**Q5: 新增或修改 `registries.yaml` 後，需要做什麼動作才會讓設定生效？**
**A5**: 必須重新啟動 RKE2 服務以觸發 `containerd` 重新讀取設定檔。若是 Master 節點，請執行 `sudo systemctl restart rke2-server`；若是 Agent 節點，請執行 `sudo systemctl restart rke2-agent`。