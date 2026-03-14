# Chameleon 協議審查報告
*(針對當前 `design.md` 的核心協議可行性 review)*

## 0. 審查定位

這份 review 不討論語言、UI、工程框架或部署細節，只討論當前
`design.md` 裡的核心協議是否：

- 能被不同實作者獨立實作並互通
- 在線上格式上自洽
- 在效率上站得住
- 在可靠性上可恢復
- 在狀態機上可推理

這份 review 也沿用目前已形成的共識：

- 協議本身必須語言無關
- 協議必須有明確的 wire format
- 性能應以 goodput、延遲與恢復為核心
- 控制面與資料面應清晰分離
- 狀態機必須覆蓋正常路徑與錯誤路徑

## 1. 總體判斷

`design.md` 的方向比早期版本成熟很多，因為它終於回到了真正的協議
問題：握手、幀格式、狀態與多路復用。

但它目前仍然**不是一份可直接實作的協議規格**。

最大的原因不是缺細節，而是幾個核心結構還沒有被定義完整：

- 握手訊息如何定界
- 記錄層如何封裝
- 會話與通道的邊界在哪裡
- 重放保護靠什麼資料結構實作
- 多路復用如何做流控與錯誤處理
- 底層承載與上層 session 的責任如何分割

也就是說，v7 的問題不是「想法不夠多」，而是「關鍵抽象還沒鎖死」。

## 2. 高優先級攻擊點

### 2.1 握手沒有定界，協議就無法可靠解析

`design.md` 在握手段寫了「沒有長度前綴，長度由底層 Stream 決定」。
這在可靠位元組流上是站不住的。

攻擊：

- 在 TCP / WebSocket 這類位元組流裡，接收方不知道該讀多少 bytes
  才能判定一個完整的握手訊息
- 如果握手消息本身有可變長 payload，就更無法只靠底層 stream 邊界
  來解析
- 一旦服務端已經讀掉部分 bytes，就不可能再把這個 stream「轉交」
  給別的 fallback 協議處理器，因為前面的資料已經消耗掉了

修正建議：

- 握手必須有明確的 record envelope
- 無論是否使用 Noise，外層都要先有可解析的訊息邊界
- fallback 決策必須發生在消耗應用層 bytes 之前，或根本不應依賴
  同一條 stream 的協議轉交

### 2.2 `Noise IK + 零標識 + 靜默回退` 這組合在工程上不自洽

當前設計想同時得到：

- 無協議標識
- 可直接用 Noise IK 做首包
- 解不開時靜默回退

攻擊：

- `IK` 代表客戶端事先知道伺服器靜態公鑰，這不是小前提，而是巨大
  的控制面前提
- 如果握手沒有外層 envelope，服務端就無法可靠知道「讀到哪裡」
- 所謂靜默回退，在已讀取隨機 bytes 之後幾乎不可能對同一條位元組流
  做出一個正常協議的完整回應

修正建議：

- 不要把「無標識」和「不可解析」混成同一件事
- 握手需要明確定界與明確所有權
- 如果保留 Noise，應先寫清楚 server key 的分發、輪換與失效模型

### 2.3 `Server-Assisted Epoch` 設計既昂貴又不嚴謹

當前文檔把時間戳、簽名與 session lifetime 放進握手回應中。

攻擊：

- Ed25519 簽名不是 32 bytes，而是 64 bytes，文檔本身就已經不一致
- 每次握手都帶一個控制面簽名，會把資料面熱路徑和控制面信任綁得
  過緊
- 單靠「簽過的絕對時間」不能解決完整的 freshness 問題
- 沒有定義允許的時鐘偏差、過期容忍窗、重放窗口與離線行為

修正建議：

- 不要在每次握手熱路徑都附加控制面時間簽名
- 把時間校準、憑據有效期與 session lifetime 分離
- 如果需要 freshness，應明確定義 nonce、窗口與憑據失效規則

### 2.4 記錄層其實還沒被定義

文檔定義了 TLV frame，但沒有真正定義 record layer。

攻擊：

