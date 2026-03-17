# Chameleon Review
## 對 `design.md` v10.0、`protocol.md` v10.0 與 `control_plane.md` v1.0 的新一輪聯合審查

這一輪和上一輪相比，最大的正面變化不是單個欄位，而是規範集合終於開始成形：
`control_plane.md` 已經出現，`protocol.md` 也補上了 `Offered/Required Caps`，
概述與 reference 也做了同步。這些都是實質進展。

但這一輪新的主問題也更明確了：

1. 控制面物件現在看起來像規格，但其實還只是「帶註解的 JSON 範例」。
2. `Caps` 雖然被拆成 `Offered/Required`，但協議對 mismatch 的處置仍然互相打架。
3. `Recovery Matrix` 比上一輪更細，但仍然沒有把「誰檢測、誰能發信號、誰只能本地中止」拆開。
4. `design.md` 和規格集的職責分工還沒有完全同步。

下面只攻擊本輪仍然阻塞收斂的點，不再重複已經修掉的舊題。

---

## 0. 本輪已可正式降級的舊問題

### 0.1 `control_plane.md` 已經出現
上一輪最大的阻塞點之一是 OOB 信任物件根本沒有獨立規格。這一輪這件事已經被補上，
不能再當作「缺文件」來批評。

### 0.2 `protocol.md` 的概述與密碼學 reference 已同步修正
握手層不再被泛化成「整體身份認證層」，Noise / X25519 的 reference 也已經拆開。
這兩個問題可以降級。

### 0.3 `Caps` 至少已經有了 `Offered/Required` 的骨架
這代表你們終於承認「偏好能力」和「必要能力」不能混在一起。雖然還沒完全收口，
但方向已經對了。

### 0.4 `Recovery Matrix` 的一個硬錯碼已被修正
`Session Flow Control Overflow` 已回到 `GOAWAY 0x03`，這個錯碼問題可以正式關閉。

---

## 1. 阻塞級問題

## 1.1 控制面簽名物件仍然不是「規格」，因為 canonical body、簽名上下文與欄位語義都還沒釘死

### 攻擊
`control_plane.md` 現在提供的是 JSON 格式範例，但還不是可互通的簽名物件規格。
原因有三個：

1. 它是範例，不是 canonical serialization。
2. `root_signature` / `manifest_signature` 的簽名輸入只寫成「對上方欄位簽名」，
   但沒有定義字段順序、字串編碼、數字編碼、空欄位處理。
3. `CPSK Certificate` 裡的 `signer_key_id` 實際上是 CPSK 自己的 key id，
   不是 signer 的 key id；這個欄位命名本身就會誤導實作。

更直接一點說：只要簽名物件還建立在「一個帶註解 JSON 例子」上，不同團隊就一定會
序列化出不同位元組，最後誰也驗不過誰。

### 為什麼目前還不夠
現在三種被簽物件都存在同一個問題：

1. `CPSK Certificate`
2. `Node Manifest`
3. `Epoch Cert`

其中前兩個在 `control_plane.md`，第三個在 `protocol.md`。這意味著問題不是單個文件
寫得不夠嚴，而是整個「簽名物件模型」還沒有統一。

另外，`control_plane.md` 在 `Signer Key ID` 的生成上仍然使用「建議為」這種語氣。
但 `protocol.md` 的驗證流程需要的是確定規則，不是建議。

### 具體修改方案
下一輪應把「範例 JSON」升級成正式簽名物件模型，至少做四件事：

1. 為每個被簽物件定義 **canonical body**，不要用「上方欄位」這種描述。
2. 為每個簽名定義 **domain-separated context string**。
3. 把 `Key ID` 生成方式從「建議」改成 `MUST`。
4. 修正 `CPSK Certificate` 的欄位命名，不要再用 `signer_key_id` 指代 subject key。

建議直接寫成：

```text
CPSKCertBody = ...
CPSKCertSigMessage = "chameleon-cpsk-cert-v1" || Canonical(CPSKCertBody)

NodeManifestBody = ...
NodeManifestSigMessage = "chameleon-node-manifest-v1" || Canonical(NodeManifestBody)

EpochCertBody = ...
EpochCertSigMessage = "chameleon-epoch-cert-v1" || EpochCertBody
```

