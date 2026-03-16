# Chameleon Review
## 對最新 `design.md` 與 `protocol.md` 的新一輪聯合審查

這一輪的情況比上一輪更清楚了：有幾個核心問題確實被修掉了，但新的主要阻塞
點已經不再是 record framing 這種底層缺口，而是「規範的責任分界與恢復語義
正在互相打架」。

先給總結：

1. `Record AAD`、`Provisional Bytes`、`Session Flow Control` 的部分修正是實質進展。
2. 最大的新問題是 `Recovery Matrix` 已經不是單純不夠細，而是混淆了誰在檢測、
   誰能發信號、誰該本地中止。
3. `control_plane.md` 仍然缺失，讓兩級 PKI 依舊沒有正式規格落點。
4. `Caps`、`Epoch Cert`、`Session Pool` 這三塊仍然各自保留了一個會在後面放大的
   語義裂縫。

下面只攻擊目前真正還沒收口的點，不重複已經被修掉的部分。

---

## 0. 本輪可以正式降級的舊問題

### 0.1 Record Layer 的完整性閉環已基本成立
`protocol.md` 現在明確把 `Plain_Header = Length || Flags` 納入 AEAD `AAD`，
這是上一輪最重要的協議級缺口。這一點可以降級，不再作為主攻點。

### 0.2 Provisional Bounds 已回到協議層可觀測範圍
`Max Inbound Bytes` 已經改成純 Chameleon bytes，不再混入 TCP/QUIC/WebSocket
下層開銷。這是正確修正。

### 0.3 Recovery Matrix 的一個硬錯碼被修掉了
上一輪我指出 `Session Flow Control Overflow` 被錯寫成 `GOAWAY 0x04`，
現在已改回 `0x03`。這是有效修正。

### 0.4 `GOAWAY 0x00` 的高層恢復思路比上一輪更合理
`Server Maintenance/ID Exhaust` 現在已經朝向「把新 channel 轉派到 pool」
的方向收斂，這比直接 `Switch Node` 更像正確的 session-pool 行為。

---

## 1. 阻塞級問題

## 1.1 `control_plane.md` 仍然不存在，兩級 PKI 依然沒有正式規格落點

### 攻擊
`design.md` 已經明確把 OOB 信任物件規格外包給 `control_plane.md`，但倉庫裡
目前仍然沒有這個文件。這表示兩級 PKI 在設計上被承認了，在規格上仍然是空的。

這不是文檔整理問題，而是結構性阻塞，因為 `protocol.md` 的 `Epoch Cert`
驗證已經顯式依賴：

1. 有效的 `CPSK`
2. `Signer Key ID` 到 `CPSK` 的查找規則
3. OOB 下發的 responder static key
4. 對 OOB 物件的 anti-rollback

只要 `control_plane.md` 不存在，這些就都還不是規格，而只是前提假設。

### 為什麼目前還不夠
現在的文檔分工實際上已經變成三份：

1. `design.md`: 邊界與取捨
2. `protocol.md`: 資料面 wire format
3. `control_plane.md`: OOB 信任物件與生命週期

但倉庫實際只有前兩份。這意味著當前規範集合本身不自洽。

### 具體修改方案
下一輪最優先不是再改 `protocol.md` 細節，而是把 `control_plane.md` 真正補出來，
至少包含：

1. `CPSK Certificate`
   - `Version`
   - `Key ID`
   - `Public Key`
   - `Not Before`
   - `Not After`
   - `Key Usage`
   - `Root Signature`
2. `Node Manifest`
   - `Manifest Version`
   - `Manifest Sequence`
   - `Issued At`
   - `Expires At`
   - `Signer Key ID`
   - `Node Entries[]`
3. `Node Entry`
   - `Node ID`
   - `Route Group ID`
   - `Current Static Pubkey`
   - `Current Key Not After`
   - `Next Static Pubkey`
   - `Next Key Not Before`
   - `Endpoint Metadata`
4. anti-rollback 規則
5. OOB 更新與撤銷規則
6. `Signer Key ID` 的生成與碰撞處理

在這份文件出現前，我不認為兩級 PKI 可以被視為已收斂。

