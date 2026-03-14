# Chameleon 聯合審查報告
*(針對當前 `design.md` 與 `protocol.md` 的 review)*

## Findings

### 1. 協議目前沒有真正的客戶端授權模型，任何拿到伺服器公鑰的人都能接入

當前 `protocol.md` 的握手模式是 `Noise_NK`，明確定義為客戶端預先持有
伺服器靜態公鑰的單向信任模型。[protocol.md:27-28]
[protocol.md:38-39]

這代表目前握手只完成了「client 驗證 server」，沒有完成：

- client 身份驗證
- client 授權
- 設備撤銷
- 設備分級權限

`design.md` 同時又強調控制面、Root CA、零信任邊緣節點與安全邊界，
這種語氣預設系統具備可治理的身份模型。[design.md:10-13]
[design.md:34-40]

但以目前 wire spec 來看，只要任何人拿到 server public key，就可以
完成合法握手。這使得：

- OOB 分發的 server public key 變相成了「共享入場券」
- client 端設備撤銷在協議層根本無從實施
- 控制面雖然存在，但沒有落到真正的授權邊界

修正建議：

- 要嘛在握手中加入明確的 client authentication / authorization
  機制
- 要嘛在握手後第一個受保護階段引入強制的 client auth frame
- 無論採用哪種方式，都必須明確定義：
  - 設備身份
  - 設備撤銷
  - 金鑰輪換
  - 未授權 client 的標準失敗路徑

在這件事補上前，協議更像是「伺服器認證的安全隧道」，不是一個
有完整接入控制的系統。

### 2. 設計文檔的 edge trust 模型與 `NK` 握手的實際密鑰需求仍然衝突

`design.md` 明確聲稱：

- 邊緣節點預設不可信且短命
- 線上節點僅持有短期流量金鑰與限時 `Epoch Cert`

[design.md:10-13]
[design.md:38-40]

但 `protocol.md` 的 `NK` 握手要求 responder 在邊緣節點上持有
`Server Static Private Key`，否則無法完成 `es` 解密與握手回應。
[protocol.md:27-28]
[protocol.md:38-57]

這代表目前有一個核心矛盾：

- 設計層想要短命 edge
- 協議層卻要求 edge 持有高價值長期秘密

如果這把 responder static key 洩露，攻擊者就能：

- 持續偽裝成合法 edge
- 直到 client OOB 更新信任材料之前都有效

這和「edge 只持有短期材料」是衝突的。

修正建議：

- 要嘛承認 responder static key 就是 edge 長期高價值秘密，並重寫
  設計邊界
- 要嘛把握手改成「client 信任短期 responder 憑據，由上層根信任
  簽發」的模式，讓 edge 不必持有長期根據任
- 不要讓 `Epoch Cert` 只是 freshness window；它更合理的用途應是
  綁定短期 responder key 與控制面信任鏈

目前這是阻塞級架構問題，不只是文案不一致。

### 3. `Epoch Cert` 只解決 freshness 的說法仍然不夠，握手重放仍未被真正處理

這一輪你們已經把 `Epoch Cert` 收斂成 freshness window，這是比前幾輪
清楚的。[design.md:24-25]
[protocol.md:62-70]

但目前規格仍然沒有回答更實際的問題：

- server 如何拒絕被重放的 client first flight
- server 是否維護 replay cache
- `Epoch_Window_Start/End` 與握手 replay 的關係是什麼
- replay 攻擊是否被明確接受為 threat model 外

當前協議的 strict lockstep 只保護握手後的 records。[protocol.md:89-93]
它不能自動保護握手本身。

以現在的規格，重放一個曾經抓到的合法 client 首包，server 很可能仍會：

- 成功完成握手計算
- 回送 server first response
- 消耗 CPU / 帶寬 / 狀態資源

如果你們接受這個風險，就應該明說它是設計取捨。
如果不接受，就要在規格裡定義 server-side anti-replay。

修正建議：

- 明確二選一：
  - 握手 replay 不防，只防 post-auth replay
  - 握手 replay 需要 server-side anti-replay cache / token
- 無論選哪條路，都要把這件事寫進 `design.md` 的安全邊界

### 4. Record layer 仍然欠缺最後一層精確性，現在還不足以保證獨立互通

`protocol.md` 這一輪明顯比前面更完整，但 record layer 還有幾個會直接
造成實作者分歧的缺口。[protocol.md:87-105]

主要問題：

- `Length` 沒有明確說是：
  - plaintext payload length
  - ciphertext length
  - 還是 `ciphertext + tag` 的總長度
