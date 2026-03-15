# Chameleon 核心協議與線上規格白皮書 v9.0
*(Wire Format & Protocol Core Specification)*

## 0. 設計哲學與架構演進

### 0.1 專案設計目標與核心思想
Chameleon 的核心目標是構建一個**在極端惡劣與高壓干預的公網環境中，依然能夠穩定、安全、高效轉發流量的覆蓋網路 (Overlay Network)**。
在經歷了多個版本的推演與重構後，我們徹底放棄了「依靠巧妙偽裝（Camouflage）或隨機擾動來逃避審查」的天真幻想，並確立了以下不可動搖的設計思想：

1.  **安全邊界與密碼學保證 (Cryptographic Security Bounds)**：我們不使用籠統的「絕對安全」來過度承諾。系統的存活性建立在精確劃分的五層安全邊界上：
    *   **機密性與完整性 (Confidentiality & Post-Auth Integrity)**：握手後的所有流量受 AEAD 嚴格保護，並具備鎖步防重放能力。
    *   **伺服器單向認證 (Responder Authentication)**：透過帶外分發 (OOB) 的信任鏈，確保客戶端連線至合法的邊緣節點。
    *   **防主動探測的絕對靜默 (Pre-Auth Silence)**：未經授權的連線嘗試或探測器，在密碼學驗證失敗時，伺服器絕不暴露任何協議特徵。
    *   **有限的重放容忍 (Replay Limits)**：我們主動接受握手請求 (Pre-Auth) 可被重放的風險，以此換取邊緣節點的無狀態與高效能；但握手後的流量 (Post-Auth) 絕對不可重放。
    *   **惡劣網路存活性 (Operational Survivability)**：在面臨高丟包、封鎖與頻繁斷線的敵意網路中，透過精準的狀態機與路由策略實現快速自癒。
2.  **密碼學證明的零信任架構 (Zero-Trust Architecture)**：邊緣節點預設為不可信且隨時會失陷的消耗品。控制面信任鏈必須進行物理隔離（離線 Root CA），線上節點僅持有極短效期的派生憑證，確保單點失陷不會導致歷史流量解密或控制面信任崩潰。
3.  **防主動探測的絕對靜默 (Silent Drop under Active Probing)**：面對任何未經授權的連線嘗試、畸形探測或重放攻擊，伺服器在密碼學驗證（AEAD MAC 驗證）失敗後，必須保持**絕對靜默**，絕不在網路上暴露出任何協議專屬的錯誤信號或行為相變。在特定的部署拓撲下（如 TCP 承載），伺服器可選擇將無效流量無縫轉交 (Fallback) 給本機真實的 Web 服務，但這被視為**可選的部署策略**，而非協議本身的強制保證。
4.  **極致效能與強健可用性的統一 (Extreme Efficiency & Robust Availability)**：我們對協議的傳輸速度、解析效率與低延遲有著極致的追求。這種高效不來自於無腦搶佔頻寬，而是來自於極低開銷的 TLV 幀結構、以及無贅餘的密碼學握手。同時，協議必須具備超強的健壯性（Robustness），透過精準的狀態機快速自癒。需要注意的是，諸如「消除隊頭阻塞」或「無縫 IP 切換」等特性，本質上**繼承自底層承載 (如 QUIC)**，Chameleon 協議保證的是**不破壞**這些底層能力，並在其之上提供會話與通道管理。
5.  **語言與平台的徹底解耦 (Language-Agnostic Protocol)**：Chameleon 的核心是一套嚴謹定義的網路傳輸二進位規格 (Wire Format)，而非綁定特定程式語言的軟體實作。協議將底層視為不可靠的位元組流承載，在協議層內部提供端到端的安全與多路復用控制。所有狀態轉換、欄位長度、端序 (Endianness) 必須達到「不同團隊盲寫也能 100% 互通」的精確度。

