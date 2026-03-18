# Chameleon Review
## 對 `design.md` v14.0、`protocol.md` v14.0 與 `control_plane.md` v14.0 的新一輪聯合審查

這一輪和上一輪相比，協議已經明顯往「完整系統規格」走，而不是只是在補欄位。
`Auth Realm Manifest` 的生命週期、`Node_Priority / Node_Weight` 的兩階段調度、
以及 `Bearer_Params` 的枚舉封口，說明你們已經開始處理真正會影響互通與運營的深水區。

但也正因為如此，剩下的問題不再是「有沒有欄位」，而是：
這些新引入的機制是否真的形成了**可收斂的狀態機**、**可維持的控制面負載模型**，
以及**可預期的故障恢復行為**。

---

## 0. 本輪可以正式關閉的舊問題

### 0.1 `Auth Realm Manifest` 已經不再只是靜態資料結構
上一輪我打的是它沒有 freshness、anti-rollback、fail-closed。
這一輪 `control_plane.md` 已經補上：

1. `Issued_At / Expires_At`
2. `Realm_Sequence`
3. 與 `Node Manifest` 對齊的 issuer contract
4. verifier fail-closed

這個大洞可以正式降級。

### 0.2 `Route_Group` 已不再完全缺跨 node 調度語義
你們現在至少明確寫出了兩階段調度：

1. 先在 route group 內選 `Node_ID`
2. 再在 node 內選 `Endpoint`

這讓 route-group selection 首次有了結構。

### 0.3 `Bearer_Params` 已不再停留在“欄位存在但語義靠猜”
`SNI_Mode`、`TLS_Mode`、`WS Path`、`Host_Header`、`ALPN` 都已經有了第一版硬規則。
這使得上輪那種“大家自己補默認值”的風險已經明顯下降。

### 0.4 `design.md` 的過度宣稱已經被回收
你們把 v13.0 那種接近「完美閉環」的語氣收斂到了更工程化的說法，這是對的。

---

## 1. 阻塞級問題

## 1.1 兩階段調度現在只有“選擇結構”，還沒有“失敗回饋狀態機”，在真實敵意網路下仍然會表現為不可預測

### 攻擊
`control_plane.md` 現在定義了：

1. 第一階段：按 `Node_Priority / Node_Weight` 選 `Node_ID`
2. 第二階段：按 `Endpoint Priority / Weight` 選 `Endpoint`
3. 安全狀態與 failure cache 以 `Node_ID` 為主鍵

這在靜態情況下沒問題，但一旦進入真實失敗場景，規格立刻又變模糊了。

目前還沒被定義的是：

1. 在同一 `Node_Priority` bucket 內，是否必須做 weighted-random，還是可做任意排序
2. 當某個 `Node_ID` 一次撥號失敗後，它是：
   - 只在本次撥號中暫時排除
   - 還是要進入短期 backoff
   - 還是要寫入某種 health cache
3. 若某個 `Node_ID` 被選中，但它下面所有 endpoint 都失敗了：
   - 是否立即回到同一 node bucket 裡選其他 node
   - 是否升到下一個 `Node_Priority`
4. 若某個 endpoint 失敗：
   - 是否只懲罰該 endpoint
   - 還是懲罰整個 `Node_ID`
5. node-level failure cache 與 rollover cache 的優先順序如何疊加

這些不是實作細節，而是 availability 行為本身。

### 為什麼目前還不夠
你們現在有了 route selection 的骨架，但沒有 failure feedback 的閉環。
結果就是，兩個實作都能聲稱自己“遵守了優先級與加權”，卻在失敗後做出完全不同的收斂：

1. 有的 client 會在同一個壞 node 上反覆打不同 endpoint
2. 有的 client 會過早把整個 node 拉黑
3. 有的 client 會在同 priority bucket 裡充分探索
4. 有的 client 會直接升級到下一個 bucket

在高丟包、局部黑洞、單入口被干擾的環境裡，這些差異會直接決定：

1. 連線建立延遲
2. 控制面壓力
3. session pool 補連速度
4. 使用者體感是否像“可用”

### 具體修改方案
這一塊我建議不要再留給實作者自由發揮，直接把 state machine 寫死：

1. **Node Selection**
   - 先取最小 `Node_Priority` bucket
   - 在 bucket 內按 `Node_Weight` 做 weighted-random
2. **Endpoint Selection**
   - 在選中的 `Node_ID` 內取最小 `Endpoint Priority` bucket
   - 在 bucket 內按 `Weight` 做 weighted-random
3. **Endpoint Failure**
   - 單個 endpoint 失敗只暫時排除該 endpoint
   - 不立即懲罰整個 `Node_ID`
