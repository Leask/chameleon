# Chameleon Review
## 對 `design.md` v10.0、`protocol.md` v10.0 與 `control_plane.md` v1.0 的新一輪聯合審查

這一輪和上一輪相比，最大的變化不是新加了多少欄位，而是規格的問題開始從
「缺文件」轉成了「文件雖然存在，但還沒收斂到可互通、可運營」。

這是正向進展，但也意味著批評要更具體。現在真正值得打的點，已經不是那些
抽象方向問題，而是幾個會直接讓實作分叉、或者讓系統在真實網路下跑偏的
精確缺口。

先給總結：

1. `control_plane.md` 的出現、canonical body 的引入、以及 `design.md` 對三份文檔
   的職責分離，都是實質收斂。
2. `Session Pool` 的過度宣稱、文檔職責分工、Noise/X25519 reference 這些老問題，
   這一輪可以正式降級。
3. 新的最大問題有五個：
   - `Node Manifest` 的 canonical body 沒有覆蓋 endpoint / route metadata
   - rollover 算法依賴了在 `NK` 下實際上不會出現的失敗信號
   - `Capability Mismatch` 的處置仍然有三套互相衝突的語義
   - `Recovery Matrix` 仍然不是 actor-aware
   - `Epoch Cert` 仍然沒有 domain-separated signature message

下面只攻擊這些真正還沒收口的點。

---

## 0. 本輪已可正式降級的舊問題

### 0.1 三份規範文件的分工已經進入正軌
`design.md` 現在已經把 `design.md / protocol.md / control_plane.md` 的職責分開。
上一輪關於「規範集合本身沒有對齊」的批評，這一輪可以正式降級。

### 0.2 控制面不再只是 JSON 範例
`control_plane.md` 已經引入 `Canonical Body`、簽名上下文與明確的被簽物件邊界。
這比上一輪只有展示性 JSON 例子要好得多。

### 0.3 `Session Pool` 的官方表述已經收斂
`design.md` 的反駁段落現在已經承認：HOL 不能在協議層被消滅，只能把故障域隔離在
單一 session 內。這一條可以降級。

### 0.4 概述與 reference 的同步問題已被修掉
`protocol.md` 的 handshake layer 描述與 cryptographic references 已經比前幾輪嚴謹。
這一項不再是主攻點。

---

## 1. 阻塞級問題

## 1.1 `Node Manifest` 的 canonical body 目前沒有覆蓋 endpoint / route metadata，控制面還不能真正支撐節點發現與路由策略

### 攻擊
`control_plane.md` 現在對 `NodeEntry` 做了 canonical 化，這是進步。
但新的關鍵問題是：它目前只簽了下面這些東西：

1. `Node ID`
2. `Route Group ID`
3. `Current / Next Static Pubkey`
4. key 的時窗欄位

它**沒有**把以下對真實撥號和路由決策至關重要的資料納入 canonical body：

1. endpoint 列表
2. priority / weight
3. transport class
4. 任何節點連線元資料

這意味著現在會落入兩種壞情況之一：

1. 這些資料根本不存在於規格裡，client 無法從 manifest 完成節點發現
2. 這些資料存在於 JSON wrapper 或其他欄位裡，但沒有被簽名保護

如果是第二種，那就是嚴重問題，因為攻擊者可以不碰 key，直接改 endpoint 或權重，
把流量導向錯誤目標或扭曲 route selection。

### 為什麼目前還不夠
`design.md` 已經把控制面的職責明確寫成：

1. 節點發現
2. 路由策略下發
3. 版本治理

但 `control_plane.md` 目前的 canonical `NodeEntry` 還不足以承載這三件事。
也就是說，控制面雖然有了簽名物件，但還沒有把「會影響 client 真正連去哪裡」
的元資料簽進去。

### 具體修改方案
這一項不要再拖。你們必須二選一：

1. 把 endpoint / route metadata 直接納入 `NodeEntry` 的 canonical body
2. 單獨定義另一個被 CPSK 簽名的 `RoutingManifest`

如果選第一種，至少補上：

```text
Endpoint_Count (1 or 2 bytes) ||
Endpoint[0] || Endpoint[1] ... ||
Priority (4 bytes) ||
Weight (4 bytes) ||
Transport_Class (1 byte)
```

