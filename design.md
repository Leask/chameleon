# Chameleon 核心協議與線上規格白皮書 v8.0
*(Wire Format & Protocol Core Specification)*

## 0. 設計哲學與架構演進

### 0.1 專案設計目標與核心思想
Chameleon 的核心目標是構建一個**在極端惡劣與高壓干預的公網環境中，依然能夠穩定、安全、高效轉發流量的覆蓋網路 (Overlay Network)**。
在經歷了多個版本的推演與重構後，我們徹底放棄了「依靠巧妙偽裝（Camouflage）或隨機擾動來逃避審查」的天真幻想，並確立了以下不可動搖的設計思想：

1.  **安全建立在密碼學與狀態機上，而非「偽裝」 (Cryptography Over Obscurity)**：我們承認不存在完美的流量隱形。系統的存活性不依賴「把流量包裝得像別的協議」，而是依賴嚴格的 Noise 密碼學握手、明確的會話邊界、以及無懈可擊的狀態轉換。我們透過 AEAD 和帶外信任分發（OOB Trust），確保協議在數學層面上不可被探測和重放。
2.  **密碼學證明的零信任架構 (Zero-Trust Architecture)**：邊緣節點預設為不可信且隨時會失陷的消耗品。控制面信任鏈必須進行物理隔離（離線 Root CA），線上節點僅持有極短效期的派生憑證，確保單點失陷不會導致歷史流量解密或控制面信任崩潰。
3.  **防主動探測的絕對靜默 (Silent Drop under Active Probing)**：面對任何未經授權的連線嘗試、畸形探測或重放攻擊，伺服器在密碼學驗證（AEAD MAC 驗證）失敗後，必須保持**絕對靜默**，或將流量無縫代理給本機真實的 Fallback 服務，絕不在網路上暴露出任何協議專屬的錯誤信號或行為相變。
4.  **極致效能與強健可用性的統一 (Extreme Efficiency & Robust Availability)**：我們對協議的傳輸速度、解析效率與低延遲有著極致的追求。這種高效不來自於無腦搶佔頻寬，而是來自於消除隊頭阻塞（HOLB）、極低開銷的 TLV 幀結構、以及無贅餘的密碼學握手。同時，協議必須具備超強的健壯性（Robustness），在面臨極高丟包、頻繁斷線、IP 切換等惡劣網路條件下，能保持高度可用（High Availability）並透過精準的狀態機快速自癒，確保互動流的「絲滑體感」。
5.  **語言與平台的徹底解耦 (Language-Agnostic Protocol)**：Chameleon 的核心是一套嚴謹定義的網路傳輸二進位規格 (Wire Format)，而非綁定特定程式語言的軟體實作。協議將底層視為不可靠的位元組流承載，在協議層內部提供端到端的安全與多路復用控制。

### 0.2 本版架構演進
在吸收了 v7 以及最新一輪嚴厲的協議審查 (Review) 後，Chameleon 協議正式從「概念框架」收斂為「精確的二進位規格」。

本版本的核心演進在於：
1.  **嚴格的四層協議劃分**：徹底拆分握手、記錄、多路復用與控制層。
2.  **定界與零特徵的完美平衡**：拒絕使用明文長度前綴（Plaintext Envelope），透過**固定長度的密碼學載荷**解決位元組流上的握手定界問題。
3.  **狀態與會話生命週期的修復**：糾正了「一個 QUIC Stream 一個會話」的效率災難，確立了「一條底層連線對應一個加密會話，會話內部包含多個邏輯通道」的標準模型。
4.  **策略與機制的解耦**：將「確定性裝箱 (Bucketing)」移出核心協議，降級為傳輸策略；協議核心僅提供 `PAD` 幀機制。

---

## 1. 協議分層模型 (Four-Layer Protocol Architecture)

Chameleon 協議運行在任何提供可靠位元組流 (Reliable Byte Stream) 的承載（如 TCP, WebSocket, QUIC Stream）之上。協議嚴格分為四層：

