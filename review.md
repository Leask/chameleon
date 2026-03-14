# Chameleon 聯合審查報告
*(本輪重點：承認已修掉的大洞，轉向更細的控制面、record 完整性、pool 契約與長連線語義)*

## 0. 總體判斷

這一輪的質量比前幾輪高很多。你們不只是修文案，而是真的開始把核心閉環補上：

- `NK` 的前置公鑰前提被正式承認
- 兩級 PKI 已經進入設計與協議
- `Epoch Cert` 已開始有 verifier algorithm
- `Auth Scheme 0x01` 已有 domain separation
- `Channel ID` exhaustion 已開始被當作長連線問題處理

所以這一輪我不再重複打那些已經收斂的舊點。我只保留當前版本仍然會阻塞
「穩定實作、可運維、可規模化」的問題，而且每個點都直接給出修改方向。

## 1. 阻塞級問題

### 1. `Epoch Cert` 的兩級 PKI 方向對了，但 `CPSK` 鏈條還沒有被定義成真正可運作的物件

#### 問題本身

你們這一輪正式引入了兩級 PKI：

- `Offline Root CA`
- `Online CPSK`
- `Epoch Cert` 由 `CPSK` 簽發

[design.md:42](/Users/leask/Documents/chameleon/design.md:42)
[protocol.md:111](/Users/leask/Documents/chameleon/protocol.md:111)

這是正確方向，我接受。

但當前規格仍然只定義了 `Epoch Cert`，沒有真正定義
`CPSK Certificate` 自己是什麼東西。現在 verifier algorithm 只說：

- client OOB 拿到 `CPSK` 憑證
- 用 `Signer Key ID` 找對應 `CPSK`

[protocol.md:114](/Users/leask/Documents/chameleon/protocol.md:114)

問題是，這還不是一個可互通的鏈。缺的內容至少包括：

- `CPSK Certificate` 的 wire / bundle 格式
- `CPSK` 有效期欄位
- `Root` 對 `CPSK` 的簽名內容
- `Signer Key ID` 的生成與唯一性規則
- 多把 `CPSK` 同時有效時如何選擇
- `CPSK` 輪換與 overlap window

#### 為什麼這會卡住實作

如果這些不定死，client 端實際上無法做出一致的 signer lookup：

- 有的實作會把 `Signer Key ID` 當隨機整數
- 有的會當公鑰 hash 截斷
- 有的會只保留一把當前 `CPSK`
- 有的會保留 current + next

結果就是同一份 `Epoch Cert`，不同客戶端可能恢復策略都不一樣。

#### 建議怎麼改

最直接的做法是新增一個 OOB 物件定義，哪怕不放進 `protocol.md`，
也至少應在設計層補出一個正式控制面規格骨架。建議定義：

`CPSK Certificate`
- `Signer Key ID`
- `Signer Public Key`
- `NotBefore`
- `NotAfter`
- `Signature by Root`

同時寫死：

- `Signer Key ID` 是控制面分配的 64-bit 唯一值，還是公鑰 hash 的截斷值
- 若使用截斷值，碰撞處理規則是什麼
- client OOB bundle 至少必須同時攜帶 `current + next CPSK`
- `Epoch Cert` 驗證時若 `Signer Key ID` 不存在於本地 bundle：
  恢復策略必須是 `Config Refresh Required`

如果你們想減少之後來回，我建議下一輪直接新增一份
`control_plane_bundle.md` 或在 `design.md` 補一節 `OOB Trust Objects`。

### 2. `Node Manifest` 被提出來了，但還沒有成為可執行的控制面契約

#### 問題本身

`design.md` 已經把裸公鑰預分發升級成 `Node Manifest`，這是對的：

[design.md:47](/Users/leask/Documents/chameleon/design.md:47)

但現在它還是概念詞，不是契約。缺少的關鍵點是：

- manifest 的最小欄位集合
- manifest 的簽名方式
- manifest 的版本號與過期時間
- region / route group 的語義
- current / next key 的切換窗口
- 若 manifest 與 `Epoch Cert` 對同一 `Node ID` 給出不一致 key，如何處理

#### 為什麼這是核心問題

