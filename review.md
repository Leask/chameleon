# Chameleon 聯合審查報告
*(針對當前 `design.md` 與 `protocol.md` 的 review)*

## Findings

### 1. `Noise NK` 的握手轉錄仍然不精確，這會直接阻塞互通

`protocol.md` 已將握手模式收斂到 `Noise_NK_25519_ChaChaPoly_BLAKE2s`，
但握手請求仍寫成 `-> e, d, es`。[protocol.md:27-28,39-40]

問題不在字面漂亮與否，而在這會直接造成實作者分歧：

- `d` 不是標準 Noise pattern token
- 不同實作者會各自猜測這是筆誤、私有擴展，還是額外 payload 階段
- 一旦 transcript 理解不同，固定長度、KDF 輸出與 MAC 驗證都會不一致

建議：

- 把握手模式的消息序列寫成唯一且完整的最終版本
- 不要在正文保留任何可能是草稿殘留的 token
- 最好在 `protocol.md` 補一個小節，列出：
  - initiator 前提
  - responder 前提
  - 每條消息的確切 transcript 元素
  - 每條消息的固定長度來源

### 2. Session 級 flow control 只有增量更新，沒有初始條件，規格仍然不閉環

`protocol.md` 新增了 `SESSION_WINDOW_UPDATE`，方向是對的。
[protocol.md:164-170]

但目前仍缺三個關鍵規則：

- session 初始 credit 是多少
- `DATA` 同時如何消耗 channel credit 和 session credit
- control frame / `PAD` 是否消耗 session credit

這不是小洞。沒有初始 session window，實作者只能自己猜：

- 是握手後默認無限額
- 還是默認 0，必須先等 `SESSION_WINDOW_UPDATE`
- 還是沿用某個隱含常數

這會直接導致互通問題和記憶體風險。

建議：

- 在握手完成後明確定義 `Initial Session Window`
- 明確規範 credit 消耗順序
- 明確規範 `PAD` 和 control frames 是否計入 session budget

### 3. Record 頭部混淆算法仍然欠缺精確定義，短 record 甚至無法按當前文字實作

Record layer 這一版有進展，但還沒有到「可精確實作」。
[protocol.md:95-105]

主要問題有三個：

- `Length` 究竟表示 plaintext payload 長度、ciphertext 長度，還是
  `ciphertext + tag` 的總長度，文檔沒有明確說死
- `Mask = ChaCha20(HP_Key, Ciphertext_Sample[0..15])[0..2]` 缺少對
  ChaCha20 nonce/counter 用法的定義
- 當 record 很短時，`Ciphertext_Sample[0..15]` 可能根本不存在
  例如只有一個很短的 control frame

這是阻塞級問題，因為 header protection 一旦寫得模糊，互通性就會
第一時間破裂。

建議：

- 明確定義 `Length` 的單位與範圍
- 明確定義 HP 的 nonce / counter / sample 來源
- 明確定義最小 record 大小；若不足 16 bytes sample，規定如何補足
  或改從 `ciphertext || tag` 取樣

### 4. `KEY_UPDATE` 只輪換了 traffic key，沒有定義 HP key 如何同步輪換

當前規格已定義：

- `Next_Traffic_Key = HKDF-Expand(Current_Traffic_Key, "rekey", 32)`
- 下一個 record 開始翻轉 phase

[protocol.md:177-181]

但 header 仍然依賴獨立的 `HP_Key`。[protocol.md:28-30,103-105]

也就是說，目前規格清楚定義了 payload key 的輪換，卻沒有定義
header protection key 的輪換。這會導致兩種實作分歧：

- 一種認為 HP key 與 traffic key 同步 rekey
- 一種認為 HP key 固定不變

這會直接導致 phase 翻轉後 record 頭部解析失敗。

建議：

- 把 rekey 擴展為完整 key schedule
- 明確定義每次 rekey 同時導出的：
  - next traffic key
  - next HP key
