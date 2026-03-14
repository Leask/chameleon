# Chameleon 聯合審查報告
*(本輪重點：接受已經合理收斂的反駁，只攻擊仍未閉環的信任、授權與恢復語義)*

## Findings

### 1. `Noise NK`、短期 `Responder Static Key` 與 `Epoch Cert` 的信任圖仍然沒有閉環

這仍然是目前最大的結構問題。

`design.md` 的主張還是：

- client 單向預知 server static key
- edge 持有的是短期 `Server Static Key`
- client 透過驗證 `Epoch Cert` 來信任這個短期 key

[design.md:23](/Users/leask/Documents/chameleon/design.md:23)
[design.md:37](/Users/leask/Documents/chameleon/design.md:37)

`protocol.md` 仍然選擇 `Noise NK`：

- client 首包是 `-> e, es`
- 這要求 initiator 在送出第一包前已知 responder static pubkey

[protocol.md:27](/Users/leask/Documents/chameleon/protocol.md:27)
[protocol.md:53](/Users/leask/Documents/chameleon/protocol.md:53)

但 `Epoch Cert` 仍然是在 server 回包裡才帶回：

[protocol.md:78](/Users/leask/Documents/chameleon/protocol.md:78)

這代表現在仍然有兩條互相衝突的敘事：

- 一條是 `NK` 所要求的「先知道 responder static key」
- 一條是文檔描述出的「回包中的 `Epoch Cert` 幫 client 建立對短期 key 的信任」

這兩件事不能同時作為主敘事存在。你們必須選一條。

修正建議：

- 若保留 `NK`，就明確寫死：
  client 必須在握手前從控制面拿到短期 responder static pubkey
- 同時補出 verifier algorithm：
  client 必須比對「本次 `NK` 使用的預置 static key」與 `Epoch Cert` 內公鑰
  完全一致，否則握手失敗
- 若不接受預分發短期 key，就不要再維持 `NK`

### 2. `Epoch Cert` 的驗證語義仍然沒有被正式定義

即使先不談 `NK` 的前提，`Epoch Cert` 本身的驗證也還沒閉環。

目前規格只定義了 `Epoch Cert` 的結構：

[protocol.md:78](/Users/leask/Documents/chameleon/protocol.md:78)

但沒有定義這些最關鍵的步驟：

- client 用哪把控制面公鑰驗證 Ed25519 簽名
- signer key 如何輪換
- clock skew 容忍多少
- `Epoch_Window_Start/End` 擦邊時如何處理
- 本地時間明顯錯誤時是硬失敗、受限重試，還是要求 config refresh

目前 `design.md` 只說 `Epoch Cert` 是 freshness window，
不是 clock sync。[design.md:25](/Users/leask/Documents/chameleon/design.md:25)
這是合理收斂，但還不是 verifier algorithm。

修正建議：

- 在 `protocol.md` 補一個 `Epoch Cert Verification` 小節
- 至少定義：
  - signer trust root 的來源
  - clock skew tolerance
  - pubkey mismatch 的處置
  - window invalid / expired / not-yet-valid 的恢復策略

### 3. `CLIENT_AUTH` 已經不再是占位符，但 `Auth Scheme 0x01` 仍未成為可互通規格

這一輪 `CLIENT_AUTH` 的進展是真實的：

- 有 `Auth Scheme`
- 有 `Credential ID`
- 有 `Proof Length`
- 文檔明確說預設方案會綁定握手 transcript hash

[protocol.md:138](/Users/leask/Documents/chameleon/protocol.md:138)
[protocol.md:149](/Users/leask/Documents/chameleon/protocol.md:149)

所以我上一輪「授權只是空殼」的批評，現在應該降級。
但它還沒有收口成真正可互通的協議，因為還缺：

- transcript hash 究竟是哪個值
- `Proof` 的精確序列化
- `Credential ID` 的命名空間與查找規則
- 撤銷與輪換如何影響 verifier
- `Proof Length` 的最大值

這些不是附錄細節，而是跨實作能否互通的核心。

修正建議：

- 把 `Auth Scheme 0x01` 補成逐步算法
- 明確：
  - 簽名輸入
  - 驗證查找流程
  - `Proof Length` 上限
  - 驗證失敗分類與對應錯誤

### 4. `Provisional Session` 的方向是對的，但現在仍是「示意規則」，不是「規範規則」

你們本輪正式寫進了：

- auth 前禁止處理 `OPEN_CHANNEL` / `DATA`
- provisional timeout
- provisional 最大接收字節數

[protocol.md:145](/Users/leask/Documents/chameleon/protocol.md:145)
[protocol.md:147](/Users/leask/Documents/chameleon/protocol.md:147)
[protocol.md:148](/Users/leask/Documents/chameleon/protocol.md:148)

這是明顯進步。

但問題在於現在的語句仍然是：

- `如 3 秒`
- `如 1KB`

這不是規範值，而是例子。結果就是不同實作的 admission envelope
仍然會漂移。

另外，現在還有一個細節沒定死：

- 第一個 record 的第一個 frame 必須是 `CLIENT_AUTH`
- 但這個 record 在 auth frame 後是否允許帶其他 frames，文檔沒有明說

如果允許，就要定義 server 是否可在同一 record 內「先驗 auth，再處理後續
`OPEN_CHANNEL`」。如果不允許，就應直接寫死「第一個 record 只能包含
`CLIENT_AUTH`」。

修正建議：

- 把 provisional timeout / byte budget 改成：
  - 規範常數
  - 或控制面可配置值，但有明確默認值與上下界
