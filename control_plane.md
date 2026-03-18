# Chameleon 控制面與信任分發規格 v16.0
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
  **定義**: 一次「撥號嘗試」(Dial Attempt) 為對單一 `(Node_ID, Endpoint, KeyChoice)` 的一次物理連線與完整握手嘗試。一次「節點嘗試輪次」(Node Attempt Round) 為對同一 `Node_ID` 下所有合法 Endpoint 與允許的 KeyChoice 探索完畢的過程。
  **限制**: `Node_Weight` 與 Endpoint `Weight` **絕對禁止為 0**。若為 0 則視為 Manifest 格式錯誤 (Invalid)。
  
  1. **第一階段 (Node Selection)**: 在同一 `Route_Group_ID` 內，取最小 `Node_Priority` 集合，並在集合內按 `Node_Weight` 執行加權隨機選擇 `Node_ID`。
  2. **第二階段 (Endpoint Selection)**: 在選定的 `Node_ID` 內，取最小 `Priority` 的 Endpoint 集合，在集合內按 `Weight` 執行加權隨機選擇撥號入口。
  3. **失敗回饋與 Rollover 組合狀態機 (Failure & Rollover State Machine)**:
     * **決策順序**:
       1. 根據該 Node_ID 的 Rollover Cache 決定首選 Key (Current 或 Next)。
       2. 在該 Key 下選擇 Endpoint。
       3. 若該 Endpoint 出現 `Handshake Timeout / Pre-Auth AEAD Fail / Silent Drop`，且當前處於 Overlap Window，且**尚未**做過 `Current -> Next` 切換，才允許在**同一 Endpoint** 上立即重試一次 `Next`。
       4. 只有當 `Current` 與 `Next` 在該 Endpoint 上都失敗後，才計入 **Endpoint Failure** (僅將該 Endpoint 從當前嘗試輪次中排除，繼續重試同 Node 內其他 Endpoint)。`TCP Connect Refused/Reset` 直接視為 Endpoint Failure，不參與 Rollover 推斷。
     * **Node Failure**: 僅當某個 `Node_ID` 下的所有合法 Endpoint 都用盡了合法 Key Path 且皆宣告失敗時，才將該 `Node_ID` 標記為「本輪不可用」，寫入 **Node Health Backoff Cache**。隨後返回第一階段繼續選擇其他 `Node_ID`。
     * **Bucket Escalation**: 當前 `Node_Priority` 集合內的所有 `Node_ID` 均已耗盡或在 Backoff 中時，才允許升級至下一 `Node_Priority` 集合。
  4. **退避與快取覆寫 (Backoff & Cache Reset)**: 
     * **Rollover Cache 覆寫**: 若某 Endpoint 用 `Next` 成功，立即將該 `Node_ID` 的 Rollover Cache 設為 `Prefer Next`，TTL 內同 Node 其他 Endpoint 不得再試 `Current`。若明確有 `Current` 成功，則清除 `Prefer Next`。
     * **Node Health Backoff Cache**: Cache Key 為 `Node_ID`。退避公式為 `min(Base * 2^N, Max_Backoff) * Jitter(0.8~1.2)`，例如 Base=5s, Max=300s。
     * **成功重置**: 任一 Node 成功完成握手並通過 `CLIENT_AUTH` 後，該 `Node_ID` 的 Health Backoff Tier 立即清零。
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

#### Verifier 時間語義 (Verifier Time Semantics)
為了確保全球 Verifier 與控制面具備單一且不可分叉的時間真相，所有時鐘校驗必須遵循以下硬性常數與公式：
* **時間常數**: 
  * `MAX_VERIFIER_CLOCK_SKEW = 300s` (最大時鐘偏差容忍值)
  * `MIN_REFRESH_MARGIN = 60s` (刷新安全視窗的最小預留量)
  * `MAX_REFRESH_JITTER_RATIO = 10%` (刷新隨機抖動上限)