1.  **Handshake Layer (握手層)**：基於 Noise 協議，處理相互認證、金鑰交換與能力協商。
2.  **Record Layer (記錄加密層)**：處理位元組流的定界、AEAD 加密、隱式序號 (Sequence Number) 與金鑰輪換相位 (Key Phase)。
3.  **Channel Multiplex Layer (通道多路復用層)**：處理具體代理請求的創建、數據傳輸、流量控制與關閉。
4.  **Control Signaling Layer (控制信令層)**：處理會話級別的保活 (Ping)、優雅退出 (GoAway) 與金鑰更新 (KeyUpdate)。

---

## 2. Handshake Layer (握手層)

為防禦主動探測，握手階段必須是**零明文特徵 (Zero-Signature)**。我們採用 `Noise_IK_25519_ChaChaPoly_BLAKE2s` 模式。

### 2.1 握手定界：固定長度載荷 (Fixed-Length Handshake)
我們不使用明文長度前綴。因為握手 Payload 長度固定，Noise 產生的密文長度在數學上是絕對可預測的。

*   **Client -> Server (Handshake Request):**
    *   **Payload 定義**: `[Version(1 byte)] [Capabilities(4 bytes)] [Padding(59 bytes)]` = 精確 64 Bytes。
    *   **Noise IK 第一條訊息 (e, es, s, ss)**:
        長度 = `ephemeral_key`(32) + `static_key_ciphertext`(32+16) + `payload_ciphertext`(64+16) = **精確 160 Bytes**。
    *   **行為**: 伺服器在建立連線後，盲讀正好 160 Bytes。如果 AEAD 解密失敗，伺服器不回覆任何 Chameleon 錯誤幀，而是將這 160 Bytes 與後續的流資料**原封不動地代理給本機的 Fallback Web Server (如 Caddy)**。

*   **Server -> Client (Handshake Response):**
    *   **Payload 定義**: `[Version(1)] [Capabilities(4)] [Epoch Cert (112 bytes)] [Padding(11 bytes)]` = 精確 128 Bytes。
    *   **Noise IK 第二條訊息 (e, ee, se)**:
        長度 = `ephemeral_key`(32) + `payload_ciphertext`(128+16) = **精確 176 Bytes**。

### 2.2 預簽發紀元憑證 (Pre-issued Epoch Certificate)
解決時鐘漂移與伺服器效能問題，伺服器不再每次握手進行非對稱簽名。
*   `Epoch Cert` 結構 (112 bytes): `[Server_Static_Pubkey (32)] [Not_Before_UTC (8)] [Not_After_UTC (8)] [Ed25519_Signature_by_Control_Plane (64)]`
*   客戶端驗證此憑證，並根據 `Not_Before` 校準本地的時鐘偏移量 (Clock Offset)，徹底解決休眠設備時間不準的問題。

---

## 3. Record Layer (記錄層)

握手完成後，所有的資料被封裝為 Encrypted Records。

### 3.1 Record 結構
為了避免明文長度暴露特徵，Record 的頭部使用類似 QUIC 的 Header Protection（利用 AEAD 的 Ciphertext 採樣生成 Mask 進行 XOR 混淆）。

```text
+-------------------------+----------------------------+
| Obfuscated Length (2)   | Obfuscated Key Phase (1)   | <-- 混淆的明文頭 (3 Bytes)
+-------------------------+----------------------------+
| Ciphertext (Length)                                  | <-- 負載密文
+------------------------------------------------------+
| AEAD Tag / MAC (16 bytes)                            | <-- 認證標籤
+------------------------------------------------------+
```
*   **Implicit Sequence Number**: 雙方各自維護一個 64-bit 的發送/接收序號，參與 AEAD 的 Nonce 計算。不透過網路傳輸，**原生提供絕對的防重放保護 (Replay Protection)**。
*   **Key Phase**: 用於標識當前使用的金鑰代代碼 (0 或 1)，支援無縫的金鑰輪換。

---

## 4. Frame Types (通道與控制層)

Record 解密後的明文 Payload 可以包含一個或多個 Frames。

