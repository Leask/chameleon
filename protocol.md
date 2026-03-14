# Chameleon 網路通訊協議規格書 v3.0
*(Chameleon Network Protocol Specification)*

**狀態**: 草案 (Draft)
**標籤**: 嚴謹二進位規格 (Strict Binary Specification), 語言無關 (Language-Agnostic)

---

## 1. 協議概述 (Overview)

Chameleon 是一個運行於可靠位元組流（Reliable Byte Stream，如 QUIC Stream、TCP、WebSocket）之上的安全隧道與多路復用協議。

協議分為四層：
1. **握手層 (Handshake Layer)**: 負責身份認證、金鑰交換與能力協商。
2. **記錄層 (Record Layer)**: 負責加密、嚴格防重放、定界與頭部混淆。
3. **通道層 (Channel Layer)**: 負責多路復用、流控與生命週期管理。
4. **控制層 (Control Layer)**: 負責會話級信令、金鑰輪換與錯誤恢復。

### 1.1 位元組序與約定
* 所有整數欄位（16-bit, 32-bit, 64-bit）均使用**大端序 (Big-Endian, Network Byte Order)**。
* 基礎承載假設為**無損、保序的可靠位元組流**（Reliable in-order byte stream）。
* 密碼學金鑰與雜湊值以位元組陣列 (Byte Array) 處理。

### 1.2 密碼學標準參考 (Cryptographic Standards)
* **ChaCha20-Poly1305**: RFC 8439
* **Noise Protocol Framework**: RFC 7748 (Curve25519)
* **HKDF**: RFC 5869

---

## 2. 密碼學原語與密鑰導出 (Cryptographic Primitives)

* **Handshake Pattern**: Noise `NK` (`Noise_NK_25519_ChaChaPoly_BLAKE2s`)
* **Prologue**: `b"chameleon-v1"` (ASCII 編碼，不含 null 終止符，共 12 Bytes)。雙方在初始化 Noise 狀態機時必須輸入此 Prologue，防止跨協議重放攻擊。

### 2.1 初始金鑰導出 (Initial Key Schedule)
握手完成後，Noise 狀態機的 `Split()` 函數會輸出兩個 32 Bytes 的對稱金鑰（記為 `cs1`, `cs2`）。
* `cs1` 用於 **Initiator to Responder (Client -> Server)** 方向。
* `cs2` 用於 **Responder to Initiator (Server -> Client)** 方向。

Chameleon 不直接使用 `cs1/cs2` 進行加密，而是將其作為 PRK (Pseudorandom Key) 進行 `HKDF-Expand` (Hash = BLAKE2s) 以分離 Traffic Key 和 HP Key：

1. **Client -> Server 方向**:
   * `C2S_Traffic_Key = HKDF-Expand(cs1, "c2s_traffic", 32)`
   * `C2S_HP_Key = HKDF-Expand(cs1, "c2s_hp", 32)`
   * `C2S_ChaCha_IV = HKDF-Expand(cs1, "c2s_iv", 12)`
2. **Server -> Client 方向**:
   * `S2C_Traffic_Key = HKDF-Expand(cs2, "s2c_traffic", 32)`
   * `S2C_HP_Key = HKDF-Expand(cs2, "s2c_hp", 32)`
   * `S2C_ChaCha_IV = HKDF-Expand(cs2, "s2c_iv", 12)`

*(註：Info 字串轉換為 ASCII bytes，不含 Null 終止符。)*

---

## 3. 握手層規格 (Handshake Layer)

採用**固定長度無前綴 (Fixed-Length, Prefix-Free)** 密文設計。

### 3.1 握手請求 (Client -> Server)
採用 Noise `NK` 模式（單向信任：Client 預先擁有 Server Static Pubkey）。
**Noise Transcript (Initiator -> Responder)**: `-> e, es`

