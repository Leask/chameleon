# Chameleon 聯合審查報告
*(本輪重點：細拆每個修改點，保留已被合理解決的問題，只攻擊仍未閉環的核心結構)*

## 0. 總體判斷

這一輪和前幾輪不一樣。你們已經開始真正修 wire-level 問題，而不是只在
高層設計上換措辭。這是實質進展。

但進展越深，問題也越集中。現在真正阻塞落地的，不再是「還缺幀」，
而是以下四條主線：

- 控制面信任鏈還沒有變成真正可運作的簽發體系
- record layer 還沒有完成最後一層完整性綁定
- `Session Pool` 被提出來了，但還只是願景，不是可執行架構
- auth / recovery / rotation 仍然缺少幾個會直接影響實作一致性的硬規則

下面我按這個順序展開。

## 1. 阻塞級問題

### 1. Root CA 離線與 `Epoch Cert` 直接由 Root 驗證，這兩件事互相衝突

#### 問題本身

`design.md` 明確寫：

- Root CA 絕對離線
- 控制面負責簽發 `Epoch Cert`

[design.md:41](/Users/leask/Documents/chameleon/design.md:41)

但 `protocol.md` 的 `Epoch Cert Verification Algorithm` 又寫成：

- client 用 Root CA 的 Ed25519 公鑰直接驗證 `Epoch Cert` 上的簽名

[protocol.md:109](/Users/leask/Documents/chameleon/protocol.md:109)
[protocol.md:112](/Users/leask/Documents/chameleon/protocol.md:112)

這代表現在的 wire spec 等價於：

- Root CA 直接簽短期 edge key

這和你們自己的信任模型相衝突。Root 一旦真離線，就不可能持續為
24 小時級別的短期 edge key 做線上簽發。

#### 為什麼這不是小問題

如果你們不修這條鏈，後面所有關於：

- signer rotation
- config refresh
- edge key 短期化
- 大規模節點滾動更新

都會變成自相矛盾。

這不是文檔表述問題，是控制面架構沒有落地。

#### 建議怎麼改

最直接的可行方案是引入兩級簽發：

1. `Root CA` 只離線簽發 `Control Plane Signing Key Certificate`
2. 線上控制面持有短期 `CPSK`，由它來簽 `Epoch Cert`

建議新增的信任鏈物件：

- `Root Public Key`
  由 client OOB 固定釘住
- `CPSK Certificate`
  內容至少包含：
  - `Signer Public Key`
  - `Signer Key ID`
  - `NotBefore`
  - `NotAfter`
  - `Signature by Root`
- `Epoch Cert`
  內容至少包含：
  - `Server Static Pubkey`
  - `Epoch Window Start`
  - `Epoch Window End`
  - `Signer Key ID`
  - `Signature by CPSK`

對應 verifier algorithm 應改成：

1. 用 Root 驗證 `CPSK Certificate`
2. 檢查 `CPSK` 是否在有效期
3. 用 `CPSK` 驗證 `Epoch Cert`
4. 再做 responder key binding 與 freshness 驗證

如果你們想更進一步減少 bundle 更新成本，還應加：

- `CPSK overlap period`
- `current + next signer` 雙簽發窗口

不然 signer 輪換會直接變成全網 config 抖動源。

### 2. 短期 responder key 的預分發模型方向對了，但規模化後的控制面成本還沒有被承認

#### 問題本身

這一輪你們正式解決了 `NK` 的前提矛盾：

- client 必須透過 OOB 預先拿到短期 responder static pubkey

[design.md:42](/Users/leask/Documents/chameleon/design.md:42)

這個方向是對的，我接受。

但它立刻帶來了一個新的、更現實的問題：

- 如果每個 edge node 都有自己的 24h 短期 static key，
  client 到底要提前拿多少把 key？
- 全量節點都預分發，config bundle 會爆炸
- 只分發部分節點，路由靈活性會受限
- key rotation 時，session pool 還在運作中的舊 session 如何與新 session 共存

現在文檔只解釋了「邏輯上可行」，還沒解釋「大規模上如何運作」。

#### 會在哪裡出問題

如果這一塊不補，你們會在三個地方踩雷：

1. 配置體積
   全球節點越多，client bundle 越大
2. 滾動更新
   edge key 一換，client config 必須同步
3. 連線池一致性
   pool 中舊 session 還在用舊 key，新的 warm session 可能已切到新 key

這不是邊角問題。你們現在正式引入了 `Session Pool`，
這會把 key rollout 的複雜度再放大一層。

