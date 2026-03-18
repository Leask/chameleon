# Chameleon 聯合審查意見 v15.0

本輪審查不再重複早已指出的欄位缺失。`v15.0` 的進步是明確的：
兩階段調度終於補上了失敗回饋狀態機，`Auth Realm Manifest`
補上了 Verifier 的輪詢/持久化合約，單條客戶端憑證也有了
`Not_Before / Not_After`，而且 `0x06 / 0x07` 也開始嘗試把
「Verifier 未同步」與「Verifier 安全事件」拆開。這些都不是小修補，
而是結構上的前進。

但 `v15.0` 仍然沒有完全閉環。現在的主要問題已經不是「少一個欄位」，
而是三類更硬的規範缺口：

1. `Auth Realm` 已經被提升為資料面授權的核心依賴，但 verifier
   本地時間、freshness、fail-closed 的時間語義還沒有被正式封死。
2. 兩階段調度的失敗回饋，和 `Current -> Next` key rollover 的本地
   fallback，仍然是兩套各自成立、但沒有組合規則的狀態機。
3. `0x06 / 0x07` 雖然被引入，但 `design.md`、`protocol.md`、
   recovery matrix、客戶端恢復策略四者之間仍然存在語義分叉。

下面只攻擊仍然會導致實作分叉、誤恢復、或運營雪崩的點。

---

## 0. 本輪已解決或可正式降級的問題

以下問題可從主審查列表中降級，不需要下一輪再重複打：

1. **兩階段調度不再只是靜態結構**。`control_plane.md`
   現在至少定義了 Node/Endpoint 兩層選擇和 Node 級 backoff。
2. **`Auth Realm` 不再只有資料結構**。現在已有 `Issued_At`、
   `Expires_At`、`Realm_Sequence`、持久化與 refresh/backoff contract。
3. **單條憑證生命週期已補齊**。`ClientCredentialRecord`
   已有 `Not_Before / Not_After`，先前「只能靠全量清單過期」的批評
   可以降級。
4. **`0x06 / 0x07` 的方向是對的**。問題不在於要不要拆，而在於目前
   拆得還不夠精確。

---

## 1. 阻塞級問題

### 1.1 Verifier 的時間語義仍未正式規範化

這是本輪最重要的阻塞點。`protocol.md` 已經給了
`Epoch Cert` 的 `MAX_CLOCK_SKEW = 300` 秒，但這個時鐘容忍模型只存在於
客戶端驗證 `Epoch Cert` 的流程裡，沒有延伸到控制面的 `Auth Realm`
驗證與 verifier 的授權決策。現在有三組時間欄位已經成為資料面行為的
硬依賴：

1. `AuthRealmBody.Issued_At / Expires_At`
2. `ClientCredentialRecord.Not_Before / Not_After`
3. Verifier 本地的 `Refresh_Margin`、輪詢重試窗口與 fail-closed 邊界

目前規格缺的是這些時間如何與 verifier 本地時鐘互動。這會直接造成四個
實作分叉：

1. **快時鐘節點**：
   verifier 可能提前把仍有效的 `Auth Realm` 視為過期，進而對所有新
   `CLIENT_AUTH` 回 `0x06` 或直接 fail-closed。
2. **慢時鐘節點**：
   verifier 可能長時間接受已過期的 realm 或已失效的單條 credential。
3. **跨節點語義不一致**：
   同一份 `Auth Realm`，不同節點因本地時鐘偏差而對同一客戶端做出不同
   授權決策，結果就是 client 在同一 `Route_Group` 內反覆命中互相矛盾
   的節點。
4. **冷啟動/時鐘異常下的雪崩**：
   若節點啟動時系統時間嚴重偏移，`Expires_At - Refresh_Margin`
   會立刻失真。實作 A 可能立刻瘋狂輪詢，實作 B 可能直接 fail-closed，
   實作 C 可能繼續服務舊 realm。這會讓控制面與資料面同時不穩。

#### 為什麼 v15.0 的修補還不夠

`control_plane.md` 的 refresh contract 目前只說：

1. 在 `Expires_At - Refresh_Margin` 時刷新
2. 加 jitter
3. 使用指數退避
4. 過期或簽章失敗時 fail-closed