**1. 構造明文 Payload (64 Bytes 固定長度):**
```text
+-------------------+-------------------+-----------------------------------+
| Version (1 byte)  | Caps (4 bytes)    | Padding (59 bytes, 填 0x00)       |
+-------------------+-------------------+-----------------------------------+
```
* `Version`: 必須為 `0x01`。
* `Caps`: 能力位元遮罩，目前保留為 `0x00000000` (將 Datagram 等未來擴充標記為 Reserved)。

**2. 密文傳輸 (112 Bytes 固定長度):**
Noise `NK` 第一條訊息的序列化：
* `e` (Ephemeral Pubkey, 明文發送): 32 Bytes
* `es` (執行 DH，產出金鑰，無輸出 bytes)
* `payload` (使用產生的金鑰進行 AEAD 加密): 64 Bytes + 16 Bytes MAC = 80 Bytes
**總長度**: `32 + 80 = 112 Bytes`。

**伺服器行為**: 盲讀 112 Bytes。如果 AEAD 解密失敗，**必須保持絕對靜默 (Silent Drop)**。這表示伺服器可選擇直接中斷底層連線（如 TCP RST），或將流靜默轉交給本機服務（Fallback），但**絕對不可 (MUST NOT)** 返回任何 Chameleon 錯誤幀。

### 3.2 握手回應 (Server -> Client)
**Noise Transcript (Responder -> Initiator)**: `<- e, ee`

**1. 紀元憑證 (Epoch Cert) 結構 (120 Bytes):**
```text
+-----------------------------------+-----------------------------------+
| Server Static Pubkey (32 bytes)   | Epoch_Window_Start (8 bytes, 秒數)|
+-----------------------------------+-----------------------------------+
| Epoch_Window_End (8 bytes, 秒數)  | Signer Key ID (8 bytes)           |
+-----------------------------------+-----------------------------------+
| Ed25519 Control Plane Sig (64 b)                                      |
+-----------------------------------+-----------------------------------+
```
*(提供防重放的時間新鮮度視窗與兩級信任鏈綁定)*

**2. 構造明文 Payload (128 Bytes 固定長度):**
```text
+-------------------+-------------------+-------------------+---------------+
| Version (1 byte)  | Selected Caps (4) | Epoch Cert (120 b)| Pad (3 bytes) |
+-------------------+-------------------+-------------------+---------------+
```

**3. 密文傳輸 (176 Bytes 固定長度):**
Noise `NK` 第二條訊息的序列化：
* `e` (Ephemeral Pubkey, 參與後續加密): 32 Bytes
* `ee` (執行 DH)
* `payload` (使用更新後的金鑰進行 AEAD 加密): 128 Bytes + 16 Bytes MAC = 144 Bytes
**總長度**: `32 + 144 = 176 Bytes`。

### 3.3 Epoch Cert 驗證演算法 (Epoch Cert Verification Algorithm)
接收到伺服器的 `Epoch Cert` 後，客戶端**必須 (MUST)** 執行以下嚴格的驗證步驟。任何一步失敗，必須立即中斷連線並丟棄金鑰：

1. **Signer Trust Root 校驗 (兩級 PKI)**: 客戶端必須預先透過 OOB 獲取由離線 Root CA 簽發的線上控制面簽發金鑰 (`CPSK`, Control Plane Signing Key) 憑證。客戶端提取 `Epoch Cert` 中的 `Signer Key ID`，從本地信任庫中找出對應的有效 `CPSK` 公鑰。使用此 `CPSK` 公鑰驗證 `Epoch Cert` 後 64 Bytes 的 `Ed25519 Control Plane Sig`。簽名的 Message 內容為 `Epoch Cert` 的前 56 Bytes (`Server Static Pubkey` || `Epoch_Window_Start` || `Epoch_Window_End` || `Signer Key ID`)。
2. **Responder Key 綁定校驗**: 提取 `Epoch Cert` 中的 32 Bytes `Server Static Pubkey`，並與本次 Noise `NK` 握手前客戶端所使用的預期伺服器公鑰進行絕對比對 (恆等校驗)。若不一致，代表伺服器身份被劫持或節點串線，立即失敗。
3. **新鮮度視窗與時鐘容忍 (Freshness Window & Clock Skew)**:
   * 獲取客戶端目前的絕對時間 (UTC 秒數)，記為 `Now`。
   * 定義最大時鐘偏差容忍值 `MAX_CLOCK_SKEW = 300` (秒)。
   * **生效檢查**: `Now + MAX_CLOCK_SKEW >= Epoch_Window_Start`。若不滿足，視為憑證尚未生效 (可能時鐘嚴重落後或配置錯誤)。
   * **過期檢查**: `Now - MAX_CLOCK_SKEW <= Epoch_Window_End`。若不滿足，視為憑證已過期 (可能遭逢重放攻擊或節點未更新)。
   * 違反上述時間視窗時，觸發 `Config Refresh Required` 的恢復策略。