- `Mask = ChaCha20(HP_Key, Sample)[0..2] (以 0 為 Nonce 與 Counter)`
  這個寫法仍不夠精確，因為 ChaCha20 的輸入模型沒有被完整定義
- `Sample` 對短 record 的補零規則已經補上，但這是取樣規則，不是
  record 最小長度規則
- 沒有 record 最大長度與接收端上限策略

這些問題不屬於「可以靠實作默契解決」的範圍。

修正建議：

- 用一節單獨定義 record header 語義：
  - `Length` 的確切含義
  - `Flags` 的編碼
  - tag 是否包含在 length 中
- 用一節單獨定義 header protection 算法：
  - HP key
  - sample
  - nonce/counter
  - 輸出截斷方式
- 定義 `MAX_RECORD_SIZE`

如果這些不寫死，兩個獨立團隊仍可能無法互通。

### 5. `PAD` 不消耗任何 credit 的設計會打穿 session 預算

你們這一輪明確寫了：

- `DATA` 同時消耗 channel credit 與 session credit
- `PAD` 與所有 control frames 不消耗任何 credit

[protocol.md:154-170]

這裡的 control frame 免計費可以接受，但 `PAD` 完全免計費會帶來
直接問題：

- 發送方可以無限制插入 padding
- session window 失去對總傳輸資源的約束
- 在單一 reliable byte stream 上，padding 甚至會直接拖慢真正的
  互動流

這和你們想用 session flow control 防止資源耗盡的目標相衝突。

修正建議：

- `PAD` 至少應消耗 session-level budget
- 或者定義單獨的 shaping budget / padding budget
- 不要讓 `PAD` 完全脫離資源約束

### 6. `GOAWAY` 的語義仍然太薄，無法乾淨處理 drain 與 in-flight channels

當前 `GOAWAY` 只有：

- type
- error code

[protocol.md:183-185]

而它的語義卻寫成：

- 不再接受新的 `OPEN_CHANNEL`
- 現有通道允許排空

[protocol.md:183-185]

這會留下典型歧義：

- 某個 `OPEN_CHANNEL` 是在 `GOAWAY` 前已被接受，還是 race 失敗
- 發起方是否可以安全重試最後幾個開啟請求
- drain 的邊界在哪裡

HTTP/2 / HTTP/3 之所以在 `GOAWAY` 中帶更明確的邊界，不是裝飾，
而是為了避免這種模糊。

修正建議：

- `GOAWAY` 增加「最後已接受的 Channel ID」之類的界線欄位
- 或明確規範：
  `GOAWAY` 之後所有尚未收到 `OPEN_ACK` 的 `OPEN_CHANNEL` 都視為未成功

### 7. `design.md` 的邊界文檔方向是對的，但 still over-claims 效率

這一輪 `design.md` 已經比之前乾淨很多，這是正向進展。
[design.md:17-26]

但它仍然在高層敘述裡保留了比較強的效率口徑：

- 極致效能
- 低延遲
- 強健可用性

[design.md:13]

而你們同時又正式接受：

- 單一 reliable byte stream
- strict lockstep
- HOL 重新引入

[design.md:21-23]

這不是說設計一定錯，而是高階敘述應更誠實：

- 它對簡潔性、安全邊界和可推理性更友好
- 但它對跨流隔離與高丟包下的互動體感並不佔優

修正建議：

- 在 `design.md` 直接把這個取捨寫成顯式代價
- 把「高效」改成更精確的描述：
  - 協議簡潔
  - 解析成本低
  - 但在多類型混合流上不追求最佳 tail latency

## Open Questions

1. client authentication / authorization 最終放在握手內，還是握手後第一個
   受保護階段？
2. responder static key 是否被視為可長期存在於 edge 的高價值秘密？
3. 握手 replay 是被接受的設計取捨，還是需要正式 anti-replay 機制？
4. `PAD` 最終是走 session budget，還是走獨立 shaping budget？
5. `GOAWAY` 是否需要攜帶最後接受的 Channel ID？

## Summary

這一輪的正面進展是明確的：

- `design.md` 基本完成了向邊界文檔的收縮
- `protocol.md` 的握手、channel lifecycle、session flow control 和
  rekey 都比上一輪更接近真正可實作的規格

但現在暴露出的問題也更本質了：

- 協議還沒有真正的 client auth
- edge trust 模型與 `NK` 的長期 responder key 需求衝突
- freshness 不等於 handshake anti-replay
- record / HP 還差最後一層精確化
- `PAD` 免 credit 會打穿資源控制
- `GOAWAY` 邊界仍然模糊

如果下一輪只做一件事，我建議先把**身份與授權模型**定死。
因為在目前這個版本裡，這已經是比 record 微調更大的結構問題。
