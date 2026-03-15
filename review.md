# Chameleon Review
## 對 `design.md` v9.0 與 `protocol.md` v3.0 的新一輪聯合審查

這一輪的文檔比前幾版更接近可收斂的狀態。值得正式確認的進展有四個：

1. 兩級 PKI 方向已經走對了。`Offline Root CA -> CPSK -> Epoch Cert`
   的鏈路，至少在高層設計上已經不再自相矛盾。
2. `CLIENT_AUTH` 已經補上 transcript binding 的 domain separation，
   這比前幾版只寫「簽握手 hash」要嚴謹得多。
3. `Channel ID` 的奇偶分配、單調遞增、耗盡水位線，以及
   `KEY_UPDATE` 的基礎狀態機，都比前幾輪明顯更像能實作的規格。
4. `Session Pool` 不再只是口號，已經開始有最小契約的影子。

但這一輪剩下的問題也更集中，基本都屬於「最後一公里不閉環就會卡住互通
或卡住運營」的級別。下面不再重複已經解決的舊問題，只攻擊這一輪新增後
仍然沒有收口的點，並盡量給出直接可落地的修改方案。

---

## 0. 本輪已可接受的設計結論

以下幾個爭議點，現在可以視為暫時成立，不應再來回震盪：

### 0.1 無明文 Envelope 的固定長度握手
這一點目前是成立的。只要 `protocol.md` 繼續把 `NK`、112 Bytes
首包、176 Bytes 回包、固定 payload 長度這幾個條件鎖死，握手解析就不需要
再引入額外明文頭。

需要保留的前提只有一個：後續版本不要再把可變長度欄位偷偷塞回握手層。

### 0.2 Fallback 降級為部署策略
這個收斂是正確的。協議核心只保證 `Pre-Auth Silent Drop`，
不再聲稱所有承載上都能提供同等形式的 socket 轉交。這比之前把
Fallback 當協議能力合理得多。

### 0.3 Bucketing 不寫死在核心協議
這個方向也成立。`PAD` 作為機制，策略由控制面決定，這種「機制/策略分離」
是對的。前提是 `PAD` 不能穿透容量控制，也不能把調度器拖成不可預測狀態。

### 0.4 單一 Session 不追求消滅 HOL，而是縮小故障域
這一點目前已經接近正解，但 `design.md` 內部還沒有完全說清楚。
正確表述應該是：

`Session Pool` 能把 HOL 的影響限制在單一 Session 內，並把其代價控制到
產品可接受範圍；它不能「完美消滅 HOL」。

---

## 1. 阻塞級問題

## 1.1 兩級 PKI 的方向對了，但物件模型仍然沒有規範化，信任鏈還不能真正落地

### 攻擊
`design.md` 已經把 `Offline Root CA -> CPSK -> Epoch Cert` 的大方向補上，
但現在仍然只有概念，沒有可互通的 OOB 物件模型。這會直接造成三個問題：

1. 客戶端無法用一致方式解釋「有效的 CPSK 憑證」到底長什麼樣。
2. `Node Manifest` 仍然只是名詞，不是規範對象，無法支撐
   `NK` 對 responder static key 的預分發。
3. `Signer Key ID` 已經進了 `Epoch Cert`，但 `Key ID` 是怎麼生成、如何對應、
   如何防回滾，依然沒有最終規則。

現在最危險的不是方向錯，而是文檔分工仍然沒收口：
`design.md` 說 `protocol.md` 是線上規格唯一真相來源，但支撐握手信任的
`CPSK Certificate` 與 `Node Manifest` 既不在 `protocol.md`，也沒有一份獨立的
控制面規格。這會讓不同實作各自補洞，最後互不相容。

### 為什麼目前修改還不夠
`protocol.md` 第 3.3 節現在已經要求客戶端從本地信任庫中依 `Signer Key ID`
找出對應 `CPSK` 公鑰，這說明 `CPSK` 已經不是純設計概念，而是協議運作的
前提資料。只要它是協議前提，就必須有規範化格式與驗證規則。

同樣地，`Node Manifest` 現在承擔了 `NK` 的前置信任分發職責，它也必須有：

1. 版本號
2. 簽發者
3. 發布時間/失效時間
4. 反回滾欄位
5. 每個節點的 `Node ID`
6. `Current/Next Static Pubkey`
7. 靜態公鑰的切換時窗

否則「預分發 responder static key」這件事仍然只是口號。

### 具體修改方案
建議不要再把這部分繼續塞回 `design.md` 散文段落，而是正式做一個規範：

