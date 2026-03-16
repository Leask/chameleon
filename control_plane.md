# Chameleon 控制面與信任分發規格 v1.0
*(Control Plane & Out-of-Band Trust Objects Specification)*

**狀態**: 草案 (Draft)

## 1. 兩級 PKI 信任模型 (Two-Tier PKI)

Chameleon 採用兩級簽發體系，以平衡 Root CA 的絕對安全（離線）與邊緣節點證書的高頻更新需求。

1. **Offline Root CA**: 產生並保存在離線冷錢包中的 Ed25519 密鑰對。其公鑰被硬編碼或透過安全的系統更新 (OOB) 分發給所有客戶端。
2. **Control Plane Signing Key (CPSK)**: 控制面持有的中期線上簽發金鑰（例如有效期 30 天）。由 Root CA 簽發 `CPSK Certificate`。
3. **Epoch Certificate**: 邊緣節點持有的短期憑證（例如有效期 24 小時），由 `CPSK` 簽發，包含在 Noise 握手的回應中，用於證明 Server Static Key 的合法性。

---

## 2. OOB 信任物件規格 (OOB Trust Objects)

控制面必須透過帶外通道 (Out-of-Band, 例如 HTTPS API) 向客戶端預先分發以下兩個核心配置物件。

### 2.1 CPSK Certificate (控制面簽發憑證)

客戶端的信任庫 (Trust Store) 必須維護一個有效 `CPSK Certificate` 的清單。為了支持平滑輪換，通常會同時包含 Current 與 Next 兩把金鑰。

**資料結構 (JSON 格式範例)**:
```json
{
  "version": 1,
  "signer_key_id": "a1b2c3d4e5f60708",  // 8 Bytes (Hex). 建議為 CPSK PubKey 的 BLAKE2s Hash 截斷前 8 Bytes
  "public_key": "...",                  // 32 Bytes Ed25519 PubKey (Base64)
  "not_before": 1700000000,             // UTC 秒數
  "not_after": 1702592000,
  "key_usage": ["CPSK_SIGNING"],
  "root_signature": "..."               // 64 Bytes (Base64). Root CA 對上方欄位 (不含 signature 本身) 的 Ed25519 簽名
}
```
* **Signer Key ID**: 用於在 `Epoch Cert` 中快速定位校驗用的 CPSK。規定為 `CPSK Public Key` 進行 `BLAKE2s` 雜湊後的前 8 個 Bytes。

### 2.2 Node Manifest (節點清單與路由表)

`Node Manifest` 解決了 Noise `NK` 模式中 Responder Static Key 的預分發問題。客戶端透過此清單獲知可用的節點以及對應的靜態公鑰。

**資料結構 (JSON 格式範例)**:
```json
{
  "manifest_version": 1,
  "manifest_sequence": 10045,           // 單調遞增的世代號，用於 Anti-rollback
  "issued_at": 1700100000,
  "expires_at": 1700186400,
  "signer_key_id": "a1b2c3d4e5f60708",  // 簽發此 Manifest 的 CPSK ID
  "nodes": [
    {
      "node_id": "edge-tokyo-01",
      "route_group": "asia-premium",
      "current_static_pubkey": "...",   // 32 Bytes X25519 PubKey (Base64)
      "current_not_after": 1700143200,
      "next_static_pubkey": "...",      // (可選) 節點即將輪換的下一把金鑰
      "next_not_before": 1700140000,
      "endpoints": ["tcp://198.51.100.1:443"],
      "priority": 10,
      "weight": 100
    }
  ],
  "manifest_signature": "..."           // 64 Bytes (Base64). 由 CPSK 對上方欄位進行的 Ed25519 簽名
}
```

---

## 3. 防回滾與更新語義 (Anti-Rollback & Updates)

1. **Manifest Sequence 檢查**: 客戶端本地持久化存儲已接受的最高 `manifest_sequence`。如果接收到新的 Manifest，其 Sequence 小於或等於本地存儲的值，必須拒絕更新（防止攻擊者重放舊的合法配置以利用已洩漏的歷史金鑰）。
2. **平滑金鑰輪換 (Key Rollover)**:
   * 控制面在下發新的 Manifest 時，如果某個 Node 準備更換 Static Pubkey，會在 `next_static_pubkey` 中提前下發。
   * 在 `current_not_after` 與 `next_not_before` 重疊的時間視窗 (Overlap Window) 內，客戶端允許使用兩把金鑰中的任意一把發起新連線。
   * 對同一個 `Node ID`，客戶端在同一時刻只能選擇**一把**金鑰作為 Noise `NK` 握手的預期公鑰。若伺服器回傳的 `Epoch Cert` 中的公鑰不屬於 Manifest 中宣告的 Current 或 Next 金鑰，客戶端必須視為配置失配，立即中止，不得默默接受。
3. **撤銷語義 (Revocation)**:
   * 若 `Epoch Cert` 校驗失敗（過期、簽名錯誤），或者 Auth Token 被拒絕 (`GOAWAY 0x05`)，客戶端必須觸發 `Config Refresh Required`，主動向控制面 API 拉取最新的 `CPSK Certificate` 和 `Node Manifest`。