## Maven vs Maven Wrapper 的差異

### Maven（傳統安裝方式）

需要在每台機器上手動安裝 Maven，並設定環境變數（`MAVEN_HOME`、`PATH`）。團隊中每個人或每台 CI 伺服器都需要自行管理 Maven 版本。

### Maven Wrapper（`mvnw`）

Maven Wrapper 是一個隨專案附帶的小型腳本（`mvnw` / `mvnw.cmd`），讓你**不需要預先安裝 Maven** 就能執行建構。專案裡會包含以下檔案：

```
.mvn/
  wrapper/
    maven-wrapper.jar
    maven-wrapper.properties   ← 指定使用的 Maven 版本
mvnw        ← Linux/Mac 腳本
mvnw.cmd    ← Windows 腳本
```

---

## Maven Wrapper 的優勢

**版本一致性**是最核心的優勢。`maven-wrapper.properties` 明確鎖定 Maven 版本，所有開發者、CI/CD 環境都使用完全相同的版本，避免「在我電腦上可以跑」的問題。

**零安裝門檻**。新成員 clone 專案後直接執行 `./mvnw clean install`，不需要額外安裝任何東西，Wrapper 會自動下載對應版本。

**CI/CD 更簡潔**。不需要在 pipeline 腳本裡特別安裝或指定 Maven 版本，直接用 `./mvnw` 即可。

**版本升級更安全**。升級 Maven 版本只需修改 `maven-wrapper.properties` 並提交，所有人同步更新，不會有人忘記。

---

## 簡單比較

| 項目 | Maven | Maven Wrapper |
|------|-------|---------------|
| 安裝需求 | 需手動安裝 | 不需要 |
| 版本控制 | 各自管理，容易不一致 | 鎖定於專案中 |
| CI/CD 設定 | 需額外步驟 | 直接使用 `./mvnw` |
| 新人上手 | 需先設定環境 | Clone 即可執行 |

---

**結論**：現代 Java 專案（尤其是 Spring Boot）幾乎都推薦使用 Maven Wrapper，Spring Initializr 預設也會產生 Wrapper 檔案。

---

## 受限環境使用 Maven Wrapper 的常見問題與設定

---

### 核心問題：自動下載被封鎖

Maven Wrapper 預設行為是**從網路下載指定版本的 Maven**，在封閉的企業環境中這通常會失敗，原因包括：

- 無法連線到外部網路（air-gapped 環境）
- 需要通過 Proxy 才能上網
- 公司防火牆封鎖 `https://repo.maven.apache.org`

---

### 問題一：需要設定 Proxy

如果公司有 HTTP Proxy，需在以下地方設定：

**`~/.m2/settings.xml`**
```xml
<settings>
  <proxies>
    <proxy>
      <id>company-proxy</id>
      <active>true</active>
      <protocol>https</protocol>
      <host>proxy.company.com</host>
      <port>8080</port>
      <username>user</username>       <!-- 如果需要認證 -->
      <password>password</password>
      <nonProxyHosts>localhost|internal.company.com</nonProxyHosts>
    </proxy>
  </proxies>
</settings>
```

**或透過 JVM 參數（`MAVEN_OPTS`）**
```bash
export MAVEN_OPTS="-Dhttps.proxyHost=proxy.company.com -Dhttps.proxyPort=8080"
```

---

### 問題二：Wrapper 下載來源需改為內部位址

`maven-wrapper.properties` 預設指向外部，需改為公司內部的 Artifact Repository（如 Nexus、Artifactory）：

**`.mvn/wrapper/maven-wrapper.properties`**
```properties
# 原本（外部）
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip

# 改為內部 Mirror
distributionUrl=https://nexus.company.com/repository/maven-public/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
```

同樣地，`maven-wrapper.jar` 本身也可以指定來源：
```properties
wrapperUrl=https://nexus.company.com/repository/maven-public/org/apache/maven/wrapper/maven-wrapper/3.2.0/maven-wrapper-3.2.0.jar
```

---

### 問題三：完全離線環境（Air-gapped）

如果根本無法下載，可以採用**預先下載並內嵌**的方式：

**方法 A：將 Maven 壓縮檔放進 Repo**

把 `apache-maven-x.x.x-bin.zip` 上傳到 Nexus/Artifactory，再修改 `distributionUrl` 指向它。

**方法 B：使用 `distributionPath` 本地路徑（不推薦共用）**
```properties
distributionUrl=file:///opt/maven/apache-maven-3.9.6-bin.zip
```

**方法 C：直接 commit Maven 解壓縮檔進 Repo（極端做法，不推薦）**

---

### 問題四：SSL 憑證問題

公司內部若使用自簽憑證（Self-signed Certificate），Wrapper 下載時會出現 SSL 錯誤。

**解法：將憑證加入 JVM truststore**
```bash
keytool -import -alias company-ca \
  -keystore $JAVA_HOME/lib/security/cacerts \
  -file company-ca.crt \
  -storepass changeit
```

**或臨時停用驗證（不推薦用於正式環境）**
```properties
# maven-wrapper.properties
wrapperProperties.skipSslValidation=true
```

---

### 問題五：Maven 相依套件下載來源（settings.xml Mirror）

Wrapper 只解決 Maven 本身的下載，**專案的 dependencies 也需要導向內部 Repository**：

**`~/.m2/settings.xml` 或專案指定的 `settings.xml`**
```xml
<mirrors>
  <mirror>
    <id>nexus-central</id>
    <mirrorOf>*</mirrorOf>   <!-- 攔截所有外部請求 -->
    <url>https://nexus.company.com/repository/maven-public/</url>
  </mirror>
</mirrors>
```

執行時指定 settings：
```bash
./mvnw -s .mvn/settings.xml clean install
```

> 建議將公司的 `settings.xml` 放進 `.mvn/` 目錄一起版控，確保所有人使用相同設定。

---

### 整體建議流程

```
公司環境設定清單：

1. Nexus/Artifactory 建立 Maven proxy repo
2. 上傳 maven wrapper 所需的 zip 至內部 repo
3. 修改 maven-wrapper.properties → distributionUrl 指向內部
4. 設定 settings.xml mirror 攔截所有 dependency 請求
5. 處理 SSL 憑證信任問題
6. 將 .mvn/settings.xml 納入版控供團隊共用
```

這樣就能讓 Maven Wrapper 在完全受控的環境下正常運作，同時維持「clone 即可建構」的優勢。