---

## 4. 記錄層規格 (Record Layer)

### 4.1 隱式序號與鎖步防重放 (Strict Lockstep Sequence)
* 雙方各自維護 `SendSeq` 與 `RecvSeq` (初始為 0)。
* **絕對順序保證**: 因為底層是可靠流，`RecvSeq` 必須與收到的 Record 嚴格一一對應。**不允許亂序，不允許跳號**。
* AEAD 每次加密/解密的 `Nonce` (12 Bytes) 計算方式：將 64-bit 的 Sequence Number 編碼為 8 Bytes 的大端序 (Big-Endian) 位元組陣列，在其左側填充 4 Bytes 的 `0x00` 構成 12 Bytes。然後將這 12 Bytes 與對應方向的 `ChaCha_IV` 進行 XOR 運算：
  `Nonce = ChaCha_IV XOR ( 0x00000000 || BigEndian(Sequence_Number) )`。
* 任何 AEAD MAC 驗證失敗，意味著位元組流已被污染或發生了未經授權的篡改，**必須立即硬關閉底層傳輸 (Hard Close)**。

### 4.2 Record 結構與頭部混淆 (Header Protection)
```text
+--------------------------+-----------------------+
| Obfuscated Length (2)    | Obfuscated Flags (1)  |  <-- 3 Bytes
+--------------------------+-----------------------+
| AEAD Ciphertext (Payload + 16 bytes MAC)         |  
+--------------------------------------------------+
```
* `Length` (2 bytes): 表示整個 Ciphertext 的總長度（**包含 16 Bytes 的 MAC Tag**）。即 `Payload Length + 16`。最大允許值 `16384` (16KB)。最小允許值 `16` (Payload 為空)。
* `Flags` (1 byte): `[Key Phase (1 bit)] [Reserved (7 bits, 須為 0)]`。
* **精確混淆演算法**: 我們使用 `Sample` 作為 ChaCha20 的 Nonce 輸入，確保每個 Record 的 Mask 都是動態且不可預測的。
  1. `Sample` 擷取：取 `Ciphertext` 的前 12 Bytes。如果 `Ciphertext` 總長度大於等於 16 Bytes，直接取前 12 Bytes。如果小於 12 Bytes（協議邏輯上最小為 16，但為防禦極端惡意截斷），不足部分在末尾補 `0x00`。
  2. `Mask` 生成：使用標準 ChaCha20 區塊加密。
     * `Key` = `HP_Key` (32 Bytes)
     * `Nonce` = `Sample` (12 Bytes)
     * `Block Counter` = `0`
     * 生成 64 Bytes 的 Keystream，截取前 3 Bytes 作為 `Mask`。
* `Obfuscated_Header = (Length || Flags) XOR Mask`。

* **解析器行為 (Parser Discipline)**: 接收方先讀取 3 Bytes 的 `Obfuscated_Header`。接著向前**預讀 (Peek)** 後續的 12 Bytes 作為 `Sample`。使用 `HP_Key` 和 `Sample` 計算出 Mask 後，反解出 `Length` 與 `Flags`。最後根據 `Length` 繼續讀取完整的 Ciphertext 進行 AEAD 解密。

---

## 5. 幀格式規格 (Frame Formats)

