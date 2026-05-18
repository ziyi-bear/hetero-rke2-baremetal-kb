# 📖 網路配置知識庫：Netplan 乙太網路與 Wi-Fi 共存設定指南

> **Metadata:**
> * **Tags:** `Linux`, `Ubuntu`, `Netplan`, `Network Configuration`, `DHCP`, `IPv4`, `IPv6`, `Wi-Fi`
> * **Target Interface:** `enp8s0` (Ethernet), `wlp9s0` (Wireless)
> * **Configuration File:** `/etc/netplan/50-cloud-init.yaml`
> 
> 

## 📄 1. 主文件 (Main Document)

### 1.1 系統與情境概述

本指南適用於基於 Ubuntu/Debian 系統且使用 Netplan 作為網路管理工具的設備（例如節點：`ziyi-bear@d1581with6600xt`）。情境涵蓋設備同時配備有線網路卡與無線網路卡，且需要兩者並存運行。

### 1.2 核心目標

1. 維持現有的 Wi-Fi (`wlp9s0`) 連線配置不變。
2. 啟動並設定有線乙太網卡 (`enp8s0`)。
3. 確保 `enp8s0` 能夠透過 DHCP 自動獲取 **IPv4** 與 **IPv6** 雙棧（Dual-stack）IP 位址。

### 1.3 網路拓撲與介面定義

* **`lo` (Loopback):** 本機迴環介面。
* **`enp8s0` (Ethernet):** 有線網卡，目標設定為 DHCPv4 與 DHCPv6 自動獲取。
* **`wlp9s0` (Wi-Fi):** 無線網卡，目標設定為維持 DHCPv4 及指定的 SSID (軼稟家族) 連線。

---

## 🛠️ 2. SOP 操作標準流程 (Standard Operating Procedure)

### Step 1: 編輯 Netplan 配置文件

使用具備 root 權限的文字編輯器開啟 Netplan 核心設定檔。

```bash
sudo nano /etc/netplan/50-cloud-init.yaml

```

### Step 2: 注入 YAML 網路配置

在 `version: 2` 層級下方，新增 `ethernets` 區塊，並與原有的 `wifis` 區塊保持平級。
**⚠️ 嚴重警告：** YAML 語法極度依賴正確的「空格」縮排，請勿使用 Tab 鍵。

```yaml
network:
  version: 2
  ethernets:
    enp8s0:
      dhcp4: true
      dhcp6: true
  wifis:
    wlp9s0:
      dhcp4: true
      access-points:
        "軼稟家族":
          auth:
            key-management: "psk"
            password: "12345678"

```

### Step 3: 測試配置 (安全防護)

在正式套用前，務必先測試配置檔的語法正確性，避免網路中斷導致設備失聯。

```bash
sudo netplan try

```

*執行後，系統會進行倒數。若無錯誤訊息且網路未斷線，請按 `Enter` 鍵確認保留配置。*

### Step 4: 驗證網路狀態

確認 `enp8s0` 網卡狀態已從 `DOWN` 轉為 `UP`，並成功取得 IP。

```bash
ip address show enp8s0

```

*預期結果：輸出結果中應包含 `inet` (代表 IPv4) 與 `inet6` (代表 IPv6) 的動態分配位址。*

---

## ❓ 3. QA 問答集 (Q&A for RAG)

**Q1: 如何在 Netplan 中讓特定網卡同時自動取得 IPv4 和 IPv6 位址？**
**A1:** 在 Netplan 的 `.yaml` 設定檔中，定位到該網卡（例如 `enp8s0`）的設定區塊，並同時將 `dhcp4: true` 與 `dhcp6: true` 寫入配置中即可。

**Q2: 為什麼修改完 Netplan YAML 檔後，執行套用會報錯？最常見的原因是什麼？**
**A2:** 最常見的原因是「縮排錯誤」。YAML 檔案嚴格要求層級之間的縮排必須一致，且**只能使用空白鍵（Space）**，絕對不能混用 Tab 鍵。

**Q3: 如果我透過 SSH 遠端修改網路配置，怕改錯導致斷線連不回去該怎麼辦？**
**A3:** 強烈建議使用 `sudo netplan try` 指令取代 `sudo netplan apply`。`try` 指令會套用新配置並開啟倒數計時（通常為 120 秒），如果在此期間內沒有按下 Enter 鍵確認（例如因為設定錯誤導致你斷線無法操作），時間一到，系統會自動還原回上一次正常的網路配置。

**Q4: 我要怎麼用指令快速查看 `enp8s0` 這張網卡現在的 IP 資訊？**
**A4:** 可以使用 `ip address show enp8s0` 或簡寫 `ip a s enp8s0` 來查看該網卡當下的實體狀態（UP/DOWN）以及所有分配到的 IPv4 與 IPv6 位址。