1. 新增一份 `control_plane.md`，作為 OOB 信任物件的唯一規範。
2. 在該文檔裡定義 `CPSK Certificate`：
   - `Version`
   - `Key ID`
   - `Public Key`
   - `Not Before`
   - `Not After`
   - `Key Usage = CPSK_SIGNING`
   - `Root Signature`
3. 定義 `Node Manifest`：
   - `Manifest Version`
   - `Manifest Sequence` 或 `Generation`
   - `Issued At`
   - `Expires At`
   - `Signer Key ID`
   - `Node List[]`
4. 每個 `Node Entry` 至少包含：
   - `Node ID`
   - `Route Group ID`
   - `Current Static Pubkey`
   - `Current Key Not After`
   - `Next Static Pubkey`（可選）
   - `Next Key Not Before`（可選）
   - `Transport Class / Endpoint Metadata`
5. 明確定義 `Key ID` 的生成方式。
   - 若保留 8 Bytes，至少規定它是某個固定 hash 的截斷值或控制面分配值。
   - 若不願承擔截斷碰撞風險，改成 16 Bytes，但這會牽動 `Epoch Cert`
     長度，需要一次性改乾淨。
6. 增加 anti-rollback 規則：
   - 客戶端必須記住最近接受的 `Manifest Sequence`
   - 若收到更低版本的 manifest，必須拒絕

這一項如果不補齊，`NK` 的信任基礎仍然只是設計意圖，不是能部署的系統。

---

## 1.2 `Epoch Cert` 已經有 `Signer Key ID`，但簽章上下文仍然不完整

### 攻擊
這一輪 `Epoch Cert` 確實進步了，因為它終於開始走兩級信任鏈。
但目前的簽章訊息仍然是「前 56 Bytes 直接拼接」，這仍然不夠嚴謹。

問題不是 Ed25519 本身，而是缺少**簽章上下文 (domain separation)**。
如果未來 `CPSK` 還要簽別的控制面物件，而那些物件剛好也有相同前綴布局，
那就會產生跨物件語義混淆風險。

`CLIENT_AUTH` 已經補上 `"chameleon-auth-v1" || h`，
但 `Epoch Cert` 反而沒有同級別的上下文綁定，這不一致。

### 為什麼目前修改還不夠
現在的 `Epoch Cert` 驗證演算法只寫了：

`Sign( ServerStaticPubkey || Epoch_Window_Start || Epoch_Window_End || Signer Key ID )`

這仍然缺三樣東西：

1. 物件類型上下文
2. 結構版本
3. 規範化序列化聲明

沒有這三樣，簽章雖然存在，但還不算是可長期維護的證書格式。

### 具體修改方案
把 `Epoch Cert` 的簽章訊息改成明確的規範文本，例如：

```text
EpochCertBody =
    ServerStaticPubkey(32) ||
    Epoch_Window_Start(8) ||
    Epoch_Window_End(8) ||
    Signer_Key_ID(8)

EpochCertSigMessage =
    "chameleon-epoch-cert-v1" || EpochCertBody
```

然後明確規定：

1. 所有整數均為大端序。
2. 字串為 ASCII bytes，不含 null。
3. `Ed25519 Control Plane Sig` 必須對 `EpochCertSigMessage` 簽章。

如果你們希望未來 `Epoch Cert` 還能擴展欄位，則不要再用「直接對前 56 Bytes
簽章」這種依賴固定位移的寫法，而要明確把「被簽內容」定義成一個版本化 body。

---

## 1.3 Record Layer 還差最後一個關鍵步驟：`Plain_Header` 尚未被明確納入 AEAD `AAD`

### 攻擊
這一輪最大的協議級缺口仍然在 Record Layer。你們已經把 header protection
從錯誤方向拉回來了，但現在仍然沒有把最重要的完整性綁定寫死：

`Length || Flags` 必須作為 AEAD 的 `AAD`。

如果這一點不在規格裡明說，就會出現兩種實作：

1. 一部分實作把 header 當 AAD。
2. 一部分實作只加密 payload，不綁 header。

這會直接破壞獨立互通。

### 為什麼目前修改還不夠
現在的文字已經有：

1. `Obfuscated_Header = (Length || Flags) XOR Mask`
2. 解析器先 peek 12 Bytes sample，再解 header，再按 length 讀 ciphertext

但整個流程裡仍然沒出現一句規範化語言說：

`AEAD_Encrypt(Traffic_Key, Nonce, Plaintext_Frames, AAD = Plain_Header)`

這不是語義細節，而是 Record 完整性是否真正閉環的核心。