### 0.2 本版架構演進與邊界確立 (v9.0)
在吸收了最新一輪的聯合審查 (Review) 後，Chameleon 的設計文檔 (`design.md`) 與協議規格 (`protocol.md`) 進行了徹底的職責分離：
*   **`design.md`**：專注於設計哲學、威脅模型、安全邊界與架構取捨 (Trade-offs)。不再包含任何具體的位元組佈局、長度常數或密碼學參數。
*   **`protocol.md`**：作為唯一的 Single Source of Truth，負責定義精確的線上格式 (Wire Format)、狀態機跳轉與二進位欄位。

本版本的核心架構確立與取捨在於：
1.  **會話池化架構 (Multi-Session Connection Pooling)**：我們明確定義 Chameleon 單一 Session 運行在**單一**的可靠位元組流 (Single Reliable Byte Stream) 上。為了在維持「嚴格密碼學鎖步 (Strict Lockstep)」的同時解決單一流帶來的隊頭阻塞 (HOL) 效能瓶頸，Chameleon 在架構層面正式引入 **會話池 (Session Pool)** 概念。客戶端不依賴單一連線，而是向伺服器維持一個動態的、包含多條獨立 Session 的連線池。這**無法完美消滅 HOL，但能將 HOL 的故障域 (Fault Domain) 嚴格隔離在單一 Session 內，從而顯著緩解 (Significantly Mitigates)** 其對整體可用性的影響。為確保不同實作間的效能預期一致，客戶端必須遵守最小池化契約 (Minimal Pool Contract)：
    *   **Class/Placement**: 強制將高延遲敏感的互動流/控制流 (Interactive/Control) 與大流量傳輸 (Bulk) 路由到池中不同的 Session。新通道一旦分配，不可跨 Session 遷移。
    *   **Bounds**: 依據設備能力與策略維持最小/最大活躍連線數 (Min/Max Sessions)。
    *   **Health & Replenishment**: 監控單一 Session 的 Channel ID 消耗與可用 Credit，在 Session 逼近耗盡或觸發 `GOAWAY` (Draining) 時，必須提前主動建立 (Replenish) 新的 Warm Session，絕不允許 Draining Session 接受新的 `OPEN_CHANNEL`。
2.  **單向信任與零特徵握手**：明確 Chameleon 的安全邊界是「客戶端單向預知伺服器靜態公鑰」以實現零特徵探測防護。具體的 Noise Pattern (如 `NK`) 與固定長度握手的實作細節，全權交由 `protocol.md` 定義。
3.  **澄清 Fallback 的邊界**：承認 Fallback (流量轉交) 高度依賴底層承載特性（如 TCP socket 移交），將其明確降級為「部署層的可選策略」，而非協議核心的強制行為。協議只保證 Pre-Auth 失敗時的絕對靜默 (Silent Drop)。
4.  **紀元憑證 (Epoch Cert) 的語義收斂**：明確紀元憑證僅用於提供防重放的**時間新鮮度視窗 (Freshness Window)**，不再承擔解決客戶端時鐘嚴重漂移的「時鐘同步」責任。
5.  **Datagram 的未來擴展**：架構上保留對 UDP/Datagram 代理的支援，但在當前穩定版本的規格中，將其標記為保留能力 (Reserved Capability)，優先確保 TCP Stream 代理與流控的絕對穩定。

---

## 1. 架構模型與信任邊界 (Architecture & Trust Boundaries)

Chameleon 採用嚴格的控制面與資料面分離架構，並在邊緣節點的信任假設上做出了務實的取捨。

### 1.1 控制面與兩級信任分發 (Two-Tier Trust Distribution)
*   **職責**：節點發現、憑證簽發、路由策略下發、版本治理。
*   **兩級 PKI 架構 (Two-Tier PKI)**：為解決「Root CA 必須絕對離線」與「邊緣節點需要頻繁簽發短期憑證」的運營矛盾，Chameleon 強制採用兩級簽發體系：
    1.  **Offline Root CA**：絕對離線，僅用於簽發 `Control Plane Signing Key (CPSK)` 憑證。客戶端透過安裝包或系統安全更新固定釘住 (Pin) 此 Root 公鑰。
    2.  **Online Control Plane (CPSK)**：線上控制面持有由 Root 簽發的短期/中期 `CPSK`。控制面使用 `CPSK` 來為全球邊緣節點高頻自動簽發 `Epoch Cert`。