#### 建議怎麼改

不要把預分發物件定義成「一堆裸公鑰」，而要改成**節點 manifest**：

- `Node ID`
- `Region / Route Group`
- `Current Static Pubkey`
- `Next Static Pubkey` 可選
- `NotBefore / NotAfter`
- `Priority / Weight`
- `Manifest Version`
- `Signer Key ID`

client 不必拿全量全球節點，而是拿：

- 路由策略允許的節點子集
- 當前 region 的 current + next key
- 一個明確的 manifest version

同時定義兩條運營規則：

1. edge key rotation 必須有 overlap window
2. client 必須允許 current / next key 在短時間內並存

如果不這樣做，`Session Pool` 一旦存在，就會在 key rollover 期間把問題放大。

### 3. Record header 仍然只有混淆，沒有被明確納入 AEAD 的完整性保護

#### 問題本身

`protocol.md` 現在已經修掉了上一輪的常量 mask 錯誤：

[protocol.md:142](/Users/leask/Documents/chameleon/protocol.md:142)

這是實質進步。

但現在還有一個更底層的問題：文檔從頭到尾都沒有明確寫出
record encryption 的 `AAD` 是什麼。

目前你們定義了：

- plaintext header: `Length || Flags`
- header 會被 HP obfuscate
- payload 會做 AEAD

但沒有說：

- `Length || Flags` 是否作為 AEAD Associated Data 參與認證

這意味著當前規格下，header 可能只是「被混淆」，而不是「被認證」。

#### 為什麼重要

如果 header 不進 AAD，至少會有這幾個問題：

- `Flags` 中的 `Key Phase` 沒有被 ciphertext 完整性綁定
- `Length` 被篡改時，接收方行為只能依賴 parser/AEAD fail 間接收斂
- 不同實作可能對 header 篡改採取不同處置，互通性和恢復語義都會漂移

這不是理論潔癖，是 record layer 定義還差最後一步。

#### 建議怎麼改

直接把 record 加解密流程寫成算法，明確：

發送方：

1. 構造 `Plain_Header = Length || Flags`
2. 以 `AAD = Plain_Header` 執行 AEAD seal，得到 `Ciphertext`
3. 取 `Sample`，計算 `Mask`
4. 產生 `Obfuscated_Header = Plain_Header XOR Mask`

接收方：

1. 讀 3 bytes `Obfuscated_Header`
2. peek 12 bytes `Sample`
3. 反混淆得到 `Plain_Header`
4. 驗證 `Length` / `Flags` 合法
5. 讀完整 `Ciphertext`
6. 以 `AAD = Plain_Header` 執行 AEAD open

同時補三條硬規則：

- `Length < 16` 或 `Length > 16384` -> 立即 `Abort Transport`
- `Flags` 的保留位非 0 -> 協議違規
- `Key Phase` 與本地方向 state machine 不一致 -> 明確進入 rekey 驗證路徑

不把 header 納入 AAD，record layer 仍然不算真正閉環。

### 4. `Session Pool` 是本輪最大的架構新增，但目前只是一個口號，不是一個可執行模型

#### 問題本身

`design.md` 本輪最重要的新主張是：

- 引入 `Multi-Session Connection Pooling`
- 透過 5-10 條獨立 sessions，動態調度 channels
- 「完美兼顧」安全與效能
- 「完美解決」HOL 問題

[design.md:27](/Users/leask/Documents/chameleon/design.md:27)
[design.md:81](/Users/leask/Documents/chameleon/design.md:81)

這裡有兩個層面的問題。

第一，結論過度承諾。

Session Pool 不是「完美解決 HOL」，而是：

- 把單 session 內的 HOL 問題
- 轉成跨 session 的調度問題

它能顯著減輕，但不能完美消除。

第二，規則完全不夠。

目前文檔沒定義：

- pool 的最小/最大 session 數
- session 類型是否分級
- open channel 時如何選 session
- channel 是否允許中途 migration
- session 何時補充、何時回收
- session unhealthy 的判定標準
- GOAWAY / maintenance / key rollover 如何與 pool manager 互動

這代表目前的 `Session Pool` 還不是架構，只是方向。

#### 為什麼這一塊現在必須補

因為你們已經把它提到設計總綱了。如果它只是「實作建議」，
就應該降級措辭；如果它是「核心架構」，就必須給最小行為契約。

否則不同實作的 pool 行為會完全不一樣，最終：

- 效能對比無法複現
- 故障恢復不可預測
- key rotation 與 session draining 會互相打架

