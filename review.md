# Chameleon Review
## 對 `design.md` v11.0、`protocol.md` v11.0 與 `control_plane.md` v11.0 的新一輪聯合審查

這一輪和上一輪相比，真正值得肯定的是：你們已經把前一輪最重要的幾個阻塞點
真正做成了規格，而不只是寫了更好聽的描述。這包括：

1. `Node Manifest` 已經把 route metadata 簽進 canonical body
2. `Deterministic Key Rollover` 不再依賴明顯錯誤的 `Responder Key Mismatch` 觸發條件
3. `Capability Mismatch` 已基本收斂到 client-local abort
4. `Recovery Matrix` 已開始變成 actor-aware 結構
5. `Epoch Cert` 已對齊到 context-separated signature message
6. `KEY_UPDATE` 和 provisional session 都開始有硬性上限

這些都不是小修小補，而是把規格從「大致可說通」拉到了「開始接近真正可互通」。

但也正因為如此，這一輪的剩餘問題會更偏向最後一公里：
不是大方向錯，而是幾個語義還不夠精確，足以讓不同實作做出不同決策。

---

## 0. 本輪可以正式關閉的舊問題

### 0.1 `Node Manifest` 已不再缺路由元資料
上一輪最大的控制面問題是：route metadata 沒有被正式簽進 canonical body。
這一輪 `control_plane.md` 已把 `Endpoint / Priority / Weight / Transport_Class`
納入 `NodeEntry`，這個批評可以正式關閉。

### 0.2 `Capability Mismatch` 的大方向已經對了
`protocol.md` 現在已經明確寫出：server 只回 `Selected Caps`，client 本地比對後 abort。
這比上一輪三套語義併存進步很多。

### 0.3 `Recovery Matrix` 已不再完全混淆 actor
它仍然有可挑的地方，但至少表頭與主體意識已經開始成形，不再是上一輪那種
完全把 client-local 驗證失敗和 server 可見動作混成一鍋的狀態。

### 0.4 `Epoch Cert` 已對齊到獨立簽名訊息模型
`EpochCertSigMessage` 現在已經被明確寫進 `protocol.md`。這個問題不再是主阻塞。

---

## 1. 阻塞級問題

## 1.1 路由元資料雖然已經被簽進 `Node Manifest`，但其**語義**仍然沒有被規範化，客戶端仍然可能合法地做出不同選擇

### 攻擊
這一輪你們解決的是「這些欄位有沒有被簽」，但還沒有完全解決「這些欄位到底怎麼用」。
目前最明顯的三個未定義語義是：

1. `Endpoint_String` 的語法與正規化規則
2. `Transport_Class` 的枚舉值與它和 endpoint scheme 的關係
3. `Priority / Weight` 的實際調度語義

現在的 `NodeEntry` 雖然簽了 route metadata，但 client 仍然不知道：

1. `Priority` 是不是數值越小越優先
2. `Weight` 是在同 priority 內做 weighted-random，還是只做 tie-break
3. `Transport_Class` 是一個硬性約束、偏好標記，還是與 endpoint scheme 重複
4. `tcp://host:443` 和 `Transport_Class = 0x02` 若互相矛盾，哪個欄位優先
5. 一個 `NodeEntry` 內若有多個 `Endpoint`，但只有一組 `Priority / Weight /
   Transport_Class`，那麼這些 metadata 到底是作用於整個 node，還是作用於每個
   endpoint 的候選集合

這些語義不收斂，最終結果就是：簽名保護了資料，但沒有保護「客戶端會如何解讀資料」。

### 為什麼目前還不夠
Chameleon 的設計目標不是只保證 parser 互通，而是希望不同團隊盲寫也能得到
可預期的行為。如果 route metadata 的使用語義不固定，那麼：

1. 同一份 manifest 在不同客戶端上會導致不同的節點選擇
2. route_group 的故障切換時機與順序會漂移
3. A/B 測試、性能調優和事故定位都會變得不可重現
4. 多 endpoint 的 `NodeEntry` 會因為「先選 node 還是先選 endpoint」的差異而產生
   額外分叉

### 具體修改方案
下一輪應至少把下面這些規則寫死：