如果選第二種，也要明確說：

1. `NodeManifest` 只負責 key material 與 node identity
2. `RoutingManifest` 負責 endpoint / weight / route policy
3. 兩者都必須有 canonical body 和 context-separated signature

在 endpoint / route metadata 被正式簽名之前，我不認為控制面已經可以支撐真實部署。

---

## 1.2 `Deterministic Key Rollover` 現在依賴了錯誤的失敗信號，在 `NK` 下它並不會按文檔描述那樣工作

### 攻擊
這一輪 `control_plane.md` 補上了 key rollover 算法，方向是對的。
但現在它有一個更深的問題：它把 fallback 的觸發條件寫成了
`Responder Key Mismatch`。

這在 Noise `NK` 裡通常不是正確的失敗信號。

原因很簡單：在 `NK` 模式下，client 在發首包前就必須選定 responder static key。
如果 client 選了 `Current`，但 server 其實已經切到了 `Next`，那麼最常見的結果不是
「握手成功後收到一個不匹配的 Epoch Cert」，而是：

1. server 直接無法解密 client first flight
2. 握手在 pre-auth 階段就失敗
3. client 看到的是 timeout / silent drop / handshake AEAD fail

也就是說，現在文檔寫的這條：

`僅當本輪連線因為 Responder Key Mismatch 失敗時，才允許 Current -> Next fallback`

在真實 `NK` 失配場景下，很可能根本不會被觸發。

### 為什麼目前還不夠
你們這一輪已經正確認識到：

1. rollover 必須 deterministic
2. fallback 不能無限抖動
3. 同一輪撥號只能有限次切換

但如果 fallback 的觸發條件本身就依賴一個通常不會出現的錯誤類型，
那這個算法在紙面上是閉環的，在實際網路上卻不是。

### 具體修改方案
把 rollover 算法改成基於**撥號嘗試結果**而不是單一 `Responder Key Mismatch`：

1. `Now < next_not_before`
   - 只能使用 `Current`
2. `next_not_before <= Now < current_not_after`
   - 第一次嘗試必須使用 `Current`
   - 若該次嘗試以 `Handshake Timeout / Handshake AEAD Fail / Silent Drop` 結束，
     且此 `Node ID` 有 `Next`，允許**一次** `Current -> Next` fallback
3. `Now >= current_not_after`
   - 只能使用 `Next`

再補三條硬規則：

1. 同一個 `Node ID` 在同一輪撥號中最多只允許一次 `Current -> Next` fallback
2. fallback 後若仍失敗，不允許在 `Current / Next` 之間往返抖動
3. 只有在 overlap window 內才允許這種特殊 fallback，窗口外不允許

這樣才符合 `NK` 真實的失敗模式。

---

## 1.3 `Capability Mismatch` 的處置仍然有三套互相衝突的語義，現在已經不是邊角問題，而是直接會打壞互通

### 攻擊
這一輪 `protocol.md` 引入了 `Offered Caps / Required Caps`，這本身是正確方向。
但 mismatch 的處置仍然沒有收口，現在同時存在三套說法：

1. 握手請求段落說：server 若不滿足 `Required Caps`，握手可完成，之後 server
   必須發 `GOAWAY 0x02`
2. 握手回應段落說：client 比對後若 mismatch，client 必須 `Abort Transport`
3. `Recovery Matrix` 又寫成：`靜默或 GOAWAY (0x02)`

這三條不可能同時為真。

### 為什麼目前還不夠
`Capability Mismatch` 不是模糊錯誤，它是 deterministic negotiation failure。
既然 deterministic，就必須指定唯一的行動主體。

現在的寫法會造成三種不同實作：

1. server 主動 `GOAWAY`
2. client 本地 abort
3. 有的實作靜默，有的發信號

這直接等於協議分叉。

### 具體修改方案
這一項我仍然建議你們直接收斂到：

**client-local abort only**

也就是：

1. server 回 `Selected Caps`，不再額外為 mismatch 發 `GOAWAY`
2. client 執行：
   `if (RequiredCaps & ~SelectedCaps) != 0 -> Abort Transport`
