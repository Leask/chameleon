# Chameleon 聯合審查報告
*(新一輪：聚焦核心協議可行性、效率、可靠性與 hostile-network 邊界)*

## Findings

### 1. `Noise NK` 與短期 `Responder Static Key` 的信任分發模型仍然沒有真正閉環

`design.md` 現在的核心說法是：

- client 單向預知 server static pubkey
- edge 節點持有的是短期 `Server Static Key`
- 控制面以 `Epoch Cert` 對短期公鑰簽名

[design.md:23](/Users/leask/Documents/chameleon/design.md:23)
[design.md:34](/Users/leask/Documents/chameleon/design.md:34)

`protocol.md` 則明確採用 `Noise NK`：

- client 首包是 `-> e, es`
- 這要求 initiator 在送出第一包前就已知 responder static pubkey

[protocol.md:27](/Users/leask/Documents/chameleon/protocol.md:27)
[protocol.md:49](/Users/leask/Documents/chameleon/protocol.md:49)

但 `Epoch Cert` 仍然是在 server 回包內才帶回：

[protocol.md:74](/Users/leask/Documents/chameleon/protocol.md:74)

這裡的結構性矛盾仍然沒解：

- 如果 client 事前不知道短期 responder static pubkey，`NK` 根本無法成立
- 如果 client 事前已經知道，那 `Epoch Cert` 就不是用來「建立」這個信任，
  而只是用來「確認它仍有效」

現在文檔同時在說這兩件事，導致握手信任圖不閉環。

修正建議：

- 若保留 `NK`，就明確寫死：
  短期 responder static pubkey 必須由控制面或配置通道預先分發到 client
- 將 `Epoch Cert` 的語義收斂為：
  對預分發短期公鑰的有效期、控制面簽名與撤銷狀態做確認
- 若不願要求預分發短期 responder static pubkey，就不要再維持 `NK`
  這個前提

這一點不解決，其他討論都會建立在錯的握手模型上。

### 2. `CLIENT_AUTH` 仍然不是一個可互通的授權協議，且 design/protocol 對認證分層仍不同步

`design.md` 明確說：

- Noise 握手只建立匿名加密隧道
- client auth 發生在握手後第一個加密 frame

[design.md:39](/Users/leask/Documents/chameleon/design.md:39)

但 `protocol.md` overview 仍寫「握手層負責身份認證」：

[protocol.md:13](/Users/leask/Documents/chameleon/protocol.md:13)

wire spec 本身對 `CLIENT_AUTH` 的定義也仍然過於空泛：

```text
Type(1) | Device ID (16) | Auth Token / Signature (32)
```

[protocol.md:126](/Users/leask/Documents/chameleon/protocol.md:126)

這不是「細節待補」，而是授權協議尚未被定義。它至少缺：

- `Auth Scheme ID`
- `Credential ID`
- `Proof Length`
- proof 的實際格式
- 失效時間或版本
- 與本次握手或 session 的綁定方式

最嚴重的問題是：目前 `Auth Token / Signature (32)` 沒有任何 transcript
binding。這意味著授權材料很可能被跨連線重放。

修正建議：

- 先修正文檔分層：
  `Handshake` 不負責 client auth；它只負責 responder auth、key exchange
  與 capability selection
- 將 `CLIENT_AUTH` 改成明確的 TLV 或 length-prefixed 結構
- 強制 `Proof` 綁定握手 transcript hash 或 session binder
- 在 `design.md` 明確授權模型：
  設備密鑰、短期票據、控制面簽發 token，還是混合模式

如果 `CLIENT_AUTH` 只是個占位欄位，整個「後置授權」設計目前仍然不可實作。

### 3. 新增的 `Initial Key Schedule` 是進步，但目前寫法在密碼學與互通層面都還不夠嚴謹

你們這一輪補上了：

- `C2S_Traffic_Key`
- `C2S_HP_Key`
- `C2S_ChaCha_IV`
- `S2C_*`

[protocol.md:29](/Users/leask/Documents/chameleon/protocol.md:29)

方向是對的，但規格還是沒有收口，主要有四個問題：

- 文中說 `cs1` / `cs2` 作為 IKM 進行 `HKDF-Expand`
  但 `HKDF-Expand` 的輸入語義應是 PRK，不是 IKM
- 沒有定義 `cs1` / `cs2` 與 initiator/responder 傳輸方向的最終映射規則
- `ChaCha_IV` 的來源有了，但 `Nonce = ChaCha_IV XOR Pad_Zero(Sequence_Number)`
  仍未定義精確的補零方向、位元寬度與端序
- 沒有寫明這套 KDF 是直接建立在 Noise `Split()` 的輸出之上，
  還是對某個額外 secret 再做一次 extract-expand

