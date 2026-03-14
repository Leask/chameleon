# Chameleon 協議設計與連線規格白皮書 v7.0
*(Language-Agnostic Protocol Specification)*

## 0. 內化與共識：回歸協議的本質

在經歷了從 v1 到 v6 的演進後，我們必須再次對齊我們的核心思想。在前幾個版本中，我們從「天真的協議包裝」走向了「嚴謹的威脅模型」，再到「具體的軟體工程實作」。但我們必須在此確立一個更高的共識：

**Chameleon 的核心資產是一套「協議 (Protocol)」，而不是某個特定語言的「代碼庫 (Codebase)」。**

1.  **語言無關性 (Language Agnosticism)：** 一個強大、穩定、安全的協議，不應依賴於它是由 Rust、Go 還是 C 寫成的。協議的價值在於其**線上資料格式 (Wire Format)**、**狀態機轉換 (State Transitions)** 與 **密碼學證明 (Cryptographic Proofs)**。
2.  **安全與高效的統一：** 真正的安全不來自於代碼層面的混淆，而來自於數學與協議設計。真正的高效不來自於特定語言的零拷貝 (Zero-copy) 技巧，而來自於協議減少了不必要的 RTT、消除了頭部阻塞 (HOLB)，並在最底層支持了精準的流量控制與排程。
3.  **依賴降級：** 我們將底層的網路承載（如 QUIC、WebTransport）視為「不可信且特性各異的傳輸管道」。Chameleon 協議運行在這些管道內部，提供端到端 (End-to-End) 的安全、偽裝與多路復用控制。

---

## 1. 對 `design_6.md` 的嚴厲攻擊

`design_6.md` 雖然極大地推動了工程落地，但從「協議設計」的角度來看，它犯了一個致命的錯誤：**「它用軟體架構圖取代了網路協議規格。」**

### 1.1 混淆了「軟體實作」與「協議設計」
v6 大量篇幅在討論 Rust Core、C-FFI、GUI 宿主域。
**攻擊：** 如果今天有兩個完全獨立的團隊，一個用 C++，一個用 Go，他們拿到 `design_6.md`，絕對無法寫出能夠互相通訊的客戶端和伺服器。因為 v6 根本沒有定義封包長什麼樣子、Byte 怎麼排列、握手發送的第一個 Byte 是什麼。

### 1.2 狀態機缺乏線上對映 (Wire-Level Mapping)
v6 定義了優美的狀態機（如 `SESSION_MIGRATING`），但沒有說明這些狀態在網路上是如何表達的。
**攻擊：** 當設備需要遷移連線時，它在網路上發送的是什麼指令？是一個特定的 Frame？還是利用了 QUIC 底層的 Migration？如果底層不支持，Chameleon 協議內部該如何表達這個狀態轉移？v6 對此隻字未提。

### 1.3 策略配置與傳輸協議的耦合
v6 試圖把「策略 (Policy)」塞進核心，變成路由查找表。
**攻擊：** 協議本身不應該關心「這個封包要不要走代理」，那是軟體路由層的事。協議只應該定義：「當你決定要把一段數據塞進 Chameleon 隧道時，它會被如何加密、分片、填充、並在對端被正確還原。」

---

## 2. Chameleon v7.0 核心協議設計 (Protocol Design)

v7 放棄所有與平台、語言相關的討論，專注於**位元組 (Bytes)**、**幀 (Frames)** 與 **狀態 (States)**。

### 2.1 協議層次定位 (Layering)

Chameleon 是一個 L5/L6 的隧道協議，它預設運行在一個**提供可靠、有序、雙向位元組流 (Byte Stream)** 的基礎傳輸層之上（如 QUIC Streams 或 WebTransport Streams）。

```text
+---------------------------------------------------+
| Application Data (HTTP, DNS, SOCKS Payload, etc.) |
+---------------------------------------------------+
| Chameleon Framing & Padding Layer (幀與填充)      |  <-- v7 核心
+---------------------------------------------------+
| Chameleon Cryptographic Session Layer (會話與加密)|  <-- v7 核心
+---------------------------------------------------+
| Base Transport Stream (QUIC Stream, TCP, WS)      |
+---------------------------------------------------+
```

### 2.2 密碼學與握手協議 (Cryptographic Handshake)

我們不發明加密算法，而是嚴格基於 **Noise Protocol Framework**（具體建議使用 `Noise_IK_25519_ChaChaPoly_BLAKE2s` 模式）。

#### 2.2.1 零特徵握手 (Zero-Signature Handshake)
為了達到「隱匿於人群」的目標，握手階段不能有任何明文的協議標識符（Magic Bytes）。

*   **Client -> Server (Handshake Request):**
    發送的純粹是 Noise `IK` 模式的第一條訊息。這是一段看起來完全隨機的二進位數據。沒有長度前綴（長度由底層 Stream 決定），沒有協議名稱。
*   **Server 處理 (靜默回退):**
    伺服器嘗試用自己的私鑰解密。如果解密失敗、校驗和 (MAC) 錯誤，伺服器**不返回任何錯誤協議幀**，而是直接將此底層 Stream 的控制權轉交給本機的 Fallback Web Server（例如返回一個正常的 HTTP 400 響應，或直接關閉 Stream，行為取決於底層承載）。

