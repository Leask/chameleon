# Chameleon 網路通訊協議規格書 v2.0
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
* 所有整數欄位均使用**大端序 (Network Byte Order)**。
* 基礎承載假設為**無損、保序的可靠位元組流**（Reliable in-order byte stream）。

---

## 2. 密碼學原語與密鑰導出 (Cryptographic Primitives)

* **Handshake Pattern**: Noise `NK` (`Noise_NK_25519_ChaChaPoly_BLAKE2s`)
* **流量金鑰導出 (Traffic Keys)**: 握手完成後，導出以下金鑰：
  * `Client_to_Server_Key`, `Server_to_Client_Key` (32 Bytes, 用於 AEAD 加密 Payload)
  * `Client_to_Server_HP_Key`, `Server_to_Client_HP_Key` (32 Bytes, 用於 Header 混淆)

---

## 3. 握手層規格 (Handshake Layer)

採用**固定長度無前綴 (Fixed-Length, Prefix-Free)** 密文設計。

### 3.1 握手請求 (Client -> Server)
發送 Noise `NK` 第一條訊息 (`-> e, d, es`)。

**1. 構造明文 Payload (64 Bytes 固定長度):**
```text
+-------------------+-------------------+-----------------------------------+
| Version (1 byte)  | Caps (4 bytes)    | Padding (59 bytes, 填 0x00)       |
+-------------------+-------------------+-----------------------------------+
```
* `Version`: `0x01`。
* `Caps` (Capabilities): 位元遮罩。
  * `Bit 0`: 支援 Datagram Channel。
  * `Bit 1-31`: 保留，必須設為 0。未知 Bit 必須被忽略。

**2. 密文傳輸 (112 Bytes 固定長度):**
* `e` (Ephemeral Pubkey): 32 Bytes
* `payload` (Ciphertext): 64 Bytes + 16 Bytes MAC = 80 Bytes
**總長度**: 112 Bytes。

**伺服器行為**: 盲讀 112 Bytes。如果 AEAD 解密失敗，**必須保持絕對靜默 (Silent Drop)**，不返回任何 Chameleon 錯誤幀。伺服器可選擇切斷底層連線或將 Socket 轉交 (Fallback)。

### 3.2 握手回應 (Server -> Client)
發送 Noise `NK` 第二條訊息 (`<- e, ee`)。

**1. 紀元憑證 (Epoch Cert) 結構 (112 Bytes):**
```text
+-----------------------------------+-----------------------------------+
| Server Static Pubkey (32 bytes)   | Epoch_Window_Start (8 bytes, 秒數)|
+-----------------------------------+-----------------------------------+
| Epoch_Window_End (8 bytes, 秒數)  | Ed25519 Control Plane Sig (64 b)  |
+-----------------------------------+-----------------------------------+
```
*(提供防重放的時間新鮮度視窗)*

**2. 構造明文 Payload (128 Bytes 固定長度):**
```text
+-------------------+-------------------+-------------------+---------------+
| Version (1 byte)  | Selected Caps (4) | Epoch Cert (112 b)| Pad (11 bytes)|
+-------------------+-------------------+-------------------+---------------+
```
* `Selected Caps`: 伺服器同意啟用的能力交集。

**3. 密文傳輸 (176 Bytes 固定長度):**
* `e` (Ephemeral Pubkey): 32 Bytes
* `payload` (Ciphertext): 128 Bytes + 16 Bytes MAC = 144 Bytes
**總長度**: 176 Bytes。

---

## 4. 記錄層規格 (Record Layer)

### 4.1 隱式序號與鎖步防重放 (Strict Lockstep Sequence)
* 雙方各自維護 `SendSeq` 與 `RecvSeq` (初始為 0)。
* **絕對順序保證**: 因為底層是可靠流，`RecvSeq` 必須與收到的 Record 嚴格一一對應。**不允許亂序，不允許跳號**。
* `Nonce = ChaCha_IV XOR Pad_Zero(Sequence_Number)`。
* 任何 AEAD MAC 驗證失敗，意味著位元組流已被污染或發生了未經授權的篡改，**必須立即硬關閉底層傳輸 (Hard Close)**。