### 具體修改方案
把第 4 章直接補成精確演算法，至少要有下面幾句：

```text
Plain_Header = Length(2) || Flags(1)

Ciphertext = AEAD_Encrypt(
    key   = Traffic_Key,
    nonce = Nonce(Sequence_Number),
    aad   = Plain_Header,
    plaintext = FramePayload
)

Obfuscated_Header = Plain_Header XOR HeaderMask(HP_Key, Sample)
```

接收方流程明確為：

1. 讀取 3 Bytes `Obfuscated_Header`
2. Peek 後續 12 Bytes 作為 `Sample`
3. 解出 `Plain_Header`
4. 先做結構檢查：
   - `Length` 必須在 `[16, 16384]`
   - `Flags.Reserved` 必須為 0
5. 根據 `Length` 讀取完整 ciphertext
6. 使用 `AAD = Plain_Header` 做 AEAD 解密
7. 若 MAC 驗證失敗，立即 `Abort Transport`

這一項不應再留到下一輪。它現在已經是規格能否獨立實作的最後一塊硬缺口。

---

## 1.4 `Caps` 的子集規則補上了，但「請求能力」與「必要能力」仍然混在一起

### 攻擊
你們這一輪補了 `Selected Caps 必須是客戶端 Caps 子集`，這是正確的。
但目前仍然沒有把「客戶端想要的能力」和「客戶端不能失去的能力」分開。

現在的寫法是：

1. Client 發 `Caps`
2. Server 回 `Selected Caps`
3. Client 若關鍵能力未滿足，可主動中止

這裡的「關鍵能力」沒有形式化定義。這會讓不同實作在未來 capability
增長後產生嚴重分歧。

### 為什麼目前修改還不夠
現在只有一個保留的 Datagram bit，所以問題還沒爆出來。
但只要之後引入更多能力位，例如：

1. 某種 channel 類型
2. 某種壓縮/填充機制
3. 某種恢復語義

那麼「只是偏好」和「必須有」的差異就會立刻變成互通問題。

### 具體修改方案
最乾淨的做法是直接把握手請求改成：

```text
Version (1)
Offered Caps (4)
Required Caps (4)
Padding (...)
```

並且規範：

1. `Selected Caps` 必須是 `Offered Caps` 的子集。
2. 若 `Required Caps & ~Selected Caps != 0`，客戶端必須立即中止。
3. 若 server 不接受任何 `Required Caps`，它不應嘗試部分成功。

如果你們現在不想改首包語義，至少也要在現有規格中明確增加：

1. `Critical Cap Mask` 的概念
2. 不滿足 critical cap 時的恢復動作
3. 這不是 `Retry Different Node`，而是 `Capability Mismatch`

否則現在的 capability negotiation 仍然只是配置比對，不是嚴格協議語義。

---

## 1.5 `Provisional Session Bounds` 仍然把底層頭部算進來，這和語言/承載解耦目標衝突

### 攻擊
`protocol.md` 現在寫 `Max Inbound Bytes = 2048 Bytes (包含底層協議頭)`。
這個定義仍然不成立。

因為底層可以是：

1. TCP
2. QUIC Stream
3. WebSocket

這三種承載的「底層頭部」根本不是同一個概念，也不在 Chameleon 協議層的
觀測面內。把它寫進規格，只會讓不同實作用不同方法估算，最後不一致。

### 為什麼目前修改還不夠
`Provisional Session` 的目的很明確：限制未完成 `CLIENT_AUTH` 前的資源消耗。
這個目標應該由 Chameleon 自身可觀測的 bytes/records 來實現，而不是引用
下層封裝。

現在這種寫法最大的問題是：

1. QUIC stream 實作無法精確知道上層這 2048 Bytes 對應了多少 transport bytes
2. WebSocket 會多出 frame 封裝
3. TCP 幾乎沒有 per-record header 概念

這會讓同一協議在不同底層下得到不同的 provisional budget。

### 具體修改方案
把該規則改成純協議內可計量的定義：

1. `Max Inbound Chameleon Bytes = 2048`
   - 計數範圍：從握手完成後的第一個 Record 開始，
     累計 `3-byte record header + ciphertext length`
   - 不包含 TCP/QUIC/WebSocket 的下層封裝
2. 新增 `Max Records Before Auth = 2`
   - 防止攻擊者用大量極小 `PAD` record 拖 parser
3. 明確 `CLIENT_AUTH` 必須出現在第一個 Record 的第一個 frame
   這一條你們已經有了，但應和 byte/record bound 一起形成閉環