如果你們堅持用 JSON 作為 OOB 交換格式，那就必須明確指定 canonical JSON 規則；
否則就不要把 JSON 視為簽名輸入，而應把它只當作展示格式。

這一項不解決，`control_plane.md` 的出現只會讓大家誤以為信任鏈已經規範化，
實際上仍然沒有。

---

## 1.2 `Node Manifest` 的 key rollover 規則仍然不夠確定，客戶端選鍵會出現實作分叉

### 攻擊
`control_plane.md` 現在定義了 `current_static_pubkey`、`next_static_pubkey` 與 overlap window，
這比沒有規則好得多。但目前的核心問題是：

`在 overlap window 內，客戶端允許使用兩把金鑰中的任意一把發起新連線。`

這句話太寬了。它等於把 key selection policy 丟給實作者自由發揮。
結果會是：

1. 有的實作永遠優先 current
2. 有的實作一進 overlap 就切 next
3. 有的實作失敗後在 current/next 之間來回抖動

這不是小問題。對 `NK` 來說，client 發首包前到底選哪把 responder static key，
會直接決定連線是否成功。

### 為什麼目前還不夠
你們已經明確要求：

1. 同一時刻只能選一把 key 作為預期公鑰
2. 如果 server 回來的 key 不屬於 manifest 中的 current/next，client 必須 fail

這說明 key selection 已經不是「實作細節」，而是連線正確性的組成部分。
既然如此，就不能只說「任意一把都可以」。

### 具體修改方案
把 rollover policy 收斂成一個確定算法，例如：

1. `Now < next_not_before`：只能使用 `current_static_pubkey`
2. `next_not_before <= Now < current_not_after`：
   - 默認仍使用 `current_static_pubkey`
   - 只有當本輪連線因 `Responder Key Mismatch` 失敗，才允許對同一 `Node ID`
     用 `next_static_pubkey` 再試一次
3. `Now >= current_not_after`：只能使用 `next_static_pubkey`

再補兩條硬規則：

1. 同一個 `Node ID` 在同一輪撥號中，最多只允許一次 current -> next fallback
2. 不允許無限在 current / next 之間來回切換

這樣才能把 rollover 從「概念可行」變成「不同實作會做同一件事」。

---

## 1.3 `Required Caps` 雖然進了協議，但 mismatch 的處置現在是三套互相衝突的語義

### 攻擊
這一輪 `Caps` 的骨架進步很大，但 mismatch 的處理現在同時存在三種說法：

1. 握手請求段落說：若 server 無法滿足 `Required Caps`，握手可完成，之後 server
   必須發 `GOAWAY 0x02`
2. 握手回應段落說：client 比對後若 mismatch，client 必須 `Abort Transport`
3. `Recovery Matrix` 又寫成：`靜默或 GOAWAY (0x02)`

這是明確的規格衝突，不是不同描述角度。

### 為什麼目前還不夠
`Capability Mismatch` 本質上是握手後立即可由雙方推導出的 negotiation failure。
你們現在最大的問題不是少了某個錯誤碼，而是沒有決定「誰是這個錯誤的唯一行動主體」。

如果 server 發 `GOAWAY`，client 也本地 abort，那就會出現 race。
如果 `Recovery Matrix` 允許「靜默或 GOAWAY」，那實作者就會各寫各的。

### 具體修改方案
這一項必須選一個唯一模型。我建議直接選：

**client-local abort only**

也就是：

1. server 回 `Selected Caps`，僅作能力回報，不再額外因 mismatch 發 `GOAWAY`
2. client 收到後執行：
   `if (RequiredCaps & ~SelectedCaps) != 0 -> Abort Transport`
3. `Recovery Matrix` 把 `Capability Mismatch` 寫成：
   - Detected By: Client
   - Local Action: `Abort Transport`
   - Peer-Visible Signal: None
   - Retry: `Do Not Retry Same Capability Set`

這個模型最乾淨，也最符合你們一貫強調的「少暴露、不引入多餘信號」。

如果你們堅持要 server 發 `GOAWAY`，那也行，但就必須把 client 的自主 abort 刪掉，
而且把 ordering 寫死。現在這種三套語義併存的狀態不能接受。

---

## 1.4 `Recovery Matrix` 仍然沒有真正拆開「偵測方」與「可見動作」，現在只是比上一輪更像表，但仍然不夠可實作

