# Chameleon 聯合審查報告
*(本輪重點：回應 `design.md` 中的自我質疑，並只保留仍然阻塞落地的問題)*

## Findings

### 1. Header protection 現在變成了常量遮罩，這是本輪最嚴重的新錯誤

`protocol.md` 現在先定義了：

- `Sample` = `Ciphertext` 前 16 bytes

[protocol.md:118](/Users/leask/Documents/chameleon/protocol.md:118)

但在後面的「精確混淆演算法」裡，又寫成：

- `Key = HP_Key`
- `Nonce = 12 bytes of zero`
- `Block Counter = 0`
- 取生成 keystream 的前 3 bytes 當 `Mask`

[protocol.md:119](/Users/leask/Documents/chameleon/protocol.md:119)

這意味著 `Sample` 根本沒有被用到。結果就是：

- 在同一個 key phase 內，所有 records 的 header mask 都是常量
- `Length` 與 `Flags` 的遮罩不再與每個 record 的 ciphertext 綁定
- 只要攻擊者猜到或觀察到一個 record 的明文長度，就能反推出 mask，
  之後可對其他 records 的 header 做可預測修改

這不是小瑕疵，是直接把 header protection 變成了「固定 XOR」。

修正建議：

- 立即重寫 header protection 算法，讓 mask 與每個 record 的 sample 綁定
- 如果你們想沿用 `Sample` 概念，就必須精確定義它如何進入 HP 算法
- 如果不想自己發明，就直接採用一個已知構造，逐項映射到你們的 header

這一條的嚴重性高於前幾輪的所有 parser 級問題。

### 2. `Noise NK` 與短期 `Responder Static Key` 的信任分發模型仍然沒有閉環

`design.md` 的說法仍然是：

- client 單向預知 server static key
- edge 節點實際持有的是短期 `Server Static Key`
- 控制面以 `Epoch Cert` 對這個短期公鑰簽名

[design.md:23](/Users/leask/Documents/chameleon/design.md:23)
[design.md:34](/Users/leask/Documents/chameleon/design.md:34)

`protocol.md` 仍然採用 `Noise NK`，client 首包是 `-> e, es`：

[protocol.md:27](/Users/leask/Documents/chameleon/protocol.md:27)
[protocol.md:49](/Users/leask/Documents/chameleon/protocol.md:49)

但 `Epoch Cert` 還是在 server 回包裡才帶回：

[protocol.md:74](/Users/leask/Documents/chameleon/protocol.md:74)

這個結構矛盾仍然存在：

- `NK` 要求 client 在發第一包前就知道 responder static pubkey
- 但現在的文檔又把對短期 key 的信任建立描述成依賴回包裡的 `Epoch Cert`

修正建議：

- 如果堅持 `NK`，就明確寫死：
  client 事前必須透過控制面拿到短期 responder static pubkey
- `Epoch Cert` 的語義收斂為：
  確認這個已知短期 key 仍在有效期、未撤銷、屬於正確控制面信任鏈
- 如果不接受預分發短期 key，就不要繼續維持 `NK`

### 3. `CLIENT_AUTH` 有了基本骨架，但還不是完整授權協議

這一輪 `CLIENT_AUTH` 比上一版明顯進步：

- 新增了 `Auth Scheme`
- 新增了 `Credential ID`
- 新增了 `Proof Length`
- 明確聲稱預設方案會綁定握手 transcript hash

[protocol.md:131](/Users/leask/Documents/chameleon/protocol.md:131)
[protocol.md:142](/Users/leask/Documents/chameleon/protocol.md:142)

這使得我上一輪對「授權只是占位符」的質疑不再完全成立。這個方向是對的。

但它仍然沒有收口為可互通規格，因為以下關鍵點還沒定死：

- transcript hash 到底是哪個值，`Noise h` 還是額外摘要
- `Proof` 的精確序列化格式
- `Credential ID` 的命名空間與查找規則
- 驗證失敗是否區分「憑證不存在」「簽名錯誤」「憑證撤銷」
- `Proof Length` 的最大值，否則 provisional state 仍有記憶體壓力面

修正建議：

- 直接把 `Auth Scheme 0x01` 寫成逐步算法
- 補上 `Proof` 最大長度與 verifier algorithm
- 明確 `Credential ID` 與控制面信任鏈的映射規則