這些方向都對，但還不足以讓不同團隊獨立實作後得到相同行為，因為缺少：

1. **統一的 verifier clock model**
2. **時間比較時的 skew tolerance**
3. **wall clock 與 monotonic clock 的分工**
4. **當本地時鐘明顯不可信時的強制處置**

#### 建議的具體演進方向

應在 `control_plane.md` 新增一個專門小節，例如
`Verifier Time Semantics`，至少寫死以下內容：

1. **定義統一常數**
   - `MAX_VERIFIER_CLOCK_SKEW = 300s`
   - `MIN_REFRESH_MARGIN = 60s`
   - `MAX_REFRESH_JITTER_RATIO = 10%`
   - `MAX_CLOCK_STEP_FORWARD` / `MAX_CLOCK_STEP_BACKWARD`
     的本地容忍門檻

2. **定義時間比較公式**
   - `Realm valid if Now + SKEW >= Issued_At AND Now - SKEW <= Expires_At`
   - `Credential valid if Now + SKEW >= Not_Before AND Now - SKEW <= Not_After`
   - 明確寫死 `Issued_At` 與 `Not_Before` 是「可提前接受多少」，
     `Expires_At` 與 `Not_After` 是「可延後接受多少」

3. **定義 clock source**
   - `wall clock` 用於 UTC validity checks
   - `monotonic clock` 用於 refresh/backoff timer
   - 不允許把 retry/backoff 直接綁在可跳變的 wall clock 上

4. **定義時鐘異常的 fail-closed 行為**
   - 若本地時間早於某個最小可信值，verifier 必須直接進入 maintenance /
     unsafe-clock state，而不是繼續服務
   - 若本地時鐘在短時間內向後跳變超過門檻，必須清空 refresh timer、
     重新計算 freshness 狀態，必要時暫停接受新 `CLIENT_AUTH`

5. **重新拆分錯誤映射**
   - `Credential Not Yet Valid` 應該更像 `[C] verifier clock / config fault`
     或 `[I] issuer misconfig`
   - `Credential Expired` 才是明確的授權失敗
   - 否則 server 目前會把所有這類情況都打包進 `0x05`，這會把
     verifier clock drift 誤報成 client credential problem

如果這一塊不先定死，`Auth Realm` 雖然看起來已經有生命週期，但實際上仍
然沒有運營上的單一真相。

---

### 1.2 `Node Health Backoff Cache` 與 `Current -> Next` fallback 還沒有組合規則

這是本輪第二個阻塞點，而且是純狀態機問題。`control_plane.md`
現在同時定義了兩套本地決策機制：

1. **調度失敗回饋**
   - endpoint failure: 當前輪次排除 endpoint
   - node failure: 同 node 全部 endpoint 失敗後才標記 node backoff
   - bucket escalation: 同 priority bucket 全耗盡後才升級

2. **key rollover fallback**
   - 在 overlap window 內，先試 `Current`
   - 單一 endpoint 上若出現 timeout / handshake AEAD fail / silent drop，
     才允許一次 `Current -> Next`
   - 結果寫入 `Node_ID` 級 cache

單獨看兩套都能自圓其說，但規格沒有定義它們怎麼組合。這在 hostile
network 下會導致非常大的差異。以下是一個真實的分叉場景：

1. client 在 `Route_Group G` 內選到 `Node A`
2. `Node A` 有 `Endpoint A1`、`Endpoint A2`
3. `Node A` 在 overlap window 中，`Current` 正在被替換成 `Next`
4. client 先用 `Current` 撥 `A1`
5. 結果是 timeout

此時至少有四種合理但互相衝突的實作：

1. **實作 A**：先把 `A1` 判為 endpoint failure，再在 `A2` 上繼續試
   `Current`
2. **實作 B**：在 `A1` 上立刻做一次 `Current -> Next`
3. **實作 C**：先把整個 `Node A` 標記為可疑 rollover，之後整個 node
   一律走 `Next`
4. **實作 D**：因 timeout 屬於網路噪聲，不立即做 rollover，優先把
   `A1` 排除、繼續 `A2`