---

## 1.2 `Recovery Matrix` 現在最大的問題不是不夠細，而是動作主體混亂，部分條目根本不可能成立

### 攻擊
這一輪最嚴重的新問題在 `protocol.md` 第 6 章。你們把恢復矩陣補得更細了，
但現在它混淆了「誰檢測到錯誤」「誰能發送對端可見的錯誤信號」「誰只能本地中止」。

這會直接導致實作分叉，因為某些條目在物理上就不成立。

最明顯的例子有：

1. `Epoch Cert Signature Invalid -> 發送 GOAWAY (0x02) -> Abort`
2. `Responder Key Mismatch -> 發送 GOAWAY (0x02) -> Abort`
3. `Epoch Cert Not Yet Valid -> 發送 GOAWAY (0x02) -> Abort`
4. `Epoch Cert Expired -> 發送 GOAWAY (0x00) -> Drain`

這些錯誤都是**客戶端在驗證伺服器握手回包時本地得出的結論**。它們不是伺服器
主動檢測到的錯誤，因此矩陣不能把它們寫成「發送 GOAWAY」這種對端可見動作。

更糟的是，在 hostile-network 的目標下，client 在這種 server-auth 失敗場景下
主動回送 `GOAWAY` 反而會增加可觀測性，而不是降低它。

### 為什麼目前還不夠
這一輪的 `Recovery Matrix` 實際上把三種完全不同的錯誤類型混在一個欄位裡：

1. **Server-side detected error**
   - 例如 unsupported auth scheme、channel flow control overflow
2. **Client-side local verification failure**
   - 例如 `Epoch Cert Signature Invalid`、`Responder Key Mismatch`
3. **Transport corruption**
   - 例如 record header/MAC fail

這三類錯誤的可用動作集合根本不同。

此外，`Unsupported Auth Scheme` 在 `CLIENT_AUTH` 章節寫的是：

`GOAWAY (0x02) 並硬關閉`

但在 `Recovery Matrix` 裡卻寫成：

`GOAWAY (0x02) -> Drain`

這是另一個直接的內部矛盾。

### 具體修改方案
不要再試圖用一張「單欄位 immediate action 表」覆蓋所有錯誤。下一輪應把
矩陣改成至少四欄：

1. `Error Scenario`
2. `Detected By`
3. `Local Action`
4. `Peer-Visible Signal`
5. `Retry / Refresh Policy`

然後按類型重寫：

1. `Epoch Cert Signature Invalid`
   - Detected By: Client
   - Local Action: `Abort Transport`, discard keys
   - Peer-Visible Signal: None
   - Retry: `Security Failure, Config Refresh First`
2. `Responder Key Mismatch`
   - Detected By: Client
   - Local Action: `Abort Transport`
   - Peer-Visible Signal: None
   - Retry: `Hard Security Failure`
3. `Epoch Cert Not Yet Valid`
   - Detected By: Client
   - Local Action: `Abort Transport`
   - Peer-Visible Signal: None
   - Retry: `Check Clock / Refresh Config`
4. `Epoch Cert Expired`
   - Detected By: Client
   - Local Action: `Abort Transport`
   - Peer-Visible Signal: None
   - Retry: `Refresh Config or Retry Different Node Same Group`
5. `Unsupported Auth Scheme`
   - Detected By: Server
   - Local Action: hard close
   - Peer-Visible Signal: `GOAWAY 0x02`
   - Retry: `Do Not Retry Until Client Upgrade`

只要這一塊不拆清楚，後面所有實作都會對「什麼時候能發錯誤幀、什麼時候只能
本地 drop」做出不同判斷。

---

## 1.3 `Epoch Cert` 仍然沒有 domain-separated signature message

### 攻擊
這一輪 `Epoch Cert` 還是沿用「驗證前 56 Bytes」的模型。這仍然不夠。

問題不是它今天不能用，而是它仍然把簽章語義綁在當前欄位位移上，而不是綁在
一個明確的、版本化的物件類型上。只要未來 `CPSK` 需要簽別的控制面資料，
或者 `Epoch Cert` 需要擴欄位，這個設計就會立刻變脆。

