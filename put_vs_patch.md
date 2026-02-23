## PUT vs PATCH 在 Spring Boot 中的比較

### 核心差異

| | PUT | PATCH |
|---|---|---|
| 語意 | **完整替換**資源 | **部分更新**資源 |
| 冪等性 | ✅ 冪等 | ⚠️ 不一定冪等 |
| 請求體 | 需包含完整欄位 | 只需包含要更新的欄位 |
| 缺少欄位 | 會被設為 null | 保持原值不變 |

---

### PUT 範例 — 完整替換

```java
@PutMapping("/users/{id}")
public ResponseEntity<User> updateUser(@PathVariable Long id, @RequestBody UserDto dto) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));

    // 整個物件全部替換
    user.setName(dto.getName());
    user.setEmail(dto.getEmail());
    user.setAge(dto.getAge());

    return ResponseEntity.ok(userRepository.save(user));
}
```

**請求體必須完整：**
```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "age": 30
}
```
若少傳 `age`，資料庫中 `age` 會被更新為 `null`。

---

### PATCH 範例 — 部分更新

**方法一：手動判斷 null（簡單但繁瑣）**

```java
@PatchMapping("/users/{id}")
public ResponseEntity<User> patchUser(@PathVariable Long id, @RequestBody UserDto dto) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));

    // 只更新非 null 的欄位
    if (dto.getName() != null) user.setName(dto.getName());
    if (dto.getEmail() != null) user.setEmail(dto.getEmail());
    if (dto.getAge() != null) user.setAge(dto.getAge());

    return ResponseEntity.ok(userRepository.save(user));
}
```

**方法二：使用 `Map` 接收（更彈性）**

```java
@PatchMapping("/users/{id}")
public ResponseEntity<User> patchUser(@PathVariable Long id,
                                       @RequestBody Map<String, Object> updates) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));

    updates.forEach((key, value) -> {
        switch (key) {
            case "name"  -> user.setName((String) value);
            case "email" -> user.setEmail((String) value);
            case "age"   -> user.setAge((Integer) value);
        }
    });

    return ResponseEntity.ok(userRepository.save(user));
}
```

**方法三：使用 `JsonMergePatch`（最符合 RFC 7396 標準）**

```java
@PatchMapping(value = "/users/{id}", consumes = "application/merge-patch+json")
public ResponseEntity<User> patchUser(@PathVariable Long id,
                                       @RequestBody JsonMergePatch patch) throws JsonPatchException, JsonProcessingException {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));

    JsonNode patched = patch.apply(objectMapper.convertValue(user, JsonNode.class));
    User updatedUser = objectMapper.treeToValue(patched, User.class);

    return ResponseEntity.ok(userRepository.save(updatedUser));
}
```

---

### 使用場景建議

**用 PUT：**
- 表單送出完整資料（如編輯頁面儲存）
- 資源結構簡單，欄位不多

**用 PATCH：**
- 只想更新單一欄位（如修改密碼、狀態切換）
- 資源欄位多，避免傳輸大量不必要資料
- 行動裝置 API，節省頻寬

---

### 常見陷阱

**PATCH 用 null 判斷有問題：** 若業務上合法需要把某欄位設為 `null`，用 null 判斷就無法區分「沒傳」和「刻意清空」，此時建議改用 `Map` 或 `JsonMergePatch`。