### 攻擊
上一輪我攻擊的是：矩陣把 client 本地驗證失敗寫成 server 發 `GOAWAY`。
這一輪雖然矩陣更細了，但根本問題還在，因為欄位設計仍然沒有把「誰在做事」
說清楚。

目前幾個最明顯的條目仍然有問題：

1. `Epoch Cert Signature Invalid`
2. `Responder Key Mismatch`
3. `Epoch Cert Not Yet Valid`
4. `Epoch Cert Expired`
5. `Capability Mismatch`

這些都應該是 client-local 的判斷與處置，但表格仍然用單一的「立即動作」欄位把
本地行為和對端可見行為混在一起。

此外，`Unsupported Auth Scheme` 仍然有內部矛盾：

1. `CLIENT_AUTH` 章節寫的是 `GOAWAY 0x02 + 硬關閉`
2. `Recovery Matrix` 寫的是 `GOAWAY 0x02 -> Drain`

這是明確矛盾，還沒有修掉。

### 為什麼目前還不夠
只要矩陣還是這種四欄模型：

1. `Error Scenario`
2. `Immediate Action`
3. `Retry Policy`
4. `Config Refresh?`

它就無法精確描述 protocol 在 hostile-network 場景下最核心的東西：

1. 哪些錯誤只能本地 drop
2. 哪些錯誤可以對外發控制幀
3. 哪些錯誤需要 refresh config
4. 哪些錯誤應該切 route、切 pool、或根本停止重試

### 具體修改方案
不要再補現有四欄表，直接改表頭：

1. `Error Scenario`
2. `Detected By`
3. `Local Action`
4. `Peer-Visible Signal`
5. `Retry / Refresh Policy`

然後至少把下面幾條重寫清楚：

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

只要這一塊不拆開，後面任何 reference implementation 都會各自補洞。

---

## 1.5 `Epoch Cert` 的簽章上下文仍然沒有補齊

### 攻擊
雖然 `control_plane.md` 已經出現，但 `protocol.md` 的 `Epoch Cert` 仍然沿用：

`簽前 56 Bytes`

這仍然不夠。它還是把簽章語義綁在當前欄位位移上，而不是綁在版本化的物件類型上。

### 為什麼目前還不夠
現在你們其實同時有三種簽名物件：

1. `CPSK Certificate`
2. `Node Manifest`
3. `Epoch Cert`

既然前兩者都已經進入控制面規格，那第三者就不應該還停留在「對固定欄位範圍簽章」
這種過渡模式。

### 具體修改方案
把 `Epoch Cert` 也對齊到同一個模型：

```text
EpochCertBody =
    ServerStaticPubkey(32) ||
    Epoch_Window_Start(8) ||
    Epoch_Window_End(8) ||
    Signer_Key_ID(8)

EpochCertSigMessage =
    "chameleon-epoch-cert-v1" || EpochCertBody
```

然後在 `protocol.md` 裡明確規定：

1. 整數全為大端序
2. `Ed25519 Control Plane Sig` 驗證的 message 就是 `EpochCertSigMessage`
3. 未來擴欄位要靠版本升級，不靠沿用舊位移

---

## 2. 高優先級但非阻塞問題

## 2.1 `design.md` 的職責分工仍然停留在兩份文檔模型，沒有把 `control_plane.md` 納入總體規範結構

### 攻擊
`design.md` 現在還寫：

1. `design.md`
2. `protocol.md`

這是文檔職責分離的全部。

但實際上你們現在已經有三份規範：

1. `design.md`
2. `protocol.md`
3. `control_plane.md`

也就是說，規範集合本身已經變了，但 `design.md` 對外宣示的職責分工還沒同步。

### 具體修改方案
把 `0.2` 的職責分離改成三份文件：

1. `design.md`: 設計哲學、威脅模型、安全邊界、取捨
2. `protocol.md`: 資料面 wire format、狀態機、錯誤語義
3. `control_plane.md`: OOB 信任物件、簽發與更新生命週期

這是文檔治理問題，但如果不改，後續審查會一直在「哪份文件應該寫什麼」上打轉。

---

## 2.2 `design.md` 的反駁段落仍然過度宣稱 `Session Pool`

