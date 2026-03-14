# Chameleon 設計與協議聯合審查報告
*(針對當前 `design.md` 與 `protocol.md` 的新一輪 review)*

## 0. 審查定位

這次 review 仍然只看核心協議，不討論語言、UI 或具體工程框架。

本輪判準只有四個：

- `design.md` 是否與 `protocol.md` 一致
- 協議是否能被不同實作者獨立實作並互通
- 協議是否在效率與可靠性上自洽
- 協議是否具備可演進性，而不是只能跑通單一版本

## 1. 總體判斷

這一輪和上一輪相比，`protocol.md` 確實有實質進展：

- 握手模式改成了 `NK`
- 補了 `OPEN_ACK`
- 補了 `FIN_CHANNEL` / `RESET_CHANNEL`
- 補了 `SESSION_WINDOW_UPDATE`
- `KEY_UPDATE` 至少開始定義 KDF 與切換時機
- 錯誤處理開始收斂成矩陣

但整體上仍然不能算「協議已經收斂」。

本輪最大的問題變成了兩類：

1. `design.md` 基本沒有跟上 `protocol.md`，文檔再度失去單一真相來源
2. `protocol.md` 雖然補了很多洞，但新的主結構暴露出更深層的效率與
   語義問題，特別是 HOL、channel 語義與 rekey 完整性

## 2. Findings

### 2.1 高嚴重度：當前模型會重新引入跨通道 HOL，與設計目標直接衝突

`design.md` 明確聲稱 Chameleon 不應破壞底層承載的能力，尤其像 QUIC
這種能避免 HOL 的承載。[design.md:13](/Users/leask/Documents/chameleon/design.md:13)

但 `protocol.md` 把整個協議明確建立在單一「可靠、保序的位元組流」
之上，並在這條位元組流內自行多路復用所有 channels。
[protocol.md:11](/Users/leask/Documents/chameleon/protocol.md:11)
[protocol.md:121](/Users/leask/Documents/chameleon/protocol.md:121)

這意味著：

- 如果底層是 TCP，HOL 當然存在
- 如果底層是單一 QUIC stream，HOL 依然存在
- 任何一個 record 因底層重傳而卡住時，所有邏輯 channel 都會一起停

這不是實作細節，而是目前協議模型本身的直接後果。

也就是說，協議現在**確實在破壞底層 QUIC 的跨流隔離能力**。

修正建議：

- 要嘛承認 Chameleon session 是「單一 byte-stream 隧道」，放棄不破壞
  QUIC HOL 這個主張
- 要嘛重新設計承載映射：
  - session control 放在單一控制流
  - data channels 在支持多流的承載上直接映射到底層多個 streams
  - 只有在不支持多流的承載上才退化到協議層多路復用

如果不處理這個問題，`OPEN_ACK`、flow control、priority 再漂亮，
高丟包下整體體感仍然會被單一丟包拖死。

### 2.2 高嚴重度：`design.md` 仍然是舊規格，已經不能當設計依據

`design.md` 前言說本輪已修正為單向預知 server key 的握手模式，
不再使用 `IK`。[design.md:19](/Users/leask/Documents/chameleon/design.md:19)

但正文仍然保留：

- `Noise_IK_25519_ChaChaPoly_BLAKE2s`
  [design.md:41](/Users/leask/Documents/chameleon/design.md:41)
- 160-byte `IK` client 首包
  [design.md:46](/Users/leask/Documents/chameleon/design.md:46)
- `Epoch Cert` 用 `Not_Before` 做時鐘校準
  [design.md:57](/Users/leask/Documents/chameleon/design.md:57)
- channel frames 仍然只有 `OPEN_CHANNEL/DATA/WINDOW_UPDATE/CLOSE_CHANNEL`
  [design.md:89](/Users/leask/Documents/chameleon/design.md:89)

而 `protocol.md` 已經改成：

- `Noise_NK`
  [protocol.md:27](/Users/leask/Documents/chameleon/protocol.md:27)
- 112-byte client 首包
  [protocol.md:53](/Users/leask/Documents/chameleon/protocol.md:53)
- `Epoch_Window_Start/End` 做 freshness window
  [protocol.md:64](/Users/leask/Documents/chameleon/protocol.md:64)
- `OPEN_ACK/FIN_CHANNEL/RESET_CHANNEL/SESSION_WINDOW_UPDATE`
  [protocol.md:140](/Users/leask/Documents/chameleon/protocol.md:140)
  [protocol.md:149](/Users/leask/Documents/chameleon/protocol.md:149)
  [protocol.md:157](/Users/leask/Documents/chameleon/protocol.md:157)
  [protocol.md:191](/Users/leask/Documents/chameleon/protocol.md:191)

目前最危險的事實是：

**`design.md` 不只是落後，而是在描述另一個協議。**