現在規格沒有告訴實作者哪一個才是對的。這不是小差異，因為它直接決定：

1. key rollover 是否會被普通網路噪聲誤觸發
2. node health cache 是否會被 key mismatch 污染
3. 同一 `Node_ID` 內不同 endpoint 是否共享 fallback 決策
4. `Session Pool` 在 replenish 時是否會沿用錯誤的 node-level cache

#### 為什麼目前寫法仍不夠

`control_plane.md` 第 2.2 節把所有安全綁定、key rollover、failure cache
都綁到 `Node_ID`，但沒有給出明確的**決策優先順序**。只靠
`Cache Key = Node_ID` 還不足以定義行為，因為你仍然要回答：

1. 同一 endpoint 上是否要先試 `Next`
2. 試完 `Next` 失敗後，這次失敗要算 endpoint failure 還是 node failure
3. 若 `A1` 走 `Next` 成功，`A2` 是否也必須直接切到 `Next`
4. node health backoff 與 rollover cache 誰先檢查，誰覆蓋誰

#### 建議的具體演進方向

應把 `Node_ID` 級 rollover 與 scheduling failure 合併成一張正式表格，
至少寫死以下狀態機：

1. **定義嘗試單元**
   - 一次 `Dial Attempt` = 對單一 `(Node_ID, Endpoint, KeyChoice)` 的一次
     物理撥號與完整握手嘗試
   - 一次 `Node Attempt Round` = 對同一 `Node_ID` 下所有 eligible
     endpoint 與允許的 key choice 探索完畢的完整過程

2. **定義決策順序**
   - 第一步：根據 rollover cache 決定該 node 的首選 key
   - 第二步：在該 key 下選 endpoint
   - 第三步：若該 endpoint 出現 timeout / silent drop / handshake AEAD fail，
     且當前處於 overlap window，且尚未做過一次 `Current -> Next`，
     才允許在**同一 endpoint** 上立即重試一次 `Next`
   - 第四步：只有在 `Current` 與 `Next` 都失敗後，這個 endpoint
     才能計入 endpoint failure
   - 第五步：只有當所有 endpoint 都用盡了當前 node 的合法 key path，
     才能宣告 node failure 並寫入 node health backoff

3. **定義成功後的 cache 覆寫**
   - 若某 endpoint 用 `Next` 成功，應立即把 `Node_ID` 的 rollover cache
     設為 `Prefer Next`
   - 後續同 node 的其他 endpoint 在 TTL 內不得再回頭試 `Current`
   - 若後續明確有某次 `Current` 成功，才允許覆寫清除 `Prefer Next`

4. **定義 failure classification**
   - `TCP connect refused / reset` 更像 endpoint reachability fault
   - `silent drop / handshake timeout / pre-auth AEAD fail` 才是可以參與
     rollover 推斷的信號
   - 如果把所有 failure 都一視同仁，rollover cache 就會被普通撥號錯誤污染

5. **定義與 session pool 的關係**
   - rollover cache 與 node health cache 是否為 process-global
   - 同一 `Route_Group` 下多個 warm session 是否共享這些 cache
   - session pool replenishment 是否允許繞過現有 backoff

現在這塊如果不補，`v15.0` 的兩階段調度雖然比 `v14.0` 先進，但仍然不具備
真正的 deterministic convergence。

---

### 1.3 `0x06 / 0x07` 已引入，但 wire-level 語義仍然分叉

`design.md` 在總結裡宣稱：

1. `0x06` = verifier config lag / out of sync
2. `0x07` = verifier rollback / security incident
3. client 應能據此精準決定「切點」還是「停止」

這個方向是正確的，但 `protocol.md` 目前還沒有真正做到。現在至少有三處
語義不一致：

1. **錯誤碼表**
   - `0x06`: `Auth Realm Out of Sync / Freshness Lag`
   - `0x07`: `Auth Realm Rollback Detected / Signature Invalid`

2. **Recovery Matrix**
   - `Auth Realm Sig Invalid / Expired` -> `GOAWAY (0x06)`
   - `Auth Realm Rollback Detected` -> `GOAWAY (0x07)`

