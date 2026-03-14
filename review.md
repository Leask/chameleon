# Chameleon 設計與協議聯合審查報告
*(針對當前 `design.md` 與 `protocol.md` 的新一輪 review)*

## 0. 審查定位

本輪 review 仍然只看核心協議的可行性，不討論語言、UI 或實作框架。

本次審查的標準是：

- `design.md` 的高階設計是否與 `protocol.md` 的具體規格一致
- 兩個獨立團隊是否能按文檔實作並互通
- 協議在可靠性、效率與狀態機一致性上是否站得住
- 文檔是否真的從「概念」收斂到「規格」

## 1. 總體判斷

這一輪最大的問題不是缺新內容，而是**文檔一致性發生了明顯回退**。

你們在 `design.md` 前言裡宣稱已經修正了幾個重要問題：

- 握手模式從 `IK` 修正為單向預知 server key 的模式
- `Epoch Cert` 不再承擔時鐘校準
- 補齊 `OPEN_ACK`、`FIN`、`RESET`
- 區分 channel 級與 session 級 flow control
- 補完 rekey 與 capability negotiation

但當前正文與 `protocol.md` 並沒有把這些修正落實下去。

因此，我對本輪版本的結論是：

**它不是在穩定收斂，而是在「設計層往前走、規格層留在原地、部分段落
又回退到舊版本」的混合狀態。**

這比單純缺細節更危險，因為它會直接導致不同實作者按照不同段落實作，
最後彼此不兼容。

## 2. 最高優先級問題

### 2.1 握手模式已經自相矛盾，這是阻塞級問題

`design.md` 在演進摘要裡說已修正為「客戶端單向預知伺服器靜態公鑰」
的模式，不再使用雙向前提的 `IK`。[design.md:19](/Users/leask/Documents/chameleon/design.md:19)

但 `design.md` 的握手正文仍然明確寫的是
`Noise_IK_25519_ChaChaPoly_BLAKE2s`，並且還列出 `IK` 的訊息長度。
[design.md:41](/Users/leask/Documents/chameleon/design.md:41)

`protocol.md` 更嚴重：

- 在密碼學原語段落仍寫 `Handshake Pattern: Noise IK`
  [protocol.md:27](/Users/leask/Documents/chameleon/protocol.md:27)
- 但在握手正文又改成 `Noise NK`
  [protocol.md:40](/Users/leask/Documents/chameleon/protocol.md:40)

這不是措辭問題，而是**協議根本前提不一致**。

實際影響：

- client 是否需要 long-term static key 不明
- server 是否需要預註冊 client 身份不明
- 首包密文長度不明
- 握手 transcript 與 KDF 輸出不明

修正建議：

- 先凍結一個單一 Noise pattern
- 在 `design.md` 只保留最終模式，不要同時出現舊模式
- 在 `protocol.md` 寫出該模式對 initiator / responder 的前提、消息序列、
  固定長度與 KDF 輸出

在這件事修好之前，其他規格都不應繼續往下推。

### 2.2 握手長度定義又回到了「概念正確，規格不成立」

`design.md` 的正文仍然寫 client 首包是 `160 Bytes` 的 `IK` 訊息。
[design.md:46](/Users/leask/Documents/chameleon/design.md:46)

`protocol.md` 則改成 `NK`，但又寫：

> 此長度僅為概念計算，實際採用 `NK` 模式時，首包長度可能不同，
> 核心約束是長度必須固定不變

[protocol.md:53](/Users/leask/Documents/chameleon/protocol.md:53)

這句話本身就說明：**你們還沒有真的定義握手長度。**

如果 wire spec 裡的握手長度只是「概念上應固定」，那麼伺服器就無法
實作精確讀取，也無法實作 pre-auth 錯誤處理。

更糟的是，`protocol.md` 的錯誤處理段落仍然寫「160 Bytes (Client) /
176 Bytes (Server)」[protocol.md:211](/Users/leask/Documents/chameleon/protocol.md:211)，
顯然是舊版殘留，已與 `NK` 草案不一致。

修正建議：

- 握手消息長度必須是規範值，不允許「概念計算」
- pre-auth 錯誤處理中的長度常數必須與握手規格來源一致
- 如果長度會隨模式變化，就不能在主規範裡混用多個模式

### 2.3 `Epoch Cert` 的語義在兩份文檔裡互相打架

`design.md` 已明確聲稱：

- 不再把 `Not_Before` 視為時鐘校準工具
- 它只承擔 freshness / replay window 語義

[design.md:22](/Users/leask/Documents/chameleon/design.md:22)

`protocol.md` 也在 `Epoch Cert` 段落旁註寫了同樣意思。
[protocol.md:72](/Users/leask/Documents/chameleon/protocol.md:72)

但後面又直接寫：

- 客戶端解密後，根據 `Not_Before_UTC` 與本地時鐘計算 `Clock_Offset`

[protocol.md:86](/Users/leask/Documents/chameleon/protocol.md:86)

