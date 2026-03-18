# Chameleon Review
## 對 `design.md` v13.0、`protocol.md` v13.0 與 `control_plane.md` v13.0 的新一輪聯合審查

這一輪和上一輪相比，已經不是「方向有沒有改對」的問題，而是你們開始真正把
control plane object 補成能支撐 data plane 的系統規格。

v13.0 的實質進展可以正式承認：

1. `Bearer_Type / Bearer_Params` 取代了之前錯位的 `Transport_Class`
2. `Auth Realm Manifest` 正式進入控制面，`CLIENT_AUTH` 不再依賴私有 verifier DB
3. `Manifest_Sequence` 的 issuer-side contract 已被寫進規範
4. `Node_ID` 與 `Endpoint` 的複合調度單位已被引入
5. `Current -> Next` 的快取 key / TTL / invalidation 已有正式規則
6. `design.md` 對底層承載的描述已與 `protocol.md` 對齊

所以這一輪 review 的焦點，不再是上一版那三個大裂口，而是：
你們剛補進來的新規格，還有哪些地方沒有真正形成閉環。

---

## 0. 本輪可以正式關閉的舊問題

### 0.1 控制面 endpoint 與資料面承載模型的大方向已經對齊
上一輪最嚴重的問題之一，是 control plane 還在混用 `TCP / UDP / WebSocket`
這種不在同一層級的 transport 概念。現在改成 `Bearer_Type + Bearer_Params`，
方向是正確的，這個老問題可以正式降級。

### 0.2 `CLIENT_AUTH` 已不再建立在 verifier 私有資料庫上
`Auth Realm Manifest` 的出現，至少把 `Credential_ID -> Public_Key` 的查驗依據
正式放回了規格層。這個結構性缺口已經不再是空白。

### 0.3 `Manifest_Sequence` 不再只靠 client 端硬扛
你們現在不只規定了 client 要嚴格 anti-rollback，也補上了 issuer 端不得本地自增、
必須具備 monotonic contract 的要求。這比上一輪完整很多。

### 0.4 `Node_ID` / `Endpoint` / rollover cache 的耦合關係已開始成形
`Cache Key`、`TTL`、`Invalidation` 都被寫進控制面，這讓 `Current -> Next`
fallback 不再只是策略建議，而開始接近真規格。

---

## 1. 阻塞級問題

## 1.1 `Auth Realm Manifest` 雖然出現了，但它還沒有自己的 freshness、anti-rollback 與 verifier fail-closed 語義

### 攻擊
這一輪最大的新增是 `Auth Realm Manifest`，但它現在只定義了：

1. `Realm_Version`
2. `Realm_Sequence`
3. `Signer_Key_ID`
4. `Record_Count`
5. `ClientCredentialRecord[]`

它沒有定義的是：

1. `Issued_At / Expires_At`
2. `Realm_Sequence` 的 anti-rollback 規則
3. signer rotation 下 `Realm_Sequence` 是否同樣全域單調
4. verifier 在 `Auth Realm Manifest` 缺失、過期、簽章無效、或輪詢失敗時應採取什麼動作

這會直接產生兩個安全與運營後果：

1. 被撤銷的憑證可能因為 edge 持有過時的 `Auth Realm Manifest` 而被長時間繼續接受
2. 被回放的舊 `Auth Realm Manifest` 可能重新讓 `Revoked` credential 變成 `Active`

也就是說，`CLIENT_AUTH` 的 verifier source 有了，但 verifier source 本身仍然沒有生命週期規格。

### 為什麼目前還不夠
你們在 `Node Manifest` 上已經接受了：

1. freshness 需要 `Issued_At / Expires_At`
2. anti-rollback 需要 sequence
3. issuer 端需要 monotonic contract

但在 `Auth Realm Manifest` 上，這三件事又幾乎全部缺席。

這會造成非常不對稱的系統狀態：

1. 節點公鑰輪換是 bounded-staleness 的
2. 客戶端憑證撤銷卻可能是 unbounded-staleness 的

對一個把 `CLIENT_AUTH` 作為 data plane 第一個加密 record 的系統來說，這是不成立的。

### 具體修改方案
我建議直接照 `Node Manifest` 的嚴格度補齊，不要另起爐灶：

1. 在 `AuthRealmBody` 中加入：
   - `Issued_At`
   - `Expires_At`
   - `Realm_Sequence`
2. 明確規定：
   - edge / verifier 必須對 `Realm_Sequence` 執行 anti-rollback
   - signer rotation 時不得重置 `Realm_Sequence`
