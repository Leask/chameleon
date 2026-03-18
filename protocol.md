# Chameleon 網路通訊協議規格書 v14.0
*(Chameleon Network Protocol Specification)*

**狀態**: 草案 (Draft)
**標籤**: 嚴謹二進位規格 (Strict Binary Specification), 語言無關 (Language-Agnostic)

---

## 1. 協議概述 (Overview)

Chameleon 是一個運行於可靠位元組流（Reliable Byte Stream，如 QUIC Stream、TCP、WebSocket）之上的安全隧道與多路復用協議。

協議分為四層：
1. **握手層 (Handshake Layer)**: 負責伺服器側信任建立 (Server-side Trust Establishment)、金鑰交換與能力協商。客戶端授權在握手後的 `CLIENT_AUTH` 階段完成。
2. **記錄層 (Record Layer)**: 負責加密、嚴格防重放、定界與頭部混淆。
3. **通道層 (Channel Layer)**: 負責在單一加密會話中多路復用多個邏輯連線。
4. **控制層 (Control Layer)**: 負責會話級別的信令、金鑰輪換與錯誤恢復。

### 1.1 位元組序與約定
* 所有整數欄位（16-bit, 32-bit, 64-bit）均使用**大端序 (Big-Endian, Network Byte Order)**。
* 基礎承載假設為**無損、保序的可靠位元組流**（Reliable in-order byte stream）。
* 所有的密碼學金鑰與雜湊值以位元組陣列 (Byte Array) 處理。

### 1.2 密碼學標準參考 (Cryptographic Standards)
* **ChaCha20-Poly1305**: RFC 8439
* **Noise Protocol Framework**: Noise Specification (Revision 34)
* **X25519**: RFC 7748
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
+-------------------+-------------------+-------------------+-------------------------------+
| Version (1 byte)  | Offered Caps (4)  | Required Caps (4) | Padding (55 bytes, 填 0x00)   |
+-------------------+-------------------+-------------------+-------------------------------+
```
* `Version`: 必須為 `0x01`。
* `Offered Caps` (4 bytes): 客戶端支援的能力位元遮罩（Bit 0: Datagram，其餘保留填 0）。
* `Required Caps` (4 bytes): 客戶端**強制要求**伺服器支援的能力位元遮罩。伺服器若無法滿足，**不會主動發送錯誤**，只會在回應中返回其實際支援的 `Selected Caps`。客戶端負責進行本地比對與中斷。

**2. 密文傳輸 (112 Bytes 固定長度):**
Noise `NK` 第一條訊息的序列化：
* `e` (Ephemeral Pubkey, 明文發送): 32 Bytes
* `es` (執行 DH，產出金鑰，無輸出 bytes)
* `payload` (使用產生的金鑰進行 AEAD 加密): 64 Bytes + 16 Bytes MAC = 80 Bytes
**總長度**: `32 + 80 = 112 Bytes`。

**伺服器行為**: 盲讀 112 Bytes。如果 AEAD 解密失敗，或 `Version` 欄位不為 `0x01` (版本不匹配)，**必須保持絕對靜默 (Silent Drop)**。這表示伺服器可選擇直接中斷底層連線（如 TCP RST），或將流靜默轉交給本機服務（Fallback），但**絕對不可 (MUST NOT)** 返回任何 Chameleon 錯誤幀。

### 3.2 握手回應 (Server -> Client)
**Noise Transcript (Responder -> Initiator)**: `<- e, ee`

**1. 紀元憑證 (Epoch Cert) 結構 (120 Bytes):**

為了與控制面的防回滾機制對齊，`Epoch Cert` 的簽章必須基於嚴格的 `EpochCertSigMessage`。

```text
EpochCertBody = 
    Server Static Pubkey (32 bytes) || 
    Epoch_Window_Start (8 bytes, 秒數) || 
    Epoch_Window_End (8 bytes, 秒數) || 
    Signer Key ID (8 bytes)

EpochCertSigMessage =
    "chameleon-epoch-cert-v1" || EpochCertBody