*   **Responder Key 的預分發模型 (解決 `NK` 矛盾)**：在 Noise `NK` 模式中，客戶端必須在發送首包前知道伺服器的靜態公鑰。為了貫徹「邊緣節點不可信且短命」的原則，這個 Server Static Key 是一個由邊緣節點生成的**短期密鑰對（例如 24 小時壽命）**。**核心架構前提**：控制面必須透過帶外通道 (OOB) 將這個短期公鑰預先分發給客戶端。伺服器在握手回包中夾帶的 `Epoch Cert`，其語義僅限於確認這個已知的短期公鑰目前仍在授權紀元內。
*   **OOB 信任物件規格**：關於 `CPSK Certificate` 與預分發的 `Node Manifest` 的精確資料結構、防回滾 (Anti-rollback) 機制與簽發生命週期，由獨立的 **`control_plane.md`** 進行規範，不與資料面傳輸協議 (`protocol.md`) 混淆。

### 1.2 資料面與客戶端授權 (Data Plane & Client Auth)
*   **職責**：加密、多路復用、流控、錯誤恢復。
*   **握手層與授權的徹底解耦**：Chameleon 的握手層 (**Handshake Layer**) 僅負責伺服器側的信任建立 (Server-side Trust Establishment)、金鑰交換與能力協商。為了追求握手階段的絕對零特徵 (Zero-Signature)，我們**刻意不在 Noise 握手層包含任何客戶端身份驗證資料**。真正的客戶端授權 (Client Auth) 發生在握手成功後的第一個加密 Record 中。
*   **臨時會話邊界 (Provisional Session Bounds)**：握手成功但尚未收到 `CLIENT_AUTH` 幀的狀態被定義為**臨時會話 (Provisional Session)**。為防止資源耗盡攻擊，伺服器對此狀態施加極度嚴格的硬性資源限制（詳見 `protocol.md`）。超時或越界將立即觸發硬關閉 (Hard Close)。

### 1.3 握手重放防護的架構取捨 (Handshake Replay Trade-off)
*   **取捨聲明**：Chameleon 協議在 Record Layer (Post-Auth) 提供絕對的防重放保護 (Strict Lockstep Sequence)。然而，在握手階段 (Pre-Auth)，為了維持邊緣節點的極致無狀態 (Stateless)，我們**主動接受「握手請求 (Client First Flight) 可被重放」的風險**。
*   **安全邊界**：這是一個經過計算的架構妥協。重放合法的 Client 首包只會導致伺服器消耗極少量的 CPU 進行 DH 計算並回覆第二包。由於攻擊者沒有 Client 的 Ephemeral Private Key，絕對無法解密後續流量，也無法完成後續的 `CLIENT_AUTH`，連線將在 Provisional 階段被迅速掐斷。營運層可透過簡單的 IP Rate Limiting 來緩解此類資源消耗攻擊。

---

## 2. 錯誤處理與暴露面策略 (Error Handling & Surface Management)

Chameleon 對錯誤處理採取極端保守的態度，將錯誤嚴格劃分為三個階段，這構成了系統存活性的核心防線：

1.  **握手與認證前 (Pre-Auth)**：面對探測器、重放攻擊或版本不符，伺服器在密碼學驗證通過前，**絕對靜默 (Silent Drop)**。這使得 Chameleon 伺服器在互聯網掃描器眼中，與一個超時斷線的死節點或普通的 Web 伺服器毫無二致。
2.  **記錄層解密失敗 (Record Integrity Failure)**：一旦進入加密傳輸，任何 MAC 驗證失敗、序號錯亂、或頭部反混淆失敗，均視為連線被中間人污染。觸發**底層傳輸硬關閉 (Abort Transport)**，絕不嘗試在被污染的流上進行局部恢復。
3.  **邏輯錯誤 (Post-Auth Logical Failure)**：解密成功但協議語義錯誤（如未知的 Channel ID、授權失敗）。此時才會發送加密的控制幀 (如 `GOAWAY`) 進行優雅降級。