- 明確定義 phase 翻轉與兩組 key 的綁定關係

### 5. Datagram 仍然是半定義狀態，現在寫進正式 capability 會讓規格變脆

`design.md` 已經把 Datagram 降成未來擴展，優先保證 stream 代理穩定。
[design.md:25-26]

但 `protocol.md` 仍然把它放進正式能力與正式 `Kind`：

- `Caps Bit 0: 支援 Datagram Channel`
- `Kind 0x02: UDP Datagram`

[protocol.md:48-50,120]

可目前完全沒有配套語義：

- datagram 最大大小
- 是否允許 fragmentation
- datagram 是否遵守與 stream 相同的 flow control
- `FIN_CHANNEL` 對 datagram 是否有意義
- datagram path 的錯誤與丟棄語義

在這種情況下把 Datagram 寫成已存在 capability，會讓後續版本演進變難。

建議：

- 如果近期不做，將 `Bit 0` 與 `Kind 0x02` 明確標為 reserved
- 如果要保留，就必須新增 datagram 專屬規則，而不是只加一個枚舉值

### 6. 單一 reliable byte stream 的取捨已被清楚承認，但文檔還沒有把效率後果寫透

這一輪 `design.md` 終於正面承認：

- 整個 session 跑在單一 reliable byte stream 上
- 這會重新引入 HOL
- 這是為了 strict lockstep 的明確取捨

[design.md:21-24]

這個取捨本身可以成立，但目前文檔還沒有把代價寫透：

- 任一底層重傳都會阻塞整個 session 的所有 channels
- 大流量 channel 和互動流 channel 之間的互相影響會被放大
- 在高丟包下，`OPEN_ACK`、`WINDOW_UPDATE`、`PING/PONG` 等控制流
  也會被資料流一起卡住

這不是 bug，而是需要更誠實的適用邊界。

建議：

- 在 `design.md` 補一句明確結論：
  單 session 應只承載同類型或相近時延敏感度的流量
- 如果要同時承載 bulk 和 interactive，建議靠上層切分成多個 session
  而不是寄希望於單 session 排程完全化解 HOL

### 7. `design.md` 已退回邊界文檔，但反駁段仍殘留舊的常數和雙版本描述

這一輪 `design.md` 大方向是對的：它大部分已不再重複 wire-level 常數。
[design.md:17-19]

但反駁段仍然留有：

- `160 Bytes 或 112 Bytes`

[design.md:58-60]

這說明 `design.md` 仍然殘留協議草稿時期的文字痕跡。這不會立即破壞
互通，但會削弱 `protocol.md` 作為唯一真相來源的地位。

建議：

- `design.md` 裡刪掉所有具體長度常數
- 反駁段也只保留抽象結論，例如：
  「固定長度握手由 `protocol.md` 定義」

## Open Questions

1. 你們是否接受「互動流與 bulk 流應拆到不同 session」作為正式部署建議？
2. Datagram 是近期目標，還是明確延後到未來版本？
3. `PAD` 是否計入 session credit？如果不計，如何避免 padding 佔滿輸出緩衝？
4. `OPEN_CHANNEL` 被拒絕時，是否用 `RESET_CHANNEL` 作為標準負向回應？

## Summary

這一輪相較上一輪是前進的，因為：

- `design.md` 終於開始像真正的邊界文檔
- `protocol.md` 補上了 channel lifecycle、session flow control 和基礎
  錯誤矩陣

但新的主要風險已經很清楚：

- 握手 transcript 仍不夠精確
- record layer 還差最後一層精確化
- rekey 沒有覆蓋 HP key
- datagram 還沒準備好就被寫進正式能力
- 單一 byte-stream 模型的 HOL 代價需要更明確地納入設計邊界

如果下一輪只做一件事，我建議先把 `protocol.md` 的
`handshake + record + rekey` 三個部分補成真正可互通的最終規格，
不要急著再擴能力面。
