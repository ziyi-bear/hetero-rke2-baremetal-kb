# 設備評測文件：AMD RX 460 運行本地 AI 模型可行性報告

**標籤 (Tags)：** `Hardware Evaluation`、`AMD RX 460`、`LLM`、`Stable Diffusion`、`Vulkan`、`DirectML`、`Home-lab`
**文件狀態：** 技術驗證與硬體極限測試
**摘要：** 本文件探討 AMD Polaris 架構（RX 460，發布於 2016 年）在現代大型語言模型（如 Gemma、Qwen）與圖像生成模型（如 Stable Diffusion）上的執行可行性、軟體環境配置及硬體瓶頸。

## ▍ 主文件 (Main Document)

### 1. 核心硬體與軟體支援度評估

AMD RX 460 屬於較早期的 Polaris 架構，目前已**不被 AMD 官方機器學習運算平台 (ROCm) 支援**。因此，所有本地 AI 應用的部署皆無法依賴標準的 ROCm 加速，必須透過其他底層 API 進行任務卸載。

* **VRAM (顯示記憶體) 瓶頸：**
* **2GB 版本：** 完全不具備運行現代 LLM 或圖像生成模型的條件，極易觸發 OOM（Out of Memory）而被迫使用 CPU 運算。
* **4GB 版本：** 處於技術驗證的最低及格線，僅能勉強運行高度量化的微型語言模型或最基礎的生圖模型。



### 2. 大型語言模型 (LLM) 執行方案

在 RX 460 上執行 LLM 屬於技術驗證性質，無法提供流暢的生產力體驗。

* **API 依賴：** 必須依賴 **Vulkan** 或 **OpenCL** 後端框架（如 `llama.cpp` 或支援 Vulkan 的最新版 Ollama）。
* **模型相容性 (基於 4GB VRAM 版本)：**
* **Gemma 家族：** 支援 `Gemma 2B`（4-bit 量化，約佔 1.5GB~2GB VRAM）。不支援 `Gemma 7B`。
* **Qwen 家族：** 支援 `Qwen1.5 4B` 或 `Qwen2.5 1.5B / 3B`（4-bit 量化，約佔 1.5GB~3.5GB VRAM）。不支援 7B 以上版本。


* **效能表現：** 受限於舊架構算力，Tokens 生成速度偏慢，且必須嚴格限制上下文長度（Context Window）以防爆顯存。

### 3. 圖像生成 (Stable Diffusion) 執行方案

在 RX 460 上執行圖像生成面臨極大挑戰，體驗比執行 LLM 更差。

* **API 依賴：** 在 Windows 環境下，必須依賴 **DirectML** API。需使用特定軟體分支（如 AUTOMATIC1111 WebUI 的 DirectML 版本）。
* **模型相容性與限制：**
* **SD 1.5：** 4GB VRAM 版本有機率成功，但必須嚴格限制解析度於 `512x512`。
* **SD 2.1 / SDXL：** 完全不可行（VRAM 不足）。


* **系統優化參數：** 啟動服務時必須加入環境變數限制 VRAM 使用，例如：`--use-directml --medvram`（或 `--lowvram`） `--always-batch-cond-uncond --opt-split-attention-v1`。
* **效能表現：** 生成一張 512x512 (20 Steps) 的圖片，預估速度約為 `0.1~0.2 it/s`，單張耗時約 1.5 至 3 分鐘。

---

## ▍ 問答集 (Q&A)

**Q1: AMD RX 460 顯示卡可以使用官方的 ROCm 來跑 AI 模型嗎？**
**A1:** 不行。AMD 官方的 ROCm 平台已經不再支援 RX 460 的 Polaris 架構。必須改用 Vulkan (針對 LLM) 或 DirectML (針對圖像生成) 等替代 API 來驅動。

**Q2: 我的 RX 460 是 2GB 版本的，可以跑 Gemma 或 Qwen 嗎？**
**A2:** 無法。2GB 的 VRAM 甚至無法載入最基礎的模型層，絕大部分運算會被迫交給 CPU 和系統 RAM 處理，速度會慢到無法正常使用。

**Q3: 如果有 4GB 的 RX 460，可以跑哪些具體的 LLM 模型？**
**A3:** 搭配 Ollama 或 llama.cpp 並強制啟用 Vulkan 加速後，可以執行經過 4-bit 高度量化（GGUF）的微型模型，例如 `Gemma 2B`、`Qwen2.5 3B` 或是 `Qwen1.5 4B`。模型參數若超過 7B 則會超出 VRAM 限制。

**Q4: 可以用 RX 460 (4GB) 來跑 Stable Diffusion 產生圖片嗎？**
**A4:** 可以，但限制極大且速度非常緩慢。你只能使用 Stable Diffusion 1.5 的基礎模型，且必須將解析度嚴格限制在 `512x512`。同時，必須使用支援 DirectML 的生圖前端軟體，並開啟 `--medvram` 或 `--lowvram` 等顯存優化參數。

**Q5: RX 460 跑 Stable Diffusion 1.5 的產圖速度大約多快？**
**A5:** 在 4GB VRAM 版本搭配 DirectML 與顯存優化設定下，生成一張 512x512、20 步 (Steps) 的基礎圖片，預期速度約為每秒 0.1~0.2 Iterations。這表示產生單張圖片需要花費大約 1.5 到 3 分鐘。

**Q6: RX 460 適合用來建置本地 RAG 系統或生圖工作站嗎？**
**A6:** 不適合。RX 460 僅適合用於 Home-lab 的「技術可行性驗證」實驗。由於算力與 VRAM 的雙重瓶頸，它無法提供流暢的 RAG 檢索生成體驗，也無法滿足反覆調試提示詞的生圖需求。若需實際應用，建議升級硬體。