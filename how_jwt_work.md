太棒了，我們直接用你的專案「**JWT 檔案管理系統**」來做一個完整的 Bean 連結示範。

這份程式碼會展示一個請求如何從 **Controller** 出發，經過 **Service** 處理邏輯，最後由 **Repository** 存入資料庫。這三個類別在 Spring 啟動時會被自動建立為 **Bean** 並互相「連結」。

---

### 🛠️ 1. 定義實體 (Entity)

這不是 Bean，它只是一個資料模型。

```java
@Data
@Entity
public class FileMetadata {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String fileName;
    private String filePath;
    private String owner; // 這裡之後會存 JWT 裡面解析出來的使用者名稱
}

```

---

### 💾 2. 建立 Repository Bean

加上 `@Repository` 標籤，Spring 會自動幫你實作資料庫操作邏輯。

```java
@Repository
public interface FileRepository extends JpaRepository<FileMetadata, Long> {
    // Spring 會自動把這個介面變成一個 Bean
}

```

---

### ⚙️ 3. 建立 Service Bean (核心邏輯)

使用 `@Service` 標籤，並用 **Lombok** 的 `@RequiredArgsConstructor` 來「連結」剛剛的 Repository。

```java
@Service
@RequiredArgsConstructor // 自動連結下面的 final 變數 (Bean 注入)
public class FileService {

    private final FileRepository fileRepository; // 這是自動連結進來的 Repository Bean

    public FileMetadata saveFileInfo(String fileName, String path, String username) {
        FileMetadata meta = new FileMetadata();
        meta.setFileName(fileName);
        meta.setFilePath(path);
        meta.setOwner(username);
        
        // 呼叫連結進來的 Repository Bean
        return fileRepository.save(meta);
    }
}

```

---

### 🎮 4. 建立 Controller Bean (API 入口)

使用 `@RestController` 標籤，並連結 `FileService`。

```java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
public class FileController {

    private final FileService fileService; // 連結 Service Bean

    @PostMapping("/upload")
    public ResponseEntity<FileMetadata> uploadFile(@RequestParam String name, @AuthenticationPrincipal String username) {
        // 假設檔案存儲邏輯已完成，取得路徑後存入資料庫
        String mockPath = "/storage/uploads/" + name;
        
        FileMetadata savedFile = fileService.saveFileInfo(name, mockPath, username);
        return ResponseEntity.ok(savedFile);
    }
}

```

---

## 🏗️ 這段程式碼中發生了什麼連結？

當你執行 `mvn spring-boot:run` 時，Log 會顯示啟動過程，背後的連結動作如下：

1. **實例化 Repository**：Spring 看到 `FileRepository`，幫你拼好一個資料庫存取工具。
2. **實例化 Service**：Spring 準備建立 `FileService`，發現它需要 `FileRepository`，於是把剛剛拼好的零件**「塞入」**建構子中。
3. **實例化 Controller**：Spring 準備建立 `FileController`，發現它需要 `FileService`，於是把剛剛拼好的 Service 零件**「塞入」**進去。
4. **發布 API**：Tomcat 啟動，將 `/api/files/upload` 這個路徑連結到 `FileController` 的方法上。

---

## 💡 總結：Bean 連結的三個關鍵字

* **掃描 (Scan)**：確定哪些類別要變成 Bean（`@Service`, `@Repository`...）。
* **容器 (Container)**：存放這些 Bean 實例的地方。
* **注入 (Injection)**：把 A Bean 塞進 B Bean 的過程（連結）。

---

### 🔎 下一步：JWT 的特殊 Bean

在你的專案中，**JWT 的過濾器 (Filter)** 也是一個 Bean。但它比較特別，它必須被插在 **Spring Security** 的過濾鏈條中。

**你想看如何把 `JwtAuthenticationFilter` 這個 Bean 設定到 Spring Security 的配置中嗎？這涉及到安全防禦的「連結」。**

---

好的，這部分是整個專案最「進階」的連結。我們要把一個自定義的 **JWT 攔截器（Filter）** 變成一個 Bean，並把它「插隊」進去 Spring Security 的原生過濾鏈條（Security Filter Chain）中。

這就像是在大樓（應用程式）的入口處，除了原本的警衛（Spring Security），再額外聘請一個專門檢查「特製識別證（JWT）」的專員。

---

## 🛠️ 1. 建立 JWT 攔截器 Bean (`JwtAuthenticationFilter`)

這個 Bean 繼承了 `OncePerRequestFilter`，確保每個 HTTP 請求只會被檢查一次。

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtils jwtUtils; // 連結 JWT 工具類 Bean
    private final UserDetailsService userDetailsService; // 連結 Spring 內建的使用者查詢 Bean

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {
        
        // 1. 從 Header 抓取 Authorization: Bearer <token>
        final String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response); // 沒帶 Token，直接放行給後面的警衛處理
            return;
        }

        String jwt = authHeader.substring(7);
        String userEmail = jwtUtils.extractUsername(jwt); // 解析 Token

        // 2. 如果解析成功且目前還沒認證過
        if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(userEmail);
            
            if (jwtUtils.isTokenValid(jwt, userDetails)) {
                // 3. 認證成功！把使用者資訊塞進 Spring 的「認證存錢筒」
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                );
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response); // 繼續往後走
    }
}