### 4. `Provisional Session` 方向是正確的，但目前寫法仍然像示意，不像規格

你們本輪正式引入了 `Provisional Session`，這是正向修正：

- auth 前禁止處理 `OPEN_CHANNEL` / `DATA`
- 有 provisional timeout
- 有 provisional 最大接收字節數

[protocol.md:138](/Users/leask/Documents/chameleon/protocol.md:138)
[protocol.md:140](/Users/leask/Documents/chameleon/protocol.md:140)
[protocol.md:141](/Users/leask/Documents/chameleon/protocol.md:141)

但這裡仍有一個規格層面的問題：

- 文檔寫的是「如 3 秒」「如 1KB」

這種寫法不是規範值，而是例子。對 wire protocol 來說，這會導致：

- 不同實作的 provisional resource envelope 不一致
- 客戶端很難預估 auth 期限
- 邊緣行為會在不同部署間漂移

修正建議：

- 二選一：
  - 把 timeout / byte budget 寫成明確常數
  - 或把它們定義為控制面配置項，但要求最小/最大邊界與默認值
- 同時補一張正式 state machine：
  `PreHandshake -> Provisional -> Authorized -> Draining -> Closed`

目前這一塊方向正確，但還沒達到可互通的規格精度。

### 5. `Initial Key Schedule` 仍然不夠嚴謹，現在只是比上一輪少了一半空白

本輪 `protocol.md` 終於把初始 KDF 寫出來了：

[protocol.md:29](/Users/leask/Documents/chameleon/protocol.md:29)

這是進步，但還有三個阻塞級缺口：

- `cs1` / `cs2` 是直接來自 `Noise Split()`，還是某個額外步驟後的值，沒寫清楚
- `HKDF-Expand` 的輸入語義與 `IKM` 的表述混在一起，不夠嚴格
- `Nonce = ChaCha_IV XOR Pad_Zero(Sequence_Number)` 仍缺少精確端序與補零規則

[protocol.md:30](/Users/leask/Documents/chameleon/protocol.md:30)
[protocol.md:105](/Users/leask/Documents/chameleon/protocol.md:105)

修正建議：

- 把 KDF 改寫成完整規範語句，而不是說明文
- 明確 `cs1/cs2` 對應的方向
- 給出 nonce 構造的精確 byte layout
- 最好附測試向量

### 6. Recovery matrix 已經出現，但裡面的幾個恢復決策並不合理

這一輪最明顯的進步之一，是你們終於把 client recovery strategy
寫進規格：

[protocol.md:247](/Users/leask/Documents/chameleon/protocol.md:247)

這個方向完全正確，我接受這個修正。

但表裡有幾條決策目前並不合理：

- `Handshake AEAD / MAC Fail -> Retry Different Node`
  這不一定對。它可能是配置陳舊、錯誤 key、節點輪換，也可能是網路干預。
- `Unknown Frame Type -> Retry Same Node (Version mismatch check)`
  如果真是版本不匹配，重試同一節點通常沒有意義，應優先 config refresh。
- `Session Flow Control Overflow -> Retry Same Node`
  如果 overflow 來自實作 bug、對端錯誤或策略不匹配，重試同一節點可能直接重演。

修正建議：

- recovery matrix 不要只填單一步驟
- 至少改成：
  - immediate action
  - retry policy
  - whether config refresh is required
- 對 `silent drop` 類錯誤，應把「配置可能過期」列為一級判斷路徑，
  不能只寫成 `Retry Different Node`

### 7. `PAD` 的機制與策略分離這個反駁在原則上成立，但協議面仍未完成

`design.md` 對 `PAD` 的反駁是：

- bucket/shaping 不該寫死在核心協議裡
- 核心協議只提供 `PAD` 機制
- 具體策略由控制面下發

[design.md:76](/Users/leask/Documents/chameleon/design.md:76)

這個反駁在架構原則上是合理的，我接受。

但 `protocol.md` 仍然維持：

- `PAD` 不消耗任何 channel/session credit
- `PAD` 只受獨立 shaping budget 約束

[protocol.md:195](/Users/leask/Documents/chameleon/protocol.md:195)
[protocol.md:241](/Users/leask/Documents/chameleon/protocol.md:241)

所以問題不再是「是否要把 bucketing 寫死在協議裡」，
而是「既然不寫死，是否仍然保證效率邊界不被打穿」。