* **時間校驗公式**:
  * `Realm Valid if: Now + SKEW >= Issued_At AND Now - SKEW <= Expires_At`
  * `Credential Valid if: Now + SKEW >= Not_Before AND Now - SKEW <= Not_After`
* **時鐘來源分工**: 所有 UTC 有效期 (`Issued_At / Expires_At / Not_Before / Not_After`) 必須使用 **Wall Clock**。所有輪詢計時、指數退避 (Retry Backoff) 必須使用 **Monotonic Clock**，以防止 Wall Clock 跳變導致重試雪崩。
* **時鐘異常應對**: 若 Verifier 本地的 Wall Clock 在短時間內向後跳變超過 `MAX_VERIFIER_CLOCK_SKEW`，或時間早於編譯期的最小可信值，Verifier 必須強制清空現有快取並進入 **Fail-Closed (Maintenance) State**，直至時間重新與 NTP 同步。

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

**未來演進方向 (Future Evolution Notes)**:
當前的全量清單模型 (Monolithic Realm) 為初期穩定版本。考慮到高頻撤銷與節點規模增長，控制面協議預留了以下演進路徑，實作者應在此模型基礎上保留抽象餘裕：
1. **Sharded Auth Realm**: 依據 `Credential_ID` 的前綴切割 Shard，Verifier 僅輪詢對應的 Shard 清單，降低單一節點記憶體與輪詢壓力。
2. **Snapshot + Delta**: 保持全量快照作為基線 (Base)，高頻推送帶有簽章的 `RealmDelta` 物件以實現增量更新。

---

## 3. 防回滾與更新語義 (Anti-Rollback & Updates)

1. **Manifest Sequence 檢查**: 客戶端本地持久化存儲已接受的最高 `Manifest_Sequence`。如果接收到新的 Manifest，其 Sequence 小於或等於本地存儲的值，必須拒絕更新（防止攻擊者重放舊的合法配置以利用已洩漏的歷史金鑰）。
   * **命名空間與輪換語義 (MUST)**: `Manifest_Sequence` 是在**整個控制面命名空間內全域單調遞增**的數值。即使控制面發生 `CPSK` 輪換 (Signer Rotation)，新 CPSK 簽發的 Manifest 也**絕對不可**重置 Sequence。客戶端必須跨 Signer 執行嚴格的 Anti-Rollback 校驗。
   * **生成端合約 (Issuer Contract)**: 控制面必須保證全域單調性。這要求控制面採用**單寫者模型 (Single-Writer Model)** 或外部分配器 (Monotonic Allocator)。若發生 Signer Rotation 或 Region Failover 時無法保證全域單序，控制面寧可暫停發佈，也絕對不可發佈回退的 Sequence，否則會導致所有客戶端永久拒絕更新。
2. **平滑金鑰輪換算法 (Deterministic Key Rollover)**:
   對同一個 `Node ID`，客戶端的允許公鑰必須遵守嚴格的時間視窗：
   * 若 `Now < Next_Not_Before` (或不存在 Next Key)：只能使用 `Current_Static_Pubkey`。
   * 若 `Next_Not_Before <= Now < Current_Not_After` (重疊視窗 Overlap Window)：允許使用 Current 或 Next。具體的 fallback 觸發時機與快取策略已統合至 `2.2 節` 的「失敗回饋與 Rollover 組合狀態機」。
   * 若 `Now >= Current_Not_After`：只能使用 `Next_Static_Pubkey`。
   * **硬性安全約束**: 若伺服器回傳的 `Epoch Cert` 中的公鑰既不等於客戶端選定的 `Current`，也不等於 `Next`，客戶端必須視為配置失配/遭劫持，立即中斷，不得默默接受。
3. **撤銷語義 (Revocation)**:
   * 若 `Epoch Cert` 校驗失敗（如 Signature Invalid），客戶端不得盲目重試同一節點，必須觸發 `Config Refresh Required`。