#### 2.2.2 伺服器輔助紀元 (Server-Assisted Epoch) 在握手中的實作
*   **Server -> Client (Handshake Response):**
    包含 Noise 第二條訊息，並在加密的 Payload 中夾帶以下結構：
    ```text
    [8 bytes: Server Absolute Timestamp (UTC)]
    [32 bytes: Ed25519 Signature of Timestamp by Control Plane CA]
    [2 bytes: Session Lifetime in minutes]
    ```
    客戶端解密後驗證時間戳簽名，計算出時鐘偏移量，確保後續的憑證驗證與金鑰輪換具有精確的時間基準。

### 2.3 Chameleon 幀結構 (Framing Format)

一旦握手完成，雙方進入對稱加密的資料傳輸階段。為了實現極致的效率與特徵收斂，我們定義緊湊的 TLV (Type-Length-Value) 幀結構。

每個加密記錄 (Encrypted Record) 解密後的明文結構如下：

```text
+---------+-----------+-----------------------+
| Type(1) | Length(2) | Payload (Variable)    |
+---------+-----------+-----------------------+
```

#### 定義的 Frame Types：
*   `0x01 DATA`: 承載實際的用戶數據。Payload 包含 Stream ID (如果底層不支持多路復用) 和原始資料。
*   `0x02 PING`: 用於測量 RTT 與保持活躍。Payload 包含 8 bytes 的時間戳或序號。
*   `0x03 PONG`: 回應 PING。
*   `0x04 REKEY_REQ`: 請求金鑰輪換。
*   `0x05 REKEY_ACK`: 確認金鑰輪換。
*   `0x06 CLOSE_STREAM`: 優雅關閉某個虛擬通道。
*   `0xFF PAD`: 填充幀。此幀的 Payload 內容必須被接收方完全忽略。

### 2.4 協議級特徵收斂：確定性裝箱 (Deterministic Bucketing)

v6 提到了「封包大小裝箱」，在 v7 的協議層，我們將其精確定義為**多幀聚合與填充 (Multi-Frame Aggregation & Padding)**。

*   **目標:** 使網路上傳輸的每一個加密記錄 (Encrypted Record) 的總大小，嚴格等於預設的 Bucket 大小（例如：`256`, `512`, `1024`, `1350` Bytes）。
*   **協議行為:**
    當發送方需要發送 300 Bytes 的 `DATA` 幀時，它會選擇下一個 Bucket 尺寸（512 Bytes）。
    發送方會在一個加密記錄中打包：
    `[DATA Frame: 300 bytes] + [PAD Frame: 209 bytes (Type(1) + Length(2) + 206 bytes dummy)] = 509 bytes`
    加上 3 Bytes 的加密頭/MAC，總大小精確達到 512 Bytes。
*   **優勢:** 攻擊者（DPI）只能看到大批量的 512 Bytes 或 1350 Bytes 密文塊，徹底消滅基於封包大小的應用指紋（如 TLS 握手包大小、特定影片 chunk 大小）。

### 2.5 協議狀態機與錯誤信令 (Error Signaling)

協議內部維護嚴格的狀態。與 v6 不同，v7 明確定義了錯誤在協議層如何傳遞：

*   **靜默關閉原則 (Silent Drop on Fatal Errors):**
    對於任何密碼學錯誤（解密失敗、MAC 驗證失敗、重放攻擊檢測命中），協議**不發送任何錯誤幀**。直接在底層丟棄該 Stream。這能最大程度防止攻擊者透過錯誤探測 (Error Oracle) 推測協議內部狀態。
*   **優雅降級 (Graceful Degradation):**
    當邊緣節點憑證即將過期（由 Server-Assisted Epoch 計算得出），伺服器可主動發送帶有特定 Payload 的 `PING` 幀暗示客戶端：「我快失效了，請不要再建立新的 Stream，並準備切換節點」。客戶端收到後，將協議狀態標記為 `DRAINING`。

### 2.6 多路復用授權 (Multiplexing Delegation)

Chameleon 協議極具彈性。
*   如果底層承載（如 QUIC）**原生支援** Stream 多路復用，Chameleon 協議的 `DATA` 幀就不包含 Stream ID。每個 QUIC Stream 內部就是一個獨立的 Chameleon 會話（握手 + 數據）。這樣完全消除了隊頭阻塞 (HOLB)。
*   如果底層承載（如 WebSocket）**不支援**原生多路復用，Chameleon 協議則在 `DATA` 幀的 Payload 前 4 Bytes 啟用虛擬 Stream ID，在協議層內部實作多路復用（類似 HTTP/2 的機制）。

---

## 3. 總結與下一步

`design_7.md` 將 Chameleon 徹底還原為一個純粹的**網路傳輸協議**。

我們排除了軟體架構的干擾，明確定義了：
1. **零特徵握手** (Noise IK + 靜默回退)。
2. **解決時鐘漂移的密碼學載荷** (Server-Assisted Epoch)。
3. **TLV 幀結構與 PAD 填充算法** (Deterministic Bucketing)。
4. **狀態與錯誤的線上表達** (Silent Drop & Draining)。

這份協議白皮書現在是語言中立的。無論未來團隊決定用什麼技術棧開發，只要嚴格遵守 v7 的位元組結構與狀態定義，不同語言寫出的 Chameleon 端點就能夠完美互通、並展現出一致的抗審查能力。

下一個版本，我們是否應該針對 **「動態路由策略與配置下發 (Policy & Config Provisioning)」** 這個控制面的核心協議進行詳細的定義？