而且欄位名稱在前面已經從 `Not_Before_UTC / Not_After_UTC`
改成 `Epoch_Window_Start / Epoch_Window_End`，後面卻又回到舊名。
[protocol.md:64](/Users/leask/Documents/chameleon/protocol.md:64)

這說明兩件事：

- 文檔沒有收斂成單一語義
- 這一塊不是「還沒寫完」，而是不同版本內容被拼在一起了

修正建議：

- 明確二選一：
  - `Epoch Cert` 只表示 freshness window
  - 或它同時承擔時間對齊，但那就要完整定義校準模型
- 一旦選定，刪掉另一套敘述，不要保留舊文字
- 欄位名稱必須全篇一致

### 2.4 `design.md` 宣稱補齊了 channel lifecycle，但 `protocol.md` 並沒有

`design.md` 在演進摘要裡寫：

- 引入 `OPEN_ACK`
- 引入 `FIN`
- 引入 `RESET`
- 區分 channel 級與 session 級 flow control

[design.md:23](/Users/leask/Documents/chameleon/design.md:23)

但 `protocol.md` 的實際 frame 仍然只有：

- `OPEN_CHANNEL`
- `DATA`
- `WINDOW_UPDATE`
- `CLOSE_CHANNEL`

[protocol.md:127](/Users/leask/Documents/chameleon/protocol.md:127)

這代表設計層已經承認上一版 review 的批評是對的，但規格層根本沒有
跟進。

實際後果：

- 開啟方仍然無法知道通道是否建立成功
- 沒有 half-close 語義
- 正常關閉與錯誤關閉仍然混在同一個 frame
- 單通道恢復策略無法寫乾淨

修正建議：

- `protocol.md` 補上 `OPEN_ACK`
- 用 `FIN` 表達半關閉，用 `RESET` 表達錯誤終止
- `CLOSE_CHANNEL` 若保留，就必須重新定義其責任，不能再與 `FIN/RESET`
  混疊

### 2.5 flow control 仍然停留在單通道 credit，沒有落實 session 邊界

`design.md` 明確說本版已區分 channel 級與 session 級 flow control。
[design.md:23](/Users/leask/Documents/chameleon/design.md:23)

但 `protocol.md` 裡只有 per-channel 的 `Init Window` 和 `WINDOW_UPDATE`。
[protocol.md:129](/Users/leask/Documents/chameleon/protocol.md:129)

沒有任何 session-level credit / receive budget / send budget 的規格。

這不是小缺口，因為沒有 session 級限額，就意味著：

- 只靠多個 channel 疊加就可能吃光整個連線緩衝
- 你無法保證控制流不被 bulk 流壓死
- 單通道的錯誤最終還是容易上升為整個 session 的擁塞問題

修正建議：

- 加一組 session 級流控欄位或 control frame
- 定義 channel 觸頂時的局部處理方式
- 不要把 `GOAWAY 0x03` 當成單通道 overflow 的默認後果

### 2.6 `KEY_UPDATE` 仍然只是一個標記，不是完整的 rekey 協議

`design.md` 宣稱本版已經「明確金鑰輪換的相位切換狀態機」。
[design.md:24](/Users/leask/Documents/chameleon/design.md:24)

但 `protocol.md` 的 `KEY_UPDATE` 還是只有：

- `Type`
- `Next Phase`

[protocol.md:176](/Users/leask/Documents/chameleon/protocol.md:176)

沒有寫：

- 新 key 如何導出
- phase 何時生效
- 接收端接受舊 phase 的窗口
- 雙方同時發起時的衝突處理
- phase bit 翻轉後如何避免歧義

也就是說，設計層說「已完成」，規格層實際上還在起點。

修正建議：

- 把 rekey 獨立成一節
- 寫出 KDF、觸發條件、發送方切換規則、接收方重疊窗口
- 給 rekey 補一張最小狀態轉換表

### 2.7 capability negotiation 仍然只是欄位，不是規則

`design.md` 說本版已經完善 capabilities 協商。
[design.md:24](/Users/leask/Documents/chameleon/design.md:24)

實際上兩份文檔裡的 `Caps` 都還是：

- 固定 `0x00000000`
- 沒有 must-understand bits
- 沒有 optional bits
- 沒有 mismatch 行為
- 沒有 server 如何選擇或回顯的規則

[protocol.md:50](/Users/leask/Documents/chameleon/protocol.md:50)

這表示目前根本不能稱之為 negotiation。

修正建議：

- 明確 capability bit map
- 明確 unknown bit 的處理
- 明確 server response 是 echo、mask 還是 selection result
- 版本與能力不兼容的失敗模式要明確規範

### 2.8 記錄層仍然過度依賴「隱式同步」，恢復能力很弱

兩份文檔都仍然沿用隱式 sequence number：

- 不透過網路傳輸
- 每次成功處理 record 後遞增

[design.md:80](/Users/leask/Documents/chameleon/design.md:80)
[protocol.md:94](/Users/leask/Documents/chameleon/protocol.md:94)

