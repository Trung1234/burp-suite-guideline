# Hướng dẫn Test Lỗi 500 Tomcat trên Spring MVC bằng Burp Suite

## Mục tiêu

Kiểm tra xem ứng dụng Spring MVC chạy trên Tomcat có để lộ thông tin nhạy cảm qua trang lỗi 500 (Internal Server Error) hay không — bao gồm stack trace, tên class, đường dẫn file, version Tomcat/Spring.

---

## 1. Chuẩn bị

- Burp Suite (Community hoặc Pro)
- Trình duyệt đã cấu hình proxy qua Burp (`127.0.0.1:8080`)
- URL ứng dụng Spring MVC đang chạy trên Tomcat (ví dụ: `http://localhost:8080/app`)

---

## 2. Bật Intercept trong Burp Suite

1. Mở **Burp Suite** → tab **Proxy** → **Intercept**.
2. Bật **Intercept is on**.
3. Truy cập bất kỳ endpoint nào của ứng dụng qua trình duyệt để bắt request.

---

## 3. Các kỹ thuật kích hoạt lỗi 500

### 3.1 Gửi Content-Type sai định dạng

Spring MVC thường bị lỗi khi không thể deserialize body do Content-Type không khớp.

**Bước thực hiện:**

1. Bắt một request `POST` có body JSON.
2. Trong **Repeater**, đổi header:
   ```
   Content-Type: application/xml
   ```
   Giữ nguyên body JSON → Spring sẽ cố parse JSON như XML → ném exception.

3. Hoặc đổi thành:
   ```
   Content-Type: text/plain
   ```
   Với endpoint `@RequestBody` → Tomcat/Spring trả 415 hoặc 500 kèm stack trace.

---

### 3.2 Gửi JSON/XML bị lỗi cú pháp (Malformed Body)

**Bước thực hiện:**

1. Bắt request `POST` gửi JSON hợp lệ.
2. Trong **Repeater**, sửa body thành JSON bị thiếu dấu đóng:
   ```json
   {"username": "admin", "password": 
   ```
3. Gửi request — Spring's `HttpMessageConverter` ném `HttpMessageNotReadableException` → Tomcat render trang lỗi 500 với stack trace đầy đủ nếu chưa cấu hình error handling.

---

### 3.3 Truyền kiểu dữ liệu sai cho tham số

Spring MVC tự động bind tham số. Nếu endpoint nhận `int` mà gửi chuỗi ký tự → `MethodArgumentTypeMismatchException`.

**Bước thực hiện:**

1. Với endpoint: `GET /api/user?id=1`
2. Trong **Repeater**, đổi thành:
   ```
   GET /api/user?id=abc HTTP/1.1
   ```
3. Hoặc giá trị cực lớn vượt giới hạn:
   ```
   GET /api/user?id=99999999999999999999 HTTP/1.1
   ```

---

### 3.4 Gửi request thiếu tham số bắt buộc

Nếu controller dùng `@RequestParam(required = true)` (mặc định) mà không gửi param → `MissingServletRequestParameterException`.

**Bước thực hiện:**

1. Request bình thường: `POST /login` với body `username=admin&password=123`
2. Trong **Repeater**, xóa hoàn toàn `password`:
   ```
   POST /login HTTP/1.1
   Content-Type: application/x-www-form-urlencoded

   username=admin
   ```

---

### 3.5 Gửi giá trị NULL / ký tự đặc biệt vào path variable

**Bước thực hiện:**

1. Endpoint: `GET /api/order/{id}`
2. Thử các giá trị:
   ```
   GET /api/order/null HTTP/1.1
   GET /api/order/%00 HTTP/1.1
   GET /api/order/../../etc HTTP/1.1
   GET /api/order/<script> HTTP/1.1
   ```

---

### 3.6 Gửi Header không hợp lệ

```
Accept: application/vnd.invalid+json;version=99
X-Custom-Header: %{7*7}
Content-Length: -1
```

---

### 3.7 Dùng Burp Intruder để Fuzz hàng loạt

1. Gửi request vào **Intruder**.
2. Chọn vị trí payload (param value, path variable, header value).
3. Payload list: dùng wordlist **Fuzzing - Full** có sẵn trong Burp hoặc thêm:
   ```
   null
   undefined
   NaN
   999999999
   -1
   0
   ' OR '1'='1
   <svg/onload=1>
   ${7*7}
   ../../../
   %00
   ```
4. Trong **Options** → **Grep - Match**: thêm từ khóa để phát hiện lỗi:
   - `java.lang`
   - `org.springframework`
   - `Tomcat`
   - `Exception`
   - `stack trace`
   - `at com.`
5. Chạy attack → cột **Match** sẽ đánh dấu response nào chứa thông tin lỗi.

---

### 3.8 Chặn request rồi sửa URL thành giá trị linh tinh

Khi một request bị chặn (intercept) trong Burp và URL bị sửa thành giá trị không hợp lệ, Tomcat có thể trả về trang lỗi 500 thay vì 404 nếu filter/security chain xử lý không đúng cách.

