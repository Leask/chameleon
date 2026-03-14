# Chameleon 設計與協議聯合審查報告
*(針對當前 `design.md` 與 `protocol.md` 的聯合 review)*

## 0. 審查定位

這次 review 不再只看單一文檔，而是把：

- `design.md` 視為設計目標、設計哲學與高階協議邊界
- `protocol.md` 視為真正約束實作者的 wire-level 規格

因此本次審查的判準不是「想法是否漂亮」，而是：

- 兩份文檔是否自洽
- 不同團隊是否能依此互通
- 協議在可靠性上是否站得住
- 協議在效率上是否不自我抵消
- 錯誤處理與演進空間是否被正確規格化

我們延續目前已形成的共識：

- 協議必須語言無關
- 協議必須有嚴格的 wire format
- session 與 channel 必須分離
- 控制面假設必須明確寫出，不得暗藏前提
- 性能應該以 goodput、延遲、恢復與可預測性為核心

## 1. 總體判斷

這一版最大的進步，是 `design.md` 已經不再停留在口號，而
`protocol.md` 也開始承擔真正的二進位規格責任。

但目前兩份文檔一起看，仍然**不能算是可實作、可互通、可長期演進
的完整協議規格**。

原因不是缺少更多功能，而是有幾個核心位置仍然同時存在：

- 設計層過度自信
- 規格層過度簡化
- 設計與規格之間存在明顯不一致

最嚴重的問題集中在五個地方：

1. 握手前提與 Noise pattern 不匹配
2. fallback / 靜默語義在不同承載上根本不成立
3. record layer 仍然不夠完整
4. channel model 還不能承載真實代理語義
5. rekey / 版本演進 / 能力協商尚未真正閉環

## 2. 高優先級問題

### 2.1 `Noise IK` 的前提被寫錯了，這不是小問題

`design.md` 與 `protocol.md` 都把核心握手定為 `Noise_IK`，並把主要
前提描述為「客戶端預先知道伺服器靜態公鑰」。

這不夠。

攻擊：

- `IK` 不是只要求 initiator 知道 responder static
- 它還要求 responder 也事先知道 initiator static
- 兩份文檔都沒有定義 client static key 如何預註冊、如何輪換、
  如何撤銷、如何與設備身份綁定
- 這意味著目前文檔對握手模式的前提描述與實際 Noise pattern
  不一致

這不是術語瑕疵，而是會直接影響：

- 是否能建立連線
- 是否支援未預註冊設備
- 控制面如何發放身份

修正建議：

- 要嘛把 `IK` 的雙向靜態前提完整寫清楚
- 要嘛選擇真正符合你們信任分發模型的握手模式
- 在 `design.md` 裡不要再把「只需 OOB 分發 server key」描述成 `IK`
  的完整前提

### 2.2 `固定長度握手 + 失敗後轉交 fallback` 在多種承載上不成立

`design.md` 第 2.1 節與 `protocol.md` 第 3.1 節都把 fallback 當成
握手失敗後的標準行為。

攻擊：

- 在 TCP 上，即使理論上可以把已讀出的 160 bytes 再轉交給別的處理
  邏輯，實作上也要求你完全控制 socket 與上游處理器的讀取邊界
- 在 WebSocket 上，所謂「轉交給本機 Web Server」根本不是同一種
  協議語義
- 在 QUIC Stream 上更不成立，因為一條 QUIC stream 不是一個可直接
  丟給通用 HTTP server 的原始 TCP 連線
- 也就是說，fallback 不是「協議層普遍成立的能力」，而是某些承載
  和某些部署拓撲下才可能成立的局部技巧

目前設計卻把它當成通用協議保證，這是錯的。

修正建議：

- fallback 必須被降級為部署策略，而不是協議核心行為
- 協議只能規定「pre-auth 失敗時不得回送協議特定錯誤」
- 不能規定「一定能把流轉交給某個本機服務」
- `design.md` 應明確把 fallback 分成：
  - 可用於特定承載 / 特定部署的可選能力
  - 無法 fallback 時的標準行為

### 2.3 `Not_Before` 不能拿來做時鐘校準

`design.md` 和 `protocol.md` 都把 `Epoch Cert` 的 `Not_Before` 當成
客戶端時計修正依據。

攻擊：

- 憑證有效期下界不是「當前時間」
- 它最多只能告訴你「現在不應早於這個值」
- 如果客戶端時間嚴重漂移，你無法從單個 `Not_Before` 推出正確 offset
- 文檔卻把它描述成「徹底解決休眠設備時間不準」

這個判斷是錯的，而且會導致實作者在會話有效期與 freshness 上做出
錯誤假設。

修正建議：

- 把時間校準與憑證有效性分開
- 如果需要 freshness，定義明確的時間來源與容忍窗口
- 如果沒有可靠時間來源，就不要把時間同步硬塞給握手證書來解決