這樣 `Provisional Session` 才真正成為 transport-agnostic 的規範。

---

## 1.6 `Session Pool` 已經進入正軌，但 `design.md` 仍然對它做了過度承諾

### 攻擊
`design.md` 第 0.2 節現在用了正確表述：`Session Pool` 只能
`Significantly Mitigate` HOL。
但同一份文檔後面的反駁段落又說它能「完美解決 HOL」。

這是這一輪最明顯的文檔自我打架。

更嚴重的是，即使先不談措辭，現在的 `Session Pool Contract` 仍然還不夠細。
它已經有 `Class/Placement`、`Bounds`、`Health & Replenishment`，
但仍然沒有把「控制流量如何避免被 bulk session 拖住」寫成明確要求。

### 為什麼目前修改還不夠
`Session Pool` 的真正價值，不是把資料平均分散，而是把不同故障模式隔離。
現在的描述還缺三個核心點：

1. 是否保留一條低佔用的 control/interactive session
2. 在 session 進入 draining 時，new opens 的重定向規則
3. pool manager 如何判定一條 session 已不適合承接互動流

如果沒有這些，實作方仍然可能把互動流和 bulk 流混回同一 session，
最後又把控制信令拖死。

### 具體修改方案
對 `Session Pool` 建議補成最小可執行契約：

1. 至少區分兩類 session：
   - `Latency/Control Session`
   - `Bulk Session`
2. `Latency/Control Session` 上禁止長時間高佔用 bulk transfer
3. 當 session 進入以下任一條件時，必須標記為 `No-New-Open`：
   - 收到 `GOAWAY`
   - `Channel ID` 接近水位線
   - session credit 長期低於門檻
   - 連續出現高排隊延遲
4. pool manager 必須在 `No-New-Open` 之前或同時完成 `Warm Session`
   補充，不能等 draining 開始後再臨時建連線
5. 文檔措辭必須統一改成：
   - `縮小 HOL 故障域`
   - `把 HOL 代價限制在單一 Session`
   - `不承諾消滅 HOL`

這不是文字修飾，而是產品預期管理。

---

## 1.7 恢復矩陣仍然過粗，現在的幾個恢復動作會把客戶端帶進錯誤路徑

### 攻擊
這一輪 `Recovery Matrix` 已經存在，但還不夠細。
目前最大的問題不是缺表，而是很多不同失敗被塞進了同一恢復策略。

這會導致客戶端在 hostile network 下做出錯誤選擇，例如：

1. 明明是本地信任鏈過期，卻一直 `Retry Different Node`
2. 明明是 capability mismatch，卻去盲目切節點
3. 明明是 responder key mismatch，卻被當成 transient network error

### 為什麼目前修改還不夠
當前矩陣還缺少至少以下幾類錯誤：

1. `Signer Key ID Not Found`
2. `Epoch Cert Signature Invalid`
3. `Responder Key Mismatch`
4. `Epoch Not Yet Valid`
5. `Epoch Expired`
6. `Capability Mismatch`
7. `Manifest Rollback Detected`
8. `Credential Revoked` 與 `Proof Invalid` 的區分

這些錯誤的恢復策略完全不同，不能再合併。

### 具體修改方案
建議把恢復矩陣擴成至少下面這些條目：

1. `Signer Key ID Not Found`
   - Server Action: N/A（客戶端本地驗證失敗）
   - Client Recovery: `Config Refresh Required`
   - 禁止：盲目同節點重試
2. `Epoch Cert Signature Invalid`
   - Client Recovery: `Security Failure, Switch Route Only After Config Check`
3. `Responder Key Mismatch`
   - Client Recovery: `Hard Security Failure`
   - 禁止：同一路由無限重試
4. `Epoch Not Yet Valid`
   - Client Recovery: `Check Clock / Config Refresh`
5. `Epoch Expired`
   - Client Recovery: `Config Refresh or Different Node Same Route Group`
6. `Capability Mismatch`
   - Client Recovery: `Do Not Retry Same Capability Set`
7. `Server Maintenance / ID Exhaustion / Draining`
   - Client Recovery: `Reassign New Channels To Fresh Session In Pool`
   - 不是直接切整個 node
8. `Client Credential Revoked`
   - Client Recovery: `Stop Retry, Require Re-auth or Config Update`
9. `Client Proof Invalid`
   - Client Recovery: `Local Bug / Credential Corruption`

如果恢復矩陣不補細，後續的 session pool 和 route manager 都會建立在錯誤事件分類上。

---

