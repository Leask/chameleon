# Chameleon Review
## 對最新 `design.md` 與 `protocol.md` 的新一輪聯合審查

這一輪和上一輪不同，重點已經不是把舊問題重複一遍，而是判斷哪些修改真的站住了，哪些只是把問題從一個文件搬到了另一個文件。整體結論是：

1. 有幾個關鍵點確實被修掉了。
2. 但目前最大的阻塞點已經變成「規範體系沒有真正閉環」。
3. 也就是說，大方向比上一輪更正確，但規格化程度還沒有跟上。

下面先確認這一輪已經有效收斂的部分，再集中攻擊仍然阻塞互通與運營的問題。

---

## 0. 本輪已經有效修掉的問題

### 0.1 Record Layer 的完整性綁定終於補上了
`protocol.md` 現在已經明確把 `Plain_Header = Length || Flags` 納入 AEAD `AAD`。
這是實質性修正，不是文字修飾。上一輪對 record layer 的核心批評，
這一輪可以正式降級。

### 0.2 Provisional Bounds 終於回到了協議層可計量範圍
`Max Inbound Bytes` 現在明確改成只計算 Chameleon 自身 bytes，
不再把 TCP/QUIC/WebSocket 下層開銷算進來。這是對的。

### 0.3 `design.md` 對握手層與 Client Auth 的職責劃分已經更準確
`design.md` 現在明確說明握手層只負責 server-side trust establishment、
key exchange 和 capability negotiation，客戶端授權在握手後第一個加密
record 完成。這一點比前幾輪清楚很多。

### 0.4 `Session Pool` 的主敘述已經比前幾輪更誠實
`design.md` 在正式架構段落裡已經不再聲稱 pool 能消滅 HOL，
而是退回到「縮小故障域、顯著緩解」。這個方向是正確的。

---

## 1. 阻塞級問題

## 1.1 `design.md` 現在依賴 `control_plane.md`，但倉庫裡根本沒有這個規格

### 攻擊
這一輪最嚴重的新問題不是某個 frame，而是文檔體系本身。

`design.md` 已經明確宣告：

1. `CPSK Certificate`
2. `Node Manifest`
3. anti-rollback
4. 控制面生命週期

都由獨立的 `control_plane.md` 規範。

問題是：目前倉庫裡沒有這個文件。這代表兩級 PKI 的問題並沒有真正被解決，
只是被從 `design.md` 與 `protocol.md` 移出了視野。

這是阻塞級，因為 `protocol.md` 第 3.3 節的 `Epoch Cert` 驗證流程已經依賴
本地有效 `CPSK`、`Signer Key ID` 對應、以及 OOB 下發的 responder key。
只要 `control_plane.md` 不存在，這整條鏈就仍然不是規格，只是口頭前提。

### 為什麼現在還不夠
目前實際狀態是：

1. `design.md` 說 `protocol.md` 是唯一 SSoT。
2. `design.md` 同時又說 `control_plane.md` 也是獨立規範。
3. 但 `control_plane.md` 不存在。

這裡出現了兩層問題：

1. 規範文件分工已經變了，但文檔總體說法還沒同步。
2. 最需要被規範化的 OOB 信任物件，現在仍然沒有正式規格。

### 具體修改方案
下一輪不要再繼續在 `design.md` 內描述 `control_plane.md` 的存在，
而應該直接把它補出來，至少包含：

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
5. key selection 規則
6. OOB 更新與撤銷語義

同時把 `design.md` 的職責描述改成：

1. `design.md`: 邊界與取捨
2. `protocol.md`: 資料面 wire format 與狀態機
3. `control_plane.md`: OOB 信任物件與控制面生命週期

在 `control_plane.md` 真正出現之前，兩級 PKI 不能算已收斂。

---

## 1.2 `Epoch Cert` 仍然沒有 domain-separated signature message