### 2.4 record layer 仍然沒有完整閉環

雖然 `protocol.md` 已經補了 record header，但還遠稱不上完整。

攻擊：

- header 只有 `Length + Phase`，沒有 version、record type、flags
- 一旦未來需要擴展 record 語義，沒有正規的欄位空間
- `Implicit Sequence Number` 完全不傳輸，意味著雙方必須永久嚴格
  lockstep；任何一次解析錯誤都會直接導致整條連線不可恢復
- 文檔沒有定義：sequence number 在哪些錯誤路徑上遞增、在哪些路徑
  上回滾、收到亂序或重複資料時的處理
- 頭部混淆使用 `Ciphertext` 前 16 bytes 做 sample，但沒有定義當
  payload 很短時是否可跨 MAC 取樣、最小 record 大小是多少
- `Header_Protection_Key` 從哪裡來、如何與 transport key 分離、何時
  輪換，完全沒寫

這表示目前 record layer 只是半成品。

修正建議：

- 補齊 header 語義，至少預留 flags / extensibility 空間
- 明確定義 HP key derivation
- 明確定義最小 record size
- 明確定義 seq 在成功、失敗、重試路徑上的規則

### 2.5 channel model 還不夠，現在只能算半個代理協議

`protocol.md` 已經有 `OPEN_CHANNEL / DATA / WINDOW_UPDATE / CLOSE_CHANNEL`，
這比上一版好很多，但還不夠。

攻擊：

- 沒有 `OPEN_ACK`，開啟方無法知道通道究竟已成功建立，還是只是請求
  已收到
- `CLOSE_CHANNEL` 同時承擔正常關閉與錯誤關閉，缺少 half-close 語義
- 對於長連線場景，沒有單獨的 `FIN` 會讓雙向流的關閉語義很粗糙
- `OPEN_CHANNEL` 只表達目標地址，沒有明確 channel kind，無法區分
  stream 類與 datagram 類語義
- 沒有回應碼與失敗原因，客戶端只能等 `CLOSE_CHANNEL` 猜測結果

修正建議：

- 增加 `OPEN_ACK`
- 增加單獨的 `FIN` / `RESET`
- 在 `OPEN_CHANNEL` 中明確 `Channel Kind`
- 將「通道成功建立」與「通道被關閉」分成不同狀態

### 2.6 flow control 是有了，但還沒到能保證穩定的程度

現在的 credit-based flow control 只做到最表面。

攻擊：

- 只有 per-channel window，沒有整體 session 級總量控制
- 沒有定義當 credit 歸零時，sender 的具體行為
- 沒有定義是否允許一個 channel 因 overflow 直接拖死整個 session
- `GOAWAY 0x03` 用於 flow control overflow，這代表單個邏輯通道的錯
  誤可能把整個連線一起關掉，太粗暴

修正建議：

- 補 session-level flow control
- 區分 channel-level overflow 與 session-level fatal error
- 對單通道違規優先使用 channel reset，而不是 session goaway

### 2.7 `KEY_UPDATE` 幾乎還是空的

`protocol.md` 定義了 `KEY_UPDATE [Next Phase]`，但這還遠不構成可靠的
金鑰輪換機制。

攻擊：

- 沒有寫新 key 如何導出
- 沒有寫 phase 切換點是「發出此 frame 後」還是「下一個 record 起」
- 沒有寫接收端允許舊 phase 的重疊窗口
- 沒有寫雙方同時觸發更新時如何處理
- 沒有寫 phase bit 重複翻轉後如何避免歧義

在當前規格下，只要雙方切換時機略有分歧，就會直接進入 AEAD 解密
失敗，最後連線被硬切。

修正建議：

- 下一版必須為 rekey 寫獨立狀態機
- 明確定義 KDF、trigger、overlap window、雙方同時更新規則
- `KEY_UPDATE` 不能只是一個 1-byte hint

### 2.8 設計層聲稱「極致效能」，但規格層還沒有支持這個結論

`design.md` 把「極低開銷、絲滑體感、高可用自癒」寫得很滿。

攻擊：

- 當前規格仍然跑在可靠位元組流之上，因此底層若是 TCP / WebSocket，
  協議天然繼承 HOL
- 現有 `OPEN_CHANNEL`/`DATA`/`WINDOW_UPDATE` 組合並不足以支撐真正細緻
  的優先級與公平排程
- 沒有 transport capability matrix，卻在高階設計裡對「IP 切換」、
  「高丟包自癒」做了接近通用性的宣稱

這會讓設計層承諾了協議本身根本無法保證的性能。

修正建議：

- 在 `design.md` 明確區分：
  - 協議本身保證的東西
  - 依賴底層承載才成立的東西
- 不要把 QUIC transport 的特性直接寫成 Chameleon 協議的普遍特性

### 2.9 `PAD` 已降級為策略，但文檔語氣還沒有完全跟上

