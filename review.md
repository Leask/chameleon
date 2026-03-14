# Chameleon 聯合審查報告
*(按專案原始目標： hostile-network-first、對抗高壓干預、公網存活性優先)*

## 0. 審查邊界

這一輪 review 回到專案最初的目標來看：

- 系統不是普通內網隧道，而是面向敵意公網環境的覆蓋網路
- 目標不是「看起來漂亮」，而是在高干預、高錯誤率、高觀測壓力下
  仍然能活、能用、能維持合理效率
- 我這次重點攻擊的是協議可行性、安全邊界、效率、可靠性與可互通性

以下內容本輪不展開具體工程細節：

- 任何直接面向可觀測性工程、流量特徵工程或規避效果優化的細節

這不影響對當前設計做出嚴厲判斷：如果核心協議本身不自洽，
後面的對抗性優化全部都只是建立在沙上。

## 1. 阻塞級問題

### 1. `Noise NK` 與「短期 responder static key」的分發模型仍然互相衝突

這是目前最嚴重的結構性問題。

`design.md` 現在的主張是：

- 邊緣節點啟動時生成短期 `Server Static Key`
- 控制面對這個短期公鑰簽發 `Epoch Cert`
- 客戶端驗證 `Epoch Cert` 後信任該短期公鑰

[design.md:34](/Users/leask/Documents/chameleon/design.md:34)

但 `protocol.md` 又明確選擇了 `Noise NK`：

- Client 首包使用 `-> e, es`
- 這意味著 initiator 在送出第一包前，**必須已經知道 responder static
  public key**

[protocol.md:38](/Users/leask/Documents/chameleon/protocol.md:38)

同時你們又把 `Epoch Cert` 放在 server 回包裡：

[protocol.md:63](/Users/leask/Documents/chameleon/protocol.md:63)

這裡的矛盾非常直接：

- 如果 client 在發第一包前不知道短期 responder static pubkey，
  `NK` 根本無法成立
- 如果 client 其實已經透過控制面提前拿到了短期 responder static pubkey，
  那就必須把這件事寫成正式前提，而不是讓 `Epoch Cert` 看起來像是在握手
  內部完成這個信任建立

目前文檔把這兩條路混在一起了，導致握手模型在邏輯上是裂開的。

修正建議：

- 如果堅持使用 `NK`，就必須把「短期 responder static pubkey 由控制面或
  配置通道預先分發給 client」寫成硬前提
- `Epoch Cert` 的職責應收斂為：
  驗證該預分發公鑰仍在有效紀元內、未被撤銷、且屬於正確控制面信任鏈
- 如果你們不想要求 client 預先持有短期 responder static pubkey，
  那就不能繼續假裝 `NK` 仍然是正確的選擇，必須換成與金鑰分發模型一致的
  握手模式

這個問題不解決，協議連第一步都站不住。

### 2. 設計哲學對密碼學能力的表述過度承諾，會直接誤導後續決策

`design.md` 第 0 章現在寫道：

> 透過 AEAD 和帶外信任分發（OOB Trust），確保協議在數學層面上不可被探測
> 和重放。

[design.md:10](/Users/leask/Documents/chameleon/design.md:10)

這句話在當前版本裡是不成立的，而且不是措辭問題，是安全邊界定義錯了。

原因很簡單：

- 你們自己已經明確接受 pre-auth handshake replay
  不受協議層密碼學阻止
  [design.md:43](/Users/leask/Documents/chameleon/design.md:43)
- AEAD 保證的是機密性與完整性，不保證不可觀測性，更不保證可用性
- 「不可被探測」不是一個密碼學定理，而是系統、部署、承載、時序、
  錯誤處理與運營策略共同作用的結果

如果這種過度承諾繼續留在設計哲學裡，後面每一輪討論都會被帶歪。

修正建議：

- 把安全目標拆開寫死，不要再用一個籠統句子包掉全部問題
- 至少分成五類：
  - Confidentiality
  - Responder Authentication
  - Post-auth Integrity
  - Pre-auth Silence
  - Operational Survivability under Hostile Networks
- 明確聲明：
  pre-auth replay tolerated but not cryptographically prevented

只有把邊界寫準，後續設計才不會陷入「以為已經解決」的假象。

### 3. `CLIENT_AUTH` 仍然不是一個可實作的授權協議，只是一個占位符