### 攻擊
這一輪 `Epoch Cert` 的兩級信任方向仍然是對的，但簽章訊息本身還是不夠嚴謹。
現在的規則依然是直接驗證前 56 Bytes 的拼接結果。

這個設計的問題不是不能工作，而是它太依賴當前的欄位排列，沒有明確的
物件類型上下文。只要未來 `CPSK` 需要簽別的控制面物件，或者 `Epoch Cert`
本身要擴欄位，這個簽章模型就會開始變脆。

### 為什麼現在還不夠
這一輪你們已經給 `CLIENT_AUTH` 補上了：

`"chameleon-auth-v1" || h`

也就是說，你們已經承認 domain separation 是必需品。那 `Epoch Cert`
就不應該停留在「簽固定位移前綴」這種過渡狀態。

### 具體修改方案
把 `Epoch Cert` 的驗證規則改成明確的版本化 body：

```text
EpochCertBody =
    ServerStaticPubkey(32) ||
    Epoch_Window_Start(8) ||
    Epoch_Window_End(8) ||
    Signer_Key_ID(8)

EpochCertSigMessage =
    "chameleon-epoch-cert-v1" || EpochCertBody
```

並強制規定：

1. 所有整數為大端序
2. 字串是 ASCII bytes，不含 null
3. `Ed25519 Control Plane Sig` 驗證的 message 就是上面的
   `EpochCertSigMessage`

如果未來要擴展 `Epoch Cert`，那就增加新版本，不要繼續沿用
「對前 56 Bytes 簽章」這種脆弱約定。

---

## 1.3 `Caps` 仍然不是 negotiation，而只是帶歧義的配置比對

### 攻擊
這一輪你們仍然保留：

1. client 發一個 `Caps`
2. server 回 `Selected Caps`
3. client 若「關鍵能力」未滿足，可主動 abort

問題還和上一輪一樣，但現在更明顯：文檔依然沒有形式化定義
什麼叫「關鍵能力」。

這代表當 capability 位開始增長時，不同實作會對「偏好」和「必要」
做出不同判斷，最後得到不同行為。

### 為什麼現在還不夠
目前只有 Datagram 這一個保留 bit，所以問題還沒有爆出來。
但只要未來真的有更多 capability，現在這種寫法會直接製造互通分裂。

現在的 `Caps` 其實不是 negotiation，而是：

1. client 發出能力意圖
2. server 做子集裁剪
3. client 本地用未定義規則決定要不要接受

這不是嚴格協議語義。

### 具體修改方案
有兩種可接受修法，最好二選一，不要再拖：

1. 直接把 client 首包改成：
   - `Offered Caps`
   - `Required Caps`
2. 保持現有 wire format，但新增一個明確的 `Critical Cap Mask`

如果採第一種，規則應該是：

1. `Selected Caps` 必須是 `Offered Caps` 的子集
2. 若 `Required Caps & ~Selected Caps != 0`，client 必須立即中止
3. 這種失敗在 recovery matrix 中應叫 `Capability Mismatch`

如果採第二種，也必須把 `Critical Cap Mask` 的含義寫死，
不能再用「關鍵能力」這種散文說法。

---

## 1.4 `design.md` 的正式架構段落已經修正，但反駁段落仍然宣稱 `Session Pool` 能完美解決 HOL

### 攻擊
這是目前最明顯的文檔自我衝突。

`design.md` 前面的正式架構段落已經正確寫成：

`Session Pool` 不能消滅 HOL，只能把故障域限制在單一 session。

但後面的反駁段落仍然保留「完美解決 HOL 阻塞問題」這種過度結論。
這不是措辭小問題，而是會直接污染之後所有性能討論。

### 為什麼現在還不夠
這一輪你們已經把 `Health & Replenishment`、`No-New-Open` 補進去了，
說明設計本身其實已經接受 pool 只是彌補，而不是魔法修復。

既然如此，反駁段落還保留「完美解決」只會帶來兩個壞處：

