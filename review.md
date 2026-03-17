# Chameleon Review
## 對 `design.md` v12.0、`protocol.md` v12.0 與 `control_plane.md` v12.0 的新一輪聯合審查

這一輪和上一輪相比，最大的變化不是又多了幾段辯護，而是你們真的把幾個
會導致實作分叉的硬問題往前推了一大步。

真正可以正式承認的進展有：

1. `Endpoint` 已從字串提升為結構化二進位
2. `Priority / Weight` 的調度語義已經首次被寫成明確規則
3. `Manifest_Sequence` 的命名空間與 CPSK 輪換語義已被明示
4. `Current -> Next` rollover fallback 已加入單一 endpoint 約束與本地決策快取
5. `KEY_UPDATE` 的 bytes / age 計數語義已被明確化
6. `Recovery Matrix` 已正式引入 fault category
7. `GOAWAY 0x05` 後的客戶端避險策略已不再是完全空白

所以這一輪 review 的重點，不再是追著舊漏洞重複攻擊，而是要指出：
你們解掉了上一批歧義後，新的規格邊界在哪裡又露出了裂縫。

---

## 0. 本輪可以正式關閉的舊問題

### 0.1 `Endpoint_String` 的解析歧義已經基本關閉
上一輪我對 `Endpoint_String` 的主要攻擊是 grammar、正規化和多 endpoint /
node-level metadata 的關係。這一輪把 endpoint 提升成結構化二進位後，
「字串該怎麼解讀」這個舊問題可以正式降級。

### 0.2 `Manifest_Sequence` 的命名空間歧義已經被正面回答
你們現在明確寫出 `Manifest_Sequence` 是跨 signer 的全域單調遞增值。這至少讓
client 的 anti-rollback 持久化策略不再需要靠猜。

### 0.3 `KEY_UPDATE` 的強制門檻不再是示意值
`MAX_RECORDS_PER_GENERATION`、`MAX_BYTES_PER_GENERATION`、`MAX_KEY_AGE`
現在都已經被精確到足以避免不同實作在 rekey 時機上悄悄漂移。

### 0.4 `Recovery Matrix` 已經不再只是 actor-aware，而是開始可運營
fault category 的加入是正確方向。雖然還沒完全細到理想狀態，但已不再是之前那種
只給一個籠統恢復建議的草稿水平。

---

## 1. 阻塞級問題

## 1.1 `Structured Endpoint` 解掉了舊的字串問題，卻把「控制面 endpoint 類型」和「資料面底層承載模型」直接撞在一起了

### 攻擊
`protocol.md` 仍然明確要求 Chameleon 運行在**可靠、有序的位元組流**之上，
例子寫的是 `QUIC Stream / TCP / WebSocket`。

但 `control_plane.md` 現在給出的 `Transport_Class` 是：

1. `0x01` TCP
2. `0x02` UDP
3. `0x03` WebSocket

這三個值放在一起，語義其實是衝突的：

1. `UDP` 本身不是可靠、有序的位元組流，無法直接承載目前的資料面協議
2. `WebSocket` 不是只有 `Host + Port` 就能撥號，還需要至少 `Path`，
   很多部署還會依賴 `TLS Mode / SNI / ALPN / Host Header`
3. 若 `UDP` 其實想表示的是 `QUIC`，那麼欄位命名就錯了，因為 `UDP` 和
   `QUIC Stream` 不是同一層東西

更糟的是，`Endpoint` 現在仍然包含一個通用的 `Host_Length`，但同時又說
`IPv4` / `IPv6` 的 Host 是固定 4/16 bytes。這會讓不同實作出現兩種合法分支：

1. 有人按照 `Host_Type` 直接讀固定 4/16 bytes，忽略 `Host_Length`
2. 有人嚴格按 `Host_Length` 讀，然後把 `Host_Type` 只當驗證欄位

這不是文檔措辭問題，而是 wire object 還沒有完全收口。

### 為什麼目前還不夠
這一版最大的問題不是 endpoint 不結構化，而是 endpoint 的結構仍然沒有和
資料面假設對齊。

現在的控制面物件同時混了三件事：

1. 底層網路傳輸族類
2. 具體撥號所需參數
3. 資料面協議對承載的能力要求

如果這三件事不分開，你們最後會遇到兩種壞結果：

1. control plane 下發一個 client 根本無法正確撥號的 endpoint
2. 不同 client 對同一個 endpoint 做不同的 transport interpretation

### 具體修改方案
這一塊我建議不要再用現在的 `Transport_Class` 硬撐，直接重構成二層：

1. `Bearer_Type`
   - 例如：`0x01 TCP_STREAM`
   - `0x02 QUIC_STREAM`
   - `0x03 WS_STREAM`
2. `Bearer_Params`
   - `TCP_STREAM`: `Host_Type || Host || Port`
   - `QUIC_STREAM`: `Host_Type || Host || Port || ALPN || SNI_Mode`
   - `WS_STREAM`: `Host_Type || Host || Port || Path || TLS_Mode || Host_Header`

