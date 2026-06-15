# Burp Suite Guideline

Tài liệu hướng dẫn sử dụng Burp Suite cho kiểm tra bảo mật ứng dụng web.

## Mục lục

1. [Giới thiệu](#giới-thiệu)
2. [Cài đặt](#cài-đặt)
3. [Cấu hình](#cấu-hình)
4. [Các tính năng chính](#các-tính-năng-chính)
5. [Use Cases](#use-cases)

## Giới thiệu

Burp Suite là công cụ kiểm tra bảo mật ứng dụng web chuyên nghiệp. Công cụ hỗ trợ pentester trong việc phát hiện và khai thác lỗ hổng bảo mật.burp Suite không phải là công cụ quét tự động hoàn toàn như Acunetix, nhưng có giao diện trực quan, thân thiện.

## Cài đặt

Yêu cầu:
- Java (JRE 8 trở lên)
- Tải Burp Suite Community Edition miễn phí từ trang chủ

## Cấu hình

Burp Suite hoạt động như một HTTP proxy. Để sử dụng:

1. Cấu hình trình duyệt sử dụng proxy `127.0.0.1:8080`
2. Hoặc sử dụng extension như **FoxyProxy** để dễ dàng chuyển đổi

## Các tính năng chính

### Target
Dành cho việc xác định scope và xem site map của ứng dụng.

### Proxy
- Intercept requests/responses
- Forward, drop, hoặc edit HTTP requests
- Inspector panel hiển thị chi tiết request

### Intruder
Công cụ tấn công tự động:
- Brute-forcing
- Payload testing
- Automated attack patterns

### Repeater
- Sửa đổi và gửi lại requests thủ công
- Phân tích responses
- Các chế độ xem: Raw, Rendered HTML, Pretty format

### Scanner
- Quét chủ động và bị động
- Custom audit checks
- Results trong Issue Activity panel

## Use Cases

Xem thư mục Claude-Opus(01_usecase/) để tìm hiểu các trường hợp sử dụng thực tế.

### Các file hiện có:
- [500error.md](01_usecase/500error.md) - Hướng dẫn tái hiện lỗi HTTP 500 trong Spring MVC