3. `Recovery Matrix` 只保留一種語義：
   - Detected By: Client
   - Local Action: `Abort Transport`
   - Peer-Visible Signal: None
   - Retry: `Do Not Retry Same Capability Set`

如果你們堅持 server 要發 `GOAWAY`，那也可以，但就必須：

1. 刪掉 client 自主 abort 的規範語句
2. 把 ordering 寫死
3. 刪掉 recovery matrix 裡的「靜默或」

現在這種三套語義共存的狀態不能再接受。

---

## 1.4 `Recovery Matrix` 仍然不是 actor-aware，幾個最關鍵條目依然把「本地驗證失敗」和「對端可見錯誤信號」混在一起

### 攻擊
上一輪我批評的是矩陣把 client 本地錯誤寫成 server `GOAWAY`。
這一輪雖然矩陣更像表了，但根本問題還沒解決，因為它仍然只有：

1. `Immediate Action`
2. `Retry Policy`
3. `Config Refresh Required?`

它沒有顯式標出：

1. 誰檢測到錯誤
2. 該錯誤是否允許發對端可見信號
3. local abort 和 peer-visible signal 是否是同一回事

目前最典型的錯例仍然是：

1. `Epoch Cert Signature Invalid`
2. `Responder Key Mismatch`
3. `Epoch Cert Not Yet Valid`
4. `Epoch Cert Expired`
5. `Capability Mismatch`

這些都應該首先是 client-local 的驗證與處置，但表格仍然沒有把它們和 server-side
錯誤分開。

此外，`Unsupported Auth Scheme` 的內部矛盾也還在：

1. `CLIENT_AUTH` 章節寫的是 `GOAWAY 0x02 + 硬關閉`
2. `Recovery Matrix` 寫的是 `GOAWAY 0x02 -> Drain`

這一條現在仍然是直接矛盾。

### 為什麼目前還不夠
在 hostile-network 下，錯誤矩陣不是附錄，它本身就是協議的一部分。
如果不顯式區分 `Detected By / Local Action / Peer-Visible Signal`，那 reference
implementation 和 production implementation 很容易分叉。

### 具體修改方案
下一輪不要再修補現有四欄表，直接改表頭：

1. `Error Scenario`
2. `Detected By`
3. `Local Action`
4. `Peer-Visible Signal`
5. `Retry / Refresh Policy`

然後至少重寫下面幾條：

1. `Epoch Cert Signature Invalid`
   - Detected By: Client
   - Local Action: discard keys + abort
   - Peer-Visible Signal: None
   - Retry: refresh config first
2. `Responder Key Mismatch`
   - Detected By: Client
   - Local Action: abort
   - Peer-Visible Signal: None
   - Retry: hard security failure
3. `Capability Mismatch`
   - Detected By: Client
   - Local Action: abort
   - Peer-Visible Signal: None
   - Retry: do not retry same caps
4. `Unsupported Auth Scheme`
   - Detected By: Server
   - Local Action: hard close
   - Peer-Visible Signal: `GOAWAY 0x02`
   - Retry: upgrade client

在這一塊真正 actor-aware 之前，後面的所有恢復語義都還不夠穩。

---

## 1.5 `Epoch Cert` 仍然沒有對齊到新的簽名物件模型

### 攻擊
`control_plane.md` 這一輪已經補上了 canonical body 和 context-separated signature。
但 `protocol.md` 裡的 `Epoch Cert` 還是沿用：

`簽前 56 Bytes`

這等於把三種簽名物件拆成了兩種簽名哲學：

1. 控制面物件：canonical body + context string
2. `Epoch Cert`：欄位位移導向的隱含 message

這會破壞整個信任鏈的一致性。

### 為什麼目前還不夠
現在的問題不是 `Epoch Cert` 不能驗，而是它仍然沒有進入你們新建立的
「被簽物件模型」。只要它不對齊，這個模型就不算真的統一。

### 具體修改方案
把 `Epoch Cert` 對齊到與控制面相同的模式：

```text
EpochCertBody =
    ServerStaticPubkey(32) ||
    Epoch_Window_Start(8) ||
    Epoch_Window_End(8) ||
    Signer_Key_ID(8)

EpochCertSigMessage =
    "chameleon-epoch-cert-v1" || EpochCertBody
```