你們現在已經把 `Session Pool` 引進來了。只要 pool 存在，manifest 就不只是
「客戶端拿到一把 key」這麼簡單，而是會直接影響：

- 新 session 預熱
- key rollover
- route failover
- 連線池內 session 的新舊世代並存

如果 manifest 沒有正式契約，pool manager 的行為一定漂移。

#### 建議怎麼改

把 `Node Manifest` 至少定義成以下欄位：

- `Manifest Version`
- `Node ID`
- `Route Group`
- `Current Static Pubkey`
- `Current NotBefore`
- `Current NotAfter`
- `Next Static Pubkey` 可選
- `Next NotBefore`
- `Next NotAfter`
- `Priority`
- `Weight`
- `Signer Key ID`
- `Manifest Signature`

同時寫三條硬規則：

1. client 對同一 `Node ID` 只能接受單一 `Current` key
2. `Next` key 只能在 overlap window 內使用於新建 session
3. 若 `Epoch Cert` 公鑰不屬於 manifest 中的 `Current/Next` 任一值，
   必須視為配置失配，不得默默接受

### 3. Record layer 仍然缺最後一步：`Plain_Header` 沒有被明確納入 AEAD AAD

#### 問題本身

你們這一輪沒有修這個點，record layer 仍然只說：

- header 被 HP 混淆
- ciphertext 做 AEAD 解密

[protocol.md:134](/Users/leask/Documents/chameleon/protocol.md:134)
[protocol.md:153](/Users/leask/Documents/chameleon/protocol.md:153)

但沒有明確說：

- `Plain_Header = Length || Flags`
- 是否作為 AEAD 的 `AAD`

#### 為什麼這一點不能再拖

如果 header 不進 AAD，你們現在的 record integrity 實際上是不完整的：

- `Key Phase` 只被混淆，沒被正式認證
- `Length` 的篡改處理只能依賴 parser 行為和 AEAD fail 的副作用
- 不同實作會在 header 損壞時產生不同的錯誤分支

這會直接破壞你們想要的「不同團隊盲寫也能互通」。

#### 建議怎麼改

直接把 record seal/open 流程寫成規範算法：

發送方：
1. 構造 `Plain_Header = Length || Flags`
2. 以 `AAD = Plain_Header` 執行 AEAD seal
3. 取 `Sample`
4. 產生 `Mask`
5. 輸出 `Obfuscated_Header = Plain_Header XOR Mask`

接收方：
1. 讀 `Obfuscated_Header`
2. peek `Sample`
3. 反混淆得 `Plain_Header`
4. 驗證 `Length/Flags`
5. 讀完整 `Ciphertext`
6. 以 `AAD = Plain_Header` 執行 AEAD open

同時補這幾條硬規則：

- `Length < 16` 或 `Length > 16384` -> `Abort Transport`
- `Flags` 的保留位非 0 -> `GOAWAY 0x02` 或直接 `Abort Transport`
  你們必須二選一，不要留白
- `Key Phase` 非預期翻轉 -> 進入 rekey 驗證分支，失敗則完整性錯誤

### 4. `Session Pool` 已經從口號變成方向，但仍然沒有足夠的最小契約

#### 問題本身

這一輪 `design.md` 已經比上一輪進步，至少不再說「完美解決 HOL」，
而改成「顯著緩解」，並補了最小池化契約的輪廓：

[design.md:27](/Users/leask/Documents/chameleon/design.md:27)
[design.md:28](/Users/leask/Documents/chameleon/design.md:28)
[design.md:30](/Users/leask/Documents/chameleon/design.md:30)

這個修正我接受。

但現在的 pool contract 仍然太抽象，最少還缺：

- `Min/Max Sessions` 是按 device class、route class，還是全局策略
- `interactive` / `bulk` 的分類標準是誰決定
- session health 的量化指標
- replenishment 何時觸發
- draining session 是否還允許接新 `OPEN_CHANNEL`
- pool manager 遇到 `GOAWAY 0x00`、epoch expiring、Channel ID watermark
  時的統一行為

#### 為什麼這一塊現在必須補細

因為你們已經把它提升到核心架構層面了。現在它不再是「可以之後由各實作自由發揮」
的東西，而是直接決定：

