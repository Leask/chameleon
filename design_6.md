# Chameleon 終極工程規格與實作藍圖 v6.0
*(The Definitive Implementation Specification)*

## 0. 內化與共識：從架構哲學到工程現實

經過前面五個版本的激烈對抗與重構，我們已經徹底洗去了初期的天真與對「黑魔法技術」的迷信。我們在此確立 Chameleon 的核心靈魂與最終共識：

1. **系統的本質是狀態機與錯誤恢復**：Chameleon 不是一個「用 WebTransport 偽裝流量的工具」，而是一個「在惡劣公網中，能夠在設備遺失、節點被封、時鐘漂移、憑證過期等各種災難下，依然能精準降級、安全恢復的狀態機」。
2. **信任鏈的物理隔離**：根金鑰絕對離線。線上邊緣節點等同於「隨時會自毀的消耗品」，只持有 24 小時壽命的派生憑證。
3. **資料面與控制面的徹底解耦**：資料面（Forwarding Plane）只做無腦的高效搬運；控制面（Control Plane）負責下發策略（Policy）。
4. **多端一致性的基石是 ABI**：語言之爭沒有意義。核心業務邏輯必須以 Rust 實現，並透過嚴格定義的 C-FFI 邊界向外提供沒有歧義的接口，將記憶體管理與錯誤處理封裝在核心內。

---

## 1. 對 `design_5.md` 的嚴厲攻擊

`design_5.md` 提出了非常深刻的運維視角（如時鐘漂移、版本治理、六層架構），這是巨大的進步。但它犯了一個架構師最容易犯的錯誤：**「它是一份關於『我們應該寫什麼文檔』的文檔，而不是設計本身。」**

這導致 v5 陷入了嚴重的**分析癱瘓 (Analysis Paralysis)** 與架構過度設計：

### 1.1 `Policy Layer` 作為獨立運行層的災難
v5 試圖將 `Policy Layer` 變成一個獨立的運行層（Runtime Layer），介於 Local Interface 和 Session Core 之間。這是嚴重的效能殺手。
**攻擊：** 策略（Policy）不應該是一個在每次封包到達時都被呼叫的「模組」。如果 Forwarding Plane 每轉發一個 Stream 都要去問 Policy Layer「這個要重試嗎？」，系統延遲將徹底崩潰。策略應該是宣告式的配置，被「編譯」或「加載」到 Forwarding Plane 的快速查找表中。

### 1.2 提出問題，卻拒絕給出密碼學解答
v5 反覆強調「時鐘漂移 (Clock Drift)」和「憑據續期」會搞死系統，但卻沒有給出工程解法。
**攻擊：** 告訴工程師「要注意時鐘異常」是沒有意義的。我們必須直接在協議層設計容忍時鐘漂移的機制（例如引入邏輯時鐘 Epoch 或依賴握手時的 Server Timestamp），而不是把它留給「未來的 runbook」。

### 1.3 陷入「要求定義 ABI，但不寫 ABI」的虛無
v5 正確地指出了 FFI 邊界的重要性，然後就停止了。
**攻擊：** 既然這是一份落地設計，我們現在就必須定義出 ABI 的骨架。沒有具體的介面，多端開發就是一句空話。

---

## 2. Chameleon v6.0 實作規格 (Constructive Design)

v6 不再討論「應該怎麼設計」，而是直接給出「就是這樣實作」的規格。

### 2.1 修正後的架構：策略編譯模型 (Compiled Policy Model)

我們將 v5 的六層壓縮為**三個高內聚的執行域 (Execution Domains)**：

1. **Host App Domain (宿主域)**：
   * 包含 GUI、系統 VPN API 整合、Local Proxy 監聽。
   * 負責從作業系統獲取網路狀態（如 Wi-Fi 切換）。
2. **Core Domain (Rust FFI 核心域)**：
   * 包含 **Session State Machine** 與 **Forwarding Engine**。
   * **Policy 不再是獨立層**，而是由宿主域傳入的一段 JSON/Protobuf 配置。Core 解析後，將其轉換為 Forwarding Engine 內部的 Radix Tree 路由表和 QoS 參數。
3. **Control Plane Domain (雲端控制域)**：
   * 提供強簽名的配置下發、短效憑證簽發與遙測接收。

### 2.2 密碼學容錯：解決時鐘漂移與重放攻擊

v5 擔憂的時鐘漂移問題，透過 **Server-Assisted Epoch (伺服器輔助紀元)** 解決：

* **背景問題：** 設備離線太久，本地時間比真實時間慢了 3 天。此時如果依賴純本地時間檢查邊緣節點憑證，會認為合法憑證「已在未來發行」而拒絕連接。
* **v6 解法：**
  1. 客戶端發起 `ClientHello` 時，使用 Noise 協議的 ephemeral key 生成一次性 Nonce，不依賴時間。
  2. 邊緣節點在回覆 `ServerHello` 時，必須附帶由 Control Plane CA 簽名的 **目前網路絕對時間戳 (Signed Timestamp)**。
  3. 客戶端 Core 驗證該時間戳的簽名後，以此時間戳與本地時間計算出 `Clock_Offset`（時鐘偏移量）。
  4. 後續所有的憑證驗證、過期檢查，全部使用 `Local_Time + Clock_Offset`。完美解決 IoT 設備或休眠設備的時鐘漂移問題。

