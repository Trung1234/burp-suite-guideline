# Làm thế nào để tái hiện lỗi 500 trong Spring MVC Web Application

Lỗi HTTP 500 (Internal Server Error) là lỗi phổ biến xảy ra khi server gặp vấn đề không xác định trong quá trình xử lý request. Dưới đây là các cách để tái hiện lỗi 500 trong Spring MVC.

## 1. NullPointerException trong Controller

```java
@GetMapping("/demo")
public String demo(Model model) {
    String value = null;
    model.addAttribute("data", value.toString()); // NullPointerException
    return "index";
}
```

**Cách tái hiện:**
- Truy cập `/demo`
- Kiểm tra Burp Suite Proxy: nhận thấy response status là 500
- Check server logs để thấy stack trace NullPointerException

## 2. Division by Zero

```java
@GetMapping("/divide")
public String divide(@RequestParam int a, @RequestParam int b, Model model) {
    int result = a / b; // ArithmeticException khi b = 0
    model.addAttribute("result", result);
    return "result";
}
```

**Cách tái hiện:**
- Truy cập `/divide?a=10&b=0`
- Server throw `ArithmeticException: / by zero`

## 3. Accessing Non-existent Database Record

```java
@GetMapping("/user/{id}")
public String getUser(@PathVariable Long id, Model model) {
    User user = userRepository.findById(id).get(); // NoSuchElementException
    model.addAttribute("user", user);
    return "user";
}
```

**Cách tái hiện:**
- Truy cập `/user/999999` với ID không tồn tại
- `.get()` trên Optional rỗng throw `NoSuchElementException`

## 4. Invalid Type Conversion

```java
@GetMapping("/age")
public String getAge(@RequestParam Integer age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative");
    }
    return "age-" + age;
}
```

**Cách tái hiện:**
- Gửi request với parameter không hợp lệ qua Burp Suite
- Parameter binding failure có thể gây 500

## 5. Missing @RequestBody Validation

```java
@PostMapping("/user")
public String createUser(@RequestBody User user) {
    // Nếu user null hoặc field validation fail
    return "created-" + user.getName();
}
```

**Cách tái hiện:**
- Gửi request POST với body invalid qua Repeater
- Validate @Valid annotation bị bỏ qua

## Cách kiểm tra với Burp Suite

1. **Proxy**: Intercept request đến endpoint suspect
2. **Repeater**: Modify parameters và gửi lại
3. **Decoder**: Test with different encoded values
4. **Scanner**: Tự động phát hiện các lỗi server

## Debug Tips

- Check server logs (Tomcat/Spring Boot logs)
- Sử dụng breakpoint trong IDE
- Thêm `@ControllerAdvice` để handle exceptions
- Sử dụng `@ExceptionHandler` custom