`protocol.md` 的 overview 說握手層負責身份認證：

[protocol.md:13](/Users/leask/Documents/chameleon/protocol.md:13)

但實際規格又把 client auth 放在握手後第一個 frame：

[design.md:39](/Users/leask/Documents/chameleon/design.md:39)
[protocol.md:115](/Users/leask/Documents/chameleon/protocol.md:115)

先不談這個位置是否合理，單看 wire spec 本身，`CLIENT_AUTH` 目前也還不能
直接實作：

```text
Device ID (16) | Auth Token / Signature (32)
```

[protocol.md:117](/Users/leask/Documents/chameleon/protocol.md:117)

這裡至少缺了以下東西：

- 認證方案 ID
- 憑證或金鑰 ID
- proof 的實際型別
- proof 長度
- 過期時間或版本
- 是否綁定握手 transcript
- 是否綁定 session
- 撤銷與輪換後如何驗證

更直接地說，`32 bytes` 的 `Auth Token / Signature`
不是一個可泛化的授權格式。這不是細節沒補齊，而是協議沒有真正定義授權。

修正建議：

- 把 `CLIENT_AUTH` 重寫為明確的 TLV 或長度前綴結構
- 至少包含：
  - `Auth Scheme ID`
  - `Credential ID`
  - `Proof Length`
  - `Proof`
  - 可選 metadata
- `Proof` 必須綁定本次握手的 transcript hash 或導出的 session binder，
  否則認證材料很容易被跨連線重放
- 在 `design.md` 明確授權模型：
  是設備密鑰、短期 token、控制面簽發票據，還是多者並存

如果 auth frame 只是個空殼，那整個「握手後授權」就只是把真正的難題
往後推了一步。

### 4. 後置授權意味著 pre-auth 資源消耗不可避免，但目前沒有被協議化

`design.md` 現在接受以下取捨：

- Noise 握手先建立匿名加密隧道
- 真正授權發生在第一個加密 frame

[design.md:41](/Users/leask/Documents/chameleon/design.md:41)

這個思路不是不能做，但它的代價是：

- server 必須先做一次 DH
- 必須先導出 traffic keys / HP keys
- 至少要建立一個 provisional session 上下文

也就是說，任何能走完握手的人，都能迫使 server 進入「半初始化」狀態。

`design.md` 目前只用一句話說可以靠 IP rate limit 緩解：

[design.md:45](/Users/leask/Documents/chameleon/design.md:45)

這遠遠不夠。對抗型系統不能把 admission control 當成營運備註。

修正建議：

- 把 session state machine 正式寫出來：
  `PreHandshake -> Provisional -> Authorized -> Draining -> Closed`
- 對 `Provisional` 狀態增加硬限制：
  - 最大存活時間
  - 最大未認證 inbound bytes
  - 最大 outbound bytes
  - 未在期限內收到 `CLIENT_AUTH` 即關閉
- 在授權通過前，禁止：
  - `OPEN_CHANNEL`
  - `DATA`
  - 任何需要分配大塊 state 的動作
- server 應延遲大部分 channel/session 資源配置，直到 auth 通過

你們既然主動接受「先建匿名隧道再授權」，就必須把資源邊界寫得像防火牆一樣
精準，而不是只靠一句 rate limit。

### 5. Record Layer 仍然沒有達到「兩個團隊可獨立實作並互通」的精度

`protocol.md` 在 record 層已經比前幾輪進步很多：

- `Length` 含義已明確
- `Key Phase` 已定義
- sample 太短時補零已定義
- rekey 也開始同時輪換 traffic key 和 HP key

[protocol.md:97](/Users/leask/Documents/chameleon/protocol.md:97)
[protocol.md:192](/Users/leask/Documents/chameleon/protocol.md:192)

但還差最後幾個會直接造成互通分裂的核心點：

- `Mask = ChaCha20(HP_Key, Sample)[0..2]`
  不是一個精確可實作的演算法描述
  [protocol.md:108](/Users/leask/Documents/chameleon/protocol.md:108)
- `Sample` 是 16 bytes，但 ChaCha20 的 nonce/input 介面不是這樣描述的
- 初始 traffic key / HP key 的導出流程沒有明確寫在規格裡
- record nonce 的構造只寫了抽象式，沒有完整定義 IV 來源

這意味著：

