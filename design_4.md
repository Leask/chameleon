# Chameleon 工程規格與核心狀態機藍圖 v4.0
*(Engineering Blueprint & State Machine Specification)*

## 0. 文檔定位與 v3 共識繼承

經過 `design_3.md` 的嚴厲拆解，我們已經徹底放棄了「完美偽裝」、「單一神級承載」、「無狀態根金鑰邊緣節點」以及「靠隨機擾動隱身」等不切實際的幻想。

`design_4.md` 不再進行哲學層面的辯論。這是一份**工程落地指南**。
本文件直接實作 v3 結尾要求的三大核心定義：**威脅模型 (Threat Model)**、**控制面信任鏈 (Trust Chain)** 以及 **會話狀態機 (Session State Machine)**。

如果說 v3 是一份「不要做什麼」的架構憲法，那麼 v4 就是「具體要怎麼寫代碼」的施工圖紙。

---

## 1. 具象化威脅模型 (Formal Threat Model)

我們不再用模糊的「高壓環境」來形容對手，而是明確定義對手能力邊界與系統的妥協點（Concessions）。

### 1.1 系統防禦目標 (In-Scope Defenses)
*   **抗深度封包檢測 (DPI Resistance):** 確保傳輸載荷具有極強的隨機性（經 AEAD 加密），且握手過程不包含任何明文的 Chameleon 專屬特徵字串。
*   **抗主動差分探測 (Active Probing Resistance):** 邊緣節點面對非授權的 TCP/UDP 探測、畸形 Client Hello 或半開連線時，行為必須與一個標準的、無特權的 Nginx/Caddy 伺服器在統計學上不可區分。
*   **前向保密與邊緣失陷控制 (PFS & Compromise Containment):** 邊緣節點（Edge Node）被物理拔線或取得 Root 權限後，**絕對不能**解密過去的歷史流量，也**不能**偽造控制面指令，且其影響半徑必須在 24 小時內自動失效。
*   **抗重放攻擊 (Replay Attack Resistance):** 攔截並重放過去的合法連線封包，不能導致伺服器執行重複的授權或資源消耗。

### 1.2 系統妥協點 (Explicit Concessions)
*   **放棄「指紋隱身」幻想:** 我們承認無法 100% 偽裝成 Chrome。我們的策略是「**隱匿於人群 (Hide in the crowd)**」——直接編譯使用未經魔改的標準 `quic-go` 或 `rustls` 庫，讓我們的指紋等同於「一個普通的 Go/Rust 網路應用」，而不是費力去偽裝成一個破綻百出的假 iOS 設備。
*   **放棄全局防關聯 (No Global Anonymity):** 系統不防禦具備國家級全局視野的流量關聯分析（例如跨國 ISP 節點間的流量進出時間差關聯）。
*   **放棄單點永不被封 (Assume Ephemeral Edges):** 我們預設邊緣節點的 IP 會被封鎖。系統的存活性依賴於**控制面的快速切換能力**，而非單一節點的無敵防護。

---

## 2. 控制面與分層信任鏈 (Control Plane & Trust Chain)

為了解決 v2 中「伺服器持有根金鑰」的災難性設計，v4 實作嚴格的 PKI 分層信任模型。

### 2.1 密鑰分層架構
1.  **離線根憑證 (Offline Root CA):**
    *   僅存在於冷錢包或實體硬體安全模組 (HSM) 中。
    *   唯一職責：簽發「控制面中介憑證 (Control Plane Intermediate CA)」與更新客戶端的信任根。
2.  **控制面簽發中心 (Control Plane CA):**
    *   部署於受嚴格保護的高安全區。
    *   職責：自動為全球的「邊緣轉發節點 (Edge Nodes)」簽發短期工作憑證 (Short-lived Edge Certs)。
3.  **邊緣節點短期憑據 (Ephemeral Edge Credentials):**
    *   線上邊緣節點**只持有**有效期極短（例如 12 ~ 24 小時）的 TLS 憑證與對應私鑰。
    *   如果節點失陷，攻擊者拿到的私鑰在幾小時後即成廢紙，且無法用來簽發新節點。

### 2.2 設備身份與配置下發
*   **設備私鑰 (Device Key):** 每個客戶端在安裝時生成本地 Ed25519 密鑰對，這是客戶端的「長期身份」。
*   **配置更新 (Config Provisioning):** 客戶端定期向控制面拉取最新的邊緣節點清單與路由策略。此配置清單必須由 Control Plane CA 進行 **Ed25519 簽名**。客戶端拒絕任何未簽名或簽名過期的節點清單，徹底阻斷 DNS 劫持或中間人下發惡意節點的可能。

---

## 3. 安全會話狀態機 (Session State Machine)

這是 Chameleon 的安全心臟 (Session Core)。我們嚴格剝離了「0-RTT」與「身份授權」的關係。

### 3.1 狀態定義
*   `STATE_IDLE`: 初始狀態，無安全上下文。
*   `STATE_HANDSHAKING`: 正在進行 1-RTT 的密鑰交換與身份驗證（使用 Noise 協議或 TLS 1.3 mTLS）。
*   `STATE_AUTHORIZED`: 會話建立成功，生成了雙向的 Session Keys。
*   `STATE_FORWARDING`: 資料面接管，處理多路復用 Stream。
*   `STATE_REKEYING`: 觸發金鑰輪換（無縫狀態）。