### 5.0 客戶端授權 (Client Authentication)
#### `0x00` CLIENT_AUTH (客戶端身分驗證)
```text
+---------+----------------+----------------+------------------+-------------+
| Type(1) | Auth Scheme(1) | Credential ID(8)| Proof Length (2) | Proof (Var) |
+---------+----------------+----------------+------------------+-------------+
```
**強制規範 (MUST)**: 握手完成後，客戶端進入 **Provisional Session (臨時會話)**。此狀態下：
1. 第一個 Record 的第一個 Frame **必須**是 `CLIENT_AUTH`。且該 Record **絕對禁止**包含除 `PAD` 外的任何其他 Frame（不允許將 Auth 與 `OPEN_CHANNEL` 放在同一個 Record 內打包）。
2. **硬性資源邊界 (Strict Provisional Bounds)**: 伺服器在進入 Provisional 狀態後啟動計時器與計數器。在成功驗證 `CLIENT_AUTH` 之前：
   * **絕對超時上限 (Absolute Timeout)**: `3000` 毫秒。
   * **最大接收位元組 (Max Inbound Bytes)**: `2048` Bytes (包含底層協議頭)。
   任何越界行為必須立即觸發 `Abort Transport`。
3. **驗證演算法 (Auth Scheme 0x01: Ed25519 Transcript Binding)**:
   * `Credential ID`: 作為索引，用於在控制面下發的使用者資料庫中查找對應的 Client Ed25519 Public Key。
   * `Proof Length`: 固定為 `64` (Bytes)。
   * **Domain Separation 與綁定**: 客戶端使用其 Ed25519 私鑰，對特定的 Context String 與**當前連線的 Noise 握手 Transcript Hash** 進行拼接後簽章。
     `Proof = Sign( Private_Key, "chameleon-auth-v1" || h )`
     (其中 `"chameleon-auth-v1"` 為 ASCII 編碼，不含 null，共 17 Bytes；`h` 即 Noise 狀態機握手完成時輸出的 32 Bytes Hash 變數)。
   * **安全性**: 此設計將客戶端的長期身份 (`Credential ID`) 與這一次特定的物理連線 (`Transcript Hash`) 進行了上下文隔離的密碼學綁定，徹底杜絕了 Auth Token 被跨連線、跨協議重放的可能。
4. 伺服器若驗證失敗（ID 不存在、簽名錯誤、或該 ID 已被撤銷），必須發送 `GOAWAY (Code: 0x05 Auth Failed)` 並硬關閉連線。

### 5.1 通道層 Frames (Channel Lifecycle & Multiplexing)

#### `0x01` OPEN_CHANNEL (開啟通道)
```text
+---------+--------------+---------+-----------------+--------+--------------------+-------------+---------+
| Type(1) | Channel ID(4)| Kind(1) | Init Window (4) | ATYP(1)| Target Length (1)* | Target(Var) | Port(2) |
+---------+--------------+---------+-----------------+--------+--------------------+-------------+---------+
```
* `Channel ID`: 發起方保證全域唯一，最大允許值為 `2^30 - 1`。
  * **分配規則 (MUST)**: 客戶端發起的 ID 必須是**奇數** (1, 3, 5...)，伺服器發起的 ID 必須是**偶數** (2, 4, 6...)。
  * **單調遞增 (Monotonicity)**: 雙方發起的新 `Channel ID` 必須嚴格單調遞增。若接收方收到一個小於或等於已見過最高 ID 的 `OPEN_CHANNEL`，必須視為協議違規，立即發送 `GOAWAY (Code: 0x02)` 並中斷連線。
  * **ID 耗盡防護 (Exhaustion Watermark)**: 當客戶端下一個可分配的 `Channel ID` 超過 `2^30 - 100` 時，客戶端必須主動發送 `GOAWAY (Code: 0x00)` 並進入 Draining 狀態，並通知 Session Pool Manager 建立新的替代 Session。