- 目前只有 `Type(1) + Length(2) + Payload`，這只是明文 frame 結構，
  不是加密記錄結構
- 缺少 sequence number，重放保護無從實作
- 缺少 key phase 或 epoch，rekey 後無法知道該用哪把鑰匙解密
- 缺少 record length，位元組流上根本無法切 record
- 文中提到「3 bytes 的加密頭 / MAC」，這不符合現代 AEAD 的基本現實

修正建議：

- 明確區分 `record` 與 `frame`
- record header 至少要能表達：長度、序號、flags、key phase
- frame 才是解密後的多路復用與控制單元

### 2.5 `DATA` frame 缺少通道語義，實際上無法承載代理工作負載

目前 `DATA` frame 的定義是「承載實際用戶數據」。

攻擊：

- 如果這是一個通用隧道協議，那麼接收端怎麼知道這段 data 屬於哪種
  上層語義
- 如果底層沒有原生多路復用，只靠一個 4-byte Stream ID 不夠，因為
  你還缺少：
  - 通道建立
  - 通道關閉
  - 通道類型
  - 目標位址或會話 metadata
  - 初始 flow control credits

修正建議：

- 補一個 `OPEN` 類 frame
- `DATA` 只負責已有通道內的資料
- `FIN`、`RESET`、`WINDOW_UPDATE` 需要獨立定義

### 2.6 `每個 QUIC Stream 都是一個獨立會話` 是嚴重的效率錯誤

文檔在多路復用部分提出：若底層承載支援 stream，就讓每個 stream
內部都是一個獨立的 Chameleon 會話。

攻擊：

- 這會讓每條邏輯流都重新握手，產生不必要的 CPU 與 RTT 成本
- 這會把 session state 數量直接放大到通道數量級
- 這會讓 rekey、draining、migration 幾乎無法作為「連線級」能力
  來處理
- 這其實不是消除 HOL，而是把 session 管理碎裂化

修正建議：

- 一條底層 transport connection 之上只建立一個 Chameleon session
- 在這個 session 內部承載多個 logical channels
- 底層若有原生 streams，可以作為 channel 映射優化，但不應成為
  session 邊界

### 2.7 `REKEY_REQ / REKEY_ACK` 不足以構成可用的 rekey 協議

當前文檔只列出兩個 rekey frame。

攻擊：

- 沒有定義新鑰匙如何導出
- 沒有定義誰先切相位
- 沒有定義舊 record 可接受多久
- 沒有定義 ACK 丟失時怎麼辦
- 沒有定義雙方同時發起 rekey 的衝突處理

修正建議：

- rekey 必須有明確 state machine
- 每個 record 要帶 key phase
- 新舊 key 的重疊接受窗口要明確
- rekey 最好依附在單調遞增序號或 epoch 上，而不是只靠一來一回訊息

### 2.8 `Silent Drop` 全做到底，會把可靠性與可診斷性一起殺掉

文檔主張對所有密碼學錯誤直接靜默丟棄。

攻擊：

- 在未建立會話之前，這可能讓客戶端完全無法區分：
  - 對端不存在
  - 網路中斷
  - 憑據錯誤
  - 版本不相容
- 在已建立會話之後，完全沒有 authenticated error signal，會讓
  rekey、draining、migration 的錯誤恢復變得非常難做

修正建議：

- pre-auth 可以保守，但 post-auth 應有最小必要的 authenticated
  control/error frames
- `PING` 不應承載 draining 等狀態信令，應有專用 `GOAWAY` 或
  `DRAIN` 類 frame

### 2.9 `Deterministic Bucketing` 不應寫死在核心協議裡

文檔把裝箱與填充提升到協議本體。

攻擊：

- 這會把帶寬開銷、封包等待時間與互動延遲直接寫進協議成本
- 在可靠位元組流之上，record size 並不等於真實網路 packet size，
  因此 bucket 的效果與代價都會依底層承載而變
- 固定 bucket 集合會讓協議在不同 MTU、不同 transport 上都不夠彈性

修正建議：