4. **Node Failure Promotion**
   - 僅當該 node 在當前嘗試輪次內所有 endpoint 都失敗時，才將該 `Node_ID` 標記為
     “本輪不可用”，回到同一 `Node_Priority` bucket 繼續選其他 node
5. **Bucket Escalation**
   - 只有當某個 `Node_Priority` bucket 的所有 node 都耗盡時，才升到下一個 priority
6. **Health Cache**
   - 補一個與 rollover cache 分離的 `Node Health Backoff`
   - 至少定義：cache key、TTL、失效條件、與 `Node_Weight` 的交互方式

如果不補這套東西，兩階段調度看起來已經“有算法”，實際上仍然只是半成品。

---

## 1.2 `Auth Realm Manifest` 的生命週期雖然補上了，但 verifier 的刷新狀態機仍然不夠精確，Fail-Closed 很可能在真實運營裡變成同步雪崩

### 攻擊
這一輪你們補了 `Issued_At / Expires_At / Realm_Sequence / Fail-Closed`，方向完全正確。
但新的問題是：一旦 verifier 真的依賴它，**刷新策略本身就變成系統可用性的核心路徑**，
而現在這條路徑還沒被真正定義。

目前還缺的是：

1. verifier 應在什麼時刻開始刷新
2. 若刷新失敗，多久後重試
3. 多次失敗時是否退避
4. 舊的 `Auth Realm Manifest` 能否在未過期前繼續使用
5. `Realm_Sequence` 的最高值是否必須持久化到磁碟，還是只存在記憶體
6. verifier 重啟後如何避免被舊 manifest 回放

如果這些不寫死，Fail-Closed 的理論正確性很可能在實務上變成：

1. 大量 verifier 在接近 `Expires_At` 時同時輪詢控制面
2. 一次控制面抖動導致整片邊緣節點同步失效
3. 重啟後的 verifier 因為沒有持久化最高 `Realm_Sequence`，接受了舊的 auth realm

### 為什麼目前還不夠
現在的文檔把“生命週期是否存在”補上了，但還沒有把“生命週期如何運行”補上。
這對 `Auth Realm` 特別危險，因為它不是純配置，而是：

1. 每次新連線都會走到的 verifier 決策依據
2. 失效後必須 fail-closed 的關鍵物件

換句話說，它不是邊緣上的“輔助配置”，而是實時 auth gate。

### 具體修改方案
這一塊建議直接補一個 verifier refresh contract：

1. **Soft Refresh Threshold**
   - 例如在 `Expires_At - Refresh_Margin` 時主動刷新
   - `Refresh_Margin` 必須大於最大控制面輪詢抖動
2. **Refresh Jitter**
   - verifier 必須加入隨機抖動，避免同時輪詢
3. **Retry Backoff**
   - 輪詢失敗後使用指數退避，但不得超過 `Expires_At`
4. **Fail-Closed Boundary**
   - 一旦超過 `Expires_At` 且無新 realm，立即拒絕所有新的 `CLIENT_AUTH`
5. **Sequence Persistence**
   - verifier 必須持久化已接受的最高 `Realm_Sequence`
   - 重啟後先載入持久化最高值，再接受新 realm
6. **Bootstrap Rule**
   - 冷啟動若沒有任何有效 `Auth Realm Manifest`，必須進入 fail-closed，不得接受 auth

如果不補這一層，你們的 `Auth Realm` 仍然只是“格式正確”，不是“運營可用”。

---

## 1.3 `Auth Realm Manifest` 現在是單一巨型清單模型，這和你們追求的高效率、面向未來、可擴展，已經開始正面衝突

### 攻擊
v14.0 的 `AuthRealmBody` 本質上仍然是：

1. 一份完整簽名的全量 credential 清單
2. 任一 credential 狀態變化，都要重發整個清單

這在小規模環境可以工作，但從設計目標來看，它已經碰到了明顯的效率與擴展性邊界：

1. 一個憑證撤銷，必須重新簽發整份 realm
2. verifier 每次輪詢都要拉取整份 realm
3. 記憶體佔用與輪詢帶寬對 credential 數量呈線性增長
4. `Record_Count` 越大，簽章驗證與反序列化越重

你們現在把 `Auth Realm` 做成了安全上可行的一版，但在規模上還沒有未來性。

### 為什麼目前還不夠
專案目標裡一直有兩個關鍵詞：

1. **整體效率足夠高**
2. **面向未來**

單一全量憑證清單模型，和這兩個目標天然有張力。

尤其當：

1. 憑證數量增加
2. 撤銷事件頻繁
3. verifier 節點數量擴大

控制面更新成本會迅速從“可以接受”變成“每次變更都在推整包”。

### 具體修改方案
這一塊未必要立刻大改，但下一步至少應該選一條演進路線：

