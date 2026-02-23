## 專業名詞解釋

---

### 1. HTTP Proxy 是什麼，以及部署服務使用它的風險

**Proxy（代理伺服器）** 是一個中間人，所有網路請求都先經過它再送出去，回應也經過它再回來。

```
你的電腦 → Proxy 伺服器 → 目標網站
```

公司使用 Proxy 的**目的**通常是：統一管控對外流量、過濾惡意網站、記錄存取紀錄、實作存取控制。

**部署服務使用 HTTP Proxy 的風險：**

**流量被攔截與記錄**：所有經過 Proxy 的請求內容（URL、Header、甚至 Body）都可能被記錄，如果是 HTTP（非 HTTPS），內容直接明文可見。

**憑證偽造（SSL Inspection）**：很多企業 Proxy 會做「中間人解密」，對 HTTPS 流量也進行解包檢查。Proxy 會偽裝成目標網站向你的服務回應，這代表你看到的「安全連線」其實已經被 Proxy 讀過一遍。這也是為什麼公司電腦需要安裝公司的 Root CA 憑證。

**單點故障**：所有服務都依賴 Proxy，Proxy 掛掉，所有對外連線就失敗。

**敏感資訊外洩風險**：若服務在請求中帶有 API Key、Token、密碼（例如放在 URL 或 Header），Proxy 管理員理論上都看得到。

---

### 2. Nexus 是什麼

**Nexus Repository Manager** 是由 Sonatype 開發的**私有套件倉庫管理工具**，可以架在公司內部網路。

它的核心功能是：

- **Proxy Repository**：作為外部 Maven Central、npm、PyPI 的快取鏡像，第一次下載後存在內部，之後不需再出去網路
- **Hosted Repository**：存放公司自己開發的套件（私有 library）
- **Group Repository**：將多個 Repository 合併成一個 URL，讓開發者只需設定一個來源

類似的工具還有 **JFrog Artifactory**，功能相近，兩者都是企業常見選擇。

```
開發者 → Nexus（內部）→ Maven Central（外部，只有 Nexus 需要出去）
               ↑
         快取已下載的套件
```

---

### 3. Air-gapped 是什麼

**Air-gapped（氣隙隔離）** 指的是一台機器或整個網路**完全與外部網路（包含網際網路）實體隔離**，沒有任何直接或間接的連線管道。

名稱來自「空氣間隙」的概念，即兩個系統之間連空氣都沒有傳導，完全切斷。

常見使用場景：

- 軍事、政府機密系統
- 核電廠、工業控制系統（SCADA）
- 金融核心系統
- 高度敏感的研發環境

在這種環境中，要更新軟體或傳入檔案，只能透過實體媒介（USB、光碟），且通常有嚴格的審查流程。

---

### 4. SSL 憑證是什麼，為什麼有存在的必要

**SSL/TLS 憑證** 是一個數位文件，用來證明「這個網站確實是它宣稱的那個網站」，同時建立加密連線。

它解決兩個問題：

**身份驗證（你在跟誰說話）**：憑證由受信任的第三方機構（CA，Certificate Authority，如 DigiCert、Let's Encrypt）簽發，確認這個網站的擁有者身份是真實的。沒有憑證機制，你連到 `bank.com`，無法確認對方真的是你的銀行，還是有人偽裝的。

**加密傳輸（別人看不到你們說什麼）**：建立憑證後，雙方會協商出一組加密金鑰，之後的通訊全部加密，中間的人截到封包也無法解讀內容。

**信任鏈的運作方式：**

```
Root CA（根憑證，內建在作業系統/瀏覽器）
  └─ Intermediate CA
       └─ 網站憑證（如 bank.com）
```

你的電腦信任 Root CA，Root CA 為 Intermediate CA 背書，Intermediate CA 為網站憑證背書，所以你信任這個網站。這就是為什麼企業自簽憑證需要讓每台電腦安裝公司的 Root CA，否則會出現「憑證不受信任」的錯誤。

---

### 5. 將 `settings.xml` 放進 `.mvn/` 版控的風險