1. `Endpoint_String` 的 grammar
   - 若保留 ASCII 字串，就要明確定義 scheme、host 格式、port、IPv6 表示法
   - 更好的做法是改成結構化 endpoint：
     `Transport_Kind || Host_Type || Host || Port`
2. `Transport_Class` 的枚舉表
   - 每個值代表什麼
   - 它是 hard constraint 還是 soft preference
   - 與 endpoint scheme 發生衝突時，誰優先
3. `Priority / Weight` 的調度算法
   - 例如：先選最小 `Priority`
   - 在同 priority 集合內按 `Weight` 做 weighted-random
   - 失敗時是否在同 priority 內重試，還是升到下一個 priority bucket
4. `Endpoint[]` 與 node-level metadata 的關係
   - 若 `Priority / Weight / Transport_Class` 是 node-level，則必須明確規定同一
     `NodeEntry` 內所有 `Endpoint` 都是同一 transport class 下的等價入口
   - 若不是，則這三個欄位就不應放在 `NodeEntry`，而應移入每個 `Endpoint`
     條目內部

這一項如果不補，控制面雖然“有了”，但還不算真正可操作。

---

## 1.2 `Deterministic Key Rollover` 已經接近可用，但現在的 fallback trigger 仍然過寬，會把普通網路噪聲誤判成 key rollover 事件

### 攻擊
這一輪最大的正面修正之一，是把 rollover 的 fallback trigger 改成了
`Handshake Timeout / Handshake AEAD Fail / Silent Drop`，這比上一輪正確得多。

但新的問題是：這個 trigger 現在又太寬了。

在真實網路裡，timeout / silent drop 並不只來自 key rollover，也可能來自：

1. 短時丟包或抖動
2. 單一路由上的瞬時黑洞
3. 某個 endpoint 的暫時不可達
4. 一次性的中間網路異常

如果 client 只要在 overlap window 內遇到一次 timeout，就立即做 `Current -> Next`
fallback，那它很可能不是在應對 key rollover，而是在對普通網路噪聲做錯誤解讀。

### 為什麼目前還不夠
你們現在已經加了「同一輪撥號最多切換一次」這條硬規則，這是對的。
但這還不足以保證 fallback 真的是“因為 key 切換”，而不是“因為碰巧這次網路很差”。

在高噪聲網路下，這會帶來兩個後果：

1. client 會過早切到 `Next`，把純網路失敗誤標成 key rollover 事件
2. 控制面與運營側會看到被污染的失敗分類與時序

### 具體修改方案
建議把 fallback trigger 再收窄一層，不要只依賴結果類型，還要加入上下文條件：

1. 只有在 overlap window 內才允許 `Current -> Next` fallback
2. 只有當該 `Node ID` 明確宣告了 `Next_Static_Pubkey` 才允許 fallback
3. 只有在同一 endpoint 上出現一次 pre-auth 失敗後，才允許對同一 `Node ID`
   做一次 `Current -> Next` 重試
4. fallback 之後應在本地短時間 cache 此決策，避免同一個 node 在短時間內反覆
   current/next 交替
5. 若 client 在同 route_group 的多個節點上同時觀察到同類 timeout，應優先視為
   route 問題，而不是 key rollover

更直接一點說：
你們現在已經把「錯誤的信號」改成了「可工作的信號」，下一步要做的是把它改成
「不容易誤觸發的信號」。

---

## 1.3 `Recovery Matrix` 雖然已經開始 actor-aware，但仍然缺少一層“錯誤分類粒度”，尤其是 auth / config / implementation fault 仍然混在一起

### 攻擊
這一輪 `Recovery Matrix` 的表頭已經比前幾輪成熟很多，但它仍然有一個殘留問題：
它把幾類本質不同的錯誤，仍然壓成了同一個 bucket。

最典型的是：

1. `Client Auth Fail (Token 錯)`
2. `Unsupported Auth Scheme`
3. `Epoch Cert Signature Invalid`
4. `Epoch Cert Expired`

這些錯誤雖然都與 auth / trust 有關，但實際上對應的是四種不同來源：