*(註：具體的 TCP/QUIC 關閉語義由部署環境綁定，核心協議僅定義抽象動作，詳見 `protocol.md`。)*

---

## 3. 對歷史質疑的總結與反駁 (Rebuttals & Responses)

在先前的審查 (review.md) 中，我們內化了大量有價值的批評（如：補齊 Record 層、分離 Channel 與 Session、修正 `Epoch Cert` 語義、完善 Flow Control 與 Channel Lifecycle 等）。但有部分質疑存在視角上的差異，在此明確反駁並確定為最終設計：

### 反駁一：關於「握手必須有明確外層 Envelope (如明文長度前綴)」的質疑
*   **質疑內容**：沒有長度前綴的位元組流無法可靠解析。
*   **協議反駁**：在 Chameleon 中，我們透過限制 Noise Payload 為固定長度，使得 Client 的握手請求在數學上成為**絕對固定的密文塊**。伺服器盲讀定長 Bytes 即可定界（具體長度由 `protocol.md` 嚴格定義）。這**不需要任何明文 Envelope**，從根本上消除了「明文頭」這個最容易被深度封包檢測 (DPI) 寫入特徵庫的致命弱點。

### 反駁二：關於「協議必須提供通用的靜默轉交 (Fallback) 保證」的質疑
*   **質疑內容**：Fallback 在 QUIC Stream 或 WebSocket 等多種承載上根本無法實作，因此它不應作為協議的核心。
*   **協議反駁**：我們接受了這個質疑的「事實部分」，即 Fallback 確實高度依賴底層承載（例如它在 TCP 上可行，但在 QUIC Stream 上極難）。但我們反駁「這就讓 Chameleon 不完整」的說法。**Fallback 被正式降級為「部署層的可選策略 (Deployment Strategy)」**。協議核心的唯一強制保證是**Pre-Auth 失敗時的絕對靜默 (Silent Drop)**。至於靜默後是直接切斷 socket，還是將 socket pipe 給 Nginx，由部署環境決定，協議本身不被此綁架。

### 反駁三：關於「單一流 (Single Stream) 導致控制信令也被 HOL 阻塞」的質疑
*   **質疑內容**：將所有邏輯通道塞入單一可靠位元組流，不僅資料流互相卡住，連 `KEY_UPDATE` 或 `OPEN_ACK` 等控制信令也會被大流量資料 (Bulk Data) 拖累，破壞了高可用性。
*   **協議反駁**：我們承認如果只使用單一連線，這的確會是個深層代價。但我們反駁「這就意味著協議無法提供極致效能」的想法。為了換取絕對的密碼學鎖步與防重放安全，單一流架構是必須的。作為架構層面的官方彌補：我們正式引入了**會話池化架構 (Multi-Session Connection Pooling)**。客戶端被要求動態維護一個包含多條獨立 Session（例如 5-10 條）的連線池。客戶端的負載均衡器會將高延遲敏感的互動流 (Interactive Traffic) 與大流量傳輸 (Bulk Transfer) 智慧地路由到**池中不同的 Chameleon Sessions** 上，利用底層網路的並發能力完美解決了 HOL 阻塞問題，同時毫不妥協協議內部的安全性與簡潔性。

### 反駁四：關於「確定性裝箱 (Bucketing) 不應寫死在核心協議裡」的延伸質疑
*   **質疑內容**：裝箱邏輯會影響延遲，放在協議層太死板。
*   **協議反駁**：我們完全接受將 Bucketing 移出核心協議。但我們反駁「協議不需要考慮特徵混淆」的想法。Chameleon 協議核心**必須提供 `PAD` 幀 (Padding Frame)** 作為基礎機制。控制面下發的策略 (Policy) 會告訴 Forwarding Engine 應該在何時、插入多長的 `PAD` 幀來實現 Bucketing。這是「機制與策略分離 (Separation of Mechanism and Policy)」的完美體現，保持了核心協議的極致輕量。