```

---

## 🛡️ 2. 連結到 Spring Security 配置 (`SecurityConfig`)

這是最關鍵的「連結」步驟。我們要在這裡定義哪些路徑（如檔案上傳）需要檢查，並把上面的 `JwtAuthenticationFilter` 插進去。

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter; // 連結我們寫的攔截器 Bean

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // 檔案管理系統通常會關閉 CSRF
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll() // 登入 API 不用檢查
                .requestMatchers("/api/files/**").authenticated() // 檔案 API 必須登入
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 告訴 Spring 不要用 Session，改用 JWT
            )
            // 💡 重點連結：在「使用者名稱密碼過濾器」之前，先執行我們的「JWT 攔截器」
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationToken.class);

        return http.build();
    }
}

```

---

## 📊 這段連結的運作流程

當一個請求進來時，Bean 之間的協作順序如下：

1. **Tomcat** 接收請求並交給 **Spring Security**。
2. **`JwtAuthenticationFilter` (Bean)** 優先啟動，它會調用 **`JwtUtils` (Bean)** 來拆解 Token。
3. 如果 Token 合法，它會把身分資訊存入 **`SecurityContext`**。
4. 最後請求才會到達你的 **`FileController` (Bean)**。此時 Controller 就能確保目前的操作者是合法的。

---

## 💡 總結：檔案管理系統的安全性

在這種架構下，你的檔案管理系統會非常安全：

* **攔截 (Intercept)**：沒有 Token 的人連 `FileController` 的邊都摸不到。
* **識別 (Identify)**：Controller 透過 `SecurityContext` 知道是誰在操作。
* **無狀態 (Stateless)**：伺服器不需要存 Session，重啟也不會導致使用者斷線。

**目前的結構已經涵蓋了 Controller、Service、Repository 和 Security 的連結。你需要我幫你寫一份 `JwtUtils` 的具體內容，讓你能直接跑起 Token 的生成與解析嗎？**

---

這是你的專案中最關鍵的「工具類」Bean。在 **Spring Boot 3.5** 中，我們使用 **JJWT 0.12.x** 版本，這個版本的語法與舊版（0.9.x）有很大的不同，主要強化了型別安全與密鑰管理的標準。

以下是 `JwtUtils` 的完整實作，它負責 **生成 (Generate)**、**解析 (Extract)** 與 **驗證 (Validate)** 三大功能。

---

## 🛠️ 1. `JwtUtils` 工具類實作

請將此類別放在 `com.jpeng.security` 或 `com.jpeng.util` 套件下。

```java
@Component
public class JwtUtils {

    // 💡 密鑰：在正式環境中，建議放在 application.properties 並用 @Value 讀取
    // 注意：HS256 演算法要求密鑰長度至少需 256 bits (32 字元)
    private static final String SECRET_KEY = "your-super-secret-key-at-least-32-characters-long!!";
    private static final long EXPIRATION_TIME = 86400000; // 24小時 (單位: 毫秒)

    // 取得簽名用的 Key 物件 (JJWT 0.12.x 新寫法)
    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(Base64.getEncoder().encodeToString(SECRET_KEY.getBytes()));
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // 1. 生成 Token (給登入成功後調用)
    public String generateToken(String username) {
        return Jwts.builder()
                .subject(username)
                .issuedAt(new Date(System.currentTimeMillis()))
                .expiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME))
                .signWith(getSigningKey()) // 自動識別 HS256
                .compact();
    }

    // 2. 從 Token 中解析出使用者名稱 (給 Filter 調用)
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    // 3. 驗證 Token 是否合法且未過期
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }

    // --- 以下為內部輔助工具方法 ---

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey()) // JJWT 0.12.x 使用 verifyWith 代替 setSigningKey
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}

```

---

## 🧩 2. JWT 的結構詳解

當你呼叫 `generateToken` 時，產出的字串（如 `eyJhbGci...`）其實由三部分組成：

1. **Header (標頭)**：標註演算法（如 HS256）與類型（JWT）。
2. **Payload (載荷)**：存放 `subject`（使用者名稱）、`issuedAt`（簽發時間）與 `expiration`（過期時間）。
3. **Signature (簽名)**：使用 `SECRET_KEY` 對前兩部分進行加密。**這是最安全的地方**，只要 Key 不洩漏，前端無法偽造 Payload。

---

## 🔄 3. 在專案流程中的連結

在你的 **「JWT 管理身分認證檔案管理系統」** 中，這個 Bean 會被頻繁調用：

* **登入時 (LoginController)**：
* 帳密正確 $\rightarrow$ 調用 `jwtUtils.generateToken("Jpeng")` $\rightarrow$ 回傳給前端存入 `localStorage`。


* **上傳檔案時 (JwtAuthenticationFilter)**：
* 攔截請求 $\rightarrow$ 調用 `jwtUtils.extractUsername(token)` $\rightarrow$ 確認身分 $\rightarrow$ 允許進入 `FileController`。



---

## ⚠️ 開發常見地雷 (Bug Alert)

1. **密鑰太短**：如果你的 `SECRET_KEY` 字數不足，JJWT 會在啟動時直接報錯 `WeakKeyException`。請確保至少 32 個字元。
2. **時區問題**：如果伺服器時區跟資料庫或 Token 簽發時間不一致，可能會導致一簽發就過期。
3. **解析異常**：前端如果傳來爛掉的字串，`parseSignedClaims` 會噴出 `MalformedJwtException`，建議在 `JwtUtils` 或 `Filter` 中使用 `try-catch` 處理。

---

### 💡 下一步建議

現在你已經有了 **Security 配置**、**JWT 工具類** 和 **Controller/Service 結構**。

**你需要我幫你整合一份 `LoginController` 的程式碼嗎？這會完成從「輸入帳密」到「領取 JWT」的最後一塊拼圖。**