### 攻擊
`design.md` 的正式架構段落已經正確承認：

`Session Pool` 只能隔離 HOL 故障域，不能消滅 HOL。

但反駁段落仍然保留：

`完美解決了 HOL 阻塞問題`

這會直接污染性能預期與 pool policy 討論。

### 具體修改方案
把反駁三統一改成：

1. `Session Pool` 的目標是把 HOL 影響限制在單一 session
2. control/interactive 與 bulk 必須分配到不同 session
3. 這是架構隔離，不是協議層消滅 HOL

只要正文和反駁還在打架，後續的性能討論就不會穩。

---

## 2.3 `KEY_UPDATE` 仍然只有冷卻期，沒有 mandatory trigger

### 攻擊
目前的 `KEY_UPDATE` 狀態機能描述「怎麼切」，但仍然沒有回答：

1. 什麼時候必須切
2. 什麼時候只是建議切

只有冷卻期，不是 rekey policy。

### 具體修改方案
至少先把三個常數釘住：

1. `MAX_RECORDS_PER_GENERATION`
2. `MAX_BYTES_PER_GENERATION`
3. `MAX_KEY_AGE`

然後規定：

1. 任一門檻逼近時 sender 必須發 `KEY_UPDATE`
2. 冷卻期不適用於 mandatory rekey
3. 對端長期不 rekey 時，本端可以主動發起輪換

---

## 2.4 Provisional Session 仍然建議補 `Max Records Before Auth`

### 攻擊
雖然 byte bound 已經修對，但還是只有：

1. timeout
2. byte upper bound

仍然缺少一個廉價但有效的上限：

`Max Records Before Auth`

### 具體修改方案
直接再加一條：

1. `Max Records Before Auth = 2` 或 `4`
2. 超出即 `Abort Transport`

這樣 provisional session 的資源邊界才算真正閉環。

---

## 2.5 `control_plane.md` 對 `GOAWAY 0x05` 的 refresh 語義仍然過粗

### 攻擊
`control_plane.md` 現在把：

`Auth Token 被拒絕 (GOAWAY 0x05)`

幾乎直接等價成：

`Config Refresh Required`

這太粗了。`0x05` 可能意味著：

1. credential 被撤銷
2. 本地 proof 損壞
3. 客戶端實作 bug
4. 使用者配置過期

這些場景不應全部走同一條 refresh path。

### 具體修改方案
如果你們暫時不想新增更細錯誤碼，至少也要在 `control_plane.md` 裡改成：

1. `GOAWAY 0x05` 觸發 refresh 是 `SHOULD`，不是無條件 `MUST`
2. client 應記錄本地連續失敗計數
3. 多次 refresh 後仍然 `0x05`，應升級為本地 credential / implementation 問題

不然控制面會被很多本地錯誤白白打爆。

---

## 3. 建議下一輪的直接修改順序

如果要加快迭代，下一輪建議按這個順序收口：

1. 先把三類簽名物件的 canonical body + context string 全部釘死
2. 把 `Node Manifest` 的 key rollover 選鍵規則寫成確定算法
3. 決定 `Capability Mismatch` 的唯一行動主體，刪掉其餘衝突語義
4. 重寫 `Recovery Matrix`，加入 `Detected By / Local Action / Peer-Visible Signal`
5. 在 `design.md` 補上三份規範文件的正式職責分工
6. 刪掉 `Session Pool` 的過度承諾
7. 為 `KEY_UPDATE` 補 mandatory trigger
8. 為 provisional session 補 `Max Records Before Auth`

---

## 4. 本輪結論

這一輪最大的正面變化是：

1. 控制面終於有了獨立規範
2. `Caps` 已經從概念走向了正式欄位
3. 有幾個之前的硬錯已經被修掉

但這一輪最大的阻塞點也更精確了：

不是「還少一份文件」，而是「文件開始變多之後，簽名物件模型、錯誤主體模型、
以及 negotiation 主體模型都還沒真正統一」。

當前最該先釘死的四件事是：

1. 三類簽名物件的 canonical form
2. `Node Manifest` 的 deterministic key rollover policy
3. `Capability Mismatch` 的唯一處置模型
4. `Recovery Matrix` 的 actor-aware 結構

這四件事補齊之後，下一輪 review 才值得真正往性能邊界與參數選型下沉。