3. **客戶端恢復策略**
   - 對 `0x06` 和 `0x07` 都寫成 `Switch Node`

這三者放在一起就裂開了。最明顯的矛盾是：

1. 錯誤碼表說 **signature invalid** 應屬於 `0x07`
2. recovery matrix 卻把 **Sig Invalid / Expired** 一起映射到 `0x06`
3. client 行為又把 `0x06` 和 `0x07` 視為同一種恢復建議

這表示目前 `0x06 / 0x07` 雖然存在，但其實還沒形成穩定的 wire contract。

#### 為什麼這個問題是阻塞級

因為這兩個錯誤碼代表兩種完全不同的運營語義：

1. **`0x06` out-of-sync / freshness lag**
   - 問題可能是單點 verifier 落後
   - 切到其他 node 有機會恢復
   - client 可以保守重試

2. **`0x07` rollback / signature invalid**
   - 這更像 verifier 本地 trust store 污染、控制面簽章鏈異常、
     或重大安全事件
   - 若 client 只是盲目 switch node，可能把本地重試放大成全域 storm
   - 在某些部署模型下，這反而應該觸發 route-group 級 taint
     或 operator alert

#### 建議的具體演進方向

應把這一塊徹底拆開，不要再混寫：

1. **重新定義 server-side mapping**
   - `Realm expired but signature valid` -> `0x06`
   - `Realm not yet refreshed / freshness lag` -> `0x06`
   - `Realm signature invalid` -> `0x07`
   - `Realm rollback detected` -> `0x07`

2. **把 `protocol.md` 的 recovery matrix 改成 actor-aware**
   - `Detected By`
   - `Classification`
   - `Local Action`
   - `Peer-visible Signal`
   - `Client Retry Scope`
   現在雖然表頭已經開始 actor-aware，但 `0x06 / 0x07` 的處理仍然把
   verifier 本地異常和 client retry policy 黏得太死。

3. **分離 client recovery**
   - `0x06`:
     `Switch Node` 合理，但必須帶 route-group 內退避與抖動
   - `0x07`:
     不應只寫 `Switch Node`
     應明確規定為：
     `Taint current route-group / suppress aggressive retries / require control-plane refresh or operator attention`

4. **補一句硬規範**
   - 任何節點在本地檢測到 `Realm Signature Invalid` 時，
     不應繼續以普通 out-of-sync node 的身分參與調度
   - 否則 routing layer 會把安全事件節點和普通 lagging node 等價處理

如果這一塊不修，`0x06 / 0x07` 最後只會變成表面上多了兩個 code，
實際恢復行為卻仍然和 `v14.0` 沒有本質差別。

---

## 2. 高優先級但非阻塞問題

### 2.1 兩階段調度仍然缺少幾個「算得出來」的細節

`control_plane.md` 現在已經有 Node/Endpoint 的 `Priority / Weight`，
這是很大進步，但還差最後幾個會造成實作分叉的邊界：

1. **`current round` 沒有明確定義**
   - 是一次 session establishment？
   - 一次 pool replenish？
   - 還是一次 route-group 級的整體 connect cycle？

2. **`Node_Weight = 0` / `Endpoint Weight = 0` 的語義未定**
   - 代表不可選？
   - 代表最低權重但仍可選？
   - 若一個 bucket 內全部為 0，是否視為配置錯誤？

3. **backoff 公式沒有規範化**
   - 目前只有示例：初始 5s，最大 300s
   - 缺 multiplier、jitter ratio、reset 條件

4. **success reset 條件不清楚**
   - 某 node 在 backoff 後成功一次，是否立即清零 backoff tier？
   - 還是採用 decay？

#### 建議的具體演進方向

至少補上以下硬規範：

1. `Round = one route-group connection attempt until success or all buckets exhausted`
2. `Weight = 0` 明確禁止，視為 manifest invalid
3. `backoff = min(base * 2^n, max) * jitter(0.8..1.2)`
4. 任一 node 成功完成握手並通過 `CLIENT_AUTH` 後，該 `Node_ID`
   的 health backoff tier 立即重置

這些看起來像工程細節，但若不寫死，不同 client 之間的流量分佈與故障
收斂速度會完全不同。