- rekey 看起來有規格
- 但初始 key schedule 其實還是空的

修正建議：

- 新增一節 `Initial Key Schedule`
- 明確寫出：
  - handshake chaining key / handshake hash 的最終輸出如何映射到
    `C2S_Traffic_Key`、`S2C_Traffic_Key`
  - `C2S_HP_Key`、`S2C_HP_Key` 如何導出
  - `ChaCha_IV` 的來源與長度
  - `Nonce` 的精確位元組構造
  - header protection 的確切算法，不要只寫概念
- 最好的做法不是「自己描述一個像某標準的東西」，
  而是直接採納某個已知構造並逐項映射

如果這塊再不收口，後面所有效率討論都沒有意義，因為互通都不一定成立。

### 6. 錯誤語義仍然綁死在特定承載上，與「language-agnostic / transport-agnostic」的宣稱衝突

`design.md` 一方面說 Chameleon 是語言與平台解耦的 wire protocol：

[design.md:14](/Users/leask/Documents/chameleon/design.md:14)

`protocol.md` 也說底層可以是：

- QUIC Stream
- TCP
- WebSocket

[protocol.md:11](/Users/leask/Documents/chameleon/protocol.md:11)

但錯誤矩陣裡又直接寫：

- `TCP RST`
- `Fallback`

[protocol.md:225](/Users/leask/Documents/chameleon/protocol.md:225)
[protocol.md:228](/Users/leask/Documents/chameleon/protocol.md:228)

這會直接產生兩個問題：

- 協議核心規格混入了 transport-specific 動作
- 不同承載的可觀測行為會失控

同一個「Hard Close」在 TCP、QUIC Stream、WebSocket
上根本不是一回事。`Fallback` 更是只在少數承載可行。

修正建議：

- 在核心協議裡只保留抽象動作：
  - `DropSilently`
  - `AbortTransport`
  - `SendEncryptedGoawayThenDrain`
- 針對不同承載另寫 binding appendix：
  - TCP binding
  - QUIC Stream binding
  - WebSocket binding
- `design.md` 應停止把 transport-specific 行為說成協議保證

不然「核心協議」和「部署策略」還是沒有真正分開。

### 7. `PAD`、shaping budget 與 flow control 仍然是三套邏輯，效率目標會被自己打穿

目前規格寫的是：

- `DATA` 消耗 channel credit + session credit
- `PAD` 不消耗任何 credit
- `PAD` 只受獨立 shaping / padding budget 約束

[protocol.md:169](/Users/leask/Documents/chameleon/protocol.md:169)
[protocol.md:215](/Users/leask/Documents/chameleon/protocol.md:215)

這種切法最大的問題不是安全，而是效率與調度邏輯會變得不誠實：

- transport egress 可能已被 `PAD` 佔滿，但 session credit 看起來仍很健康
- 實作端可能在「協議層健康」的同時，實際 goodput 已經被 padding 吃掉
- `PAD` 與控制幀共用「不消耗 credit」的規則，也會讓調度器很難表述優先級

目前這個模型還不足以支持「高效率」的主張。

修正建議：

- 明確定義 send scheduler 的優先級：
  `Auth / Critical Control > DATA > PAD`
- 明確規定：
  `PAD must never starve authenticated control or data frames`
- shaping budget 不一定要併入 channel credit，
  但至少必須併入一個 session 級傳輸預算或 egress budget
- 對 `PAD` 增加每個時間窗的硬上限，而不是只說「由策略決定」

不然 padding 會變成一個在設計上合法、在效率上失控的黑洞。

### 8. 單一可靠位元組流的問題不只是 HOL，而是整個控制面響應都與 bulk data 共享故障域

`design.md` 已正式接受單一 reliable byte stream：

[design.md:21](/Users/leask/Documents/chameleon/design.md:21)

這不只是「某個 channel 丟包會卡住其他 channel」這麼簡單。
更深層的代價是：

- `OPEN_ACK`
- `KEY_UPDATE`
- `GOAWAY`
- `RESET_CHANNEL`

這些控制信令雖然可以在本地排程時提高優先級，但一旦它們與大量資料共享同一
條可靠位元組流，它們仍然共享同一個 HOL / congestion / buffering 故障域。

也就是說，你們現在不是只接受「資料互相拖累」，
而是接受「控制恢復能力也被 bulk traffic 拖累」。