### 為什麼目前還不夠
你們已經承認 `CLIENT_AUTH` 需要：

`"chameleon-auth-v1" || h`

也就是說，domain separation 在你們自己的設計裡已經不是可選項。那
`Epoch Cert` 繼續停留在「簽固定前綴」只是半收斂狀態。

### 具體修改方案
直接把 `Epoch Cert` 驗證規則改成：

```text
EpochCertBody =
    ServerStaticPubkey(32) ||
    Epoch_Window_Start(8) ||
    Epoch_Window_End(8) ||
    Signer_Key_ID(8)

EpochCertSigMessage =
    "chameleon-epoch-cert-v1" || EpochCertBody
```

並規定：

1. 所有整數為大端序
2. 字串為 ASCII bytes，不含 null
3. `Ed25519 Control Plane Sig` 驗證的 message 就是 `EpochCertSigMessage`

如果未來有 `Epoch Cert v2`，就升級 context string，不要靠欄位偏移硬撐。

---

## 1.4 `Caps` 仍然不是正式 negotiation，只是 server 子集裁剪後由 client 用散文規則決定是否接受

### 攻擊
這一輪 `Caps` 依然保留舊問題：

1. client 發 `Caps`
2. server 回 `Selected Caps`
3. client 若「關鍵能力」未滿足，可主動 abort

這裡的「關鍵能力」仍然沒有形式化定義。也就是說，現在的 capability 流程
還不是 negotiation，而只是：

1. client 發出偏好集合
2. server 回一個子集
3. client 自己猜哪些 bit 是必要的

這會在 capability 增長後直接製造互通分裂。

### 為什麼目前還不夠
現在只有 Datagram 這一個保留 bit，所以問題還沒爆開。但只要之後新增：

1. 某種 channel 類型
2. 某種 control-plane-coupled feature
3. 某種 data path behavior

那麼不同實作就會對「必要 / 可選」做出不同解讀。

### 具體修改方案
兩種修法都可以，但必須二選一：

1. 改握手首包，加入
   - `Offered Caps`
   - `Required Caps`
2. 保持 wire format，但新增一個明確的 `Critical Cap Mask`

若採第一種，規則應寫死：

1. `Selected Caps` 必須是 `Offered Caps` 子集
2. 若 `Required Caps & ~Selected Caps != 0`，client 必須立即本地中止
3. 此失敗在 recovery matrix 中叫 `Capability Mismatch`

若採第二種，也要把 `Critical Cap Mask` 的語義寫死，不要再用「關鍵能力」
這種散文詞。

---

## 1.5 `design.md` 的架構正文已經修正，但反駁段落仍然過度宣稱 `Session Pool`

### 攻擊
`design.md` 的正式架構段落現在已經正確承認：

`Session Pool` 不能消滅 HOL，只能把故障域收斂到單一 session。

但反駁段落仍然保留：

`完美解決了 HOL 阻塞問題`

這不是文字小問題，而是會直接污染你們對性能上限、排隊延遲和 pool policy
的後續推理。

### 為什麼目前還不夠
你們這一輪其實已經用 `Health & Replenishment`、`No-New-Open` 等行為承認了：

1. HOL 依然存在
2. pool 只是隔離故障域
3. control/interactive 必須和 bulk 分流

那反駁段落還寫「完美解決」只會讓文檔自相矛盾。

### 具體修改方案
把反駁三重寫成：

1. `Session Pool` 將 HOL 的影響限制在單一 session
2. control/interactive 與 bulk 必須放在不同 session
3. 這是架構層隔離，不是協議層消滅 HOL

並再補一條硬規則：

一條 session 一旦被標記為 `Bulk` 或觀察到持續高排隊延遲，就不得再承接新的
control/interactive opens。

---

## 2. 高優先級但非阻塞問題

## 2.1 `protocol.md` 的概述仍然沒有同步到目前的 auth 模型

### 攻擊
`design.md` 已經把握手層職責修正了，但 `protocol.md` 開頭仍寫：

`Handshake Layer: 負責身份認證、金鑰交換與能力協商`

這和現在的實際設計不一致。當前模型是：