### 4.2 Record 結構與頭部混淆 (Header Protection)
```text
+--------------------------+-----------------------+
| Obfuscated Length (2)    | Obfuscated Flags (1)  |  <-- 3 Bytes
+--------------------------+-----------------------+
| AEAD Ciphertext (Payload + 16 bytes MAC)         |  
+--------------------------------------------------+
```
* `Flags` (1 byte): `[Key Phase (1 bit)] [Reserved (7 bits, 須為 0)]`。
* `Mask = ChaCha20(HP_Key, Ciphertext_Sample[0..15])[0..2]`。
* `Obfuscated_Header = (Length || Flags) XOR Mask`。

---

## 5. 幀格式規格 (Frame Formats)

### 5.1 通道層 Frames (Channel Lifecycle & Multiplexing)

#### `0x01` OPEN_CHANNEL (開啟通道)
```text
+---------+--------------+---------+-----------------+--------+------------+-------+
| Type(1) | Channel ID(4)| Kind(1) | Init Window (4) | ATYP(1)| Target(Var)| Port(2)|
+---------+--------------+---------+-----------------+--------+------------+-------+
```
* `Channel ID`: 發起方保證全域唯一（Client奇數，Server偶數）。
* `Kind`: `0x01` (TCP Stream), `0x02` (UDP Datagram)。
* `Init Window`: 允許對端發送的初始位元組數 (Channel-level Flow Control)。

#### `0x02` OPEN_ACK (確認通道開啟)
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Init Window (4)   |
+----------+-----------------+-------------------+
```
接收方確認資源已分配，發起方收到此幀後才可發送 `DATA`。

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

#### `0x05` RESET_CHANNEL (強制重置通道)
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Error Code (4)    |
+----------+-----------------+-------------------+
```
立即廢棄該 Channel 的所有未讀資料與狀態。不會影響 Session 內的其他 Channel。

### 5.2 流量控制 Frames (Flow Control)

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
* **發送方行為**: 發送 `KEY_UPDATE` 幀，從**下一個 Record 開始**，發送方將 `Key Phase` 位元翻轉，並使用 `Next_Traffic_Key` 加密。
* **KDF**: `Next_Traffic_Key = HKDF-Expand(Current_Traffic_Key, "rekey", 32)`。
* **接收方行為**: 收到 `KEY_UPDATE` 後，計算出 `Next_Traffic_Key`。當收到下一個 Record 發現 `Key Phase` 翻轉時，使用新金鑰解密。接收方也應盡快發送自己的 `KEY_UPDATE` 以完成雙向輪換。

#### `0x13` GOAWAY (會話級致命錯誤)
`[Type(1)] [Error Code (4 bytes)]`。
通知對端會話即將關閉。不再接受新的 `OPEN_CHANNEL`，現有通道允許排空 (Draining)。

#### `0xFF` PAD (流量特徵填充)
`[Type(1)] [Pad Length (2 bytes)] [Padding Bytes...]`
傳輸策略 (Transport Policy) 可插入此幀以湊齊特定的 Record 大小。接收方**必須安全忽略**。

---

## 6. 錯誤處理矩陣 (Error Handling Matrix)

| 錯誤情境 (Error Scenario) | 所在層級 | 是否可觀測 | 協議行為 (Protocol Action) | 關閉粒度 |
| :--- | :--- | :--- | :--- | :--- |
| **Handshake Length Mismatch** | Handshake | 否 | 靜默丟棄 (Silent Drop)，斷開底層或 Fallback。 | Session |
| **Handshake AEAD / MAC Fail** | Handshake | 否 | 靜默丟棄 (Silent Drop)，斷開底層或 Fallback。 | Session |
| **Record Header De-obfuscate Fail** | Record | 是 (TCP RST) | 硬關閉底層連線 (Transport Close)，無錯誤幀。 | Session |
| **Record AEAD MAC Fail / 亂序** | Record | 是 (TCP RST) | 視為流污染。硬關閉底層連線，無錯誤幀。 | Session |
| **Channel Flow Control Overflow** | Channel | 是 (RESET Frame)| 發送 `RESET_CHANNEL (Code: 0x03)`。 | **Channel** |
| **Unknown Frame Type** | Control | 是 (GOAWAY) | 發送 `GOAWAY (Code: 0x02)`，進入 Draining。 | Session |
| **Session Flow Control Overflow** | Control | 是 (GOAWAY) | 發送 `GOAWAY (Code: 0x04)`，硬關閉。 | Session |