1. 讓讀文檔的人對實際性能上限產生錯誤預期
2. 掩蓋單一 session 內控制流量仍可能被 bulk 影響的事實

### 具體修改方案
把反駁段落整段改成下面這種語氣：

1. `Session Pool` 的目標是把 HOL 的代價限制在單一 session
2. 控制流/互動流與 bulk 流必須分配到不同 session
3. 這是架構層彌補，不是協議層消滅 HOL

並補一條明確規則：

當某條 session 被標記為 `Bulk` 或出現持續高排隊延遲時，
不得承接新的 control/interactive opens。

---

## 1.5 Recovery Matrix 仍然過時，而且現在已經出現了內部錯碼

### 攻擊
這一輪 `Recovery Matrix` 仍然是高風險區，因為它不只不夠細，
現在還出現了明顯的內部不一致：

`GOAWAY` 錯誤碼表裡：

1. `0x03 = Flow Control Overflow`
2. `0x04 = Decrypt / Integrity Violation`

但 `Recovery Matrix` 卻寫：

`Session Flow Control Overflow -> GOAWAY (Code: 0x04)`

這是明確錯碼，不是 interpretation 問題。

### 為什麼現在還不夠
除了錯碼之外，矩陣本身仍然沒有跟這一輪的新設計同步，至少缺少：

1. `Signer Key ID Not Found`
2. `Epoch Cert Signature Invalid`
3. `Responder Key Mismatch`
4. `Capability Mismatch`
5. `Unsupported Auth Scheme`
6. `Manifest / CPSK Refresh Required`
7. `GOAWAY 0x00` 在 session pool 下的正確恢復路徑

現在最不合理的一條是：

`Server Maintenance/Epoch Expiring -> Switch Node`

在引入 session pool 之後，這不應再是第一反應。更合理的默認路徑應該是：

1. 先在池內補一條 fresh session
2. 新 opens 改派到新 session
3. 只有當 node 本身不可用時才切 route / switch node

### 具體修改方案
下一輪至少做這幾個修改：

1. 先修正錯碼：
   - `Session Flow Control Overflow -> GOAWAY 0x03`
2. 把 recovery matrix 擴成至少包含：
   - `Signer Key ID Not Found`
   - `Epoch Cert Signature Invalid`
   - `Responder Key Mismatch`
   - `Capability Mismatch`
   - `Unsupported Auth Scheme`
   - `Manifest Rollback Detected`
3. 把 `GOAWAY 0x00` 的 client 恢復改成：
   - `Replenish Fresh Session In Pool`
   - 不是直接 `Switch Node`
4. 把 `Client Auth Fail` 再拆成：
   - `Credential Revoked / Unknown`
   - `Proof Invalid`
   - `Unsupported Auth Scheme`

這一項如果不修，控制層與 pool manager 之後都會建立在錯誤事件分類上。

---

## 2. 高優先級但非阻塞問題

## 2.1 `protocol.md` 的概述仍然把握手層寫成泛化的「身份認證層」

### 攻擊
`design.md` 這一輪已經把握手層描述修正了，但 `protocol.md` 開頭仍然寫：

`Handshake Layer: 負責身份認證、金鑰交換與能力協商`

這和當前設計不一致。現在真正的模型是：

1. 握手層負責 server-side trust establishment
2. client auth 在 `CLIENT_AUTH`

### 具體修改方案
把 `protocol.md` 的概述同步改成：

`Handshake Layer: 負責伺服器側信任建立、金鑰交換與能力協商；客戶端授權在握手後的 CLIENT_AUTH 階段完成。`

這個改動很小，但應該立刻做，因為它會影響所有後續實作對狀態機的理解。

---

## 2.2 密碼學參考仍然寫錯

### 攻擊
`protocol.md` 依然寫：

`Noise Protocol Framework: RFC 7748`

這是錯的。RFC 7748 是 X25519/X448，不是 Noise 規格本身。