1. 握手層負責 server-side trust establishment
2. client auth 發生在 `CLIENT_AUTH`

### 具體修改方案
把概述改成：

`Handshake Layer: 負責伺服器側信任建立、金鑰交換與能力協商；客戶端授權於握手後的 CLIENT_AUTH 階段完成。`

這個改動很小，但它會影響整個狀態機理解，應該立刻做。

---

## 2.2 密碼學 reference 仍然是錯的

### 攻擊
`protocol.md` 仍然把：

`Noise Protocol Framework: RFC 7748`

寫成同一條。這是錯的。RFC 7748 是 X25519/X448，不是 Noise 規格本身。

### 具體修改方案
拆成兩條：

1. `Noise Protocol Framework`: 引用 Noise 規格本身
2. `RFC 7748`: 作為 Curve25519/X25519 參考

這是規格嚴肅性的基本要求。

---

## 2.3 `KEY_UPDATE` 仍然只有冷卻期，沒有 mandatory trigger

### 攻擊
現在的 `KEY_UPDATE` 狀態機已經能描述「怎麼切」，但仍然沒有描述
「什麼時候必須切」。

當前唯一硬規則是：

`兩次 KEY_UPDATE 之間至少相隔 1000 個 Records 或 1 分鐘`

這只是 anti-thrash 冷卻期，不是 rekey policy。

### 為什麼目前還不夠
如果沒有至少一組強制觸發條件，不同實作會出現完全不同的長連線行為：

1. 有的幾乎永不 rekey
2. 有的按時間 rekey
3. 有的只按 operator policy rekey

這會讓跨實作行為不可預測。

### 具體修改方案
至少先把三個常數與語義釘住：

1. `MAX_RECORDS_PER_GENERATION`
2. `MAX_BYTES_PER_GENERATION`
3. `MAX_KEY_AGE`

並規定：

1. 任一門檻逼近時 sender 必須發送 `KEY_UPDATE`
2. 冷卻期不適用於 mandatory rekey
3. 若對端長期不 rekey，本端可主動觸發輪換

---

## 2.4 Provisional Session 仍然建議補 `Max Records Before Auth`

### 攻擊
雖然 byte bound 已經修對，但仍然只有限制：

1. `3000ms`
2. `2048 Chameleon bytes`

還缺一個很便宜但很有效的保護：

`Max Records Before Auth`

### 為什麼目前還不夠
即使 bytes 有上限，攻擊者仍可用大量極小 record 增加 parser 與 scheduler 的
無效工作量。

### 具體修改方案
建議直接加一條：

1. `Max Records Before Auth = 2` 或 `4`
2. 超出即 `Abort Transport`

這樣 provisional boundary 才算真的閉環。

---

## 3. 建議下一輪直接落地的修改順序

如果你們要加快收斂，下一輪不要平均修改，直接按這個順序做：

1. 補出真正的 `control_plane.md`
2. 重寫 `Recovery Matrix`，顯式區分 `Detected By / Local Action / Peer-Visible Signal`
3. 為 `Epoch Cert` 補上 domain-separated signature message
4. 決定 `Caps` 的最終模型：`Offered/Required` 或 `Critical Cap Mask`
5. 刪掉 `design.md` 反駁段落對 `Session Pool` 的過度承諾
6. 同步 `protocol.md` 的握手層描述與密碼學 reference
7. 為 `KEY_UPDATE` 增加 mandatory trigger
8. 為 provisional session 增加 `Max Records Before Auth`

---

## 4. 本輪結論

這一輪最大的正面變化是：你們已經開始把批評真正落成規格，而不是只做口頭回應。
這是向前走的信號。

但這一輪最大的阻塞點也因此變得更具體：

不是 record layer 不能互通了，而是「規範體系還沒有對齊」，尤其是：

1. `control_plane.md` 缺失
2. `Recovery Matrix` 動作主體混亂
3. `Epoch Cert` 簽章上下文仍然過弱
4. `Caps` 仍然不是正式 negotiation

只要這四件事收口，下一輪 review 才值得真正往：

1. 參數選型
2. pool 調度策略
3. 性能邊界
4. 實作可測性

繼續下沉。現在還沒有到那一步。