1. 本地憑證或授權材料問題
2. client / server 版本能力不匹配
3. 信任鏈或控制面資料失效
4. 普通時效性問題

如果這些分類不更細，client 最終還是會把一部分本地 bug 誤當作需要 refresh config，
把一部分需要 refresh config 的情況誤當作一般重試。

### 為什麼目前還不夠
現在的矩陣已經分出了 `Detected By / Local Action / Peer-Visible Signal`，這是對的。
但要讓恢復策略真正可運營，還要再加一層：

1. 這是 **local configuration fault**
2. 這是 **control-plane freshness fault**
3. 這是 **protocol/version mismatch**
4. 這是 **security failure**

沒有這層分類，client 最後還是很難決定到底應該：

1. 立刻 refresh config
2. 停止重試
3. 只做退避後重試
4. 提示升級客戶端

### 具體修改方案
不一定要新增新表，但至少要把這一層明確加進現有矩陣或錯誤碼附註：

1. `Client Auth Fail`
   - 區分：credential revoked / credential unknown / proof invalid
2. `Unsupported Auth Scheme`
   - 明確標成 protocol-version / feature mismatch，不屬於 config refresh
3. `Epoch Cert Signature Invalid`
   - 明確標成 trust-chain or control-plane freshness failure
4. `Epoch Cert Expired`
   - 明確標成 freshness / rollout lag，而不是 generic auth fail

如果短期內不想新增錯誤碼，至少要把現有錯誤在文檔中映射到更細的本地處置分類。

---

## 1.4 `Manifest_Sequence` 的防回滾語義仍然缺少命名空間定義，到了 CPSK 輪換時很容易把合法更新錯判為 rollback

### 攻擊
`control_plane.md` 現在要求客戶端持久化「已接受的最高 `Manifest_Sequence`」，這個方向
本身是對的，但它還缺一個非常關鍵的限定：

這個 sequence 到底是：

1. 全域單調遞增
2. 針對某個 manifest lineage 單調遞增
3. 針對某個 `Signer_Key_ID` 單調遞增

這不是文書細節，而是會直接影響 CPSK 輪換後客戶端還能不能接受新 manifest。

如果未來控制面更換了新的 `CPSK`，而新簽發端從較小的 sequence 重新開始，按照目前規則，
client 會把它視為 rollback 並永久拒絕。反過來說，如果 sequence 其實只要求對單一
`Signer_Key_ID` 單調，那麼跨 signer 的 anti-rollback 力度又會被削弱。

### 為什麼目前還不夠
現在文檔把 `Manifest_Sequence` 和 `Signer_Key_ID` 都放進了 body，但沒有定義它們之間的
命名空間關係。這會讓不同實作做出三種完全不同的持久化策略：

1. 全域只存一個最高 sequence
2. 每個 signer 存一個最高 sequence
3. 每個 route group / manifest family 存一個最高 sequence

這三種策略都能“看起來合理”，但互通結果完全不同。

### 具體修改方案
這一項應直接在 `control_plane.md` 寫死，不要留給實作猜：

1. 若你們要全域 anti-rollback，則必須明確規定 `Manifest_Sequence` 在整個控制面
   命名空間內跨 `CPSK` 也必須單調遞增
2. 若你們不想把 sequence 與 signer 綁死，則應新增一個固定的 `Manifest_Family_ID`
   或 `Config_Lineage_ID`，client 以 `(Family_ID, Sequence)` 做 anti-rollback
3. 若暫時不想引入新欄位，至少要明確寫出：CPSK 輪換不得重置 manifest sequence，
   否則所有舊 client 都可能把新 manifest 當作 rollback

這個問題如果不補，控制面在“穩態”下能跑，但一到 signer rotation 就可能出現大面積
合法配置被拒絕的事故。

---

## 2. 高優先級但非阻塞問題

## 2.1 `Endpoint_String` 現在仍然是“簽名上安全、語義上脆弱”的欄位

### 攻擊
把 `Endpoint_String` 直接簽進 canonical body，確實解決了完整性問題。
但它沒有解決另外一個問題：字串型 endpoint 太容易產生語義上的解析分歧。

例如：

1. host 是域名時是否大小寫規範化
2. IPv6 是否要求 bracket 格式
3. 是否允許默認 port
4. 是否允許額外 path/query