```

* **線上二進位佈局**: `EpochCertBody` (56 Bytes) || `Ed25519 Control Plane Sig` (64 Bytes)。
*(提供防重放的時間新鮮度視窗與兩級信任鏈綁定)*

**2. 構造明文 Payload (128 Bytes 固定長度):**
```text
+-------------------+-------------------+-------------------+---------------+
| Version (1 byte)  | Selected Caps (4) | Epoch Cert (120 b)| Pad (3 bytes) |
+-------------------+-------------------+-------------------+---------------+
```
* `Selected Caps`: 伺服器同意啟用的能力。**伺服器行為 (MUST)**: 此值必須是客戶端 `Offered Caps` 的子集 (Subset)。若伺服器無法滿足 `Required Caps`，仍會返回其支援的子集。
* **客戶端行為**: 收到後比對，若 `(Required Caps & ~Selected Caps) != 0`，代表關鍵能力未被滿足，客戶端必須主動 `Abort Transport`，並標記為 Capability Mismatch。此檢查與中斷行為由**客戶端全權負責**，伺服器不再發出任何協議層級的錯誤信號。

**3. 密文傳輸 (176 Bytes 固定長度):**
Noise `NK` 第二條訊息的序列化：
* `e` (Ephemeral Pubkey, 參與後續加密): 32 Bytes
* `ee` (執行 DH)
* `payload` (使用更新後的金鑰進行 AEAD 加密): 128 Bytes + 16 Bytes MAC = 144 Bytes
**總長度**: `32 + 144 = 176 Bytes`。

### 3.3 Epoch Cert 驗證演算法 (Epoch Cert Verification Algorithm)
接收到伺服器的 `Epoch Cert` 後，客戶端**必須 (MUST)** 執行以下嚴格的驗證步驟。任何一步失敗，必須立即中斷連線並丟棄金鑰：

1. **Signer Trust Root 校驗 (兩級 PKI)**: 客戶端必須預先透過 OOB 獲取由離線 Root CA 簽發的線上控制面簽發金鑰 (`CPSK`, Control Plane Signing Key) 憑證。客戶端提取 `Epoch Cert` 中的 `Signer Key ID`，從本地信任庫中找出對應的有效 `CPSK` 公鑰。使用此 `CPSK` 公鑰驗證 `Epoch Cert` 後 64 Bytes 的 `Ed25519 Control Plane Sig`。簽名的 Message 內容必須為前述定義的 `EpochCertSigMessage`。
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
* `Length` (2 bytes): 表示整個 Ciphertext 的總長度（**包含 16 Bytes 的 MAC Tag**）。即 `Payload Length + 16`。最大允許值 `16384` (16KB)。最小允許值 `16` (Payload 為空)。若超出此範圍，**必須立即 `Abort Transport`**。
* `Flags` (1 byte): `[Key Phase (1 bit)] [Reserved (7 bits, 須為 0)]`。若保留位不為 0，視為協議違規，**必須立即 `Abort Transport`**。
* **AEAD 附加認證資料 (AAD)**: 反混淆後獲得的 3 Bytes `Plain_Header` (`Length || Flags`) **必須 (MUST)** 作為 `AAD` 輸入到 AEAD (ChaCha20-Poly1305) 的解密/加密函數中。這確保了 `Key Phase` 和 `Length` 無法被惡意篡改。

* **精確混淆演算法 (發送方)**:
  1. 構造 `Plain_Header = Length || Flags`。
  2. 以 `AAD = Plain_Header` 執行 AEAD 加密，得到 `Ciphertext`。
  3. `Sample` = 取 `Ciphertext` 的前 12 Bytes（不足 12 Bytes 則末尾補 0x00）。
  4. 使用標準 ChaCha20 區塊加密產生 Mask：
     `Key` = `HP_Key` (32 Bytes), `Nonce` = `Sample` (12 Bytes), `Block Counter` = 0。截取輸出 Keystream 的前 3 Bytes 作為 `Mask`。
  5. `Obfuscated_Header = Plain_Header XOR Mask`。

* **解析器行為 (接收方)**:
  1. 讀取 3 Bytes 的 `Obfuscated_Header`。
  2. 預讀 (Peek) 後續的 12 Bytes 作為 `Sample`（不足則補 0x00）。
  3. 根據 `HP_Key` 與 `Sample` 產出 `Mask`，反解得到 `Plain_Header`。
  4. 驗證 `Length` 與 `Flags` 合法性。
  5. 讀取完整的 `Ciphertext`。
  6. 以 `AAD = Plain_Header` 執行 AEAD 解密。若 MAC 校驗失敗，視為流污染，**立即 `Abort Transport`**。

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
   * **最大接收位元組 (Max Inbound Bytes)**: `2048` Bytes (**純 Chameleon 協議層位元組**，僅計算 3-Byte Header + Ciphertext Length，不包含底層 TCP/QUIC 開銷)。
   * **握手前最大記錄數 (Max Records Before Auth)**: `2`。
   任何越界行為必須立即觸發 `Abort Transport`。
3. **驗證演算法 (Auth Scheme 0x01: Ed25519 Transcript Binding)**:
   * `Credential ID`: 作為索引，用於在控制面下發的 **Auth Realm Manifest** (客戶端憑證清單) 中查找對應的 Client Ed25519 Public Key。伺服器必須基於此全域一致的 Manifest 進行驗證，不得依賴私有資料庫。
   * `Proof Length`: 固定為 `64` (Bytes)。全局最大允許長度為 `1024` Bytes。
   * **Domain Separation 與綁定**: 客戶端使用其 Ed25519 私鑰，對特定的 Context String 與**當前連線的 Noise 握手 Transcript Hash** 進行拼接後簽章。
     `Proof = Sign( Private_Key, "chameleon-auth-v1" || h )`
     (其中 `"chameleon-auth-v1"` 為 ASCII 編碼，不含 null，共 17 Bytes；`h` 即 Noise 狀態機握手完成時輸出的 32 Bytes Hash 變數)。
   * **安全性**: 此設計將客戶端的長期身份 (`Credential ID`) 與這一次特定的物理連線 (`Transcript Hash`) 進行了上下文隔離的密碼學綁定，徹底杜絕了 Auth Token 被跨連線、跨協議重放的可能。
4. **錯誤處置**:
   * 若 `Auth Scheme` 不支援，伺服器必須發送 `GOAWAY (Code: 0x02 Protocol Violation)` 並硬關閉連線。
   * 若支援但驗證失敗（ID 不存在、簽名錯誤、或該 ID 已被撤銷），伺服器必須發送 `GOAWAY (Code: 0x05 Auth Failed)` 並硬關閉連線。

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
  * **ID 耗盡防護 (Exhaustion Watermark)**: 當客戶端下一個可分配的 `Channel ID` 超過 `2^30 - 100` 時，客戶端必須主動發送 `GOAWAY (Code: 0x00)` 並進入 Draining 狀態。若伺服器方向的 `Channel ID` 逼近上限，伺服器也必須發送 `GOAWAY (Code: 0x00)`，交由 Session Pool Manager 建立新的替代 Session。
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

**狀態機定義 (Key Rotation State Machine)**:
每個方向 (Tx/Rx) 獨立維護以下狀態：`current_generation` (整數), `next_generation_prepared` (布林), `current_key_phase` (0 或 1)。

1. **發送方 (Sender)**:
   * 發送 `KEY_UPDATE` 幀。此時**僅標記準備就緒**，不會立即切換發送金鑰。
   * 從**下一個 Record 開始**，發送方將 `current_key_phase` 位元翻轉，並將 `current_generation` 加 1。
   * 新的發送 Record 必須使用新一代的金鑰套件 (`Next_Traffic_Key`, `Next_HP_Key`) 進行加密與混淆。
2. **KDF (Key Derivation Function)**: 
   每次 generation 推進時，同時更新 Traffic Key 與 Header Protection (HP) Key：
   * `Next_Traffic_Key = HKDF-Expand(Current_Traffic_Key, "traffic_rekey", 32)`
   * `Next_HP_Key = HKDF-Expand(Current_HP_Key, "hp_rekey", 32)`
3. **接收方 (Receiver)**:
   * 收到 `KEY_UPDATE` 幀後，將 `next_generation_prepared` 設為 True，並預先計算出下一代的兩把新金鑰，但**不切換**當前的解密狀態。
   * 接收方持續讀取下一個 Record。當解析頭部時發現 `Key Phase` 位元發生翻轉：
     * **驗證**: 若 `Key Phase` 翻轉但 `next_generation_prepared` 為 False，視為協議違規 (`Abort Transport`)。
     * **提交 (Commit)**: 使用準備好的新一代金鑰 (`Next_HP_Key`, `Next_Traffic_Key`) 進行反混淆與解密。
     * 若解密成功，將 `current_generation` 加 1，`current_key_phase` 翻轉，並將舊 generation 的金鑰徹底銷毀。
   * 接收方完成切換後，也應盡快發送自己的 `KEY_UPDATE` 幀以完成雙向輪換。
* **安全約束**: 為避免歧義，連續兩次 `KEY_UPDATE` 的發送之間必須相隔至少 1000 個 Records 或 1 分鐘，接收方若在冷卻期內收到連續的 `KEY_UPDATE`，必須 `GOAWAY 0x02`。
* **強制輪換觸發條件 (Mandatory Triggers)**: 當逼近以下任一門檻時，發送方必須主動發送 `KEY_UPDATE`：
  1. `MAX_RECORDS_PER_GENERATION = 2^30` (單一方向、單一 Generation 內的加密 Record 總數)。
  2. `MAX_BYTES_PER_GENERATION = 2^40` (1 TB, 單一方向、單一 Generation 內的 Ciphertext 位元組總數，包含 3-Bytes Obfuscated Header 與 MAC Tag)。
  3. `MAX_KEY_AGE = 1` 小時 (從該 Generation 的 Commit 時間點起算)。
  冷卻期不適用於強制輪換。當對端長期不 rekey 時，本端可主動發起輪換。

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

錯誤處理被劃分為四種**故障分類 (Fault Category)**，以確保運營側能精確定位問題：
* **[S] Security / Trust Failure**: 信任鏈斷裂、安全綁定失敗或金鑰失配。
* **[C] Config / Freshness Fault**: 控制面資料失效或尚未同步。
* **[P] Protocol / Version Mismatch**: 雙方能力或協議版本不匹配。
* **[I] Implementation / Credential Fault**: 本地邏輯、憑證生成或資源控制錯誤。

| 錯誤情境 (Error Scenario) | 檢測方 (Detected By) | 故障分類 (Category) | 本地動作 (Local Action) | 對端可見信號 (Peer-Visible Signal) | 重試/刷新策略 (Retry / Refresh Policy) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Handshake Length / AEAD Fail** | Server | [P] / [S] | Abort Transport / Fallback | None (Silent Drop) | Client: Retry Different Node |
| **Epoch Cert Signature Invalid**| Client | [S] | Discard keys + Abort | None | Refresh control-plane config first |
| **Responder Key Mismatch**    | Client | [S] | Abort Transport | None | Hard security failure, stop route |
| **Epoch Cert Not Yet Valid**  | Client | [C] | Abort Transport | None | Check local clock |
| **Epoch Cert Expired**        | Client | [C] | Abort Transport | None | Freshness lag; Retry different node, refresh config |
| **Auth Realm Sig Invalid / Expired**| Server | [C] / [S] | Fail-Closed (Reject Auth) | `GOAWAY (0x01)` | Verifier polls control-plane for sync |
| **Auth Realm Rollback Detected**| Server | [S] | Fail-Closed (Reject Auth) | `GOAWAY (0x01)` | Security Incident; Verifier halts updates |
| **Capability Mismatch**       | Client | [P] | Abort Transport | None | Feature mismatch; Do not retry same capability set |
| **Client Auth Fail (Token 錯)**| Server | [I] / [C] | Abort Transport | `GOAWAY (0x05)` | Client: Check config (Refresh possible) |
| **Unsupported Auth Scheme**   | Server | [P] | Hard Close | `GOAWAY (0x02)` | Upgrade Client |
| **Provisional Bound Exceeded**| Server | [I] | Abort Transport | None (Pre-Auth Drop) | App Bug / Rate Limit Triggered |
| **Record Header/MAC Fail**    | Both | [S] | Abort Transport | None | Retry Same Node (限1次) / Switch |
| **Channel Flow Control Over** | Both | [I] | Discard Channel State | `RESET_CHANNEL (0x03)` | Retry Channel (App Layer) |
| **Session Flow Control Over** | Both | [I] | Abort Transport | `GOAWAY (0x03)` | Retry Same Node (帶退避算法) |
| **Server Maintenance/ID Exhaust**| Both | [C] | Wait for Drain | `GOAWAY (0x00)` | Reassign New Channels to Pool |

**關於 `Client Auth Fail` 的運營避險規則**:
客戶端收到 `GOAWAY 0x05` 時，可以嘗試一次 Config Refresh。若連續 Refresh 後仍然發生 `0x05`，必須認知到以下兩種可能之一：
1. **客戶端本地憑證已撤銷 (Revoked) 或實作錯誤 (Implementation Bug)**。
2. **邊緣節點 (Verifier) 持有過期或未同步的 Auth Realm Manifest (Freshness Lag)**。
無論是哪種情況，客戶端本地的持續重試與 Refresh 均無法解決問題，因此客戶端**必須停止無意義的重試並暫停控制面輪詢**，避免對控制面發起雪崩式請求。