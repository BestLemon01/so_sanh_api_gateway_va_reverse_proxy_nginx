# PHÂN TÍCH KIẾN TRÚC: TẠI SAO KHÔNG THỂ BỎ NGINX KHI ĐÃ CÓ SPRING CLOUD GATEWAY?

**Vai trò:** Software Architect  
**Kết luận:** **Không được bỏ Nginx.** Nginx (Edge Layer) và Spring Cloud Gateway (Application Gateway Layer) bổ trợ cho nhau theo mô hình phòng thủ nhiều lớp (Defense in Depth), không trùng lặp chức năng cốt lõi.

---

## 1. Bảng so sánh thế mạnh từng thành phần

| Tiêu chí | Nginx (Edge Reverse Proxy) | Spring Cloud Gateway (API Gateway) |
| :--- | :--- | :--- |
| **Bản chất & Hiệu năng** | Viết bằng C, kiến trúc Event-driven non-blocking, tiêu tốn cực ít RAM (~10-20MB), chịu hàng chục nghìn kết nối đồng thời. | Chạy trên JVM (Spring WebFlux/Netty), tốn nhiều RAM & CPU hơn (~300-500MB). |
| **Phục vụ Static Content** | Phục vụ React/Vue HTML, JS, CSS, ảnh trực tiếp ở tốc độ hệ điều hành (`sendfile`), không cần chạm tới JVM. | Không tối ưu để phục vụ file tĩnh của Frontend. |
| **SSL/TLS Termination** | Xử lý giải mã HTTPS, HTTP/2, OCSP stapling cực nhanh ở rìa mạng; dễ tích hợp Certbot Let's Encrypt. | Cấu hình SSL trên Java Keystore phức tạp hơn và tốn CPU của JVM. |
| **Bảo mật tầng mạng & DDoS** | Chặn kết nối chậm (Slowloris), giới hạn băng thông, Rate Limiting tầng IP/TCP, chặn Bot trước khi request vào hệ thống Java. | Dễ bị tràn Heap/Thread nếu hứng trực tiếp lưu lượng tấn công DDoS tầng mạng. |
| **Logic nghiệp vụ & Discovery** | Không tự đọc được trạng thái động từ Eureka/Consul (bản Open Source), không xử lý sâu payload/JWT theo nghiệp vụ Spring. | Tích hợp chặt chẽ **Eureka Service Discovery**, **JWT Auth Filter**, **Circuit Breaker (Resilience4j)**, biến đổi Header/Body theo logic nghiệp vụ Microservices. |

---

## 2. Sơ đồ luồng dữ liệu chuẩn Production (Data Flow)

```
[Client / Browser / Mobile App]
              │
              │ HTTPS (Port 443)
              ▼
┌────────────────────────────────────────────────────────┐
│ NGINX REVERSE PROXY (Edge Network Layer)               │
│ - SSL/TLS Termination (Let's Encrypt)                  │
│ - Gzip Compression & Static Files (React Frontend)     │
│ - DDoS Protection, Rate Limit theo IP, Header Security │
└─────────────┬──────────────────────────┬───────────────┘
              │ / (Static/SPA)           │ /api/** (HTTP nội bộ)
              ▼                          ▼
     [React Frontend]       ┌────────────────────────────┐
                            │ SPRING CLOUD API GATEWAY   │
                            │ (Port 8080 - Internal Net) │
                            │ - Xác thực JWT Token       │
                            │ - Eureka Service Discovery │
                            │ - Circuit Breaker & Filter │
                            └──────┬──────────────┬──────┘
                                   │              │
                          ┌────────▼───┐    ┌─────▼───────┐
                          │ Order Svc  │    │ Payment Svc │
                          └────────────┘    └─────────────┘
```

**Tóm lại:** Nginx đóng vai trò "lá chắn thép" ở rìa mạng để xử lý kết nối, tĩnh và SSL; giúp Spring Cloud Gateway giải phóng tài nguyên, tập trung hoàn toàn vào xác thực nghiệp vụ và điều phối Microservices nội bộ.
