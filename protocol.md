# Chameleon 網路通訊協議規格書 v1.0
*(Chameleon Network Protocol Specification)*

**狀態**: 草案 (Draft)
**標籤**: 嚴謹二進位規格 (Strict Binary Specification), 語言無關 (Language-Agnostic)

---

## 1. 協議概述 (Overview)

Chameleon 是一個運行於可靠位元組流（Reliable Byte Stream，如 QUIC Stream、TCP、WebSocket）之上的安全隧道與多路復用協議。

協議分為四層：
1. **握手層 (Handshake Layer)**: 負責身份認證與金鑰交換。
2. **記錄層 (Record Layer)**: 負責加密、防重放、定界與頭部混淆。
3. **通道層 (Channel Layer)**: 負責在單一加密會話中多路復用多個邏輯連線。
4. **控制層 (Control Layer)**: 負責會話級別的信令與錯誤處理。

### 1.1 位元組序與類型約定
* 所有整數欄位（16-bit, 32-bit, 64-bit）均使用**大端序 (Big-Endian, Network Byte Order)**。
* 密碼學金鑰與雜湊值以位元組陣列 (Byte Array) 處理。

---

## 2. 密碼學原語 (Cryptographic Primitives)

協議強制依賴以下密碼學套件，不提供協商空間以減少指紋：
* **Key Exchange (KEX)**: X25519
* **AEAD Cipher**: ChaCha20-Poly1305
* **Hash**: BLAKE2s
* **Signatures**: Ed25519 (僅用於控制面簽發 Epoch Cert，不在握手熱路徑即時計算)
* **Handshake Pattern**: Noise `IK` (`Noise_IK_25519_ChaChaPoly_BLAKE2s`)

---

## 3. 握手層規格 (Handshake Layer)

握手層採用嚴格的**固定長度無前綴 (Fixed-Length, Prefix-Free)** 密文設計。

### 3.1 握手請求 (Client -> Server)
客戶端發送 Noise `NK` 模式（或等效的單向靜態公鑰認證模式）的第一條訊息。
在此模型中，**客戶端預先知道伺服器的靜態公鑰 (Server Static Key)，但伺服器不預先知道客戶端的身份**。

**1. 構造明文 Payload (64 Bytes 固定長度):**
```text
+-------------------+-------------------+-----------------------------------+
| Version (1 byte)  | Caps (4 bytes)    | Padding (59 bytes, 填 0x00)       |
+-------------------+-------------------+-----------------------------------+
```
* `Version`: 必須為 `0x01`。
* `Caps`: 能力位元遮罩，目前保留為 `0x00000000`。

**2. 生成網路上傳輸的密文 (112 Bytes 固定長度):**
基於 Noise `NK` 產生的二進位序列：
* `e` (Ephemeral Pubkey): 32 Bytes
* `payload` (Payload Ciphertext): 64 Bytes + 16 Bytes MAC = 80 Bytes
*(註：此長度僅為概念計算，實際採用 `NK` 模式時，首包長度可能不同，核心約束是**長度必須固定不變**)*

**伺服器行為**: 伺服器從位元組流中精確讀取規定長度的 Bytes。如果 AEAD 認證失敗，**必須 (MUST) 保持絕對靜默 (Silent Drop)**。這表示伺服器可選擇直接中斷底層連線（如 TCP RST），或將流靜默轉交給本機服務（Fallback），但**絕對不可 (MUST NOT)** 返回任何 Chameleon 錯誤幀。

### 3.2 握手回應 (Server -> Client)
伺服器發送 Noise 的第二條訊息。

**1. 預簽發紀元憑證 (Epoch Cert) 結構 (112 Bytes):**
```text
+-----------------------------------+-----------------------------------+
| Server Static Pubkey (32 bytes)   | Epoch_Window_Start (8 bytes, 秒數)|
+-----------------------------------+-----------------------------------+
| Epoch_Window_End (8 bytes, 秒數)  | Ed25519 Control Plane Sig (64 b)  |
+-----------------------------------+-----------------------------------+
```
*(註：不再將此視為時鐘校準工具，而是作為重放防護的新鮮度 (Freshness) 視窗)*

**2. 構造明文 Payload (128 Bytes 固定長度):**
```text
+-------------------+-------------------+-------------------+---------------+
| Version (1 byte)  | Caps (4 bytes)    | Epoch Cert (112 b)| Pad (11 bytes)|
+-------------------+-------------------+-------------------+---------------+
```

**3. 生成網路上傳輸的密文 (176 Bytes 固定長度):**
* `e` (Ephemeral Pubkey): 32 Bytes
* `payload` (Payload Ciphertext): 128 Bytes + 16 Bytes MAC = 144 Bytes
**總長度**: `32 + 144 = 176 Bytes`。

客戶端解密後，根據 `Not_Before_UTC` 與本地時鐘計算 `Clock_Offset`，後續所有時間計算依賴 `LocalTime + Clock_Offset`。

---

## 4. 記錄層規格 (Record Layer)

握手完成後，傳輸階段被封裝為加密記錄 (Encrypted Records)。

### 4.1 隱式序號與防重放 (Implicit Sequence Number)
* 雙方各自維護發送序號 `SendSeq` 與接收序號 `RecvSeq`，初始值均為 `0`，每次成功處理 Record 後遞增 `1`。序號為 64-bit 無號整數。
* AEAD 的 `Nonce` 計算方式：`Nonce = ChaCha_IV XOR Pad_Zero(Sequence_Number)`。
* 序號**不透過網路傳輸**。

### 4.2 Record 頭部混淆 (Header Obfuscation)
為隱藏 Record 長度，3 Bytes 的 Header 必須與生成的 Mask 進行 XOR 運算。
* `Sample` 取 Ciphertext 的前 16 Bytes。
* `Mask = ChaCha20(Header_Protection_Key, Sample)[0..2]`。
* `Obfuscated_Header = Header XOR Mask`。