這些問題會直接導致不同實作產生不同 key schedule。

修正建議：

- 用規範語言重寫整段 KDF
- 明確聲明：
  - `Noise Split()` 產出的是什麼
  - 哪個輸出對應 initiator->responder，哪個對應 responder->initiator
  - 是否需要額外的 `HKDF-Extract`
  - `Nonce` 的精確位元組構造
- 最好附一組固定測試向量，不然這塊仍然只能靠猜

### 4. Header protection 依然沒有精確到可直接實作

你們補了 header protection 小節，這是必要進展；但目前算法描述仍然不夠：

- `Mask = ChaCha20(HP_Key, Sample)[0..2]`
- `Sample` 為 ciphertext 前 16 bytes
- `Nonce` 與 `Counter` 取 0

[protocol.md:108](/Users/leask/Documents/chameleon/protocol.md:108)

這裡的問題很直接：

- ChaCha20 的介面不是這樣定義的
- `Sample` 在這個公式裡到底扮演 nonce、plaintext、還是 block input，
  文檔沒有說清楚
- parser 需要的最小前讀行為沒有被明確寫出
- 規格寫了「若 ciphertext 小於 16 bytes 就補零」，但你們同時又規定
  `Length` 最小為 16，這條規則實際上是死規則

修正建議：

- 把 header protection 改寫成逐步算法，不要再寫成概念式公式
- 明確定義 parser 行為：
  先讀 3-byte header，再預讀 16-byte sample，再反混淆長度，再讀剩餘 ciphertext
- 刪掉死規則，或重新調整最小長度定義，使文檔內部一致

這一塊還沒到「兩個團隊照 spec 就能互通」的程度。

### 5. `OPEN_CHANNEL` 仍然不是自定界格式，frame parser 還是做不穩

`OPEN_CHANNEL` 現在的格式是：

```text
Type | Channel ID | Kind | Init Window | ATYP | Target(Var) | Port
```

[protocol.md:137](/Users/leask/Documents/chameleon/protocol.md:137)

這裡最大的問題是 `Target(Var)` 沒有長度。

如果 `ATYP` 只有 IPv4 / IPv6，長度還能靠型別硬推；但只要有 domain name，
parser 就無法知道 `Target` 在哪裡結束、`Port` 從哪裡開始。文檔也沒定義
domain name 的編碼、最大長度與標準化規則。

修正建議：

- 最直接的做法：
  加一個 `Target Length`
- 或者對每個 `ATYP` 定義完全獨立的結構，而不是共享 `Target(Var)`
- 同時補上：
  - domain name 編碼
  - 最大長度
  - 是否要求 lower-case / punycode / canonical form

這是一個純 parser 級阻塞點，不修就談不上 wire-level 穩定。

### 6. 後置授權已成正式取捨，但 `Provisional Session` 的資源邊界仍然沒有被協議化

你們現在已正式接受：

- 先跑 Noise 握手
- 再做 `CLIENT_AUTH`
- pre-auth replay 由營運層 rate limit 緩解

[design.md:41](/Users/leask/Documents/chameleon/design.md:41)
[design.md:43](/Users/leask/Documents/chameleon/design.md:43)

這個取捨可以接受，但相應的資源控制仍然寫得太輕：

- 沒有 `PreHandshake / Provisional / Authorized / Draining / Closed`
  的正式狀態機
- 沒有 auth timeout
- 沒有 provisional 階段可接收的最大 records / bytes
- 沒有明確規範在 auth 完成前能做哪些事、不能做哪些事

目前的規格只有一句「第一個 record 的第一個 frame 必須是 `CLIENT_AUTH`」：

[protocol.md:133](/Users/leask/Documents/chameleon/protocol.md:133)

這不足以封住實作分歧，也不足以防止資源邊界漂移。

修正建議：

- 將 provisional state 正式寫入 `protocol.md`
- 補三條硬規則：
  - auth 必須在固定時間窗內完成
  - auth 前不得處理 `OPEN_CHANNEL` / `DATA`
  - auth 前可接受的 inbound bytes / records 有硬上限
- 讓 `GOAWAY 0x05` 的行為與 provisional state 關閉語義對齊

### 7. `PAD` / shaping budget / flow control 仍然是分裂模型，效率目標仍然站不住

當前規格明確寫了：

- `DATA` 消耗 channel credit 與 session credit
- `PAD` 不消耗任何 credit
- `PAD` 只受獨立 shaping / padding budget 控制

[protocol.md:180](/Users/leask/Documents/chameleon/protocol.md:180)
[protocol.md:226](/Users/leask/Documents/chameleon/protocol.md:226)

這個模型最大的問題不是安全，而是效率敘事會被自己打穿：

