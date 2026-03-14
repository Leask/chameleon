# Chameleon 核心協議與線上規格白皮書 v8.0
*(Wire Format & Protocol Core Specification)*

## 0. 設計哲學與架構演進

### 0.1 專案設計目標與核心思想
Chameleon 的核心目標是構建一個**在極端惡劣與高壓干預的公網環境中，依然能夠穩定、安全、高效轉發流量的覆蓋網路 (Overlay Network)**。
在經歷了多個版本的推演與重構後，我們徹底放棄了「依靠巧妙偽裝（Camouflage）或隨機擾動來逃避審查」的天真幻想，並確立了以下不可動搖的設計思想：

1.  **安全建立在密碼學與狀態機上，而非「偽裝」 (Cryptography Over Obscurity)**：我們承認不存在完美的流量隱形。系統的存活性不依賴「把流量包裝得像別的協議」，而是依賴嚴格的 Noise 密碼學握手、明確的會話邊界、以及無懈可擊的狀態轉換。我們透過 AEAD 和帶外信任分發（OOB Trust），確保協議在數學層面上不可被探測和重放。
2.  **密碼學證明的零信任架構 (Zero-Trust Architecture)**：邊緣節點預設為不可信且隨時會失陷的消耗品。控制面信任鏈必須進行物理隔離（離線 Root CA），線上節點僅持有極短效期的派生憑證，確保單點失陷不會導致歷史流量解密或控制面信任崩潰。
3.  **防主動探測的絕對靜默 (Silent Drop under Active Probing)**：面對任何未經授權的連線嘗試、畸形探測或重放攻擊，伺服器在密碼學驗證（AEAD MAC 驗證）失敗後，必須保持**絕對靜默**，絕不在網路上暴露出任何協議專屬的錯誤信號或行為相變。在特定的部署拓撲下（如 TCP 承載），伺服器可選擇將無效流量無縫轉交 (Fallback) 給本機真實的 Web 服務，但這被視為**可選的部署策略**，而非協議本身的強制保證。
4.  **極致效能與強健可用性的統一 (Extreme Efficiency & Robust Availability)**：我們對協議的傳輸速度、解析效率與低延遲有著極致的追求。這種高效不來自於無腦搶佔頻寬，而是來自於極低開銷的 TLV 幀結構、以及無贅餘的密碼學握手。同時，協議必須具備超強的健壯性（Robustness），透過精準的狀態機快速自癒。需要注意的是，諸如「消除隊頭阻塞」或「無縫 IP 切換」等特性，本質上**繼承自底層承載 (如 QUIC)**，Chameleon 協議保證的是**不破壞**這些底層能力，並在其之上提供會話與通道管理。
5.  **語言與平台的徹底解耦 (Language-Agnostic Protocol)**：Chameleon 的核心是一套嚴謹定義的網路傳輸二進位規格 (Wire Format)，而非綁定特定程式語言的軟體實作。協議將底層視為不可靠的位元組流承載，在協議層內部提供端到端的安全與多路復用控制。

### 0.2 本版架構演進與邊界確立 (v8.1)
在吸收了最新一輪的聯合審查 (Review) 後，Chameleon 的設計文檔 (`design.md`) 與協議規格 (`protocol.md`) 進行了徹底的職責分離：
*   **`design.md`**：專注於設計哲學、威脅模型、安全邊界與架構取捨 (Trade-offs)。不再包含任何具體的位元組佈局、長度常數或密碼學參數。
*   **`protocol.md`**：作為唯一的 Single Source of Truth，負責定義精確的線上格式 (Wire Format)、狀態機跳轉與二進位欄位。

本版本的核心架構確立與取捨在於：
1.  **放棄多流映射，接受單一流帶來的 HOL 代價 (Simplicity over Multi-stream HOL Avoidance)**：我們明確定義 Chameleon Session 運行在**單一**的可靠位元組流 (Single Reliable Byte Stream) 上。這意味著即使底層是 QUIC，我們也只使用一條 QUIC Stream 承載整個 Session 內的所有多路復用 Channel。我們充分意識到這會**重新引入隊頭阻塞 (HOL)** 的風險。但這是一個深思熟慮的架構取捨：為了維持極度嚴格的密碼學鎖步 (Strict Cryptographic Lockstep) 與防重放機制，我們選擇了協議狀態機的絕對簡潔性與安全性，放棄了複雜且容易出錯的跨流狀態同步。
2.  **單向信任與零特徵握手**：明確 Chameleon 的安全邊界是「客戶端單向預知伺服器靜態公鑰」以實現零特徵探測防護。具體的 Noise Pattern (如 `NK`) 與固定長度握手的實作細節，全權交由 `protocol.md` 定義。
3.  **澄清 Fallback 的邊界**：承認 Fallback (流量轉交) 高度依賴底層承載特性（如 TCP socket 移交），將其明確降級為「部署層的可選策略」，而非協議核心的強制行為。協議只保證 Pre-Auth 失敗時的絕對靜默 (Silent Drop)。
4.  **紀元憑證 (Epoch Cert) 的語義收斂**：明確紀元憑證僅用於提供防重放的**時間新鮮度視窗 (Freshness Window)**，不再承擔解決客戶端時鐘嚴重漂移的「時鐘同步」責任。
5.  **Datagram 的未來擴展**：架構上保留對 UDP/Datagram 代理的支援，但在當前穩定版本的規格中，將其標記為保留能力 (Reserved Capability)，優先確保 TCP Stream 代理與流控的絕對穩定。