**Bước thực hiện:**

1. Bật **Intercept is on** trong Burp Proxy.
2. Gửi bất kỳ request nào từ trình duyệt → request bị giữ trong Intercept.
3. Trong cửa sổ Intercept, sửa đường dẫn URL thành các giá trị linh tinh:
   ```
   GET /{} HTTP/1.1
   GET /[{}] HTTP/1.1
   GET /| HTTP/1.1
   GET /' HTTP/1.1
   GET /" HTTP/1.1
   GET /test;foo=bar HTTP/1.1
   GET /;test HTTP/1.1
   GET ///// HTTP/1.1
   GET /./ HTTP/1.1
   GET /../../ HTTP/1.1
   GET /% HTTP/1.1
   GET /%%00%01 HTTP/1.1
   GET /.env HTTP/1.1
   GET /.git/config HTTP/1.1
   GET /WEB-INF/web.xml HTTP/1.1
   GET /..;/ HTTP/1.1
   ```
4. Forward request qua Tomcat.
5. Quan sát response — nếu nhận được trang HTML lỗi 500 của Tomcat (có chứa `Apache Tomcat` hoặc stack trace) thay vì 404 → ứng dụng đang leak thông tin.

**Giải thích:** Khi URL chứa ký tự đặc biệt, path traversal hoặc format không hợp lệ, một số filter bảo mật (WAF, security filter) có thể throw exception không được handle → Tomcat render trang lỗi 500 kèm thông tin.

**Mẹo:** Dùng Intruder với payload list các URL path đặc biệt kết hợp Grep-Match từ khóa `Tomcat`, `500`, `Exception`, `java.lang` để nhanh chóng phát hiện lỗi.

---

## 4. Nhận biết lỗi 500 có lộ thông tin

### Dấu hiệu nguy hiểm trong response:

| Thông tin lộ | Ý nghĩa rủi ro |
|---|---|
| `java.lang.NullPointerException` | Lộ logic nội bộ |
| `org.springframework.web.servlet...` | Lộ framework version |
| `at com.example.app.controller...` | Lộ cấu trúc package |
| `Apache Tomcat/9.0.x` | Lộ version Tomcat → tra CVE |
| Đường dẫn tuyệt đối `/home/ubuntu/app/...` | Lộ cấu trúc server |
| SQL error message | Có thể dẫn đến SQL Injection |
| `HikariPool`, `JDBC URL` | Lộ thông tin kết nối DB |

### Ví dụ response nguy hiểm:
```
HTTP/1.1 500 Internal Server Error

java.lang.NumberFormatException: For input string: "abc"
    at java.base/java.lang.NumberFormatException.forInputString(NumberFormatException.java:67)
    at com.example.shop.controller.ProductController.getProduct(ProductController.java:45)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:897)
```

---

## 5. Kiểm tra trang lỗi mặc định của Tomcat

Nếu Spring không handle exception, Tomcat sẽ render trang lỗi HTML mặc định — lộ version rõ ràng:

```html
<h1>HTTP Status 500 – Internal Server Error</h1>
...
<footer>Apache Tomcat/9.0.65</footer>
```

**Cách kiểm tra nhanh:** Gửi request tới path không tồn tại:
```
GET /this-path-does-not-exist-xyz HTTP/1.1
```
Nếu response trả về HTML có chữ `Apache Tomcat` → trang lỗi chưa được tùy chỉnh.

---

## 6. Kết luận & Khuyến nghị

### Ứng dụng **dễ bị tấn công** nếu:
- Trả về stack trace trong response production
- Hiển thị trang lỗi mặc định của Tomcat
- Lộ version Tomcat, Spring, Java trong header hoặc body

### Fix đề xuất:

**Trong Spring MVC — tạo Global Exception Handler:**
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, String>> handleAll(Exception ex) {
        // Log nội bộ, KHÔNG trả ra ngoài
        log.error("Internal error", ex);
        return ResponseEntity.status(500)
            .body(Map.of("error", "Internal Server Error"));
    }
}
```

**Trong `application.properties`:**
```properties
server.error.include-stacktrace=never
server.error.include-message=never
server.error.include-exception=false
```

**Ẩn version Tomcat trong `server.xml`:**
```xml
<Connector ... server="Apache" />
```

---

## 7. Checklist kiểm tra nhanh

- [ ] Gửi body JSON bị lỗi cú pháp → kiểm tra response
- [ ] Đổi Content-Type sai → kiểm tra response
- [ ] Truyền chuỗi vào tham số kiểu số → kiểm tra response
- [ ] Xóa `@RequestParam` bắt buộc → kiểm tra response
- [ ] Truy cập path không tồn tại → trang lỗi có lộ version?
- [ ] Dùng Intruder fuzz với Grep để phát hiện stack trace
- [ ] Kiểm tra header `Server:` trong response có lộ `Apache-Coyote` hay `Tomcat` không