修正建議：

- 立刻把 `design.md` 收斂成邊界文檔，不再保存舊 wire-level 描述
- 把所有具體字段、長度、frame 集、KDF 與錯誤矩陣完全移交給
  `protocol.md`
- 如果這一輪不清理，下一輪 review 還是會優先命中同樣問題

### 2.3 高嚴重度：Datagram channel 被宣告支持，但協議語義根本沒有定義完

`protocol.md` 在 `OPEN_CHANNEL` 中新增了 `Kind`，允許：

- `0x01 TCP Stream`
- `0x02 UDP Datagram`

[protocol.md:129](/Users/leask/Documents/chameleon/protocol.md:129)

這是很大的語義擴展，但當前規格沒有把 datagram path 補完。

具體問題：

- `DATA` frame 仍然是通用 length-prefixed payload
  [protocol.md:143](/Users/leask/Documents/chameleon/protocol.md:143)
- `FIN_CHANNEL` 對 datagram 類通道沒有明確意義
  [protocol.md:149](/Users/leask/Documents/chameleon/protocol.md:149)
- `Init Window` 與 `CHANNEL_WINDOW_UPDATE` 是 byte-credit 模型，這對
  datagram 類語義未必合適
- 沒有定義單個 datagram 的大小上限、丟棄規則、是否允許分片、
  對端如何回報 datagram path 不支持

換句話說，現在只是加了 `Kind` 欄位，沒有真正定義第二種傳輸語義。

修正建議：

- 如果近期不做 datagram，先把 `Bit 0` capability 和 `Kind 0x02`
  都標為 reserved，不要假裝它已經是正式能力
- 如果要保留 datagram，就必須新增 datagram-specific 規則：
  - 最大 datagram size
  - 是否允許 fragmentation
  - 是否需要單獨 frame type
  - flow control 是否按 byte 還是按 packet
  - datagram 通道的關閉語義

### 2.4 中高嚴重度：`OPEN_ACK` 補了，但拒絕路徑仍然不乾淨

`OPEN_ACK` 的加入是正確方向。[protocol.md:140](/Users/leask/Documents/chameleon/protocol.md:140)

但目前仍然缺少一條乾淨的 open-failure path。

現在的問題是：

- `OPEN_ACK` 只有成功確認，沒有顯式失敗回應
- 失敗似乎只能靠 `RESET_CHANNEL` 補救
  [protocol.md:157](/Users/leask/Documents/chameleon/protocol.md:157)
- 但文檔沒有定義：
  - `OPEN_CHANNEL` 尚未成功前是否允許 `RESET_CHANNEL`
  - 發起方在等待 `OPEN_ACK` 期間能否超時重試
  - 失敗是 target connect fail、policy reject 還是 address parse fail

這會導致不同實作者對「拒絕建立通道」採用不同語義。

修正建議：

- 明確規範：
  - `OPEN_ACK` 只表示成功
  - `RESET_CHANNEL` 可以用作 open failure 的負向回應
  - `RESET_CHANNEL` 的 error code 要覆蓋 open-stage failure
- 或者新增單獨的 `OPEN_NACK`

### 2.5 中高嚴重度：session-level flow control 有了 frame，但還沒有初始條件

`SESSION_WINDOW_UPDATE` 是本輪真正的進步。[protocol.md:191](/Users/leask/Documents/chameleon/protocol.md:191)

但它現在仍然不夠，因為：

- 只定義了增量，沒有定義初始 session credit
- 沒有定義 channel credit 與 session credit 的消耗順序
- 沒有定義當 session credit 歸零時，sender 的具體行為
- 沒有定義 control frames 是否消耗 session credit

如果這幾件事不寫清楚，實作者仍然可能做出完全不同的 backpressure。

修正建議：

- 在握手或 session 初始化時定義 `Initial Session Window`
- 明確規範：
  - `DATA` 同時消耗 channel credit 和 session credit
  - control frames 是否豁免
  - 哪個 credit 先檢查
  - credit 不足時的標準動作

### 2.6 中高嚴重度：`KEY_UPDATE` 仍然缺一半，尤其缺 HP key 的輪換

本輪 `KEY_UPDATE` 比上一版好很多，至少補了：

- 無 payload 的 frame
- phase 從下一個 record 起翻轉
- `Next_Traffic_Key = HKDF-Expand(Current_Traffic_Key, "rekey", 32)`

[protocol.md:198](/Users/leask/Documents/chameleon/protocol.md:198)

但這裡還有一個很大的缺口：

- record header 是用 `HP_Key` 混淆的
  [protocol.md:104](/Users/leask/Documents/chameleon/protocol.md:104)
- 可目前 rekey 只定義了 `Traffic_Key` 的輪換，沒有定義 `HP_Key`
  如何同步輪換