`settings.xml` 可能含有**不應該公開的敏感資訊**：

**帳號密碼**：連接 Nexus 的認證資訊，若直接寫在檔案裡，進版控就等於全團隊（甚至未來有權限看 Repo 的人）都能看到。

```xml
<!-- 這樣的內容絕對不能進版控 -->
<server>
  <id>nexus</id>
  <username>deploy_user</username>
  <password>MySecretPassword123</password>
</server>
```

**內部網路資訊暴露**：Nexus 的內部 URL、Proxy 的 Host/Port，這些資訊對內部人員無害，但如果 Repo 有對外（如 GitHub 公開）就會洩漏網路架構。

**解決方法**：使用 Maven 的**加密密碼機制**，或將敏感部分改由環境變數注入：

```xml
<password>${env.NEXUS_PASSWORD}</password>
```

這樣檔案可以安全地進版控，實際密碼由 CI/CD 環境變數或 Secrets Manager 提供。

---

### 6. `.mvn/settings.xml` 納入版控的資安風險評估

**結論：取決於檔案內容，而不是行為本身。**

**可以安全版控的內容：**
- Mirror URL（內部 Nexus 位址）
- Proxy Host/Port（非密碼）
- Repository 設定

**絕對不能版控的內容：**
- 明文密碼
- Access Token
- 私鑰

**額外需要考量的情境：**

如果這個 Repo 是**私有的（Private）**，風險相對較低，但仍要考慮內部人員的最小權限原則，不是每個開發者都需要知道 deploy 帳號的密碼。

如果這個 Repo 有任何可能**對外公開**，所有內部網路資訊和帳密都是高風險洩漏。

**建議的安全做法：**

版控一份**模板**（`settings.xml.template`），敏感值用佔位符表示，搭配文件說明開發者如何填入自己的認證，或由 CI/CD 的 Secrets 機制自動注入，這樣既能達到「設定共用」的目的，又不會有資安疑慮。

---

## 關於 HTTP Proxy 與單點故障的釐清

你的理解方向是對的，但要分清楚**兩個不同階段**：

**安裝/建構階段**（你說的情境）：`./mvnw clean install` 時，Maven 透過 Proxy 去 Nexus 下載套件。這個階段 Proxy 確實是中間人，如果 Proxy 掛掉，下載就會失敗。所以**這裡仍然存在單點故障**。

**但是**，如果你的 Nexus 已經快取了所有需要的套件（第一次有人下載過），之後即使 Proxy 掛掉、甚至對外網路中斷，只要**開發者能直接連到 Nexus**（Nexus 在內網），就不需要經過 Proxy。

```
有 Proxy 的情境：
開發者 → Proxy → Nexus（內網）   ← Proxy 掛掉就失敗

直連內網 Nexus 的情境：
開發者 → Nexus（內網）           ← 不需要 Proxy，無單點故障
```

所以關鍵問題是：**你的 Nexus 是在內網還是外網？**
- 內網 Nexus → 通常不需要 Proxy，直接連，單點故障問題消失
- 外網服務 → 需要 Proxy，單點故障問題存在

`settings.xml` 裡設定 Proxy，主要是為了讓 Maven **能出去外網**，而不是為了連內網 Nexus。

---

