要理解 **Bean**，最簡單的方法是把它想像成**「被 Spring 容器託管的機器零件」**。

在傳統的 Java 開發中，如果你需要一個工具（物件），你會用 `new` 來建立它：
`FileService service = new FileService();`

但在 Spring Boot 中，我們不自己 `new`。我們把類別交給 Spring，讓 Spring 幫我們實例化、管理、並在需要的時候「塞」給我們。這個**由 Spring 建立並管理的物件，就叫做 Bean**。

這裡用三個層次帶你徹底理解 Bean 的連結方式：

---

## 1. 如何告訴 Spring「這是一個 Bean」？（標記）

你會在類別定義上面加上**註解 (Annotations)**。這就像是在零件上貼標籤，告訴 Spring 掃描時要把這個零件撿起來放進工具箱（IoC 容器）。

* **`@Component`**：最通用的標籤（普通的零件）。
* **`@Service`**：邏輯層的零件（專門處理檔案上傳邏輯、JWT 計算）。
* **`@Repository`**：資料層的零件（專門跟資料庫打交道）。
* **`@RestController`**：介面層的零件（專門處理 HTTP 請求）。

---

## 2. Bean 的連結方式：依賴注入 (Dependency Injection)

這就是你最關心的「連結」。當 A 零件需要用到 B 零件時，Spring 會自動把它們接起來。

### 方式 A：建構子注入 (Constructor Injection) —— **最推薦**

這是 Spring Boot 3.5 官方最推薦的方式。利用 **Lombok** 的 `@RequiredArgsConstructor` 可以寫得非常優雅。

```java
@Service
@RequiredArgsConstructor // Lombok 會自動幫你寫建構子，並把 final 欄位放進去
public class FileService {
    
    // 這裡宣告我要用到資料庫零件，Spring 會自動幫我連結
    private final FileRepository fileRepository; 

    public void saveFileMetadata() {
        fileRepository.save(...);
    }
}

```

### 方式 B：欄位注入 (Field Injection) —— **不推薦但常見**

直接在變數上加 `@Autowired`。雖然方便，但不利於測試，且容易造成循環依賴。

```java
@RestController
public class FileController {
    
    @Autowired
    private FileService fileService; // Spring 會自動把 FileService 零件塞進來
}

```

---

## 3. Bean 的生命週期：連結發生在什麼時候？

這連結了你上一個關於「執行順序」的問題。Bean 的連結過程如下：

1. **掃描 (Scanning)**：Spring 啟動時，看到你的 `@SpringBootApplication`，開始掃描所有貼了標籤的類別。
2. **實例化 (Instantiation)**：Spring 呼叫這些類別的建構子，把它們 `new` 出來變成 Bean。
3. **裝配 (Wiring)**：Spring 發現 `FileController` 需要 `FileService`，於是把剛剛 `new` 出來的 `FileService` 實例**塞進** `FileController` 裡。
4. **就緒 (Ready)**：所有的零件都接好了，Tomcat 啟動，你可以開始上傳檔案了。

---

## 💡 形象化類比：樂高 (Lego)

* **Java Class**：是樂高的**說明書**。
* **Bean**：是根據說明書拼出來的**積木實體**。
* **IoC 容器**：是你的**樂高工作檯**。
* **Dependency Injection (DI)**：是積木與積木之間的**卡榫連結**。

你不需要自己去膠水黏積木（`new`），你只需要把積木放在工作檯上（加上 `@Service`），Spring 會根據卡榫形狀（型別）自動幫你扣在一起。

---

## ⚠️ 常見的連結錯誤 (Bug 定位)

如果你在 Log 看到以下錯誤，通常就是 Bean 的連結出問題了：

1. **`NoSuchBeanDefinitionException`**：
* **原因**：你想要用到 A 零件，但你忘記在 A 類別上面貼 `@Service` 標籤，或是 A 不在 Spring 的掃描範圍內。


2. **`UnsatisfiedDependencyException`**：
* **原因**：Spring 知道你要 A 零件，但它在工具箱裡找不到，或者找到了兩個長得一樣的（型別衝突）。


3. **`BeanCurrentlyInCreationException` (循環依賴)**：
* **原因**：A 需要 B，B 又需要 A。兩個人卡在那邊等對方拼好，最後導致啟動失敗。



**在你的「JWT 檔案管理系統」中，你會有很多 Bean 需要連結（例如 `JwtUtils` 連結到 `SecurityConfig`）。需要我幫你寫一個完整的「Controller -> Service -> Repository」連結範例嗎？**