### 2.3 狀態機的錯誤分支與恢復路由 (Error Edges)

v5 要求定義錯誤狀態，v6 直接給出核心錯誤恢復矩陣：

| 當前狀態 | 觸發事件 / 錯誤 | 下一個狀態 | 核心動作 (Core Action) |
| :--- | :--- | :--- | :--- |
| `SESSION_AUTHENTICATING` | `ERR_CERT_EXPIRED` (節點憑證過期) | `CONFIG_RENEWAL` | 凍結資料面，向 Control Plane 請求最新節點清單。 |
| `SESSION_AUTHENTICATING` | `ERR_AUTH_REJECTED` (設備被撤銷) | `DEVICE_REVOKED` | 清空本地金鑰，向宿主域發送 `FATAL_REVOKED`，停止運行。 |
| `SESSION_ESTABLISHED` | 網路切換 (底層 IP 改變) | `SESSION_MIGRATING` | 暫停 Stream 排程，觸發 QUIC Connection Migration。0-RTT 發送探測包。 |
| `SESSION_MIGRATING` | `ERR_TIMEOUT` (遷移探測超時) | `SESSION_IDLE` | 丟棄 Session Key，重新發起完整的 1-RTT 握手。 |
| `CONFIG_RENEWAL` | `ERR_SIGNATURE_INVALID` (配置被污染) | `CONFIG_ROLLBACK` | 拒絕加載，退回上一份合法的本地快取配置。增加風險計數器。 |

### 2.4 C-FFI ABI 邊界定義 (The Rust Core Interface)

為了確保 iOS/Android/Desktop 邏輯絕對一致，Rust Core 必須導出極簡的 C ABI。所有的記憶體分配由 Rust 控制，對外只暴露 Handle (指標)。

```c
// Chameleon Core C-ABI (chameleon.h)

typedef void* ChameleonInstance;

// 錯誤碼矩陣 (對應狀態機的失敗模式)
typedef enum {
    CHAM_OK = 0,
    CHAM_ERR_CONFIG_INVALID_SIG = -1,
    CHAM_ERR_AUTH_REVOKED = -2,
    CHAM_ERR_NETWORK_UNREACHABLE = -3,
    // ...
} ChamError;

// 1. 初始化與配置下發 (包含 Policy)
ChamError cham_init(const char* signed_config_json, ChameleonInstance* out_instance);

// 2. 啟動資料面 (非阻塞)
ChamError cham_start(ChameleonInstance instance);

// 3. 宿主域通知網路環境改變 (如切換到 5G，觸發 SESSION_MIGRATING)
ChamError cham_notify_network_change(ChameleonInstance instance);

// 4. 停止與資源釋放
void cham_stop(ChameleonInstance instance);

// 5. 遙測指標獲取 (預設最小化)
// 返回一段 JSON，宿主域讀取後負責釋放字串記憶體
char* cham_get_minimal_metrics(ChameleonInstance instance);
void cham_free_string(char* str);
```
這套 ABI 確保了**語言邊界不會洩漏狀態**。宿主域（如 Swift / Kotlin）只需要呼叫 `cham_notify_network_change`，剩下的握手、重試、時鐘校正全部由 Rust 內部的狀態機自主完成。

### 2.5 版本治理與配置回滾 (Rollout & Versioning)

為解決 v5 提出的平滑升級與舊版本淘汰問題，v6 引入 **配置世代 (Configuration Epoch)** 與 **安全回落機制**。

* **配置 Schema (JSON)** 包含：`config_epoch` (整數遞增), `min_client_version`, `edge_nodes` (數組), `signature`。
* **版本攔截：** 如果客戶端解析配置發現自身版本低於 `min_client_version`，將拒絕連接邊緣節點，並向宿主域觸發 `REQUIRE_APP_UPDATE` 事件，引導用戶去應用商店更新，避免引發未知的協議解析崩潰。
* **容錯回滾 (Fallback Trigger)：** 如果加載新配置（例如 Epoch 42）後，連續 3 次連接邊緣節點失敗，Core 會自動將狀態標記為 `CONFIG_REJECTED`，並**無縫退回**記憶體中的上一份合法配置（Epoch 41），繼續嘗試連線，同時將異常指標記入遙測準備上報。

---

## 3. 下一步實作路徑

`design_6.md` 已經完成了從架構哲學到實作規範的跨越。所有的模糊地帶（時鐘、ABI、錯誤狀態、版本控制）都已給出具體的工程定義。

我們可以立即進入編碼階段，第一步的 PR 應該是：
**建立 Rust Workspace，定義 `chameleon-core` 的狀態機 Enum 與 C-FFI 接口，並編寫狀態跳轉的 Unit Test。**