1. **Sharded Auth Realm**
   - 按 `Credential_ID` 前綴或 hash bucket 切 shard
   - 每個 shard 有自己的 `Shard_Sequence / Issued_At / Expires_At`
   - verifier 只更新受影響 shard
2. **Delta + Base Snapshot**
   - 保留周期性全量 snapshot
   - 中間發佈簽名的增量變更集
3. **Per-Record Validity**
   - 為 `ClientCredentialRecord` 增加 `Not_Before / Not_After`
   - 降低整份 realm 因單一 credential 到期而頻繁重發的壓力

如果你們短期內不想增加太多複雜度，我建議先做最保守的一步：

1. 保留全量 `AuthRealmBody`
2. 但為 `ClientCredentialRecord` 增加 `Not_Before / Not_After`
3. 同時預留未來 shard 化的 `Realm_ID / Shard_ID`

這樣至少不會把未來的擴容路徑堵死。

---

## 2. 高優先級但非阻塞問題

## 2.1 `GOAWAY 0x01` 現在同時承載 generic internal error 與 auth realm verifier failure，對 client 來說過於粗糙

### 攻擊
`protocol.md` 目前把：

1. `Auth Realm Sig Invalid / Expired`
2. `Auth Realm Rollback Detected`

都映射到了 `GOAWAY 0x01`。

這雖然比完全沒語義好，但對 client 來說仍然過於粗糙：

1. generic internal error
2. verifier freshness lag
3. verifier security incident

都變成同一個 wire-level code。

### 具體修改方案
你們可以不立刻新增很多主錯誤碼，但至少要二選一：

1. 新增 encrypted `GOAWAY Detail Code`
2. 或在 `GOAWAY 0x01` 後附一個可選的 detail TLV

這樣 client / telemetry / operator 才能分辨：

1. 普通內部錯誤
2. auth realm 未同步
3. auth realm rollback 安全事件

---

## 2.2 `ClientCredentialRecord` 仍然只有 `Status`，沒有單條憑證級的時間語義

### 攻擊
即便暫時接受全量 `AuthRealmBody`，目前單條 record 仍只有：

1. `Credential_ID`
2. `Auth_Scheme`
3. `Client_Public_Key`
4. `Status`

沒有：

1. `Not_Before`
2. `Not_After`

這意味著任何憑證有效期管理，都只能透過整份 realm 的替換來完成。

### 具體修改方案
如果短期內不做 shard / delta，至少把 record 級時間語義補上：

1. `Credential_Not_Before`
2. `Credential_Not_After`

這可以顯著降低整份 realm 因單一 credential 生命周期變更而頻繁重發的壓力。

---

## 2.3 `Node Manifest` 與 `Auth Realm Manifest` 雖然都用了“同樣生命週期語義”的表述，但規格仍然太依賴引用而不是顯式列出

### 攻擊
`control_plane.md` 現在用一句話說：
`Auth Realm Manifest` 具有與 `Node Manifest` 完全相同的生命週期與 issuer contract 語義。

這種寫法對人類讀者夠用，但對實作者其實不夠穩：

1. 哪些規則完全相同
2. 哪些只是概念相同但對象不同
3. verifier 是否和 client 一樣都必須持久化最高 sequence

這些最好不要靠“類比理解”。

### 具體修改方案
建議在 `Auth Realm Manifest` 章節下直接顯式列出：

1. `Realm_Sequence` 的 anti-rollback
2. issuer contract
3. 持久化要求
4. fail-closed 邊界

即使內容和 `Node Manifest` 重複，也比引用式規範更穩。

---

## 3. 建議下一輪直接落地的修改順序

如果目標是繼續快收斂，下一輪我建議按這個順序改：

1. 先把兩階段調度補成帶 failure feedback / backoff 的正式狀態機
2. 再把 `Auth Realm` 的 verifier refresh contract 補齊
3. 再決定 `Auth Realm` 的規模模型是全量 + per-record validity，還是直接走 shard / delta
4. 之後補 `GOAWAY 0x01` 的 detail semantics
5. 最後把 `Auth Realm` 與 `Node Manifest` 的共享生命週期規則改成顯式規範，而不是引用式規範

---

## 4. 本輪結論

v14.0 不是在修詞，而是真的把上一輪的大部分主問題打掉了。這一點應該正式承認。

但新的主事實也非常清楚：

1. `Node Manifest` 這條線已經接近可以收口
2. 真正新的瓶頸在於 `Auth Realm` 這條線的運營狀態機與規模模型
3. 其次是 route selection 從“靜態算法”走向“失敗下可收斂算法”的最後一步

把這兩件事做完之後，下一輪 review 才值得真正深入到：

1. session pool 的補連策略
2. mixed workload 下的 HOL 隔離效果
3. 控制幀優先級與 tail latency 邊界