這一版已經把 bucketing 拿出核心協議，這是對的。

攻擊：

- `protocol.md` 仍然在 frame 規格裡暗示「發送方可將多個 frame 打包
  進同一個 record 以對齊特定的 Bucket 尺寸」
- 這會讓實作者誤以為 bucket 仍然是規範主路徑，而不是可選 transport
  policy

修正建議：

- 在 `protocol.md` 裡把 `PAD` 明確標註為 optional policy hook
- 把 bucket 相關描述從規範性文字改成非規範性說明

### 2.10 版本與能力協商還沒有真的落地

兩份文檔都提了 `Version` 和 `Capabilities`，但現在這些欄位還只是空殼。

攻擊：

- `Caps` 目前固定為 `0x00000000`
- 沒有定義 server response 中哪些 capability 是 echo、哪些是選擇結果
- 沒有 downgrade / mismatch 規則
- 沒有未知 capability bit 的處理方式
- 沒有規定未來新增 frame type 時，舊實作如何降級

修正建議：

- 把 capability negotiation 從「欄位存在」提升到「規則存在」
- 明確寫出：
  - must-understand bits
  - optional bits
  - unknown bits 的處理
  - version mismatch 的失敗模式

### 2.11 錯誤處理在兩份文檔裡存在語義衝突

`design.md` 對 pre-auth 錯誤強調靜默與 fallback，
`protocol.md` 對 record decryption failure 要求直接切底層連線。

攻擊：

- 在 TCP 上的 `RST`、在 QUIC 上的 `CONNECTION_CLOSE`，都不是「什麼都
  沒發生」
- 如果你們真的把 `record decryption failure` 視為高風險探測面，
  那麼底層關閉方式本身也屬於可觀察語義
- 目前兩份文檔沒有把「pre-auth 靜默」「post-auth 顯式錯誤」
  「record 失敗時的 transport close semantics」分成一套一致模型

修正建議：

- 在 `design.md` 裡明確切出三類錯誤：
  - handshake parse/auth failure
  - record integrity failure
  - post-auth logical failure
- 在 `protocol.md` 對應規定每一類錯誤的標準關閉行為

## 3. 我建議的修正方向

### 3.1 把 `design.md` 從「主張」改成「邊界與責任」

下一版 `design.md` 不應繼續重複「極致、高效、零特徵」這種強結論，
而應更精確地寫三件事：

- 協議核心保證什麼
- 控制面前提是什麼
- 哪些能力取決於特定承載或特定部署

這樣 `protocol.md` 才不會被迫背負設計層過度承諾。

### 3.2 把 `protocol.md` 補成真正的閉環協議

下一版 `protocol.md` 至少要補齊以下幾塊：

- 握手模式前提
- HP key derivation
- record 最小長度與錯誤路徑
- `OPEN_ACK`
- `FIN` / `RESET`
- session-level flow control
- rekey state machine
- capability negotiation rules

### 3.3 給錯誤處理一個統一模型

不要再用分散在設計文案和協議條文裡的描述來定義錯誤。

應該直接列一張矩陣：

- 錯誤類型
- 所在層
- 是否可觀測
- 是否發送控制 frame
- 關閉粒度是 channel 還是 session
- 是否允許重試

### 3.4 下一版應補一張真正的狀態機圖

我不是要抽象狀態名稱，而是要真正有事件、轉移與動作的表。

至少要有：

- handshake state machine
- record key phase machine
- channel lifecycle machine
- goaway / draining machine

沒有這四張表，協議仍然只是在靠文字敘述維持一致性。

## 4. 我認為下一版必須回答的問題

1. 你們真的需要 `IK`，還是只是需要「client 預知 server key」？
2. fallback 是協議能力，還是部署能力？
3. `Epoch Cert` 到底承擔的是 freshness、有效期，還是時間同步？
4. header protection key 從哪裡導出？
5. record sequence 在哪些錯誤路徑上遞增？
6. channel 成功建立怎麼回報？
7. 單通道失控時，為什麼要拖垮整個 session？
8. `KEY_UPDATE` 的 phase 切換如何避免雙方不同步？
9. `Capabilities` 的 bit 語義與 mismatch 規則是什麼？
10. 哪些性能特性是協議保證，哪些只是某個 transport 的特性？

## 5. 結論

這一版的真實進步，在於你們終於開始同時寫「設計層」與「規格層」。
這是對的。

但我對目前版本的最直接判斷仍然是：

**它已經比上一版更像協議了，但它還沒有到「兩個獨立團隊按規格寫，
就一定能互通且行為一致」的程度。**

當前最需要補的不是更多新點子，而是把以下幾個位置徹底做實：

- 握手模式與控制面前提
- fallback 的適用邊界
- record layer 的完整性
- channel lifecycle
- rekey 規則
- version / capability negotiation

把這六件事補齊，下一版才有資格從「像協議」走到「真協議」。