### 4.1 Channel Multiplex Layer (通道層)
*   **0x01 `OPEN_CHANNEL`**:
    `[Type(1)] [Channel ID (4)] [Target Length(1)] [Target (Host:Port)] [Initial Window (4)]`
*   **0x02 `DATA`**:
    `[Type(1)] [Channel ID (4)] [Data Length (2)] [Data...]`
*   **0x03 `WINDOW_UPDATE`**: (基於 Credit 的流量控制)
    `[Type(1)] [Channel ID (4)] [Added Credit (4)]`
*   **0x04 `CLOSE_CHANNEL`**:
    `[Type(1)] [Channel ID (4)] [Error Code (4)]`

### 4.2 Control Signaling Layer (控制信令層)
*   **0x10 `PING`** / **0x11 `PONG`**:
    `[Type(1)] [8-bytes Opaque Data]`
*   **0x12 `KEY_UPDATE`**:
    `[Type(1)] [Next Phase (1)]`。發送方通知對端，下一個發送的 Record 將使用新的 Key Phase 加密。
*   **0x13 `GOAWAY`**: (Post-auth 的安全錯誤信令)
    `[Type(1)] [Error Code (4)]`。用於通知對端憑證即將過期或協議嚴重違規，準備優雅關閉會話。
*   **0xFF `PAD`**:
    `[Type(1)] [Pad Length (2)] [Padding Bytes...]`。用於流量特徵混淆。

---

## 5. 錯誤處理狀態機 (Error Handling Strategy)

回應 v7 中對「全面靜默」的質疑，v8 明確區分了認證前後的錯誤處理：

1.  **Pre-Auth Errors (握手階段)**:
    解密失敗、長度不對、MAC 錯誤。**絕對靜默丟棄 (Silent Drop)**。不發送任何錯誤幀，防止 Active Probing 探測協議特徵。
2.  **Post-Auth Errors (加密通訊階段)**:
    如收到不合法的 Channel ID、未知的 Frame Type。此時對手已經通過了密碼學認證，屬於內部邏輯錯誤。**必須發送 `GOAWAY` 幀** 並關閉連線，以確保客戶端能進行精準的錯誤恢復與可觀測性記錄。

---

## 6. 對歷史質疑的總結與反駁 (Rebuttals & Responses)

在先前的審查 (review.md) 中，有一些觀點我們已經內化（如：補齊 Record 層、分離 Channel 與 Session、修正 0-RTT 範圍）。但有部分質疑存在技術盲點，在此明確反駁並確定為最終設計：

### 反駁一：關於「握手必須有明確外層 Envelope (如明文長度前綴)」的質疑
*   **質疑內容**：沒有長度前綴的位元組流無法可靠解析，一旦讀取部分 bytes，就無法將流轉交給 fallback 處理器。
*   **協議反駁**：在 Chameleon 中，我們透過**限制 Noise IK Payload 為精確的 64 Bytes**，使得 Client 的握手請求在數學上成為**絕對固定的 160 Bytes 密文**。伺服器只需盲讀 160 Bytes。如果解密失敗，伺服器只需將這讀出來的 160 Bytes 加上緩衝區剩餘的 Bytes，原封不動地 Pipe 給 Caddy (Fallback Server)。這**不需要任何明文 Envelope**，從根本上消除了「明文頭」這個最容易被深度封包檢測 (DPI) 寫入特徵庫的弱點。

### 反駁二：關於「IK 模式依賴客戶端預先知道伺服器靜態公鑰，是巨大的控制面前提」的質疑
*   **質疑內容**：事先知道公鑰不是小前提，會導致控制面過於複雜。
*   **協議反駁**：這是正確的描述，但不是一個「缺陷」。在抗審查的終極對抗中，**帶外分發信任 (Out-of-Band Trust Distribution) 是抵禦主動探測的唯一數學解**。任何試圖在握手階段（In-Band）進行公鑰協商的協議（如標準 TLS 1.3），都會在網路上暴露協商特徵。我們主動選擇承擔控制面的複雜度，正是為了換取資料面的「零特徵 (Zero-Signature)」。這是一筆極具性價比的安全架構交易。