#### 建議怎麼改

把 `Session Pool` 收斂成「最小可實作契約」，至少定義以下內容：

1. session class
   - `interactive`
   - `bulk`
   - `spare`

2. per-class pool bounds
   - `min_sessions`
   - `target_sessions`
   - `max_sessions`

3. placement policy
   - 新 channel 一經分配，**不可跨 session 遷移**
   - interactive 流不得與 bulk 流預設共池

4. health signal
   至少用本地可觀測值，而不是幻想可拿到底層所有資訊：
   - auth state
   - available session credit
   - pending open count
   - recent ping RTT
   - draining flag

5. replenishment policy
   - session 進入 `Draining` 時何時補一條新 warm session
   - session close 後何時重建

6. rollover policy
   - key rotation / epoch expiring / maintenance 期間如何平滑替換 pool

如果你們不想把這些寫成正式架構，那就把現在的「完美解決 HOL」
降級成「推薦實作策略」。否則這塊現在過度承諾。

### 5. `Provisional Session` 已經接近可用，但還有兩個會造成實作分歧的細節沒釘死

#### 問題本身

這一輪 `CLIENT_AUTH` 與 provisional state 的進展是明顯的：

- 第一個 record 的第一個 frame 必須是 `CLIENT_AUTH`
- 該 record 不允許攜帶除 `PAD` 外其他 frame
- 超時與字節上限已經變成固定值
- `Proof Length` 對 `Auth Scheme 0x01` 已固定為 64

[protocol.md:164](/Users/leask/Documents/chameleon/protocol.md:164)
[protocol.md:167](/Users/leask/Documents/chameleon/protocol.md:167)
[protocol.md:172](/Users/leask/Documents/chameleon/protocol.md:172)

這意味著我上一輪對 provisional state 的大部分批評，現在都應降級。

但還剩兩個細節沒封死：

1. `Max Inbound Bytes = 2048` 被寫成「包含底層協議頭」
   [protocol.md:168](/Users/leask/Documents/chameleon/protocol.md:168)
   這和你們的 transport-agnostic 目標衝突。
   TCP、QUIC Stream、WebSocket 的「底層協議頭」根本不是同一個觀測面。

2. 對未知 `Auth Scheme` 的處置沒有定義。
   如果 client 發送保留值或未支援 scheme，server 應回：
   - `GOAWAY 0x02 Protocol Violation`
   - 還是 `GOAWAY 0x05 Auth Failed`
   - 還是直接 `Abort Transport`

現在沒有寫。

#### 建議怎麼改

對第一點：

- 把 provisional budget 改成**純 Chameleon 位元組數**
- 若你們真的想控制更底層成本，應另在 transport binding 文檔定義，
  不能寫進核心規格

對第二點：

- 明確分流：
  - `unsupported Auth Scheme` -> `GOAWAY 0x02`
  - `supported scheme but invalid credential/proof` -> `GOAWAY 0x05`

同時再加一條上限：

- `Proof Length` 對所有 scheme 必須有全局最大值，例如 `<= 1024`

否則未來擴 scheme 時 provisional 邊界又會重新打開。

### 6. `Auth Scheme 0x01` 已經有基本算法，但還缺最後兩個會影響互通與安全隔離的細節

#### 問題本身

你們現在已經定義：

- `Credential ID` 查找 client Ed25519 公鑰
- `Proof` = 對 Noise transcript hash `h` 做 Ed25519 簽章

[protocol.md:170](/Users/leask/Documents/chameleon/protocol.md:170)
[protocol.md:173](/Users/leask/Documents/chameleon/protocol.md:173)

這比前幾輪強很多。但還缺兩點：

1. domain separation
   現在簽的是 raw `h`

2. credential database 的版本一致性
   server 查找的公鑰來自哪個配置版本，沒有對應欄位

#### 為什麼這兩點要補

第一，raw `h` 雖然大概率能工作，但不是好規格。
更安全的做法是對簽名 message 做上下文域分離，避免未來被其他用途重用。

第二，`Credential ID -> Public Key` 的查找現在完全依賴 server 本地資料庫，
但沒有定義：

- 資料庫版本
- 撤銷延遲
- rotation overlap

這會直接影響 auth fail 的恢復語義。

#### 建議怎麼改

把 `Auth Scheme 0x01` 的簽名輸入改成：

`Sign( "chameleon-auth-v1" || h )`

其中：

- context string 用 ASCII
- 不含 null terminator

同時在控制面模型裡增加：