- 同時補一條硬規則：
  第一個 record 是否允許在 `CLIENT_AUTH` 後攜帶其他 frames

### 5. Recovery matrix 終於出現了，但幾個恢復決策仍然不合理

這一輪把 recovery strategy 寫進 spec 是正向進展：

[protocol.md:261](/Users/leask/Documents/chameleon/protocol.md:261)

但表裡有幾條決策現在仍然很粗，甚至會把 client 帶錯方向：

- `Handshake AEAD / MAC Fail -> Retry Different Node`
  這可能是 stale config、錯誤 key、節點輪換，也可能是網路干預；
  不能直接收斂成「換節點」
- `Unknown Frame Type -> Retry Same Node (Version mismatch check)`
  如果真是版本或 capability 不匹配，同節點重試通常沒有意義，
  更合理的是先 config refresh
- `Session Flow Control Overflow -> Retry Same Node`
  如果 overflow 是策略衝突或實作 bug，重試同節點很可能只會重演

修正建議：

- recovery matrix 至少拆成三欄：
  - immediate action
  - retry policy
  - config refresh required?
- 對 `silent drop` 類錯誤，先判斷配置陳舊，再判斷是否切節點

### 6. `design.md` 的總綱仍然過度承諾，和你們自己接受的邊界相衝突

`design.md` 仍然寫：

> 透過 AEAD 和帶外信任分發（OOB Trust），確保協議在數學層面上不可被探測和重放。

[design.md:10](/Users/leask/Documents/chameleon/design.md:10)

但同一份文檔又明確接受：

- pre-auth handshake replay 是 threat model 外

[design.md:43](/Users/leask/Documents/chameleon/design.md:43)

這個矛盾還在。你們的具體設計已經比總綱更誠實了，現在反而是總綱在拖後腿。

修正建議：

- 把總綱改成分層安全目標：
  - responder authentication
  - post-auth integrity/confidentiality
  - pre-auth silence
  - replay tolerance / replay limits
  - operational survivability

## Response To Design Rebuttals

### 1. 「握手不需要明文 envelope」這個反駁是否合理？

合理。我接受。

前提是：

- 握手首包與回包在該版本內固定長度
- version / caps 不能偷偷改變握手長度

在這個前提下，握手定界本身不再是主問題。

### 2. 「Fallback 應降級為部署策略」這個反駁是否合理？

合理。我接受。

這讓核心協議終於不再被少數承載能力綁架，這一條已經不是主要攻擊點。

### 3. 「單一流 HOL 與控制耦合是可接受取捨」這個反駁是否合理？

合理，但前提是你們繼續保持語言誠實。

`design.md` 已經承認控制信令與 bulk data 共享故障域：

[design.md:72](/Users/leask/Documents/chameleon/design.md:72)

這讓反駁本身站得住。現在只剩一個表述層面的收尾：
前文裡「不破壞底層能力」這句還是應再收斂一點，因為在 session 層你們確實
重新引入了自己的耦合。

### 4. 「Bucketing 不應寫死在核心協議裡」這個反駁是否合理？

基本合理，而且這一輪你們補的調度器優先級與 session-level egress budget
已經讓它比上一輪完整很多：

[protocol.md:204](/Users/leask/Documents/chameleon/protocol.md:204)
[protocol.md:207](/Users/leask/Documents/chameleon/protocol.md:207)

所以這一條我不再作為阻塞級問題。

## What Improved

這一輪確實有幾個之前的主問題可以正式降級：

- header protection 的「常量 mask」錯誤已修正
  [protocol.md:123](/Users/leask/Documents/chameleon/protocol.md:123)
- `OPEN_CHANNEL` 的自定界問題已解決
  [protocol.md:155](/Users/leask/Documents/chameleon/protocol.md:155)
- `PAD` 的調度優先級與 session egress budget 已寫進規格
  [protocol.md:204](/Users/leask/Documents/chameleon/protocol.md:204)
- `GOAWAY` 的 draining 邊界更清楚了
  [protocol.md:244](/Users/leask/Documents/chameleon/protocol.md:244)
- 初始 nonce 構造現在已經明確到 byte layout
  [protocol.md:109](/Users/leask/Documents/chameleon/protocol.md:109)

這些都是實質進步，不是文案修飾。

## Next Round Priorities

下一輪如果只先做 5 件事，我建議按這個順序：

1. 釘死 `NK` 與短期 responder static key 的真實分發與驗證路徑。
2. 補完 `Epoch Cert Verification` algorithm。
3. 把 `Auth Scheme 0x01` 補成完整 verifier algorithm。
4. 把 provisional session 的限制從示意值升級為規範值或正式配置模型。
5. 重寫 recovery matrix，讓它成為真正可執行的恢復決策。

## Summary

這一輪的主要結論不是「沒進步」，而是：
你們已經開始把上一輪最表層的 wire-level 錯誤修掉，協議正在從
「顯然不可實作」往「開始可以實作」移動。

但真正卡住落地的問題，現在已經收斂到少數幾個核心閉環：

- `NK` 與短期 responder key 的信任圖沒有閉環
- `Epoch Cert` 還沒有 verifier algorithm
- `Auth Scheme 0x01` 還沒有 verifier algorithm
- provisional session 仍然是示意規則
- recovery matrix 還沒有到可直接執行的程度

現在最該做的不是再加功能，而是把這幾個閉環補死。只有這樣，
這份規格才會真正從「可討論」變成「可穩定實作」。
