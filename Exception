Checked Exception 最核心的設計是：「連編譯都不讓你過」。

它實際影響的情形分為這兩個階段：

1. 編譯階段 (Compile-time) —— 強制防呆，不處理就直接報錯
只要你呼叫了任何會丟出 Checked Exception 的方法（例如讀檔 IOException，連資料庫 SQLException），你 必須做出選擇：

自己用 try-catch 吃掉它。
自己加上 throws IOException 繼續往上丟給呼叫你的上一層。
如果你兩邊都不選，IDE 會立刻亮紅線，Java Compiler 也會直接拒絕編譯。你的專案根本連啟動或打包 (mvn clean package) 的機會都沒有。它設計的初衷是：這類錯誤通常是「外部因素（網路斷線、檔案不見）」造成的，程式應該要能預期並處理，所以編譯器提早逼你面對。

2. 執行階段 (Runtime) —— 它不會導致整台伺服器掛掉
假設你透過 throws 或 try-catch 讓程式成功編譯了。當系統真正上線，且剛好觸發了這個 Exception（且你選擇 throws 往外拋）時：

它 不會 讓整個 Spring Boot 應用程式強制中斷或 Crash。

它只會中斷 「當下那條正在處理請求的 Thread (執行緒)」。
這個 Exception 會一直往上丟，直到被你的 

GlobalExceptionHandler
 攔截到，並回傳 HTTP 500 給前端。
處理完這個 500 回應後，該 Thread 就收工了。伺服器依然好好的在背景運作，其他人打進來的 API 請求完全不受影響。
一句話總結： Checked Exception 是 Java 用來在**「編譯期」拿槍指著你的頭，逼你預防可預期錯誤的機制；但在「執行期」**，它跌倒的樣子跟一般 RuntimeException 完全一樣，只會弄死當下那支 API，不會弄死整個系統。