如果這是有意識的取捨，就必須升級成正式產品限制，而不是只放在反駁段。

修正建議：

- 把單一流模型的限制寫成 deployment guidance，而不是設計備註
- 明確建議至少拆分兩類 session：
  - latency-sensitive interactive / control
  - bulk transfer
- 不要再用「不破壞底層能力」這種語言包裝它
  [design.md:13](/Users/leask/Documents/chameleon/design.md:13)
  因為在應用層你們確實重新引入了自己的故障耦合

這不是否定單流，而是要求你們誠實定義它的代價。

### 9. `GOAWAY` 的 wire format 已經接近可用，但恢復語義仍然不完整

`GOAWAY` 現在已經有：

- `Error Code`
- `Last Accepted Channel ID`

[protocol.md:200](/Users/leask/Documents/chameleon/protocol.md:200)

這是正向進展，但還差最後一段最重要的規格：

- client 收到不同 `Error Code` 後應不應立即重連
- 是重連同一節點、切換新節點，還是等待控制面刷新
- `Draining` 階段已發出的 channel 如何判定成功或失敗

沒有這些語義，`GOAWAY` 仍然只是半個框架。

修正建議：

- 補一張 recovery matrix
- 每個 `GOAWAY Code` 至少定義：
  - retry same node
  - retry different node
  - require config refresh
  - never retry automatically

這是穩定實作與客戶端行為一致性的必要條件，不是運營細節。

## 2. 次高優先級問題

### 10. `Epoch_Window_Start / End` 已經不是時鐘同步工具，但仍缺少時鐘漂移容忍規則

`design.md` 已經把 `Epoch Cert` 收斂為 freshness window，而不是 clock sync：

[design.md:25](/Users/leask/Documents/chameleon/design.md:25)

這是對的。但 `protocol.md` 還缺少操作性規則：

- client 允許多少 clock skew
- 視窗剛好擦邊時如何處理
- 若本地時間明顯錯誤，是立即失敗，還是先進入受限模式

如果不補，最終會演變成各實作各自處理。

修正建議：

- 定義固定的 clock skew tolerance
- 對「證書尚未生效」與「證書已過期」分別定義處置
- 這塊至少要保證跨實作一致

### 11. Version / Capability negotiation 還只是雛形，未來很容易形成降級與互通問題

目前握手裡有：

- `Version`
- `Caps`
- `Selected Caps`

[protocol.md:42](/Users/leask/Documents/chameleon/protocol.md:42)
[protocol.md:73](/Users/leask/Documents/chameleon/protocol.md:73)

但還沒明確寫出：

- server 是否只能回覆 client 宣告能力的子集
- 未知 capability bit 如何處理
- 版本不符時是否有兼容窗口

現在因為 capability 幾乎全保留，問題還沒爆。等你們真的把 datagram、
新的 auth scheme 或不同 padding policy 放進來，這塊會立刻變成兼容性地雷。

修正建議：

- 現在就把 negotiation rule 寫死
- 不要等 capability 真開始擴張時再補

## 3. 我認為下一版必須先完成的 6 件事

1. 解決 `NK` 與短期 responder static key 的根本矛盾。
2. 把 `CLIENT_AUTH` 從占位欄位升級為真正可驗證的授權協議。
3. 補齊 `Initial Key Schedule` 與 header protection 的精確算法。
4. 寫出 `Provisional Session` 的硬資源邊界與超時規則。
5. 將 transport binding 從核心協議中拆出。
6. 為 `GOAWAY`、時鐘視窗與 capability negotiation 補齊恢復與兼容規則。

## 4. 總結

這一輪最大的問題已經不是「還缺一些 frame」，
而是有三個核心矛盾仍未真正收斂：

- 握手模式與金鑰分發模型衝突
- 授權模型只有位置，沒有真正內容
- 高效率主張與實際調度 / 流控邊界不匹配

正面的部分也很明確：

- 你們已經開始把 `design.md` 與 `protocol.md` 分工拉正
- 已經接受單一流是有代價的取捨
- 已經把 `GOAWAY`、flow control、rekey 往「能實作」的方向推進

但如果下一輪還不先把 `NK / Auth / Initial Key Schedule`
這三塊釘死，設計就會再次陷入「表面上更完整，實際上仍不可實作」的循環。