- 高延遲流量的實際表現
- 長連線補池時的穩定性
- key rollover 時會不會抖

#### 建議怎麼改

我建議你們下一輪至少把以下內容寫成正式條款：

`Session Class`
- `interactive`
- `bulk`
- `spare`

`Placement`
- 新 channel 一經分配不可遷移
- `interactive` 預設不得落在已有 bulk 壓力的 session 上

`Health Signals`
- `authorized`
- `draining`
- `available_session_credit`
- `pending_open_count`
- `recent_ping_rtt_ms`
- `channel_id_headroom`

`Replenishment Triggers`
- session 收到 `GOAWAY`
- session 進入 `draining`
- `channel_id_headroom` 低於門檻
- session credit 長期不足

`Admission Rule`
- draining session 絕對不得接受新 `OPEN_CHANNEL`

如果你們不想把這些寫進 `design.md`，那就至少新增一份
`pooling.md`。否則「會話池化」仍然只是討論概念。

### 5. `Channel ID` exhaustion 現在只處理了 client 半邊，而且 watermark 寫法還不夠嚴謹

#### 問題本身

你們這一輪新增了：

- `Channel ID` 單調遞增
- 最大值 `2^30 - 1`
- client 在 `2^30 - 100` 時主動 `GOAWAY`

[protocol.md:189](/Users/leask/Documents/chameleon/protocol.md:189)
[protocol.md:192](/Users/leask/Documents/chameleon/protocol.md:192)

方向是對的，但現在還有三個缺口：

1. 只寫了 client，沒寫 server
2. `2^30 - 100` 沒說明是否考慮 parity
3. 沒定義收到 exhaustion-triggered `GOAWAY` 後，
   pool manager 與 peer 的具體行為

#### 為什麼這會形成分歧

如果 server 也允許發起 channel，那 exhaustion 就是雙向問題，
不能只定義 client。

另外，因為 ID 有奇偶分配，真正的本地方向可用 headroom
應該按 parity 計，而不是簡單用整體最大值。

#### 建議怎麼改

把這條改成雙向規則：

- 每個方向維護自己的 `next_local_channel_id`
- 定義 `CHANNEL_ID_HIGH_WATERMARK`
  為本地方向可用 ID 空間的門檻，而不是全局整數字面值

建議寫成：

- client 若 `next_local_channel_id > CLIENT_CHANNEL_ID_HIGH_WATERMARK`
  -> 發 `GOAWAY 0x00`
- server 若 `next_local_channel_id > SERVER_CHANNEL_ID_HIGH_WATERMARK`
  -> 也同樣處理

並補：

- exhaustion 型 `GOAWAY` 收到後，對端不得再在該 session 建立新 channel
- pool manager 必須立即補一條 warm replacement session

### 6. `CLIENT_AUTH` 已經接近可互通，但還缺兩個會在擴展時重新炸開的邊角

#### 問題本身

`Auth Scheme 0x01` 現在已有：

- `Credential ID`
- `Proof Length = 64`
- `Sign("chameleon-auth-v1" || h)`

[protocol.md:172](/Users/leask/Documents/chameleon/protocol.md:172)
[protocol.md:176](/Users/leask/Documents/chameleon/protocol.md:176)

這一塊比之前強很多，我接受這個修正。

但還缺兩條會在你們未來擴展 scheme 時立刻回來咬人的規則：

1. 不支援的 `Auth Scheme` 怎麼處理，還是沒寫
2. `Proof Length` 只對 `0x01` 固定，沒有全局上限

#### 為什麼要現在補

因為現在你們的 provisional state 已經是 hard-bounded 的。
如果未來引入新的 scheme，但沒有全局 `Proof Length` 上限，
那 provisional resource boundary 會再次被打開。

#### 建議怎麼改

補兩條硬規則：

- `unsupported Auth Scheme` -> `GOAWAY 0x02 Protocol Violation`
- `supported scheme but invalid credential/proof` -> `GOAWAY 0x05 Auth Failed`

再加一條全局限制：

- 對所有 auth schemes，`Proof Length <= MAX_AUTH_PROOF_LEN`
  例如 `1024`