## settings.xml Template

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0
                              https://maven.apache.org/xsd/settings-1.2.0.xsd">

  <!-- =========================================================
       本機 Maven Repository 快取位置
       預設是 ~/.m2/repository，通常不需要改
  ========================================================= -->
  <localRepository>${user.home}/.m2/repository</localRepository>

  <!-- =========================================================
       Proxy 設定
       用途：讓 Maven 透過公司 Proxy 連外網
       如果 Nexus 在內網且不需要出外網，可以整段移除
  ========================================================= -->
  <proxies>
    <proxy>
      <id>company-proxy</id>
      <active>true</active>
      <protocol>https</protocol>
      <host>${env.PROXY_HOST}</host>           <!-- 例如 proxy.company.com -->
      <port>${env.PROXY_PORT}</port>           <!-- 例如 8080 -->
      <!-- 如果 Proxy 需要帳號密碼，取消下面兩行註解 -->
      <!-- <username>${env.PROXY_USER}</username> -->
      <!-- <password>${env.PROXY_PASS}</password> -->
      <!-- 不需要經過 Proxy 的位址，用 | 分隔 -->
      <nonProxyHosts>localhost|127.0.0.1|*.company.com|nexus.company.com</nonProxyHosts>
    </proxy>
  </proxies>

  <!-- =========================================================
       Server 認證資訊
       用途：連接需要帳號密碼的 Repository（如 Nexus）
       id 必須與下方 mirror 或 repository 的 id 對應
       
       ⚠️ 密碼請用環境變數，不要寫明文進版控
  ========================================================= -->
  <servers>

    <!-- Nexus 一般下載用帳號 -->
    <server>
      <id>nexus-public</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>

    <!-- Nexus 部署（上傳 artifact）用帳號，權限通常與下載不同 -->
    <server>
      <id>nexus-releases</id>
      <username>${env.NEXUS_DEPLOY_USER}</username>
      <password>${env.NEXUS_DEPLOY_PASSWORD}</password>
    </server>

    <server>
      <id>nexus-snapshots</id>
      <username>${env.NEXUS_DEPLOY_USER}</username>
      <password>${env.NEXUS_DEPLOY_PASSWORD}</password>
    </server>

  </servers>

  <!-- =========================================================
       Mirror 設定
       用途：將所有外部 Repository 請求攔截，導向內部 Nexus
       mirrorOf=* 代表攔截所有來源
       
       設定後，pom.xml 裡不管寫了什麼 repository，
       都會被導向這裡，確保所有流量走內網
  ========================================================= -->
  <mirrors>
    <mirror>
      <id>nexus-public</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.company.com/repository/maven-public/</url>
      <!-- 
        maven-public 通常是 Nexus 的 Group Repository，
        同時包含：
          - proxy/maven-central（快取 Maven Central）
          - hosted/maven-releases（公司自己的 release 套件）
          - hosted/maven-snapshots（公司自己的 snapshot 套件）
      -->
    </mirror>
  </mirrors>

  <!-- =========================================================
       Profile 設定
       用途：定義可重複使用的環境設定組合
       可以用來區分 dev / staging / prod 等環境
  ========================================================= -->
  <profiles>
    <profile>
      <id>nexus</id>

      <!-- 覆寫 release 套件的下載來源 -->
      <repositories>
        <repository>
          <id>nexus-public</id>
          <url>https://nexus.company.com/repository/maven-public/</url>
          <releases>
            <enabled>true</enabled>
          </releases>
          <snapshots>
            <enabled>false</enabled>
          </snapshots>
        </repository>
      </repositories>

      <!-- 覆寫 Maven Plugin 的下載來源 -->
      <pluginRepositories>
        <pluginRepository>
          <id>nexus-public</id>
          <url>https://nexus.company.com/repository/maven-public/</url>
          <releases>
            <enabled>true</enabled>
          </releases>
          <snapshots>
            <enabled>false</enabled>
          </snapshots>
        </pluginRepository>
      </pluginRepositories>

    </profile>
  </profiles>

  <!-- =========================================================
       啟用的 Profile
       將上面定義的 profile 預設啟用
  ========================================================= -->
  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>

</settings>
```

---

## 各區塊對應的使用情境整理

| 區塊 | 什麼時候需要設定 |
|------|----------------|
| `localRepository` | 想改快取位置時（預設通常夠用） |
| `proxies` | Maven 需要出外網，且公司有 Proxy |
| `servers` | Nexus 需要帳號密碼認證 |
| `mirrors` | 要把所有下載導向內部 Nexus |
| `profiles` + `pluginRepositories` | 確保 Maven Plugin 也走內網（容易被遺漏） |

`pluginRepositories` 是很多人會漏掉的地方，Maven Plugin（如 `maven-compiler-plugin`）的下載來源與一般 dependency 是分開的，如果只設定 `repositories` 沒設定 `pluginRepositories`，Plugin 可能還是會嘗試連外網。