- `Credential DB Version`
- `Revocation Epoch`

不一定要把它放進 `CLIENT_AUTH` frame，
但至少 recovery matrix 要承認 auth fail 可能來自配置版本漂移，而不只是憑證錯誤。

### 7. `KEY_UPDATE` 還沒有真正收成完整 state machine

#### 問題本身

`protocol.md` 對 `KEY_UPDATE` 已定義：

- 下一個 record 切 `Key Phase`
- 用新的 traffic key / HP key
- 連續兩次更新至少相隔 `1000 records` 或 `1 minute`

[protocol.md:256](/Users/leask/Documents/chameleon/protocol.md:256)

這比之前完整，但 still 不夠：

- 沒有明確每個方向都維護 `generation counter`
- 沒有明確何時淘汰舊一代 key
- 沒有明確如果收到 unexpected `Key Phase` 應怎麼辦
- 沒有明確兩端幾乎同時 rekey 時，狀態如何獨立演進

#### 為什麼這一塊值得現在補

你們現在有 `Session Pool`，而且 session 會長時間保活。
一旦 session 壽命拉長，rekey 不是稀有事件，而會變成日常路徑。

現在不補，後面一定出現實作漂移。

#### 建議怎麼改

為每個方向增加顯式狀態：

- `current_generation`
- `next_generation_prepared`
- `current_key_phase`

並定義：

1. 收到 `KEY_UPDATE` frame 後：
   - 只準備 next keys
   - 不立即切換 recv generation

2. 收到下一個 `Key Phase` 翻轉的 record：
   - 驗證其必須與 prepared state 對應
   - 成功後提交 generation
   - 舊 generation 立即退休

3. 如果 `Key Phase` 翻轉但本地沒有 prepared next keys：
   - 視為協議違規或完整性錯誤

這一塊不需要新 frame，只需要把 state machine 寫完整。

### 8. `Recovery Matrix` 已經比之前好多了，但還沒有細到足以支撐自動化恢復

#### 問題本身

本輪 recovery matrix 已正式存在：

[protocol.md:287](/Users/leask/Documents/chameleon/protocol.md:287)

這是對的，但現在仍然過粗。它還把不同根因混在一起：

- `Handshake AEAD / MAC Fail -> Retry Different Node`
- `Unknown Frame Type -> Retry Same Node`
- `Session Flow Control Overflow -> Retry Same Node`

[protocol.md:291](/Users/leask/Documents/chameleon/protocol.md:291)
[protocol.md:297](/Users/leask/Documents/chameleon/protocol.md:297)
[protocol.md:298](/Users/leask/Documents/chameleon/protocol.md:298)

這些判斷都不穩：

- handshake fail 可能是 stale config，不一定是節點問題
- unknown frame 很可能是版本或 capability 漂移，重試同節點沒意義
- session overflow 可能是 bug、策略衝突、或惡意對端，重試同節點可能重演

#### 建議怎麼改

把 recovery matrix 拆成更細：

- `Server Action`
- `Client Immediate Action`
- `Retry Same Node?`
- `Retry Different Node?`
- `Config Refresh Required?`
- `Backoff Required?`

同時新增缺失的錯誤條目：

- `Epoch Cert Signature Fail`
- `Epoch Cert Pubkey Mismatch`
- `Epoch Cert NotYetValid`
- `Epoch Cert Expired`
- `Unsupported Auth Scheme`
- `Provisional Timeout`
- `Provisional Byte Overflow`
- `Channel ID Exhaustion`

沒有這些條目，你們的恢復邏輯還是太靠實作者臨場發揮。

### 9. `Channel ID` 規則變嚴格是對的，但你們還沒有定義 exhaustion 策略

#### 問題本身

你們現在新增了：

- client odd / server even
- 嚴格單調遞增
- 最大 `Channel ID = 2^30 - 1`

[protocol.md:185](/Users/leask/Documents/chameleon/protocol.md:185)
[protocol.md:187](/Users/leask/Documents/chameleon/protocol.md:187)

這是好事，因為 channel lifecycle 終於開始像真正的協議。

但現在反而冒出一個新的問題：

- session 長時間存在時，channel id 會耗盡
- 尤其你們本輪正式引入 `Session Pool`
- pool 的 warm sessions 很可能比前幾輪活得更久

現在文檔沒有說：

- 逼近上限時怎麼辦
- 是否要提前 `GOAWAY`
- high watermark 是多少

#### 建議怎麼改

新增一條 session-level 規則：