如果未來某個 scheme 需要更大 proof，就明確升版本，而不是悄悄擴大 provisional 面。

### 7. `Max Inbound Bytes` 仍然寫成「包含底層協議頭」，這和 transport-agnostic 目標衝突

#### 問題本身

`protocol.md` 仍寫：

- `Max Inbound Bytes = 2048 Bytes (包含底層協議頭)`

[protocol.md:170](/Users/leask/Documents/chameleon/protocol.md:170)

這條和你們自己強調的 language-agnostic / transport-agnostic 邏輯衝突。

TCP、QUIC Stream、WebSocket 的「底層協議頭」：

- 不屬於同一層
- 也不一定是應用層可見

#### 為什麼這會造成實作漂移

不同實作會用不同觀測面去算這 2048：

- 有的只算 Chameleon bytes
- 有的算承載封裝後 bytes
- 有的算 socket read bytes

最後 provisional close 行為一定不一致。

#### 建議怎麼改

把這條改成：

- `Max Inbound Chameleon Bytes`

只計：

- record header
- ciphertext
- decrypted frames

如果你們真的需要限制更底層成本，應寫在 transport binding 文檔裡，
而不是核心規格。

### 8. `KEY_UPDATE` 還沒變成完整 state machine，長壽命 session 會在這裡漂

#### 問題本身

目前你們定義了：

- 收到 `KEY_UPDATE`
- 下一個 record 切 `Key Phase`
- 用新的 traffic / HP keys
- rekey 間隔至少 `1000 records` 或 `1 minute`

[protocol.md:261](/Users/leask/Documents/chameleon/protocol.md:261)

這仍然不夠，少的是：

- 每個方向的 generation state
- 舊 key 何時退休
- unexpected `Key Phase` 的處置
- simultaneous rekey 的獨立演進
- 最大 key age / 最大 key bytes

#### 為什麼這一塊變得更重要

你們現在正式引入 `Session Pool`，而 pool 代表 session 會更長壽。
一旦 session 壽命拉長，rekey 不再是小概率路徑，而是常態。

#### 建議怎麼改

下一輪至少補一個 state machine：

每方向本地狀態：
- `current_generation`
- `next_generation_prepared`
- `current_key_phase`

規則：
1. 收到 `KEY_UPDATE` frame 只準備 next keys，不立即切換
2. 收到 `Key Phase` 翻轉 record 才提交 generation
3. 若翻轉時無 prepared next keys -> 完整性錯誤
4. 舊 generation 在新 generation 首個成功 decrypt 後立即退休

同時再補一條 rotation policy：

- 除最小 rekey 間隔外，還要有最大 key age 或最大 encrypted bytes 門檻

### 9. Recovery matrix 仍然太粗，還不夠支撐自動化恢復

#### 問題本身

你們有了 recovery matrix，這是正向進展：

[protocol.md:292](/Users/leask/Documents/chameleon/protocol.md:292)

但它仍然過於一行一個方向，沒有細到能直接被實作成 client policy：

- `Handshake AEAD / MAC Fail -> Retry Different Node`
- `Unknown Frame Type -> Retry Same Node`
- `Session Flow Control Overflow -> Retry Same Node`

[protocol.md:296](/Users/leask/Documents/chameleon/protocol.md:296)
[protocol.md:302](/Users/leask/Documents/chameleon/protocol.md:302)
[protocol.md:303](/Users/leask/Documents/chameleon/protocol.md:303)

這些都還是過粗判斷。

#### 為什麼不夠

因為不同錯誤根因其實會共享同一表象：

- stale config
- signer rotation
- node rollover
- capability mismatch
- protocol bug

如果 recovery matrix 不細分，client 最終還是只能靠 heuristics 臨場發揮。

#### 建議怎麼改

把矩陣拆成多欄：

- `Server Action`
- `Client Immediate Action`
- `Retry Same Node?`
- `Retry Different Node?`
- `Config Refresh Required?`
- `Backoff Required?`

並新增至少這些缺失條目：