---

### 2.2 `Auth Realm` 仍然是單體全量模型，長期效率風險沒有解除

`v15.0` 已經透過 `Not_Before / Not_After` 降低了頻繁重發整份 realm 的
壓力，但本質上 `AuthRealmBody` 仍然是一個全量簽發、全量輪詢、全量替換
的模型。這在早期完全可行，但如果專案的設計目標是面向未來、易於優化，
那就應該及早承認這是未來的壓力點。

#### 問題在哪裡

1. 新增一條 credential，要重發整份 realm
2. 撤銷一條 credential，要重發整份 realm
3. verifier refresh 時，拿到的是全量快照，不是增量
4. `Realm_Sequence` 一旦足夠頻繁地遞增，控制面與 verifier 的同步成本
   會很快從「可以接受」變成「明顯偏重」

#### 為什麼這一輪還不用把它當阻塞點

因為現在最優先的是保證 correctness 與 fail-closed，不是過早優化。
但應該在文檔裡正式承認這是**下一階段演進方向**，而不是等到實作做大後
再返工。

#### 建議的演進方向

可以先不改現行穩定規格，但在 `control_plane.md` 補一段
`Future Evolution Notes`，明確列出可接受的升級路徑：

1. **Snapshot + Delta**
   - 保持全量快照作為基線
   - 新增 `RealmDelta` 物件承載小變更

2. **Sharded Auth Realm**
   - 按 `Credential_ID` 前綴或租戶維度切 shard
   - verifier 只拉自己需要的 shard

3. **Epoch-based Realm Families**
   - 用 `Realm_ID / Shard_ID / Realm_Sequence`
     取代單一全域 realm 序列

先把方向寫進去，未來擴展時才不會把現在的 wire semantics 打碎。

---

### 2.3 `design.md` 對 v15.0 的總結仍然過度樂觀

`design.md` 最後一句說：
「控制面下發的物件已經具備了完整的、抗噪聲的生命週期與失敗收斂能力。」

這句話現在還是說早了。原因不是方向錯，而是上面三個問題都還沒完全閉環：

1. verifier 時間語義未定
2. rollover 與 node health 的組合狀態機未定
3. `0x06 / 0x07` 的 wire semantics 仍分叉

建議把這句收斂成更準確的版本，例如：

`v15.0 已經補上控制面生命週期與失敗收斂的主要骨架，
但 verifier 時間模型、rollover/backoff 組合規則、以及 0x06/0x07 的
最終語義仍待收斂。`

這不是文字潔癖。設計文檔如果過早宣布閉環，後續 reviewer 和實作者都會
低估仍然存在的系統風險。

---

## 3. 建議的下一輪收斂順序

如果要加快迭代，下一輪不應再平均用力，而應按下面順序收斂：

1. **先補 `Verifier Time Semantics`**
   - 這會同時影響 `Auth Realm`、`ClientCredentialRecord`
     與 `0x05 / 0x06 / 0x07` 的分類

2. **再補 `Node Backoff x Key Rollover Composition`**
   - 把 `(Node_ID, Endpoint, KeyChoice)` 的完整狀態機寫成決策順序

3. **然後修 `0x06 / 0x07` 的 code table、matrix、client recovery**
   - 三處一起改，避免再次分叉

4. **最後補 scheduling 精確數值**
   - round 定義
   - zero-weight 規則
   - backoff 公式
   - reset 條件

5. **把 `Auth Realm` 的擴展路線寫成附錄或 future note**
   - 不必立刻改 wire format
   - 但必須明確承認未來演進方向

---

## 4. 總結

`v15.0` 是一個明確前進的版本。現在的主要問題不再是「缺一個物件」，
而是「多個已經存在的機制尚未組合成唯一行為」。這代表協議正在從概念收斂
走向真正的可互通實作，但也意味著 reviewer 的要求必須提高：下一輪需要的
不是再補名詞，而是把狀態機與時間語義寫到不同團隊盲寫仍不會分叉的程度。

如果只看方向，`v15.0` 是正確的；如果看可互通與可運營的精確度，
它還沒有完工。