這一點目前仍沒回答完。

修正建議：

- 在規格裡明確 scheduler 優先級
- 至少寫死：
  `Auth / Critical Control > DATA > PAD`
- `PAD must never starve authenticated traffic`
- shaping budget 至少要受 session egress budget 約束

### 8. `design.md` 對安全能力的表述仍然過度承諾，和你們自己的取捨相衝突

`design.md` 仍寫：

> 透過 AEAD 和帶外信任分發（OOB Trust），確保協議在數學層面上不可被探測和重放。

[design.md:10](/Users/leask/Documents/chameleon/design.md:10)

但同一份文檔又承認：

- pre-auth handshake replay 被主動接受

[design.md:43](/Users/leask/Documents/chameleon/design.md:43)

這個矛盾還在。你們的反駁段已經越來越務實，但總綱這一句仍然把安全邊界說錯了。

修正建議：

- 把這句改成分層目標，不要再用一個總句把全部包掉
- 至少分成：
  - responder authentication
  - post-auth integrity/confidentiality
  - pre-auth silence
  - replay limits
  - operational survivability

## Response To Design Rebuttals

### 1. 「握手不需要明文 envelope」這個反駁是否合理？

合理，但只在一個狹義前提下成立：

- 握手首包與回包長度在該版本內必須固定
- version/capability 不能偷偷改變握手長度

也就是說，這個反駁**對握手定界成立**，但不能被擴張成
「整個協議都不需要更精確的 parser discipline」。

### 2. 「Fallback 應降級為部署策略」這個反駁是否合理？

合理。我接受這個收斂。

把 Fallback 從核心協議移出之後，協議本身終於不再被少數承載能力綁架。
這一條現在不再是主要攻擊點。

### 3. 「單一流 HOL 是可接受取捨」這個反駁是否合理？

合理，但要付出語言誠實的代價。

你們現在已經承認：

- 單一流不只影響資料流
- 控制信令也共享同一故障域

[design.md:72](/Users/leask/Documents/chameleon/design.md:72)

這讓反駁本身成立。但同時，`design.md` 前文中
「不破壞底層能力」這種表述就應再收斂一點，因為在應用層你們確實重新引入了
自己的耦合。

### 4. 「Bucketing 不應寫死在核心協議裡」這個反駁是否合理？

原則上合理，我接受。

但這個反駁只解決了「機制與策略分離」，
沒有自動解決 scheduler、budget 與 efficiency accounting。
所以它不能被當成 `PAD` 問題已經完結的證據。

## What Improved

這一輪有幾個批評可以正式降級，不應再重複當作主問題：

- `OPEN_CHANNEL` 的自定界問題基本已解決
  [protocol.md:148](/Users/leask/Documents/chameleon/protocol.md:148)
- `CLIENT_AUTH` 不再只是單一 32-byte 黑盒
  [protocol.md:133](/Users/leask/Documents/chameleon/protocol.md:133)
- recovery matrix 已經正式進入 spec
  [protocol.md:247](/Users/leask/Documents/chameleon/protocol.md:247)
- Fallback 被降級為部署策略，這是正確收斂
  [design.md:24](/Users/leask/Documents/chameleon/design.md:24)

這些都是實質進步，不是表面修辭。

## Next Round Priorities

下一輪如果只先做 5 件事，我建議按這個順序：

1. 立即修正 header protection，去掉當前的常量 mask 問題。
2. 釘死 `NK` 與短期 responder static key 的真實分發模型。
3. 把 `Auth Scheme 0x01` 補成可互通的 verifier algorithm。
4. 將 provisional session 的限制改成真正的規範值或明確配置模型。
5. 重寫 recovery matrix，讓它成為可實際執行的恢復決策，而不是方向提示。

## Summary

這一輪最重要的結論不是「你們沒進步」，而是：
你們有幾個以前只是口頭上的修正，現在真的開始落到 spec 裡了。

所以我對這一輪的判斷是：

- 有些舊質疑已經被合理回應，應該正式降級
- 但同時你們在 header protection 和 recovery policy 上又暴露了新的、
  更靠近實作層的問題

目前最危險的點已經不是「設計哲學太空」，而是
**wire-level 細節正在開始決定這個協議到底能不能真正站住**。