- `Epoch Cert Signature Fail`
- `Epoch Cert Signer Key Unknown`
- `Epoch Cert Pubkey Mismatch`
- `Epoch Cert NotYetValid`
- `Epoch Cert Expired`
- `Unsupported Auth Scheme`
- `Provisional Timeout`
- `Provisional Byte Overflow`
- `Channel ID Exhaustion`

## 2. 對本輪設計修改的回應

### 1. 兩級 PKI

我接受這個修正方向。

這一輪最大的正向變化，就是你們終於把離線 Root 和線上短期簽發的矛盾正面收掉了。
現在剩的不是方向錯，而是鏈條物件還不夠細。

### 2. `Node Manifest`

我接受這個修正方向。

把預分發物件從「裸公鑰」升級成 manifest，是正確的規模化思路。
現在只差把 manifest 從概念名詞補成正式控制面契約。

### 3. `Session Pool`

我接受你們把「完美解決 HOL」收斂成「顯著緩解」。

[design.md:27](/Users/leask/Documents/chameleon/design.md:27)

這比上一輪更誠實。但你們現在還沒有最小可實作契約，所以它仍然不能算完成。

### 4. `Auth Scheme 0x01` 的 domain separation

我接受，這一條已經明顯改善：

[protocol.md:175](/Users/leask/Documents/chameleon/protocol.md:175)

這意味著我上一輪對 raw transcript hash 直接簽名的質疑，現在可以正式降級。

## 3. 已被合理解決或明顯改善的問題

以下幾個舊點這一輪可以正式降級：

- 兩級 PKI 已正式進設計與規格
  [design.md:44](/Users/leask/Documents/chameleon/design.md:44)
  [protocol.md:114](/Users/leask/Documents/chameleon/protocol.md:114)
- `Epoch Cert` 已加入 `Signer Key ID`
  [protocol.md:85](/Users/leask/Documents/chameleon/protocol.md:85)
- `Auth Scheme 0x01` 已有 domain separation
  [protocol.md:175](/Users/leask/Documents/chameleon/protocol.md:175)
- `Channel ID` exhaustion 已開始進入協議語義
  [protocol.md:192](/Users/leask/Documents/chameleon/protocol.md:192)
- `Session Pool` 已從口號下降到架構方向
  [design.md:27](/Users/leask/Documents/chameleon/design.md:27)

這些都是真實進展，不應再重複列為主阻塞點。

## 4. 下一輪我建議直接落的修改

為了繼續快迭代，我建議下一輪不要再泛泛討論，直接修改這些地方：

1. 補 `CPSK Certificate` / `Node Manifest` 契約
   - 至少把 OOB trust objects 定義清楚
   - 補 `Signer Key ID` 規則
   - 補 overlap / rotation 行為

2. 補 record AAD
   - 明確 `AAD = Plain_Header`
   - 補 `Length/Flags/KeyPhase` 非法處置

3. 收斂 `Session Pool`
   - 補 `Session Class`
   - 補 health signals
   - 補 replenishment triggers
   - 補 draining admission rule

4. 收緊 auth / provisional
   - `unsupported Auth Scheme` 的處置
   - `Proof Length` 全局上限
   - `Max Inbound Bytes` 改成純 Chameleon bytes

5. 補長連線狀態
   - `KEY_UPDATE` state machine
   - `Channel ID` exhaustion 對 server 方向的對稱規則

6. 重寫 recovery matrix
   - 增加 `Epoch Cert` 類錯誤
   - 增加 `Provisional` 類錯誤
   - 拆成可直接驅動 client policy 的多欄矩陣

## 5. 總結

這一輪的最大變化是：你們已經開始真正處理「規模化與長連線」問題，
而不是只補協議最表層的 parser 級錯誤。

現在卡住落地的，不再是大而空的哲學問題，而是幾個更尖銳的工程閉環：

- 兩級 PKI 的 OOB 物件還沒正式定義
- record header 還沒有被明確納入 AEAD 完整性
- `Session Pool` 還缺最小可實作契約
- auth / provisional / rekey / recovery 還缺少少量但關鍵的硬規則

你們如果下一輪把這幾條補死，之後 review 就能進一步下沉到
真正的性能模型、互通測試向量與運維邊界，而不會再卡在核心協議閉環上。