### 3.2 0-RTT 的嚴格限制
在 v4 中，**0-RTT 絕對不允許用於初始身份驗證或攜帶新的代理請求 (如新的 DNS 查詢或 TCP Connect)**。
*   **合法場景:** 0-RTT 僅限於 QUIC 的 `Connection Migration`（例如手機從 Wi-Fi 切換到 4G）。此時底層 IP 改變，但客戶端使用之前 `STATE_AUTHORIZED` 階段商議好的 Session Key 直接加密封包。
*   **安全邊界:** 即使這些封包被重放，轉發引擎 (Forwarding Engine) 的滑動窗口防重放機制也會直接丟棄 Sequence Number 已經處理過的封包，不會產生任何副作用。

---

## 4. 傳輸承載矩陣與資料面 (Transport & Forwarding)

傳輸適配器 (Transport Adapter) 退化為純粹的字節搬運工。

### 4.1 承載能力矩陣 (Adapter Matrix)

| 承載類型 (Adapter) | 建連成本 | 多路復用 | 切網恢復 | 行動端耗電 | 適用場景 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **QUIC (HTTP/3)** | 1-RTT | 原生極佳 | 原生支援 | 中/高 | 預設首選，適合高並發網頁瀏覽 |
| **WebSocket + TLS**| 2-3 RTT | 依賴應用層 | 斷線重連 | 低 | 保底方案，適合嚴格限制 UDP 的網路 |
| **WebTransport** | 1-RTT | 原生極佳 | 原生支援 | 中 | 未來擴展，替代 WS |

### 4.2 轉發引擎排程 (Forwarding Engine Scheduling)
不再依賴應用層 BBR，轉發引擎專注於**佇列管理**與**優先級排程**：
1.  **控制流絕對優先:** DNS 請求與會話管理指令 (Ping/Rekey) 擁有最高優先級 (Priority 0)。
2.  **互動流優先:** 小於 1MB 的 HTTP 請求 (如 HTML/JS/CSS) 分配為 Priority 1。
3.  **大流量退讓:** 大尺寸文件下載與視訊串流 (Priority 2) 在頻寬擁擠時必須讓渡資源給 Priority 1，確保網頁「秒開」的體感。
4.  **Bufferbloat 控制:** 轉發引擎的 Send Buffer 實作 CoDel (Controlled Delay) 算法。當佇列延遲超過 20ms 時主動丟棄部分封包（或通知底層 QUIC 減速），保持低延遲。

---

## 5. 暴露面管理 (Surface Management & Padding)

不再使用全域的隨機擾動，而是實施**有限預算的特徵收斂**。

### 5.1 封包大小裝箱 (Packet Size Bucketing)
為了解決封包大小被精準識別的問題，我們不隨機 Padding，而是將上行封包大小「裝箱」到幾個固定的 Bucket 中（例如：128B, 512B, 1024B, 1350B）。
*   **機制:** 如果真實載荷是 400B，傳輸層自動 Padding 到 512B。
*   **優勢:** 在 DPI 設備眼中，Chameleon 只有四種封包大小。這極大地破壞了透過特定封包大小推測應用類型（如 SSH 擊鍵、特定圖片加載）的機器學習模型，同時將帶寬浪費控制在數學期望的 10%~15% 內（成本可控）。

### 5.2 錯誤語義一致性 (Error Semantic Consistency)
當邊緣節點收到探測時，如果無法通過 Session Core 的認證，它的行為必須與標準伺服器一致：
*   如果收到隨機 UDP 垃圾：直接丟棄，不回覆任何 ICMP 錯誤。
*   如果收到非 Chameleon 的正常 HTTPS 請求：將請求移交給本機的 Caddy，返回真實的誘餌網站頁面。不引發 TCP RST，不引發 TLS Alert。

---

## 6. 多端落地的工程路徑

### 6.1 核心庫 (Core Library)
使用 Rust 編寫 Chameleon 的 `Session Core` 與 `Forwarding Engine`，編譯為 C-FFI / WebAssembly。
*   **原因:** 確保狀態機邏輯在全平台（Windows, macOS, iOS, Android, Linux）絕對一致，且無垃圾回收 (GC) 帶來的延遲抖動。

### 6.2 接入層 (Local Interface)
*   **桌面端 (Desktop):** 提供 HTTP/SOCKS5 本地監聽埠，利用作業系統 API 設置系統代理。
*   **行動端 (Mobile):** 將 Rust Core 打包為 iOS `NetworkExtension` 與 Android `VpnService`。
*   **無頭模式 (CLI):** 直接編譯為靜態二進位檔，接受 YAML 配置，支援 Prometheus 格式的 Metrics 暴露，供營運監控使用。

## 7. 總結

Chameleon v4 徹底完成了從「駭客玩具」到「工程級基礎設施」的轉變。它承認了網路對抗的殘酷性，將防禦重心從「花俏的流量偽裝」轉移到了「堅不可摧的密碼學狀態機」、「分層受控的信任鏈」以及「穩定的多端傳輸排程」上。

這份藍圖已經具備了直接轉化為代碼架構庫 (Repository Structure) 的成熟度。