* `Kind`: 必須為 `0x01` (TCP Stream)。`0x02` (UDP Datagram) 暫為保留能力。
* `Init Window`: 允許對端發送的初始位元組數 (Channel-level Flow Control)。
* `ATYP` (SOCKS5 衍生標準):
  * `0x01`: IPv4 (`Target` 固定 4 Bytes，**無** `Target Length` 欄位)
  * `0x03`: Domain (**包含** 1 Byte 的 `Target Length` 欄位，後接 ASCII 域名，不含 Null 終止符)
  * `0x04`: IPv6 (`Target` 固定 16 Bytes，**無** `Target Length` 欄位)

#### `0x02` OPEN_ACK (確認通道開啟)
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Init Window (4)   |
+----------+-----------------+-------------------+
```
接收方確認資源已分配，發起方收到此幀後才可發送 `DATA`。若接收方拒絕開啟通道，則直接回覆 `RESET_CHANNEL` 幀作為負向回應。

#### `0x03` DATA (通道數據)
```text
+----------+-----------------+-------------------+-----------------------+
| Type(1)  | Channel ID (4)  | Data Length (2)   | Data (Data Length)    |
+----------+-----------------+-------------------+-----------------------+
```

#### `0x04` FIN_CHANNEL (優雅半關閉)
```text
+----------+-----------------+
| Type(1)  | Channel ID (4)  |
+----------+-----------------+
```
單向 EOF。雙方均發送 `FIN` 後，通道生命週期結束。

#### `0x05` RESET_CHANNEL (強制重置通道/拒絕開啟)
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Error Code (4)    |
+----------+-----------------+-------------------+
```
立即廢棄該 Channel 的所有未讀資料與狀態。不會影響 Session 內的其他 Channel。可用於拒絕 `OPEN_CHANNEL` 請求。

### 5.2 流量控制 Frames (Flow Control)

**初始條件 (Initial Constraints)**: 握手完成後，預設的 Channel Initial Window 為 0，預設的 **Session Initial Window 為 10MB**（雙向獨立）。`DATA` 幀的發送會同時消耗 Channel Credit 與 Session Credit，任意一個 Credit 不足時，發送方必須阻塞。`PAD` 與所有控制幀（如 `PING`, `WINDOW_UPDATE`）**絕對不消耗**任何 Session 或 Channel Credit。

**調度器優先級 (Scheduler Priority)**:
發送方在組合 Record 時，必須遵守以下硬性優先級，確保流量特徵填充不會打穿資源約束：
`Auth / Critical Control (GOAWAY, KEY_UPDATE)` > `Flow Control (WINDOW_UPDATE)` > `DATA / OPEN_CHANNEL` > `PAD`
**強制規範 (MUST)**: `PAD` 幀絕對不可餓死 (Starve) 任何認證流量或資料流量。`PAD` 的發送僅受制於控制面下發的 Session-level Egress Budget（如每秒最大填充字節數）。