如果短期內不想引入變長參數區，至少先把下面幾條寫死：

1. `UDP` 改名為 `QUIC`，或明確禁止把它解讀成 raw UDP
2. `WebSocket` endpoint 必須補 `Path`
3. 若 `Host_Type = IPv4/IPv6`，則 `Host_Length` 必須分別固定為 `4/16`，
   否則整個 `Node Manifest` 驗證失敗

這一項如果不改，v12.0 雖然看起來更結構化了，但控制面和資料面仍然沒有真正對齊。

---

## 1.2 `CLIENT_AUTH` 仍然不是一個完整閉環規格，因為控制面根本還沒有定義 client credential object

### 攻擊
`protocol.md` 現在對 `Auth Scheme 0x01` 的簽章算法已經比前幾輪完整很多，
但它仍然假設了一個不存在的前提：

伺服器可以根據 `Credential ID` 去「控制面下發的使用者資料庫」查出對應的
Client Ed25519 Public Key。

問題是：在 `control_plane.md` 中，這個物件根本不存在。

現在控制面只有：

1. `CPSK Certificate`
2. `Node Manifest`

但沒有任何一份規格定義：

1. `Credential ID` 如何生成，是否全域唯一
2. Credential 與 `Auth Scheme` 如何綁定
3. Client public key 如何分發到 edge / verifier
4. 撤銷、過期、快取 TTL、版本更新如何做
5. 多租戶或多 auth realm 如何隔離

所以目前的 `CLIENT_AUTH` 雖然在 wire format 上已經像回事了，但整體系統仍然缺一塊
控制面基礎設施規格。

### 為什麼目前還不夠
如果 control plane 不定義 credential object，那麼不同實作會自己補出不同模型：

1. 有人用靜態本地檔案查 `Credential ID`
2. 有人把它放進某個外部資料庫
3. 有人把 `Credential ID` 當作公鑰雜湊
4. 有人把撤銷當刪除，有人當 denylist

最後的結果是：

1. `Auth Scheme 0x01` 看起來可驗證，但其實無法跨實作互通
2. recovery matrix 中的 `0x05` 也無法對應到穩定的控制面語義

### 具體修改方案
下一輪至少要新增一個正式控制面物件，例如：

1. `ClientCredentialRecord`
   - `Credential_ID`
   - `Auth_Scheme`
   - `Client_Public_Key`
   - `Not_Before / Not_After`
   - `Status` (`active / revoked / suspended`)
   - `Realm_ID`
2. 或者新增一份 `Auth Realm Manifest`
   - 由控制面簽章
   - 可被 edge/verifier 快取
   - 定義撤銷與刷新語義

同時 `protocol.md` 也要配套補一句：

1. `Credential ID` 不是自由格式欄位，而是由控制面物件唯一綁定
2. `GOAWAY 0x05` 的 verifier 必須基於該控制面物件，而不是某個實作私有資料庫

這一項不補，`CLIENT_AUTH` 仍然只是“局部精確”，不是端到端精確。

---

## 1.3 `Manifest_Sequence` 的語義雖然終於清楚了，但你們現在把一個非常重的控制面可用性前提直接寫進規格，卻沒有定義它怎麼成立

### 攻擊
上一輪我打的是 sequence 命名空間不清楚。這一輪你們選了一個非常明確的答案：
全域單調遞增，跨 CPSK 也不能重置。

這個答案本身在安全性上是合理的，但它現在引入了一個新的、而且很重的前提：

整個控制面必須有能力在所有 signer / region / failover 節點之間，維持單一的、
絕不回退的 global sequence order。

這不是免費的。

如果你們未來有：

1. 多區域控制面
2. 災難切換
3. signer rotation
4. 短時間 split-brain

那麼「全域單調遞增」就不再只是邏輯要求，而是要求你們擁有某種全域序號分配機制。

### 為什麼目前還不夠
現在的文檔只定義了 client 驗證規則，沒有定義 issuer 端的生成規則。
這會讓規格出現一個危險落差：

1. 驗證端是嚴格的
2. 生成端卻沒有被規範

結果就是，實作很容易在正常運營切換時自己產出一份“簽名正確但 sequence 回退”的合法
manifest，然後被所有 client 拒絕。

### 具體修改方案
這裡不能只靠一句“必須全域單調”帶過，至少要把控制面生成模型寫成下面兩種之一：

1. **單寫者模型**
   - 任一時刻只有一個 active manifest issuer 可以遞增 sequence
   - failover 前必須完成 lease/epoch 交接
2. **外部分配器模型**
   - 所有 signer 從同一個 monotonic allocator 取得 sequence
   - allocator 的單調性是控制面安全前提之一

如果短期內不想把 control plane HA 寫太細，至少要在 `control_plane.md` 補一句：

