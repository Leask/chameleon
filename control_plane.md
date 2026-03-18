# Chameleon 控制面與信任分發規格 v15.0
*(Control Plane & Out-of-Band Trust Objects Specification)*

**狀態**: 草案 (Draft)

## 1. 兩級 PKI 信任模型 (Two-Tier PKI)

Chameleon 採用兩級簽發體系，以平衡 Root CA 的絕對安全（離線）與邊緣節點證書的高頻更新需求。

1. **Offline Root CA**: 產生並保存在離線冷錢包中的 Ed25519 密鑰對。其公鑰被硬編碼或透過安全的系統更新 (OOB) 分發給所有客戶端。
2. **Control Plane Signing Key (CPSK)**: 控制面持有的中期線上簽發金鑰（例如有效期 30 天）。由 Root CA 簽發 `CPSK Certificate`。
3. **Epoch Certificate**: 邊緣節點持有的短期憑證（例如有效期 24 小時），由 `CPSK` 簽發，包含在 Noise 握手的回應中，用於證明 Server Static Key 的合法性。

---

## 2. OOB 信任物件規格 (OOB Trust Objects)

控制面必須透過帶外通道 (Out-of-Band, 例如 HTTPS API) 向客戶端預先分發以下兩個核心配置物件。雖然客戶端可透過 JSON 獲取這些物件，但所有的**數位簽章必須針對嚴格的規範化二進位實體 (Canonical Binary Body)** 進行，以確保跨實作的一致性。所有整數皆為大端序 (Big-Endian)，字串為 ASCII (無 Null 結尾)。

### 2.1 CPSK Certificate (控制面簽發憑證)

客戶端的信任庫 (Trust Store) 必須維護一個有效 `CPSK Certificate` 的清單。

**規範化被簽體 (Canonical Body): `CPSKCertBody`**
```text
CPSKCertBody =
  Version (1 byte, 值為 0x01) ||
  Subject_Key_ID (8 bytes) ||
  Public_Key (32 bytes, Ed25519) ||
  Not_Before (8 bytes, UTC 秒數) ||
  Not_After (8 bytes, UTC 秒數) ||
  Key_Usage (1 byte, 0x01 表示 CPSK_SIGNING)
```

* **Subject_Key_ID (強制作為查找鍵)**: 必須 (MUST) 為 `Public_Key` 進行 `BLAKE2s` 雜湊後截取的前 8 個 Bytes。

**簽章與封裝**:
* `Root_Signature` = `Sign(Root_Private_Key, "chameleon-cpsk-cert-v1" || CPSKCertBody)` (簽名結果為 64 Bytes Ed25519 簽名，前綴字串 22 Bytes)。
* 最終提供給客戶端的 JSON 封裝需包含上述所有明文欄位以及 Base64 編碼的 `Root_Signature`。

### 2.2 Node Manifest (節點清單與路由表)

`Node Manifest` 解決了 Noise `NK` 模式中 Responder Static Key 的預分發問題。

**規範化節點條目 (Canonical Node Entry)**:
```text
NodeEntry =
  Node_ID_Length (1 byte) || Node_ID (ASCII) ||
  Route_Group_ID_Length (1 byte) || Route_Group_ID (ASCII) ||
  Node_Priority (4 bytes) ||
  Node_Weight (4 bytes) ||
  Current_Static_Pubkey (32 bytes) ||
  Current_Not_After (8 bytes) ||
  Has_Next_Key (1 byte, 0x00 或 0x01) ||
  [Optional: Next_Static_Pubkey (32 bytes)] ||
  [Optional: Next_Not_Before (8 bytes)] ||
  Endpoint_Count (1 byte) ||
  Endpoint[0] || ... || Endpoint[N-1]
```

**結構化端點 (Structured Endpoint): `Endpoint`**
```text
Endpoint =
  Priority (4 bytes) ||
  Weight (4 bytes) ||
  Bearer_Type (1 byte) ||
  Bearer_Params (Variable Length)
```

* **Bearer_Type 與參數 (Bearer Params)**:
  * `0x01` (**TCP_STREAM**): `Host_Type (1) || Host_Length (1) || Host (Var) || Port (2)`
  * `0x02` (**QUIC_STREAM**): `Host_Type (1) || Host_Length (1) || Host (Var) || Port (2) || ALPN_Length (1) || ALPN (Var) || SNI_Mode (1) || [Optional: SNI_Length (1) || SNI (Var)]`
  * `0x03` (**WS_STREAM**): `Host_Type (1) || Host_Length (1) || Host (Var) || Port (2) || Path_Length (1) || Path (Var) || TLS_Mode (1) || Host_Header_Length (1) || Host_Header (Var)`