### 4.3 網路上傳輸的 Record 結構
```text
+--------------------------+-----------------------+
| Obfuscated Length (2)    | Obfuscated Phase (1)  |  <-- 3 Bytes (明文頭與 Mask XOR 後的結果)
+--------------------------+-----------------------+
| AEAD Ciphertext (長度由解開的 Length 決定)       |  <-- 包含明文 Payload
+--------------------------------------------------+
| AEAD MAC / Tag (16 bytes)                        |  <-- ChaCha20-Poly1305 MAC
+--------------------------------------------------+
```
* `Length` (2 bytes): 表示 Payload (Ciphertext) 的位元組數。最大允許值 `16384` (16KB)。
* `Phase` (1 byte): Key Phase，初始為 `0x00`。
* `Ciphertext`: 封裝一個或多個 Chameleon Frames。

---

## 5. 幀格式規格 (Frame Formats)

Record 的 Payload 明文由一個或多個 TLV 格式的 Frame 組成。發送方可將多個 Frame 打包進同一個 Record 以對齊特定的 Bucket 尺寸（例如：將 `DATA` 和 `PAD` 放在一起）。

所有 Frame 遵循 `[Type (1 byte)] [Payload (變長)]` 的基礎結構。

### 5.1 通道層 Frames (Channel Layer)

#### `0x01` OPEN_CHANNEL (開啟通道)
開啟一個新的雙向邏輯通道，並宣告初始的 Flow Control Window。
```text
+----------+-----------------+------------------+--------+-------------+---------+
| Type(1)  | Channel ID (4)  | Init Window (4)  | ATYP(1)| Address(Var)| Port(2) |
+----------+-----------------+------------------+--------+-------------+---------+
```
* `Channel ID`: 必須全域唯一（客戶端發起的 ID 為奇數，伺服器為偶數）。
* `Init Window`: 允許對端發送的初始位元組數 (Bytes)。
* `ATYP` (SOCKS5 標準):
  * `0x01`: IPv4 (Address 長 4 Bytes)
  * `0x03`: Domain (Address 首位元組為長度 N，後接 N Bytes 域名)
  * `0x04`: IPv6 (Address 長 16 Bytes)

#### `0x02` DATA (通道數據)
```text
+----------+-----------------+-------------------+-----------------------+
| Type(1)  | Channel ID (4)  | Data Length (2)   | Data (Data Length)    |
+----------+-----------------+-------------------+-----------------------+
```

#### `0x03` WINDOW_UPDATE (更新流量控制視窗)
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Added Credit (4)  |
+----------+-----------------+-------------------+
```
增加對端可以發送的資料量 (Bytes)。

#### `0x04` CLOSE_CHANNEL (關閉通道)
```text
+----------+-----------------+-------------------+
| Type(1)  | Channel ID (4)  | Error Code (4)    |
+----------+-----------------+-------------------+
```
正常關閉 `Error Code = 0x00000000`。

### 5.2 控制層 Frames (Control Layer)

#### `0x10` PING / `0x11` PONG (保活與測速)
```text
+----------+-------------------------+
| Type(1)  | Opaque Data (8 bytes)   |
+----------+-------------------------+
```
發送方可將 Timestamp 放於 Opaque Data 內，接收方 `PONG` 必須原樣回傳。

#### `0x12` KEY_UPDATE (金鑰輪換)
```text
+----------+-------------------------+
| Type(1)  | Next Phase (1 byte)     |
+----------+-------------------------+
```
通知對端：我發送的下一個 Record 將使用 `Next Phase` 的金鑰加密。接收方不需回覆，但必須準備使用新金鑰解密後續資料。

#### `0x13` GOAWAY (致命錯誤/優雅退出)
```text
+----------+-------------------------+
| Type(1)  | Error Code (4 bytes)    |
+----------+-------------------------+
```
一旦發送，不再接受新的 `OPEN_CHANNEL`。
**錯誤碼矩陣:**
* `0x00`: 優雅退出 (如伺服器進入維護、憑證即將到期)。
* `0x01`: 內部錯誤。
* `0x02`: 協議違規 (如未知的 Frame Type)。
* `0x03`: 流量控制溢出 (Flow Control Overflow)。

#### `0xFF` PAD (流量填充)
```text
+----------+-------------------------+-------------------------+
| Type(1)  | Pad Length (2 bytes)    | Padding Bytes (Var)     |
+----------+-------------------------+-------------------------+
```
`Padding Bytes` 的內容可以是全 `0x00`，因為外層 Record 會進行 AEAD 加密，網路上將呈現為完美的隨機數。接收方解析 Frame 時**必須完全忽略**此幀。

---

## 6. 狀態與錯誤處理規範 (Error Handling)

此規範對開發者具備強約束力：

1.  **握手階段 (Pre-Auth) 錯誤**:
    在接收並校驗 160 Bytes (Client) 或 176 Bytes (Server) 完成之前，所有的網路層中斷、長度不足、或解密/MAC 驗證失敗，**嚴禁發送任何資料**。必須直接中斷連接或轉交 fallback。
2.  **記錄層解密錯誤 (Record Decryption Failure)**:
    AEAD MAC 驗證失敗、頭部反混淆失敗。**嚴禁發送 GOAWAY**。直接切斷底層位元組流（如 TCP RST 或 QUIC CONNECTION_CLOSE）。
3.  **邏輯錯誤 (Logical Errors)**:
    解密成功，但解析 Frame 出錯（如 `Data Length` 大於剩餘 Buffer，或操作了未開啟的 `Channel ID`）。**必須發送 GOAWAY (Code 0x02)**，然後關閉連接。