如果這些不寫死，不同 client 仍然可能對同一個 `Endpoint_String` 做出不同解讀。

### 具體修改方案
如果你們想把這件事徹底做乾淨，我仍然建議最終放棄字串 endpoint，改成結構化二進位：

1. `Transport_Kind`
2. `Host_Type`
3. `Host`
4. `Port`

如果短期內不改，至少先在 `control_plane.md` 明確寫出 endpoint grammar。

---

## 2.2 `KEY_UPDATE` 的 mandatory trigger 雖然補上了，但“計數什麼”仍然不夠精確

### 攻擊
這一輪你們補上了：

1. `MAX_RECORDS_PER_GENERATION`
2. `MAX_BYTES_PER_GENERATION`
3. `MAX_KEY_AGE`

這是實質進步。但 `MAX_BYTES_PER_GENERATION = 1 TB` 這條，現在仍然缺一個關鍵定義：

到底算的是：

1. plaintext bytes
2. ciphertext bytes
3. record payload bytes
4. application bytes

同樣地，`MAX_KEY_AGE = 1 小時` 也還缺一個參考起點：

1. 從 `Split()` 開始算
2. 從上一代 commit 開始算
3. 從第一次使用該 generation 發包開始算

### 具體修改方案
把這三個常數再補精確：

1. `MAX_RECORDS_PER_GENERATION`
   - 明確是 per-direction、per-generation 的 encrypted record count
2. `MAX_BYTES_PER_GENERATION`
   - 明確是 per-direction ciphertext bytes 或 plaintext bytes
3. `MAX_KEY_AGE`
   - 明確起點是 generation commit timestamp

這樣不同實作才不會在 rekey 時機上悄悄漂移。

---

## 2.3 `Client Auth Fail` 的本地恢復策略仍然太粗，運營上會把“憑證撤銷”和“本地實作錯”混成一類

### 攻擊
現在 `Client Auth Fail (Token 錯)` 在矩陣裡仍然偏粗。對運營來說，
下面幾種情況需要完全不同的處理：

1. credential 已撤銷
2. credential id 不存在
3. proof 生成錯誤
4. client 使用了舊格式 proof

把這些都寫成 `Check config (Refresh possible)`，對實作是夠用的，對生產事故卻不夠。

### 具體修改方案
如果短期內不想增錯誤碼，至少在文檔裡補一條：

1. `GOAWAY 0x05` 只是 auth 失敗的大類
2. client 本地應維護失敗計數與最近 config version
3. 連續 refresh 後仍然 `0x05`，應提升為本地 credential / implementation 問題，而不是繼續打控制面

---

## 3. 建議下一輪直接落地的修改順序

如果目標是加快收斂，下一輪我建議按這個順序改：

1. 先把 route metadata 的**使用語義**寫死：endpoint grammar、transport class、priority/weight algorithm
2. 明確 `Manifest_Sequence` 的命名空間與 CPSK 輪換語義，避免 anti-rollback 在 signer rotation 時誤傷
3. 收窄 `Current -> Next` fallback 的觸發條件，避免把普通網路噪聲誤判成 key rollover
4. 為 recovery matrix 補一層更細的本地錯誤分類：config / protocol / security / implementation
5. 把 `KEY_UPDATE` 的 bytes/age 計數語義寫精確
6. 再決定是否要把 endpoint 最終從字串改成結構化二進位表示

---

## 4. 本輪結論

這一輪我不認為協議還停留在“概念階段”了。你們已經把幾個真正難啃的結構問題
做成了規格，這點要正式承認。

現在剩下的問題，已經不是大框架錯，而是：

1. 控制面資料怎麼被所有 client **以同樣方式理解**
2. anti-rollback 在控制面 signer rotation 時怎麼保持**既嚴格又不中斷**
3. 噪聲網路下的 rollover fallback 怎麼做到**不誤觸發**
4. auth / refresh / rekey 這些恢復語義怎麼做到**對運營足夠清晰**

把這四件事補齊之後，下一輪 review 就可以真正往參數選型、調度策略和性能邊界
繼續下沉了。