#### `0x06` CHANNEL_WINDOW_UPDATE
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Added Credit (4)  |
+----------+-----------------+-------------------+
```
增加單一通道的發送額度。

#### `0x07` SESSION_WINDOW_UPDATE
```text
+----------+-------------------+
| Type(1)  | Added Credit (4)  |
+----------+-------------------+
```
增加整個 Session 的發送總額度，防止單一惡意通道耗盡全局記憶體。

### 5.3 控制層 Frames (Control Signaling)

#### `0x10` PING / `0x11` PONG (保活與測速)
`[Type(1)] [8-bytes Opaque Data]`。接收方必須原樣回傳 `PONG`。

#### `0x12` KEY_UPDATE (金鑰輪換狀態機)
`[Type(1)]` (無 Payload)。
* **發送方行為**: 發送 `KEY_UPDATE` 幀，從**下一個 Record 開始**，發送方將 `Key Phase` 位元翻轉，並使用新一輪的金鑰套件加密與混淆頭部。
* **KDF (Key Derivation Function)**: 一旦觸發輪換，同時更新 Traffic Key 與 Header Protection (HP) Key：
  * `Next_Traffic_Key = HKDF-Expand(Current_Traffic_Key, "traffic_rekey", 32)`
  * `Next_HP_Key = HKDF-Expand(Current_HP_Key, "hp_rekey", 32)`
* **接收方行為**: 收到 `KEY_UPDATE` 後，計算出兩把新金鑰。當收到下一個 Record 發現 `Key Phase` 翻轉時，使用新金鑰解除頭部混淆與解密。接收方也應盡快發送自己的 `KEY_UPDATE` 以完成雙向輪換。為避免歧義，連續兩次 `KEY_UPDATE` 之間必須相隔至少 1000 個 Records 或 1 分鐘。

#### `0x13` GOAWAY (會話級致命錯誤)
```text
+----------+-------------------------+-------------------------------+
| Type(1)  | Error Code (4 bytes)    | Last Accepted Channel ID (4)  |
+----------+-------------------------+-------------------------------+
```
通知對端會話即將關閉。接收方收到此幀後，不再發送任何新的 `OPEN_CHANNEL`。
**Draining 與 Race Condition 消除**: 
`Last Accepted Channel ID` 明確劃定了排空 (Draining) 邊界。對於接收方來說，所有已發出且 Channel ID **大於**此值的 `OPEN_CHANNEL` 請求，皆視為被發送方**隱式拒絕 (Implicitly Rejected)**，必須立即釋放關聯資源；小於或等於此值的通道允許繼續 Draining 直至 `FIN`。
**錯誤碼矩陣:**
* `0x00`: 優雅退出 (如伺服器進入維護、憑證即將到期)。
* `0x01`: 內部錯誤。
* `0x02`: 協議違規 (如未知的 Frame Type)。
* `0x03`: 流量控制溢出 (Flow Control Overflow)。
* `0x04`: 解密/完整性違規。
* `0x05`: 客戶端授權失敗 (Auth Failed)。

#### `0xFF` PAD (流量特徵填充)
`[Type(1)] [Pad Length (2 bytes)] [Padding Bytes...]`
傳輸策略 (Transport Policy) 可插入此幀以湊齊特定的 Record 大小。接收方**必須安全忽略**。**此幀絕對不消耗任何 Channel 或 Session 的 Flow Control Credit**，它只受制於控制面下發的獨立 Shaping/Padding Budget。

---

## 6. 錯誤處理與恢復矩陣 (Error Handling & Recovery Matrix)

| 錯誤情境 (Error Scenario) | 所在層級 | 伺服器動作 (Server Action) | 客戶端恢復策略 (Client Recovery) |
| :--- | :--- | :--- | :--- |
| **Handshake Length Mismatch** | Handshake | 靜默丟棄 (Silent Drop)。 | Retry Different Node |
| **Handshake AEAD / MAC Fail** | Handshake | 靜默丟棄 (Silent Drop)。 | Retry Different Node |
| **Client Auth Fail (Token 錯誤)**| Auth     | 發送 `GOAWAY (Code: 0x05)`，然後 `Abort Transport`。 | Require Config Refresh (Do Not Retry) |
| **Record Header De-obfuscate Fail** | Record | 視為流污染。立即硬關閉 (`Abort Transport`)，無錯誤幀。 | Retry Same Node (If transient) / Switch Node |
| **Record AEAD MAC Fail / 亂序** | Record | 視為流污染。立即硬關閉 (`Abort Transport`)，無錯誤幀。 | Retry Same Node (If transient) / Switch Node |
| **Channel Flow Control Overflow** | Channel | 發送 `RESET_CHANNEL (Code: 0x03)`。 | Retry Channel (App layer decision) |
| **Unknown Frame Type** | Control | 發送 `GOAWAY (Code: 0x02)`，進入 Draining。 | Retry Same Node (Version mismatch check) |
| **Session Flow Control Overflow** | Control | 發送 `GOAWAY (Code: 0x04)`，然後 `Abort Transport`。 | Retry Same Node (Rate limit applied) |
| **Server Maintenance/Epoch Expiring**| Control | 發送 `GOAWAY (Code: 0x00)`，進入 Draining。 | Switch Node (Graceful shift) |