1. `Manifest_Sequence` 的生成不是各 signer 本地自增
2. signer rotation / region failover 若不能保證全域單調，寧可暫停發佈，也不能發布回退 sequence

這是新的阻塞點，因為 v12.0 已經不再是“沒答案”，而是“答案需要 operational contract
支撐”。

---

## 2. 高優先級但非阻塞問題

## 2.1 `Current -> Next` 的本地決策快取方向是對的，但 cache key / TTL / invalidation 仍然沒有被寫死

### 攻擊
你們這一輪加入 `Cache decision` 是進步，但目前還停留在策略語義，沒有變成規格語義。

現在仍然沒被定義的是：

1. cache key 是 `Node_ID`，還是 `(Node_ID, Endpoint)`，還是 `(Route_Group_ID, Node_ID)`
2. cache 的有效期多久
3. manifest 刷新後是否立即失效
4. overlap window 結束時是否必須清空

如果這幾個點不寫死，不同 client 仍然可能出現：

1. 有人過度 stick 到 `Next`
2. 有人在噪聲網路裡很快又退回 `Current`

### 具體修改方案
我建議直接寫死最小規則：

1. cache key = `Node_ID`
2. TTL = `min(Overlap_Window_End - Now, Fixed_Local_Max_TTL)`
3. 收到更新的 `Node Manifest` 後立即失效
4. 若握手成功使用 `Current`，則立即覆蓋掉舊的 `Next` 決策

---

## 2.2 `Route_Group` 調度語義現在按 endpoint 展開了，但 key rollover 與故障統計仍然是按 node 定義，兩者的耦合關係還沒寫明

### 攻擊
`control_plane.md` 現在要求 client 在同一個 `Route_Group_ID` 內收集所有可用的
`Endpoint`，再按 `Priority / Weight` 調度。這本身沒問題。

但 `Deterministic Key Rollover`、`Current/Next`、`Epoch Cert` 綁定、以及失敗統計，
仍然都是以 `Node_ID` 為單位。

這代表 client 內部其實需要處理的是 `(Node_ID, Endpoint)` 這個複合實體，而不是單純的
endpoint 列表。

若不明說，實作者很容易做出兩種不同模型：

1. 先選 node，再選 endpoint
2. 直接把所有 endpoint 攤平後選一個

這兩種模型都能“看起來符合文檔”，但在 fallback、cache、故障統計上結果不同。

### 具體修改方案
建議在 `control_plane.md` 直接補一句規範：

1. endpoint 的調度單位是 `(Node_ID, Endpoint)`，不是脫離 node 的裸 endpoint
2. `Priority / Weight` 只決定撥號入口選擇
3. key rollover、security binding、failure cache 一律以 `Node_ID` 為主鍵

這樣控制面調度和資料面安全綁定才不會分家。

---

## 2.3 `design.md` 和 `protocol.md` 對底層承載的描述仍然有一處明顯不一致，這會繼續污染後面的 transport 設計

### 攻擊
`protocol.md` 很清楚地寫的是 reliable in-order byte stream。
但 `design.md` 第 0.1 節仍然寫了「協議將底層視為不可靠的位元組流承載」。

這句話如果只是筆誤，應立即修掉；如果不是筆誤，那就代表 `design.md` 和
`protocol.md` 在最根本的承載假設上仍然沒有完全一致。

### 具體修改方案
把 `design.md` 的表述直接收斂成與 `protocol.md` 一致的說法，例如：

1. 底層承載在**安全上不可信**
2. 但在**傳輸語義上**，當前穩定版本要求 reliable in-order byte stream

這樣才能避免後續又把 raw UDP 之類的概念寫進控制面 endpoint。

---

## 3. 建議下一輪直接落地的修改順序

如果目標是繼續快收斂，下一輪我建議按這個順序改：

1. 先把 `Structured Endpoint` 和資料面承載模型對齊，重做 `Transport_Class`
2. 補一份正式的 client credential / auth realm 控制面物件
3. 為全域 `Manifest_Sequence` 補 issuer-side operational contract
4. 把 rollover decision cache 的 key / TTL / invalidation 寫死
5. 明確 route scheduling 的主鍵是 `(Node_ID, Endpoint)` 還是別的複合單位
6. 清理 `design.md` 裡對承載可靠性的表述不一致

---

## 4. 本輪結論

v12.0 是一次真正有效的收斂，不是表面修詞。上一輪的幾個核心批評，這一輪大多都被
正面吸收了。

但新的事實也很清楚：

1. 你們現在的主要風險已經不在 record / rekey 這種純資料面細節
2. 真正還沒閉環的地方，集中在 **control plane object 是否足以支撐資料面假設**
3. 特別是 transport endpoint、client credential、manifest issuance contract，
   這三塊現在已經變成新的結構性阻塞點

把這三塊補齊之後，下一輪 review 才值得真正往性能邊界、調度策略與故障域隔離的
更深處下沉。