3. 補 verifier fail-closed 規則：
   - 若 `Auth Realm Manifest` 簽章無效，拒絕所有新的 `CLIENT_AUTH`
   - 若 `Auth Realm Manifest` 已過期且無新版本，拒絕所有新的 `CLIENT_AUTH`
   - 不允許在 auth realm 缺失時退回任何私有本地資料庫
4. 在 `protocol.md` recovery matrix 新增至少三條：
   - `Auth Realm Signature Invalid`
   - `Auth Realm Expired`
   - `Auth Realm Rollback Detected`

這一項現在是本輪最重要的阻塞點，因為 v13.0 已經把 auth verifier 正式引入控制面，
但只補了「資料結構」，還沒有補「生命週期」。

---

## 1.2 `Route_Group` 的跨 node 調度演算法反而被你們自己切斷了，現在只剩「node 內 endpoint 調度」，沒有「route group 內 node 調度」

### 攻擊
v13.0 在 `control_plane.md` 中明確寫出：

1. 調度的最小單位是 `(Node_ID, Endpoint)`
2. `Priority / Weight` 僅決定**特定 `Node_ID` 下**不同撥號入口的順序與加權
3. key rollover / failure cache 一律以 `Node_ID` 為主鍵

這裡的問題是：你們把「node 內 endpoint 調度」寫清楚了，卻沒有再定義
「同一 `Route_Group_ID` 下多個 `Node_ID` 之間怎麼選」。

換句話說，現在還缺一個上層 selection step：

1. 先選哪個 `Node_ID`
2. 再選該 `Node_ID` 下哪個 `Endpoint`

如果這一步不明確，不同 client 仍然可能做出完全不同的 route selection：

1. 按 manifest 順序選第一個 node
2. 在所有 node 中隨機選一個
3. 基於本地失敗歷史做私有排序

這會直接影響：

1. route group 的負載分佈
2. 故障切換順序
3. 客戶端在 hostile network 下的收斂行為

### 為什麼目前還不夠
你們上一輪接受了「調度單位必須和安全主鍵分開」這個觀點。
但現在的文檔只走到了一半：

1. 安全主鍵是 `Node_ID`
2. 撥號入口是 `(Node_ID, Endpoint)`

真正還缺的是 route-group level 的 node selection 規則。

### 具體修改方案
這裡必須二選一，不要再保持模糊：

1. **兩階段調度**
   - 第一階段：在同一 `Route_Group_ID` 內按 `Node_Priority / Node_Weight` 選 `Node_ID`
   - 第二階段：在該 `Node_ID` 內按 `Endpoint Priority / Weight` 選入口
2. **單階段全域池化**
   - 直接把 `(Node_ID, Endpoint)` 作為 route-group 內的全域候選池
   - 但此時 `Priority / Weight` 就不能只對 node 內生效，而要對全域池生效

如果你們要維持目前「安全狀態以 node 為主、入口選擇以 endpoint 為主」的方向，
我更建議第一種，因為：

1. 路由策略更清楚
2. 故障統計更可控
3. 不會把 node health 與 endpoint health 混在同一層

這一項不補，`Route_Group_ID` 仍然不算真正可操作的規格。

---

## 1.3 `Bearer_Params` 雖然方向正確，但最後幾個關鍵欄位仍然沒有真正封口，不同實作仍可能做出不同撥號

### 攻擊
你們現在把 `Bearer_Params` 分成了：

1. `TCP_STREAM`
2. `QUIC_STREAM`
3. `WS_STREAM`

這比上一輪好很多，但還留下了幾個不能靠猜的欄位：

1. `QUIC_STREAM` 的 `SNI_Mode` 沒有枚舉表
2. `WS_STREAM` 的 `TLS_Mode` 沒有枚舉表
3. `WS_STREAM` 的 `Path` 沒有 grammar
4. `Host_Header` 是否可為空、是否必須等於 `Host`、是否可與 SNI 不一致，都沒被定義
5. `ALPN` 是否允許空字串、是否只允許單一值，也沒被定義

這些不是 UI 細節，而是不同 client 會不會連到同一個實際入口的問題。

### 為什麼目前還不夠
當前的 bearer schema 仍然停留在「欄位存在」，但還沒有做到「欄位語義不可分叉」。
尤其在 QUIC / WS 這兩種承載上，下面這些行為如果不寫死，最終都會出現實作私貨：

1. `SNI_Mode = 0x00/0x01/...` 分別代表什麼
2. `TLS_Mode` 是 `Plain / TLS / Inherit` 還是別的
3. `Path` 是否必須以 `/` 開頭
4. `Host_Header` 缺失時如何導出預設值

### 具體修改方案
下一輪至少把這些枚舉與 grammar 補成不可分叉的表：