### 具體修改方案
把參考拆開：

1. `Noise Protocol Framework`: 引用 Noise 規格本身
2. `RFC 7748`: 作為 Curve25519/X25519 參考

這不是吹毛求疵，而是規格嚴肅性的基本要求。

---

## 2.3 `KEY_UPDATE` 已經有狀態機，但仍然只有冷卻期，沒有強制觸發條件

### 攻擊
這一輪 `KEY_UPDATE` 的 state machine 已經能工作，但還沒有回答：

1. 什麼時候必須 rekey
2. 哪些條件下只建議 rekey

現在只有「兩次 `KEY_UPDATE` 之間至少相隔 1000 個 records 或 1 分鐘」。
這只是 anti-thrash 冷卻期，不是 rekey policy。

### 為什麼現在還不夠
如果沒有至少一組 mandatory trigger，不同實作會出現完全不同的長連線行為：

1. 有的幾乎永不 rekey
2. 有的按時間 rekey
3. 有的只在 operator 指令下 rekey

這會讓互通測試很難穩定。

### 具體修改方案
下一輪至少新增三個規範常數：

1. `MAX_RECORDS_PER_GENERATION`
2. `MAX_BYTES_PER_GENERATION`
3. `MAX_KEY_AGE`

然後規定：

1. 任一門檻逼近時，sender 必須發送 `KEY_UPDATE`
2. 冷卻期不適用於 mandatory rekey
3. 若對端長期不 rekey，本端可主動觸發

如果還不想定死數值，至少先把常數名與語義釘住。

---

## 2.4 Provisional Session 雖然已經改成純協議位元組，但還缺少 record-count 上限

### 攻擊
這一輪你們已經把 byte bound 修對了，這點很好。
但目前仍然只有：

1. 3000ms timeout
2. 2048 Chameleon bytes 上限

還缺一個很簡單但很實用的保護：

`Max Records Before Auth`

### 為什麼現在還不夠
即使 byte bound 存在，攻擊方仍然可以用大量極小 record 把 parser、
scheduler 或 state tracking 拖進不必要的工作量。

### 具體修改方案
在 `CLIENT_AUTH` 的 provisional 規則裡再加一條：

1. `Max Records Before Auth = 2` 或 `4`
2. 一旦超出，立即 `Abort Transport`

這樣 provisional session 的資源邊界才算真正閉環。

---

## 3. 建議下一輪直接落地的修改順序

如果目標是加快收斂，下一輪建議不要平均修，而是按下面順序做：

1. 先把 `control_plane.md` 真正補出來
2. 修正 `Epoch Cert` 的 signature message
3. 修正 recovery matrix 的錯碼與缺項
4. 決定 `Caps` 最終模型：`Offered/Required` 或 `Critical Cap Mask`
5. 刪掉 `design.md` 反駁段落裡對 `Session Pool` 的過度承諾
6. 同步 `protocol.md` 概述與密碼學 reference
7. 為 `KEY_UPDATE` 增加 mandatory trigger
8. 為 provisional session 增加 `Max Records Before Auth`

---

## 4. 本輪結論

這一輪最大的正面變化是：你們確實開始把前幾輪批評轉化成規格了，
而不再只是做語義回應。這很重要。

但現在最大的阻塞點也因此變得更清楚：

不是你們不知道要怎麼設計，而是規範文件還沒有真的閉環。

具體來說，當前最該先釘死的四件事是：

1. `control_plane.md` 必須出現
2. `Epoch Cert` 的簽章上下文必須版本化
3. recovery matrix 必須和錯誤碼表、session pool 行為同步
4. `Caps` 必須從「散文 negotiation」變成正式 negotiation

只要這四件事補齊，下一輪 review 才有資格進一步下沉到：

1. 參數選型
2. 連線池策略
3. 實作可測性
4. 資源上限與性能預期

否則現在仍然是規格邊界沒有封口，而不是實作細節問題。