* **Bearer Enum 與語法強制約束**:
  * **Host_Type**: `0x01` (IPv4, 固定 4 bytes), `0x02` (IPv6, 固定 16 bytes), `0x03` (Domain, ASCII)。若長度不符，整個 Manifest 驗證失敗。
  * **SNI_Mode**: `0x00` (None), `0x01` (Use Host as SNI, 此時不可附帶 SNI 欄位), `0x02` (Explicit SNI, 必須附帶後續的 `SNI_Length || SNI` 欄位)。
  * **TLS_Mode**: `0x00` (Plain), `0x01` (TLS)。
  * **WS Path**: 必須為 ASCII，不允許空字串，且必須以 `/` 開頭。
  * **Host_Header**: 若 `Host_Header_Length = 0`，代表預設使用 `Host` 的值；若長度大於 0，則允許顯式覆蓋 (可與 SNI 獨立)。
  * **ALPN**: 必須為單一 ASCII 字串。若 `ALPN_Length = 0` 則不傳遞 ALPN。

* **兩階段調度演算法與失敗回饋 (Two-Stage Scheduling & Failure Feedback)**:
  1. **第一階段 (Node Selection)**: 在同一 `Route_Group_ID` 內，取最小 `Node_Priority` 集合，並在集合內按 `Node_Weight` 執行加權隨機選擇 `Node_ID`。
  2. **第二階段 (Endpoint Selection)**: 在選定的 `Node_ID` 內，取最小 `Priority` 的 Endpoint 集合，在集合內按 `Weight` 執行加權隨機選擇撥號入口。
  3. **失敗回饋狀態機 (Failure State Machine)**:
     * **Endpoint Failure**: 單一 Endpoint 撥號失敗或超時，僅將該 Endpoint 從當前嘗試輪次中排除，並在同 Node 內繼續重試其他 Endpoint。不立即懲罰該 `Node_ID`。
     * **Node Failure**: 僅當某個 `Node_ID` 下的所有 Endpoint 在本輪內皆宣告失敗時，才將該 `Node_ID` 標記為「本輪不可用」，並記錄入 **Node Health Backoff Cache**。隨後返回第一階段，在同一個 `Node_Priority` 集合內繼續選擇其他 `Node_ID`。
     * **Bucket Escalation**: 只有當當前 `Node_Priority` 集合內的所有 `Node_ID` 均已耗盡或在 Backoff 中，客戶端才允許升級至下一個 `Node_Priority` 集合。
  4. **狀態主鍵與退避快取**: 
     * 金鑰輪換 (Key Rollover)、安全綁定 (Security Binding) 與失敗快取 (Failure Cache) **一律以 `Node_ID` 為唯一主鍵**。
     * **Node Health Backoff Cache**: Cache Key 為 `Node_ID`，TTL 使用指數退避策略 (例如初始 5s, 最大 300s)。處於 Backoff 狀態的 Node 暫不參與第一階段的加權隨機調度。
```

**規範化被簽體 (Canonical Body): `NodeManifestBody`**
```text
NodeManifestBody =
  Manifest_Version (1 byte, 值為 0x01) ||
  Manifest_Sequence (8 bytes) ||
  Issued_At (8 bytes) ||
  Expires_At (8 bytes) ||
  Signer_Key_ID (8 bytes) ||
  Node_Count (2 bytes) ||
  NodeEntry[0] || NodeEntry[1] || ... || NodeEntry[N-1]
```

### 2.3 Auth Realm Manifest (客戶端憑證清單)

為確保 `protocol.md` 中的 `CLIENT_AUTH` 有全域一致的查驗依據，控制面必須分發 `Auth Realm Manifest`。邊緣節點 (Verifier) 僅能依據此清單驗證客戶端，不得依賴私有的本地資料庫。

**規範化憑證條目 (Canonical Credential Record)**:
```text
ClientCredentialRecord =
  Credential_ID (8 bytes) ||
  Auth_Scheme (1 byte, 0x01 表示 Ed25519) ||
  Client_Public_Key (32 bytes) ||
  Not_Before (8 bytes, UTC 秒數) ||
  Not_After (8 bytes, UTC 秒數) ||
  Status (1 byte, 0x01=Active, 0x02=Revoked, 0x03=Suspended)
```

**規範化被簽體 (Canonical Body): `AuthRealmBody`**
```text
AuthRealmBody =
  Realm_Version (1 byte, 值為 0x01) ||
  Realm_Sequence (8 bytes) ||
  Issued_At (8 bytes) ||
  Expires_At (8 bytes) ||
  Signer_Key_ID (8 bytes) ||
  Record_Count (4 bytes) ||
  ClientCredentialRecord[0] || ... || ClientCredentialRecord[N-1]
