# Chameleon 協議架構設計白皮書 v2.0 
*(基於安全邊界與工程落地的重構版本)*

## 1. 演進概述與設計哲學反思

在 Chameleon v1.0 的初步構想中，我們試圖透過「WebTransport 承載 + 隨機擾動 + 應用層激進擁塞控制 + 反向代理掩護」來構建一個抗審查網路。經過嚴格的安全審查（參見 `design_1.md`），我們認知到 v1.0 存在根本性的邏輯漏洞：
1. **「擾動」不等於「隱蔽」**：隨機的 Padding 與 Jitter 會創造出全新的統計學特徵，反而更容易被機器學習模型捕捉。
2. **「相變」是致命弱點**：所謂的「階段性卸載偽裝（前 10 秒重度偽裝，後續火力全開）」會產生極其明顯的時序相變特徵（Temporal Phase-shift signature）。
3. **應用層與傳輸層的內耗**：在應用層強加 BBRv3 會與 QUIC 原生的擁塞控制產生衝突，導致嚴重的緩衝區膨脹（Bufferbloat）與 Goodput 下降。
4. **反向代理探測防護過於脆弱**：簡單的反代真實網站無法抵抗差分探測（Differential Probing）與邊界條件測試。

因此，**Chameleon v2.0 的設計哲學從「試圖用魔法欺騙審查」轉向「基於密碼學與行為一致性的工程落地」**。我們不再追求理論上的極限速度，而是追求**極致的穩定性、可觀測性與指紋一致性**。

---

## 2. 威脅模型定義 (Threat Model)

在設計任何防禦機制前，必須先明確對手（如 GFW 或企業級防火牆）的能力邊界：

*   **被動監聽與分析 (Passive Observation):**
    *   具備全流量深度封包檢測 (DPI) 能力。
    *   具備提取 TLS/QUIC 指紋 (如 JA3/JA4, QUIC Version, Cipher Suites, ALPN) 的能力。
    *   具備基於機器學習的流特徵分析能力 (封包大小分佈 Packet Size Distribution, 到達時間間隔 Inter-Arrival Time, 流量上下行比例)。
*   **主動探測與干預 (Active Probing & Intervention):**
    *   具備重放攻擊 (Replay Attacks) 能力。
    *   具備差分探測能力 (發送畸形封包、半開連線、異常 HTTP Header 來比對伺服器反應)。
    *   具備動態 QoS 限速與隨機丟包注入能力。
*   **系統級威脅:**
    *   伺服器 IP / ASN 可能被無差別封鎖。
    *   伺服器一旦失陷，不能導致歷史通訊內容被解密 (需要前向保密 Forward Secrecy)。

---

## 3. 系統核心架構：三層解耦模型

為保證系統的可維護性與多端演進，Chameleon v2.0 放棄單體協議設計，採用嚴格的三層解耦架構：

### 3.1 傳輸與偽裝層 (Transport & Camouflage Adapter - TCA)
**職責：** 負責底層連通性，並確保網路表面的指紋與主流客戶端（如 Chrome/iOS）完全一致。
*   **拒絕土砲 QUIC：** 不再自己手寫 QUIC 參數。客戶端強制使用經過嚴格指紋偽裝的網路庫（例如基於 `uTLS` 思想的 QUIC 實現），確保 Client Hello、版本協商、Pacing (發包節奏) 與標準瀏覽器毫無二致。
*   **可替換承載：** 雖然預設使用 HTTP/3 (QUIC)，但 TCA 層設計為介面化，未來可隨時熱切換為 WebSocket+TLS 或 WebRTC，而不需要修改上層邏輯。

### 3.2 安全會話層 (Secure Session Layer - SSL)
**職責：** 處理身份認證、金鑰交換、重放防護與會話生命週期。
*   **身份與傳輸剝離：** 不再將認證資訊粗暴地塞入 TLS 擴展。
*   **Noise Protocol Framework：** 採用 Noise 協議框架（如 `Noise_IK` 模式）或基於 TLS 1.3 Out-of-Band PSK 進行會話建立。保證 0-RTT 連線的同時，提供嚴格的前向保密 (PFS) 與重放保護。
*   **金鑰輪換：** 每個會話具備獨立的 Session Key，並支援基於時間或流量閾值的無縫金鑰輪換 (Key Rekeying)。

### 3.3 轉發與多路復用引擎 (Forwarding & Multiplexing Engine - FME)
**職責：** 處理實際的代理請求、多路復用、DNS 解析與路由分流。
*   **原生 QUIC Streams：** 充分利用 QUIC 原生的 Stream 機制進行多路復用，徹底解決隊頭阻塞，無需在應用層重複造輪子（拋棄 Yamux/smux）。
*   **內部路由態：** 支援 Split Routing（國內外分流），內建 DoH/DoT 解析器以防止 DNS 污染洩漏。

---

## 4. 流量特徵對抗：結構化行為模版