---

## 1. 架構模型與信任邊界 (Architecture & Trust Boundaries)

Chameleon 採用嚴格的控制面與資料面分離架構。

### 1.1 控制面 (Control Plane)
*   **職責**：節點發現、憑證簽發、路由策略下發、版本治理。
*   **信任假設**：Root CA 絕對離線。線上 Control Plane API 預設處於高風險暴露狀態，必須透過強簽名 (如 Ed25519) 保護下發的配置與 Epoch Cert。

### 1.2 資料面 (Data Plane / Forwarding Engine)
*   **職責**：加密、多路復用、流控、錯誤恢復。
*   **信任假設**：邊緣節點 (Edge Nodes) 預設不可信且短命。線上節點僅持有短期流量金鑰與限時的 Epoch Cert，絕對無法篡改控制面指令或解密歷史流量。

---

## 2. 錯誤處理與暴露面策略 (Error Handling & Surface Management)

Chameleon 對錯誤處理採取極端保守的態度，將錯誤嚴格劃分為兩個階段，這構成了系統存活性的核心防線：

1.  **握手與認證前 (Pre-Auth)**：面對探測器、重放攻擊或版本不符，伺服器在密碼學驗證通過前，**絕對靜默 (Silent Drop)**。這使得 Chameleon 伺服器在互聯網掃描器眼中，與一個超時斷線的死節點或普通的 Web 伺服器毫無二致。
2.  **認證後與加密傳輸中 (Post-Auth)**：一旦密碼學握手成功，雙方進入加密的鎖步狀態 (Lockstep)。此時的任何 MAC 驗證失敗、序號錯亂，均視為連線被中間人污染，觸發**硬關閉 (Hard Close)**，絕不嘗試在被污染的流上進行局部恢復。只有協議內部的邏輯錯誤（如未知的 Channel ID），才會發送加密的控制幀 (如 `GOAWAY`) 進行優雅降級。

*(註：具體的錯誤代碼、狀態跳轉矩陣與控制幀結構，請參見 `protocol.md`。)*
---

## 3. 對歷史質疑的總結與反駁 (Rebuttals & Responses)

在先前的審查 (review.md) 中，我們內化了大量有價值的批評（如：補齊 Record 層、分離 Channel 與 Session、修正 `Epoch Cert` 語義、完善 Flow Control 與 Channel Lifecycle 等）。但有部分質疑存在視角上的差異，在此明確反駁並確定為最終設計：

### 反駁一：關於「握手必須有明確外層 Envelope (如明文長度前綴)」的質疑
*   **質疑內容**：沒有長度前綴的位元組流無法可靠解析。
*   **協議反駁**：在 Chameleon 中，我們透過限制 Noise Payload 為固定長度，使得 Client 的握手請求在數學上成為**絕對固定的密文塊（如 160 Bytes 或 112 Bytes）**。伺服器盲讀定長 Bytes 即可定界。這**不需要任何明文 Envelope**，從根本上消除了「明文頭」這個最容易被深度封包檢測 (DPI) 寫入特徵庫的致命弱點。

### 反駁二：關於「協議必須提供通用的靜默轉交 (Fallback) 保證」的質疑
*   **質疑內容**：Fallback 在 QUIC Stream 或 WebSocket 等多種承載上根本無法實作，因此它不應作為協議的核心。
*   **協議反駁**：我們接受了這個質疑的「事實部分」，即 Fallback 確實高度依賴底層承載（例如它在 TCP 上可行，但在 QUIC Stream 上極難）。但我們反駁「這就讓 Chameleon 不完整」的說法。**Fallback 被正式降級為「部署層的可選策略 (Deployment Strategy)」**。協議核心的唯一強制保證是**Pre-Auth 失敗時的絕對靜默 (Silent Drop)**。至於靜默後是直接切斷 socket，還是將 socket pipe 給 Nginx，由部署環境決定，協議本身不被此綁架。

### 反駁三：關於「確定性裝箱 (Bucketing) 不應寫死在核心協議裡」的延伸質疑
*   **質疑內容**：裝箱邏輯會影響延遲，放在協議層太死板。
*   **協議反駁**：我們完全接受將 Bucketing 移出核心協議。但我們反駁「協議不需要考慮特徵混淆」的想法。Chameleon 協議核心**必須提供 `PAD` 幀 (Padding Frame)** 作為基礎機制。控制面下發的策略 (Policy) 會告訴 Forwarding Engine 應該在何時、插入多長的 `PAD` 幀來實現 Bucketing。這是「機制與策略分離 (Separation of Mechanism and Policy)」的完美體現，保持了核心協議的極致輕量。