## 2. 高優先級但非阻塞問題

## 2.1 協議概述仍然把「握手層」寫成泛化的身份認證層，這會誤導實作者

### 攻擊
`protocol.md` 開頭把 Handshake Layer 寫成「負責身份認證、金鑰交換與能力協商」。
這個表述仍然不準確。

現在的實際設計是：

1. 握手層完成 server-side trust establishment、key exchange、capability offer
2. client auth 在握手後第一個 record 內完成

如果不把這件事寫準，實作者很容易以為握手層已經包含雙向認證，
然後在狀態機上做出錯誤簡化。

### 具體修改方案
把概述中的一句話改為：

`握手層負責伺服器側信任建立、金鑰交換與能力協商；客戶端授權在握手後的 CLIENT_AUTH 階段完成。`

---

## 2.2 密碼學參考文獻仍然寫錯，這會污染規格的嚴肅性

### 攻擊
`Noise Protocol Framework: RFC 7748` 這個寫法是錯的。
RFC 7748 是 Curve25519/448，不是 Noise 規格本身。

這不是吹毛求疵。協議文檔如果連 normative reference 都寫錯，
會讓整份規格的嚴謹性打折。

### 具體修改方案
建議改成：

1. `Noise Protocol Framework`：引用 Noise 官方規格或穩定版本文檔
2. `RFC 7748`：單獨作為 Curve25519/X25519 參考

至少不要把兩者混成一條。

---

## 2.3 `KEY_UPDATE` 已有狀態機，但仍缺少強制觸發條件

### 攻擊
這一輪 `KEY_UPDATE` 的 state machine 已經比之前好很多，
但它仍然只定義了「如何切」，沒有定義「何時必須切」。

如果不定義至少一組 mandatory trigger，實作方可能出現：

1. 有的實作幾乎從不 rekey
2. 有的實作頻繁 rekey
3. 有的實作只在 operator 指令下 rekey

這會讓長連線行為不可預測。

### 為什麼目前修改還不夠
目前唯一的硬限制是：

`連續兩次 KEY_UPDATE 之間至少間隔 1000 個 Records 或 1 分鐘`

這只是冷卻期，不是觸發條件。

### 具體修改方案
至少新增三個協議常數，哪怕先用保守值：

1. `MAX_RECORDS_PER_GENERATION`
2. `MAX_BYTES_PER_GENERATION`
3. `MAX_KEY_AGE`

然後規定：

1. 任一門檻逼近時，發送方必須主動發送 `KEY_UPDATE`
2. 冷卻期只限制「不必要的高頻 rekey」，不能阻止 mandatory rekey
3. 若一方長期不 rekey，對端可在安全策略上主動發起輪換

這樣 `KEY_UPDATE` 才從「可用」變成「可運營」。

---

## 3. 建議下一輪直接落地的文檔改法

如果你們要加快迭代，下一輪不要再平均發力。建議直接按下面順序修改：

1. 在 `protocol.md` 補齊 `Record AEAD AAD = Plain_Header`
2. 在 `protocol.md` 為 `Epoch Cert` 增加 domain-separated signature message
3. 在 `protocol.md` 把 `Provisional Bounds` 改成只計算 Chameleon bytes/records
4. 在 `protocol.md` 把 `Caps` 拆成 `Offered/Required`，或者引入明確的
   `Critical Cap Mask`
5. 在 `design.md` 統一 `Session Pool` 表述，刪掉「完美解決 HOL」這種過度結論
6. 新增 `control_plane.md`，把 `CPSK Certificate`、`Node Manifest`、
   anti-rollback、key selection 規範化
7. 擴充 `Recovery Matrix`，把信任鏈/能力/配置失敗拆開
8. 修正文檔概述中的握手層描述與錯誤 reference

---

## 4. 本輪結論

這一輪不是方向性崩潰，而是規格化的最後收口問題。
也正因為如此，本輪不能再接受「先有概念，之後再補格式」的處理方式。

當前版本最值得肯定的是：

1. 大架構已經明顯比前幾輪穩定
2. 反駁清單中有幾條已經真正站住
3. 協議核心沒有再來回搖擺

但如果要進入真正的可互通實作，下面四件事必須先釘死：

1. OOB 信任物件的正式規格
2. `Epoch Cert` 的簽章上下文
3. Record Layer 的 `AAD` 綁定
4. `Caps` / `Provisional Bounds` / `Session Pool` 的最後語義收斂

只要這四項補齊，下一輪 review 就可以從「協議閉環審查」進一步下沉到
「實作風險與參數選型審查」。