- bucket / padding 應降級為 transport policy
- 它應該是可關閉、可調參、可按通道類型決定的外掛能力
- 核心協議只應保留「可承載 padding frame」的能力，不應綁死策略

### 2.10 缺少版本治理與能力協商

當前設計沒有定義版本與能力協商。

攻擊：

- 如果未來新增 frame type，舊節點如何處理
- 如果某個 transport 不支持 migration 或不支持 channel priority，
  雙方如何降級
- 如果 client 和 server 支持不同的 bucket / rekey / multiplex 子集，
  怎麼互通

修正建議：

- 握手必須導出一份 negotiated capabilities
- frame type 要保留擴展空間
- 版本不相容要有清晰的失敗模式

## 3. 我建議的修正方向

### 3.1 重新劃分四個協議層

下一版設計不應再把所有東西混在「握手 + TLV」裡，而應清楚拆成：

1. Handshake Layer
2. Session Record Layer
3. Channel Multiplex Layer
4. Control Signaling Layer

這四層的責任分別是：

- Handshake：身份、密鑰交換、能力協商
- Record：定界、序號、加密與 key phase
- Channel：open/data/window/fin/reset
- Control：ping/goaway/rekey/telemetry hints

### 3.2 下一版握手至少要明確這幾件事

- 握手訊息如何定界
- server static key 如何分發與輪換
- capabilities 如何協商
- resumption 是否存在，若存在邊界在哪裡
- time / freshness 問題到底由誰負責

沒有這五件事，握手就仍然只是概念。

### 3.3 下一版 record header 應至少有這些欄位

不一定非要用這個位寬，但語義上至少要有：

- version
- record_type
- flags
- key_phase
- sequence_number
- ciphertext_length

否則：

- 無法在 byte stream 上正確切 record
- 無法做 replay window
- 無法做 rekey overlap
- 無法做未來擴展

### 3.4 下一版 channel model 應至少有以下 frame

- `OPEN`
- `OPEN_ACK`
- `DATA`
- `WINDOW_UPDATE`
- `FIN`
- `RESET`
- `PING`
- `PONG`
- `GOAWAY`
- `KEY_UPDATE`

這樣通道層與控制層的責任才會分開。

### 3.5 會話邊界要明確固定

我建議直接採用：

- 一條 transport connection
- 一個 authenticated session
- 多個 logical channels

這個模型的好處是：

- rekey 有明確作用域
- migration 有明確作用域
- draining 有明確作用域
- 上下行 flow control 更容易做總量與分類控制

### 3.6 把 bucket / padding 從協議核心移出去

下一版不應再把 bucket size 當協議常數。

更合理的做法是：

- 協議允許 padding frame
- transport policy 決定何時聚合、何時補齊、何時完全不做
- 互動流和 bulk 流應可採用不同策略

這樣才不會把延遲成本永久寫死在 wire spec 裡。

## 4. 我認為下一版 `design.md` 必須回答的問題

如果下一版要真正進入可實作區間，至少要把以下問題寫死：

1. 握手 record 的確切定界方式是什麼？
2. 一條 transport connection 上有幾個 Chameleon session？
3. replay window 用什麼欄位驅動？
4. rekey 的 key phase 切換規則是什麼？
5. logical channel 的 open / close / reset / flow control 怎麼做？
6. capabilities 在握手中怎麼協商？
7. 錯誤在哪個層級靜默，哪個層級必須顯式？
8. 哪些行為是核心協議，哪些只是可選 transport policy？

## 5. 結論

當前 `design.md` 的價值在於，它第一次真正把問題收斂到了協議核心。

但我對它的最嚴厲判斷是：

**v7 目前還是一份「有協議感的設計文檔」，還不是一份可以交給兩個
獨立團隊各自實作並保證互通的協議規格。**

真正阻礙它落地的，不是缺少更多想法，而是缺少幾個不能含糊的結構：

- 握手定界
- record 層
- channel 語義
- rekey 狀態機
- 版本與能力協商

把這五件事補齊，下一版才有資格討論「完整性」。