1. `SNI_Mode`
   - 例如：`0x00 = None`
   - `0x01 = Use Host As SNI`
   - `0x02 = Explicit SNI`，若要支持，則還需要 `SNI_Length || SNI`
2. `TLS_Mode`
   - 例如：`0x00 = Plain`
   - `0x01 = TLS`
3. `WS Path`
   - 必須為 ASCII
   - 必須以 `/` 開頭
   - 不允許空字串
4. `Host_Header`
   - 若長度為 0，是否代表使用 `Host`
   - 若非 0，是否允許與 `Host` 不一致
5. `ALPN`
   - 是否允許空
   - 是否只允許單一字串

這一項我把它列成阻塞，不是因為方向錯，而是因為它現在正處於
「大家都會補自己的默認值」的危險階段。

---

## 2. 高優先級但非阻塞問題

## 2.1 `Client Auth Fail` 的 recovery 現在仍然把 client-local 問題和 verifier freshness lag 混在一起

### 攻擊
`protocol.md` 目前對 `GOAWAY 0x05` 的建議仍然是：

1. 客戶端 refresh config 一次
2. 再失敗就視為 revoked 或 implementation bug

這在 v12.0 還勉強說得過去，但 v13.0 引入 `Auth Realm Manifest` 後，
這個恢復邏輯已經不夠了。

因為現在至少還多出一種完全不同的可能：

1. edge / verifier 持有過期或回退的 `Auth Realm Manifest`

這時候 client refresh 自己的 config，可能根本沒有任何作用。

### 具體修改方案
等 `Auth Realm Manifest` 的 freshness 語義補齊後，`recovery matrix` 至少要把
`0x05` 的本地推論拆成兩層：

1. client-local credential problem
2. verifier-side auth realm freshness problem

不一定要新增錯誤碼，但至少要在恢復策略裡承認這兩種來源不同。

---

## 2.2 `design.md` 的反駁段落已經開始過度宣稱“完美閉環”，這會讓後續 review 失去抓手

### 攻擊
`design.md` 現在把 v13.0 的回應寫成：
控制面物件已能「完美且無歧義」地支撐資料面所有假設。

這個表述現在還不成立。

原因不是你們沒進步，而是：

1. `Auth Realm Manifest` 還沒有 freshness / anti-rollback
2. `Route_Group` 還沒有跨 node selection 規則
3. `Bearer_Params` 還有枚舉與 grammar 缺口

### 具體修改方案
把這種總結語氣收斂成工程上更準確的版本，例如：

1. `v13.0` 已完成主要 control-plane object 的引入
2. 尚餘 auth realm lifecycle、route-group selection、bearer enum closure 三項待收斂

這樣你們後面做 review 時，文檔本身就不會和現實打架。

---

## 2.3 `control_plane.md` 的 markdown 結構目前有明顯的 code fence 錯位，這會直接降低規格可讀性

### 攻擊
`NodeEntry` 與 `Endpoint` 那一段現在仍然有 code fence 沒收好的問題，
會讓部分 markdown renderer 把文字說明吞進 code block。

這不是協議語義問題，但對一份靠文檔驅動互通的規格來說，它不是小事。

### 具體修改方案
直接整理這一段：

1. `NodeEntry` 的 code block 單獨關閉
2. `Endpoint` 的 code block 單獨開啟與關閉
3. `Bearer_Type`、`Host_Type`、調度規則放到正常正文，不要夾在 code fence 內

---

## 3. 建議下一輪直接落地的修改順序

如果目標是繼續快收斂，下一輪我建議按這個順序改：

1. 先補 `Auth Realm Manifest` 的 freshness / anti-rollback / verifier fail-closed
2. 再定死 `Route_Group` 的跨 node selection 演算法
3. 把 `Bearer_Params` 的枚舉表與 grammar 補成不能分叉的最終版
4. 之後再回頭細化 `0x05` 的恢復策略
5. 最後清理 `design.md` 的總結語氣與 `control_plane.md` 的 markdown 結構

---

## 4. 本輪結論

v13.0 是明顯比 v12.0 更成熟的一版，這點不需要客氣。你們已經開始把 control plane
真正做成 protocol 的一部分，而不是旁白。

但新的主事實也很明確：

1. `Node Manifest` 這條線已經接近收口
2. 真正新的主戰場已經轉到 `Auth Realm Manifest`
3. 其次是 `Route_Group` 的 node-level selection 與 `Bearer_Params` 的最後封口

把這三件事補齊之後，下一輪 review 才適合真正往性能邊界、session pool 調度、
以及更細的故障域分析繼續下沉。