```

**簽章、更新與 Verifier 運營合約 (Verifier Operational Contract)**:
* `Realm_Signature` = `Sign(CPSK_Private_Key, "chameleon-auth-realm-v1" || AuthRealmBody)`。
* **Anti-Rollback 與持久化**: `Realm_Sequence` 必須在全域控制面單調遞增 (包含 CPSK 輪換)。邊緣節點必須將已接受的最高 `Realm_Sequence` 持久化至磁碟。重啟後需載入持久化狀態，嚴格拒絕 Sequence 回退，防止舊清單回放。
* **輪詢刷新狀態機 (Refresh State Machine)**:
  1. **Soft Refresh Threshold**: Verifier 必須在 `Expires_At - Refresh_Margin` (例如提前 10%) 時主動輪詢更新，並必須加入隨機抖動 (Jitter) 以避免雪崩。
  2. **Retry Backoff**: 輪詢失敗必須採用指數退避 (Exponential Backoff)，重試邊界不得超過 `Expires_At`。
* **Verifier Fail-Closed 規則**: 
  1. 若超出 `Expires_At` 仍無法拉取新清單，或 `Realm_Signature` 驗證失敗，邊緣節點必須**Fail-Closed (立即拒絕所有新的 CLIENT_AUTH)**。
  2. 冷啟動 (Bootstrap) 時若無有效的本地快取，必須進入 Fail-Closed，不允許放行任何連線。
  3. 絕對禁止在 Auth Realm 失效時降級退回任何本地私有資料庫。
  4. 若客戶端的 `Credential_ID` 不存在於此，或 `Status` 為撤銷/暫停，或不在憑證的 `Not_Before` 與 `Not_After` 有效期內，伺服器必須返回對應錯誤 (見 `protocol.md`)。

---

## 3. 防回滾與更新語義 (Anti-Rollback & Updates)

1. **Manifest Sequence 檢查**: 客戶端本地持久化存儲已接受的最高 `Manifest_Sequence`。如果接收到新的 Manifest，其 Sequence 小於或等於本地存儲的值，必須拒絕更新（防止攻擊者重放舊的合法配置以利用已洩漏的歷史金鑰）。
   * **命名空間與輪換語義 (MUST)**: `Manifest_Sequence` 是在**整個控制面命名空間內全域單調遞增**的數值。即使控制面發生 `CPSK` 輪換 (Signer Rotation)，新 CPSK 簽發的 Manifest 也**絕對不可**重置 Sequence。客戶端必須跨 Signer 執行嚴格的 Anti-Rollback 校驗。
   * **生成端合約 (Issuer Contract)**: 控制面必須保證全域單調性。這要求控制面採用**單寫者模型 (Single-Writer Model)** 或外部分配器 (Monotonic Allocator)。若發生 Signer Rotation 或 Region Failover 時無法保證全域單序，控制面寧可暫停發佈，也絕對不可發佈回退的 Sequence，否則會導致所有客戶端永久拒絕更新。
2. **平滑金鑰輪換算法 (Deterministic Key Rollover)**:
   對同一個 `Node ID`，客戶端在發起新連線 (Session) 時，**必須**使用以下確定性算法選擇 Noise `NK` 的預期公鑰 (`Expected_Server_Pubkey`)：
   * 若 `Now < Next_Not_Before` (或不存在 Next Key)：只能使用 `Current_Static_Pubkey`。
   * 若 `Next_Not_Before <= Now < Current_Not_After` (重疊視窗 Overlap Window)：
     * 第一次嘗試必須使用 `Current_Static_Pubkey`。
     * **僅當**該次嘗試在**單一 Endpoint 上**以 `Handshake Timeout / Handshake AEAD Fail / Silent Drop` 結束，且此 `Node ID` 明確宣告了 `Next_Static_Pubkey` 時，允許**一次** `Current -> Next` 的 fallback。
     * **抗噪聲規則 (Noise Resistance)**: 
       1. 若客戶端在同一個 Route Group 的多個 Node ID 上同時觀察到 Timeout，應優先視為網路路由異常或遭封鎖，而不是 Key Rollover。
       2. **本地決策快取 (Local Decision Cache)**: Fallback 後，客戶端必須將決策存入快取。
          * **Cache Key**: `Node_ID`。
          * **TTL**: `min(Current_Not_After - Now, Fixed_Local_Max_TTL)` (例如 3600 秒)。
          * **失效 (Invalidation)**: 收到包含該 `Node_ID` 的新 `Node Manifest` 後立即失效；或若後續握手成功使用了 `Current_Static_Pubkey`，則立即覆蓋清除該 `Next` 決策。
          此快取確保該 `Node ID` 短時間內不再於 Current / Next 之間反覆抖動。同一輪撥號最多允許一次切換。只有在 Overlap Window 內才允許此特殊 Fallback。
   * 若 `Now >= Current_Not_After`：只能使用 `Next_Static_Pubkey`。
   * **硬性安全約束**: 若伺服器回傳的 `Epoch Cert` 中的公鑰既不等於客戶端選定的 `Current`，也不等於 `Next`，客戶端必須視為配置失配/遭劫持，立即中斷，不得默默接受。
3. **撤銷語義 (Revocation)**:
   * 若 `Epoch Cert` 校驗失敗（如 Signature Invalid），客戶端不得盲目重試同一節點，必須觸發 `Config Refresh Required`。