針對流特徵分析，v2.0 徹底拋棄「隨機擾動」與「相變卸載」，改為採用**結構化流量模版 (Structured Traffic Profiles)**。

1.  **全局生命週期一致性：** 
    一條底層 QUIC 連線從建立到關閉，必須始終符合單一的流量模版（例如：它偽裝成一個正在看 YouTube 的連線，就必須從頭到尾保持視訊串流的特徵）。如果使用者行為發生巨大改變（例如從看網頁變成下載 BT），客戶端會**新開一條連線**套用新的模版，而不是在原連線中產生相變。
2.  **確定性填充 (Deterministic Padding)：** 
    不使用隨機亂數作為 Padding，也不在封包中明文傳輸 Padding 長度。客戶端與伺服器利用 Session Key 衍生出的偽隨機數生成器 (PRNG) 同步計算出每個封包應有的 Padding 長度。這保證了極低的 CPU 開銷，同時對外部觀察者呈現出完美的模版特徵。
3.  **模版實例：**
    *   `Profile-Web`: 模擬零碎、高併發、短時的網頁資源載入特徵。
    *   `Profile-Stream`: 模擬穩定的下行大流量、週期性的 ACK 上行特徵。

---

## 5. 認證與防探測：零信任靜默回退 (Zero-Trust Fallback)

為了徹底解決主動探測與差分比對的問題，v2.0 放棄了脆弱的「反向代理」掩護，改為**本機真實靜態掩護**。

1.  **真實的誘餌站點：** 伺服器端部署一個標準的 Web Server (例如 Caddy)，並在本機真實掛載一個完整的靜態網站（例如一個無害的個人開源技術部落格）。
2.  **單封包授權 (SPA) 與透明升級：** Chameleon 客戶端的首個請求是一個完全合法的 HTTP/3 請求，但其 HTTP Header 中包含一段經過 AEAD 加密的認證票證 (Ticket)。
3.  **防禦邏輯：**
    *   如果審查系統發送主動探測封包（無論是普通的 GET 請求，還是畸形的 TLS 握手），因為沒有合法的 Ticket，Caddy 會像一個正常的 Web Server 一樣，返回真實的靜態部落格內容或標準的 404/400 錯誤。
    *   這個誘餌網站的行為、快取策略、Header 順序完全真實，因為它**就是**一個真實的網站。審查系統無法透過差分比對找出破綻。

---

## 6. 性能優化：以 Goodput 與穩定性為核心

1.  **回歸傳輸層擁塞控制：** 刪除所有應用層的激進 BBR 邏輯。統一交由底層的 QUIC 協議堆疊處理擁塞控制（優先推薦 BBRv2 或 Cubic）。
2.  **Bufferbloat 抑制：** 在 FME 層引入 CoDel (Controlled Delay) 或 FQ (Fair Queuing) 隊列管理算法，確保在跨國弱網環境下，Ping 值與尾延遲 (Tail Latency) 保持在可用範圍內，提升網頁「秒開」的體感。
3.  **連線遷移 (Connection Migration)：** 充分利用 QUIC 的連線遷移特性。當行動裝置從 Wi-Fi 切換到 5G 網路時，底層 IP 雖然改變，但 Chameleon 會話不會中斷，代理流量無縫接續。

---

## 7. 多端落地與部署抽象

1.  **客戶端接入層 (Client Local Interface)：**
    *   提供標準的 SOCKS5 與 HTTP Proxy 介面供桌面軟體接入。
    *   提供 TUN/TAP 虛擬網卡模式，支援全局路由劫持（特別適用於 iOS/Android 系統級 VPN API）。
2.  **控制面與資料面分離 (Control / Data Plane Separation)：**
    *   資料面 (Data Plane) 只負責高效轉發。
    *   控制面 (Control Plane) 負責節點配置拉取、金鑰輪換通知與遙測 (Telemetry) 數據回傳。
3.  **無狀態伺服器 (Stateless Server)：**
    伺服器端設計為無狀態架構，僅持有根金鑰 (Root Key)。所有會話狀態由客戶端提供的 Ticket 衍生。這使得伺服器可以極其輕量地部署於 Serverless 容器或基礎 VPS 中，且支援毫秒級的災難重啟。

---

## 8. 下一步工程計畫 (Next Steps)

在進入代碼編寫前，後續的工程推進應優先完成以下驗證：
1.  **指紋一致性驗證：** 抓包比對 Chameleon TCA 層的 QUIC Client Hello 與真實 Chrome 瀏覽器的差異，確保 JA3/JA4 雜湊值完全一致。
2.  **確定性填充 PoC：** 實作基於 PRNG 的同步 Padding 算法，並測量其在 ARM 設備上的 CPU 與電量消耗。
3.  **誘餌站點差分測試：** 使用自動化掃描工具（如 Nmap, Zmap）對 Chameleon 伺服器進行掃描，確保其反應與普通 Caddy 伺服器在統計學上不可區分。