- session credit 可能顯示健康，但實際 egress 已被 `PAD` 吃掉
- `PAD` 與控制幀都被歸類為「不消耗 credit」，會讓 scheduler 很難表達優先級
- `goodput`、`wire utilization`、`session health` 會變成三套不同指標

修正建議：

- 明確定義 scheduler：
  `Auth / Critical Control > DATA > PAD`
- 寫死規則：
  `PAD must never starve authenticated control or data`
- shaping budget 不一定要併入 channel credit，
  但至少要併入 session 級 egress budget
- 對 `PAD` 增加時間窗上限，而不只是抽象地說「由策略決定」

如果不把 `PAD` 納入整體發送預算，效率目標就是自我矛盾。

### 8. `Version` / `Caps` 目前不是真正的 in-band negotiation，只是帶寬昂貴的配置校驗

握手裡現在有：

- `Version`
- `Caps`
- `Selected Caps`

[protocol.md:53](/Users/leask/Documents/chameleon/protocol.md:53)
[protocol.md:84](/Users/leask/Documents/chameleon/protocol.md:84)

但同時 pre-auth 失敗的策略又是完全靜默：

[design.md:53](/Users/leask/Documents/chameleon/design.md:53)
[protocol.md:236](/Users/leask/Documents/chameleon/protocol.md:236)

這代表：

- 版本不符時，client 收到的只是超時或中斷
- capability 不兼容時，也沒有明確的 in-band 回退路徑
- `Selected Caps` 的實際前提是：client 與 server 事前已透過控制面拿到高度同步的配置

也就是說，這不是 negotiation，而是「事前配置同步成功後的最終確認」。
這本身不是錯，但文檔現在還沒把話說清楚。

修正建議：

- 在 `design.md` 與 `protocol.md` 明確寫死：
  協議不提供通用的 in-band 版本回退
- 將 `Version` / `Caps` 定位為 config validation，而不是樂觀協商
- 定義 unknown capability bit、版本不符與回退策略的精確處理規則

### 9. `GOAWAY` 的格式比上一輪更像樣，但 client recovery matrix 仍然缺席

`GOAWAY` 現在已有：

- `Error Code`
- `Last Accepted Channel ID`

[protocol.md:211](/Users/leask/Documents/chameleon/protocol.md:211)

這是正向進展，但恢復語義仍然沒有被規範：

- 哪些 code 應重連同一節點
- 哪些 code 應切節點
- 哪些 code 必須等待控制面刷新
- draining 階段哪些 channel 可視為成功，哪些要重試

在 hostile network 場景裡，沒有 recovery matrix，
client 行為最後一定會各實作各自發明。

修正建議：

- 為每個 `GOAWAY Code` 補一個 recovery decision
- 至少要有：
  - retry same node
  - retry another node
  - require config refresh
  - do not retry automatically

## What Improved

這一輪不是沒有進步，至少有三個方向是對的：

- 你們把 transport-specific 關閉行為從核心語義裡抽象成了
  `Silent Drop` / `Abort Transport`
  [design.md:54](/Users/leask/Documents/chameleon/design.md:54)
  [protocol.md:238](/Users/leask/Documents/chameleon/protocol.md:238)
- `Initial Key Schedule` 終於被寫進 `protocol.md`
  [protocol.md:29](/Users/leask/Documents/chameleon/protocol.md:29)
- 單一流不只會造成資料 HOL，還會讓控制面與 bulk data 共享故障域，
  這件事已經被設計文檔明確承認
  [design.md:72](/Users/leask/Documents/chameleon/design.md:72)

這些改動都是朝「真正能實作」的方向走，不是裝飾。

## Next Round Priorities

下一輪如果只先解最重要的五件事，我建議按這個順序：

1. 釘死 `NK` 與短期 responder static key 的真實分發模型。
2. 把 `CLIENT_AUTH` 升級為真正的授權協議，而不是占位 frame。
3. 補齊 `Initial Key Schedule`、nonce 構造與 header protection 的精確算法。
4. 把 `OPEN_CHANNEL` 改成完全自定界格式。
5. 為 provisional session、`PAD` 調度與 `GOAWAY` 恢復補上硬規則。

## Summary

這一輪最大的正面變化，是你們終於開始把幾個原本只存在於討論裡的東西
落到規格上，例如初始 key schedule、抽象 transport action、
以及單一流故障域的正式承認。

但真正阻塞協議走向可實作的問題仍然很集中：

- 握手模式與信任分發仍然互相打架
- 授權模型仍然只有位置，沒有內容
- record/header protection 仍然沒有達到可獨立互通的精度
- flow control、padding、恢復語義仍然沒有收成單一一致模型

如果下一輪還不先把 `NK / CLIENT_AUTH / KDF+HP`
這三塊釘死，文檔會繼續看起來更完整，但實際上仍然不能穩定落地。