這個設計的問題不是不能用，而是你們還沒有把它規格化到可恢復。

缺的內容包括：

- header protection key 如何導出
- seq 在哪些失敗路徑上遞增，哪些不遞增
- 當 record parse 失敗但完整性仍合法時怎麼處理
- 是否允許有限重疊窗口或永遠 lockstep

現在文檔的語氣是「天然提供絕對 replay protection」，
這個結論說得太滿了。[design.md:80](/Users/leask/Documents/chameleon/design.md:80)

修正建議：

- 把 seq / nonce / hp key 的導出放進同一節
- 用表格列出每種錯誤對 seq 的影響
- 若你們選擇嚴格 lockstep，就直接承認這是恢復性與簡潔性的取捨

### 2.9 錯誤處理語義仍然沒有收斂到單一模型

`design.md` 現在前言裡已經把 fallback 降級為部署策略，這是對的。
[design.md:21](/Users/leask/Documents/chameleon/design.md:21)

但正文裡仍然寫：

- pre-auth 失敗後，把流量原封不動代理給本機 fallback web server

[design.md:50](/Users/leask/Documents/chameleon/design.md:50)

`protocol.md` 則寫：

- pre-auth 錯誤可直接中斷連接或轉交 fallback
- record decrypt 失敗直接切底層連線
- logical error 發 `GOAWAY`

[protocol.md:211](/Users/leask/Documents/chameleon/protocol.md:211)

問題在於這三層語義還沒有變成統一矩陣：

- 哪些錯誤一定靜默
- 哪些錯誤允許控制層信令
- 哪些錯誤關閉 channel，哪些錯誤關閉 session
- 哪些錯誤是否可重試

修正建議：

- 補一張統一錯誤矩陣
- 行至少包括：
  - handshake parse/auth failure
  - record integrity failure
  - record format failure
  - channel protocol violation
  - session fatal error
- 列至少包括：
  - 是否發送 frame
  - 關閉粒度
  - 是否可重試
  - 是否依賴部署策略

### 2.10 `design.md` 和 `protocol.md` 的版本邏輯沒有同步更新

這一輪最明顯的現象是：高階設計先寫了「已修正」，正文和規格文件卻
還保留舊版內容。

這代表你們現在缺的不是某個 frame，而是**文檔版本治理本身**。

目前表現出的問題包括：

- 前言與正文衝突
- 設計文檔與規格文檔衝突
- 欄位命名更新一半
- 錯誤處理常數未同步更新

如果這種情況持續，後面就算規格本身漸漸合理，實作者也會被文檔版本
噪音拖死。

修正建議：

- 每一輪先刪舊內容，再寫新內容，不要讓新舊段落共存
- `design.md` 只保留高階結論與邊界
- `protocol.md` 承擔唯一的 wire-level 真相來源

## 3. 我建議的下一步修正方式

### 3.1 先做一次「規格凍結清理」

不要急著加新東西，先把現有文檔裡互相衝突的地方清掉。

本輪最先要清的只有四件事：

1. 最終握手模式
2. 最終握手長度
3. `Epoch Cert` 最終語義
4. channel lifecycle 最終 frame 集

這四件事如果還不鎖死，後面所有 review 都會反覆命中同樣問題。

### 3.2 `design.md` 應收斂成邊界文檔

下一版 `design.md` 不要再重複 wire-level 細節。
它應只回答：

- 協議要解什麼問題
- 不解什麼問題
- 控制面前提是什麼
- 底層承載能力如何影響協議邊界
- 哪些是核心強保證，哪些是部署層可選策略

### 3.3 `protocol.md` 應成為唯一規格來源

下一版 `protocol.md` 應承擔唯一真相來源，至少補齊：

- 單一握手模式及消息長度
- KDF 與 key schedule
- header protection key derivation
- capability negotiation 規則
- `OPEN_ACK` / `FIN` / `RESET`
- session-level flow control
- rekey state machine
- 統一錯誤矩陣

## 4. 下一版必答問題

1. 最終握手模式到底是 `IK` 還是 `NK`？
2. client 首包固定長度到底是多少？
3. `Epoch Cert` 是 freshness window，還是時間同步工具？
4. `OPEN_ACK`、`FIN`、`RESET` 是否真的進協議？
5. session-level flow control 如何表達？
6. `KEY_UPDATE` 的 phase 切換規則是什麼？
7. capability negotiation 的 bit 語義和 mismatch 規則是什麼？
8. pre-auth / post-auth / channel / session 的錯誤矩陣是什麼？

## 5. 結論

本輪最大的問題不是協議方向錯，而是**文檔已經失去單一真相來源**。

我對當前版本的直接判斷是：

**它現在最缺的不是新的設計靈感，而是一次嚴格的規格收斂。**

如果不先把 `design.md` 和 `protocol.md` 收斂到單一一致版本，
那麼你們下一輪即使補再多內容，review 也只會反覆指出同一件事：

**文檔在說兩套不同的協議。**