- 當本地方向下一個可分配 `Channel ID` 超過 `CHANNEL_ID_HIGH_WATERMARK`
  時，實作必須主動對該 session 發送 `GOAWAY 0x00` 並進入 draining，
  由 pool manager 補一條新 session

這個 high watermark 可以不是最大值本身，例如保留一個安全餘量。

如果不補，long-lived pool session 最後一定踩到這個坑。

## 2. 對本輪反駁的回應

### 1. 「握手不需要明文 envelope」

我接受，這一條已經不是主問題。

前提仍然是：

- 首包與回包在該版本內固定長度
- version / caps 不得改變握手長度

### 2. 「Fallback 應降級為部署策略」

我接受，這一條已收斂。

### 3. 「用 Session Pool 解掉單一流 HOL」

我只接受一半。

接受的部分：

- 把 HOL 從單 session 問題轉成多 session 調度問題，方向是對的

不接受的部分：

- `design.md` 現在把它寫成了「完美解決 HOL」

[design.md:27](/Users/leask/Documents/chameleon/design.md:27)
[design.md:81](/Users/leask/Documents/chameleon/design.md:81)

這是過度承諾。它最多是「顯著緩解」，不是「完美解決」。

### 4. 「Bucketing 不寫死在核心協議」

這一條我現在接受，而且不再列為阻塞級問題。

原因很簡單：你們已經把調度優先級和 session-level egress budget 寫進去了，
這比前幾輪實質得多。

[protocol.md:230](/Users/leask/Documents/chameleon/protocol.md:230)
[protocol.md:233](/Users/leask/Documents/chameleon/protocol.md:233)

## 3. 已被合理解決或明顯改善的問題

這一輪以下幾個舊問題可以正式降級：

- `NK` 的前置公鑰前提已被 `design.md` 明確承認
  [design.md:42](/Users/leask/Documents/chameleon/design.md:42)
- `Epoch Cert Verification` 已經開始成形
  [protocol.md:109](/Users/leask/Documents/chameleon/protocol.md:109)
- `CLIENT_AUTH` 不再是黑盒 token
  [protocol.md:170](/Users/leask/Documents/chameleon/protocol.md:170)
- provisional state 已有固定 timeout / byte bound
  [protocol.md:167](/Users/leask/Documents/chameleon/protocol.md:167)
- `OPEN_CHANNEL` 的自定界問題已解決
  [protocol.md:179](/Users/leask/Documents/chameleon/protocol.md:179)
- nonce byte layout 已明確
  [protocol.md:128](/Users/leask/Documents/chameleon/protocol.md:128)

這些進展都是真實的，不應再重複當作主 finding。

## 4. 下一輪我建議你們直接改的清單

為了減少來回溝通，下一輪我建議不要再做抽象討論，直接落以下修改：

1. 改控制面信任鏈
   - 引入 `CPSK Certificate`
   - `Epoch Cert` 改由 `CPSK` 簽發
   - 補 `Signer Key ID`

2. 補 record AAD
   - 明確 `AAD = Plain_Header`
   - 補 `Length/Flags` 非法值處置

3. 收斂 `Session Pool`
   - 把「完美解決 HOL」改成「顯著緩解」
   - 加最小 pool contract：class / bounds / placement / health / replenishment

4. 補 auth 細節
   - `unsupported Auth Scheme` 處置
   - provisional bytes 不再計算底層頭
   - `Proof Length` 全局上限
   - `Auth Scheme 0x01` 改用 domain-separated message

5. 重寫 recovery matrix
   - 新增 `Epoch Cert` 類錯誤
   - 新增 `Provisional` 類錯誤
   - 拆成更細的恢復欄位

6. 補 long-lived session 規則
   - `Channel ID` high watermark
   - exhaustion 時的 `GOAWAY + drain + pool replenish`

## 5. 總結

這一輪最重要的變化是：
協議已經不是「顯然沒法實作」的狀態了。你們已經把很多前幾輪的大洞補上了。

但現在卡住它的，已經不是大而空的設計哲學，而是幾個更尖銳的工程閉環：

- 離線 Root 與線上短期簽發沒有真正連起來
- record header 還沒有被明確納入 AEAD 完整性
- `Session Pool` 被提出來了，但還沒有最小可執行契約
- auth / recovery / long-lived session 的少量硬規則仍然缺失

你們如果把這幾條在下一輪補死，之後 review 的重心就能真正下沉到
更細的相容性、效能與長期維運問題，而不是一直卡在核心閉環上。