並在 `protocol.md` 明確規定：

1. 整數全為大端序
2. `Ed25519 Control Plane Sig` 驗證的 message 必須是 `EpochCertSigMessage`
3. 未來如需擴欄位，只能透過版本化 body，不得再靠固定欄位範圍隱式延續

---

## 2. 高優先級但非阻塞問題

## 2.1 `CPSKCertBody` 的 `Signer_Key_ID` 命名仍然具有誤導性

### 攻擊
`control_plane.md` 現在把 `Signer_Key_ID` 定義成 `Public_Key` 的 hash 前 8 bytes。
這在功能上可以工作，但語義上仍然有問題。

在 `Epoch Cert` 與 `Node Manifest` 中，`Signer_Key_ID` 指的是「誰簽了這個物件」。
但在 `CPSKCertBody` 中，這個欄位實際上指的是「這把 CPSK 自己的 key id」。
也就是說，同一個欄位名在不同物件裡表示的不是同一件事。

### 具體修改方案
建議把 `CPSKCertBody` 裡的欄位改名為：

1. `Key_ID`
或
2. `Subject_Key_ID`

而把 `Signer_Key_ID` 保留給真正表示 signer 身份的物件。

這不是字眼潔癖，而是避免之後寫 verifier / trust store 時把 subject 和 signer 混掉。

---

## 2.2 `KEY_UPDATE` 仍然只有冷卻期，沒有 mandatory trigger

### 攻擊
現在的 `KEY_UPDATE` 能描述「怎麼切」，但仍然沒有回答「什麼時候必須切」。

### 具體修改方案
至少先把以下三個常數與語義釘住：

1. `MAX_RECORDS_PER_GENERATION`
2. `MAX_BYTES_PER_GENERATION`
3. `MAX_KEY_AGE`

並規定：

1. 任一門檻逼近時 sender 必須發 `KEY_UPDATE`
2. 冷卻期不適用於 mandatory rekey
3. 對端長期不 rekey 時，本端可以主動發起輪換

---

## 2.3 Provisional Session 仍然建議補 `Max Records Before Auth`

### 攻擊
現在 provisional bound 已經有 timeout 和 bytes，但還缺一個很便宜、卻很有效的上限：

`Max Records Before Auth`

### 具體修改方案
直接加一條：

1. `Max Records Before Auth = 2` 或 `4`
2. 超出即 `Abort Transport`

這樣 provisional session 的資源邊界才算真正閉環。

---

## 3. 建議下一輪直接落地的修改順序

如果目標是加快收斂，下一輪建議按這個順序改：

1. 先決定 `Node Manifest` 是否直接承載 endpoint / route metadata，並把它們簽進 canonical body
2. 修正 `Deterministic Key Rollover` 的 failure trigger，讓它符合 `NK` 真實失敗模式
3. 決定 `Capability Mismatch` 的唯一行動主體，刪掉其餘衝突語義
4. 把 `Recovery Matrix` 改成 actor-aware 結構
5. 讓 `Epoch Cert` 對齊到新的簽名物件模型
6. 修正 `CPSKCertBody` 的 key id 命名
7. 為 `KEY_UPDATE` 補 mandatory trigger
8. 為 provisional session 補 `Max Records Before Auth`

---

## 4. 本輪結論

這一輪最大的正面變化是：

1. 三份規範文檔的分工已經基本站住
2. 控制面終於從「概念」走向「被簽物件」
3. 有些之前的老問題確實被修掉了

但這一輪最大的阻塞點也因此變得更具體：

現在不再是「文件不夠」，而是「規範開始成形後，真正影響互通和運營的字段還沒有全部簽進去，
真正影響恢復語義的主體還沒有全部分開」。

當前最該先釘死的四件事是：

1. `Node Manifest` 的 canonical body 是否包含 endpoint / route metadata
2. rollover 在 `NK` 下的真實失敗觸發條件
3. `Capability Mismatch` 的唯一處置模型
4. actor-aware 的 `Recovery Matrix`

這四件事補齊之後，下一輪 review 才值得真正下沉到參數選型、pool policy 和性能邊界。