這意味著當 phase 翻轉後：

- payload 可能用新 key
- 但 header 可能仍然要用舊 HP key
- 或者實作者各自猜測一起輪換

這會直接導致互通性事故。

修正建議：

- 明確規範 rekey 對兩組 key 的影響：
  - traffic key
  - HP key
- 最簡單的做法是把它們放進同一個 key schedule，一次輪換兩組
- 同時補上 minimum spacing，避免連續 rekey 造成 phase 翻轉歧義

### 2.7 中嚴重度：strict lockstep 雖然簡潔，但你們還沒把代價寫出來

`protocol.md` 現在明確採用 strict lockstep：

- 不允許亂序
- 不允許跳號
- MAC fail 直接 hard close

[protocol.md:90](/Users/leask/Documents/chameleon/protocol.md:90)

這個設計不是不能接受，但它有明確代價：

- 任意一個 record 層 bug 都會直接放大成整個 session 斷線
- 沒有任何局部恢復空間
- 在多 channel 場景下，一處 record 級錯誤會把全部通道一起帶死

你們目前只寫了優點，沒寫這是刻意放棄恢復性的設計取捨。

修正建議：

- 在 `design.md` 裡誠實寫出這是 simplicity-over-recovery 的選擇
- 在 `protocol.md` 補一條非規範性說明：
  strict lockstep 的代價是 session 級故障半徑更大

### 2.8 中嚴重度：capability negotiation 還是太薄，現在只夠一版自用

`protocol.md` 現在至少開始定義 `Caps`：

- `Bit 0`: Datagram Channel
- `Bit 1-31`: 保留
- 未知 bit 必須忽略

[protocol.md:49](/Users/leask/Documents/chameleon/protocol.md:49)

這比上輪好，但還不夠支撐長期演進。

主要問題：

- 沒有 must-understand bit 類型
- 所有未知 bit 都忽略，意味著未來無法安全引入強制能力
- 沒有定義 version mismatch 優先還是 capability mismatch 優先
- 沒有定義 `Selected Caps` 在 server 端是如何計算的完整規則

修正建議：

- 將 capability 分成：
  - mandatory-understand
  - optional
- 明確 server selection 規則
- 定義 capability 不滿足時是握手失敗還是 capability 降級

### 2.9 中嚴重度：錯誤矩陣開始成形，但仍然混入了 transport-specific 描述

錯誤矩陣是本輪很對的方向。[protocol.md:207](/Users/leask/Documents/chameleon/protocol.md:207)

但它裡面仍然寫了像：

- `TCP RST`

[protocol.md:213](/Users/leask/Documents/chameleon/protocol.md:213)

這會讓 wire spec 與 transport behavior 混在一起。

如果底層是：

- TCP
- WebSocket
- QUIC stream

對應的 close semantics 完全不同。

修正建議：

- 錯誤矩陣裡只保留協議級語義：
  - silent transport close
  - authenticated session close
  - channel reset
- transport-specific close 方式留給承載適配層說明

### 2.10 中嚴重度：`PAD` 的角色已經合理，但還需要明確不參與語義

本輪 `PAD` 已經被降級為 transport policy hook，這是對的。
[protocol.md:203](/Users/leask/Documents/chameleon/protocol.md:203)

但還缺一句關鍵規則：

- `PAD` 不得影響 flow control accounting 之外的任何語義

否則實作者可能在以下兩件事上分歧：

- `PAD` 是否消耗 channel credit
- `PAD` 是否消耗 session credit

修正建議：

- 明確規範 `PAD` 只消耗 session 級傳輸預算，不屬於任何 channel
- `PAD` 不得攜帶任何可觀察語義

## 3. 開放問題

- 如果要保住 QUIC 的低 HOL 優勢，session 是否需要支援 multi-stream
  mapping？
- Datagram support 是近期目標，還是未來保留位？
- strict lockstep 是否值得為了實作簡潔而接受較大的 session 故障半徑？

## 4. 建議的下一步

下一輪我建議不要再加新能力，先做一次結構收斂，只處理四件事：

1. 把 `design.md` 清理成真正的邊界文檔
2. 決定是否接受單一 byte-stream 模型帶來的 HOL 代價
3. 把 datagram path 要么補完，要么正式降級為 future work
4. 補完 rekey 的 HP key 輪換與 flow control 的初始條件

## 5. 結論

這一輪不是沒有進步，`protocol.md` 的進展比上一輪真實得多。

但新的核心問題已經很清楚：

**你們現在的主要風險不再是“沒有協議”，而是“協議正在用單一
byte-stream 隧道模型，重新引入本來想避免的性能問題，並且設計文檔
還沒有同步到這個現實”。**

這件事如果不先處理，後面再補 frame、再補矩陣，整體架構仍然會卡在
錯誤的傳輸抽象上。
