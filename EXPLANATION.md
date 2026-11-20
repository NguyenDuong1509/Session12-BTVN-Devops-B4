# Giải thích chi tiết: UFW và Docker

## 1. Vấn đề: Docker bypass UFW

UFW (Uncomplicated Firewall) là front-end cho **iptables**. Khi bạn chạy:
```bash
sudo ufw deny 8080
```
UFW thêm rule vào chain `ufw-before-input` của iptables.

Nhưng Docker cũng thao tác iptables khi khởi động container:
```bash
# Docker tự thêm rule này (TRƯỚC cả UFW):
-A DOCKER -d 0.0.0.0/0 -p tcp --dport 8080 -j ACCEPT
```

**Thứ tự xử lý iptables:**
```
Packet từ internet → PREROUTING → DOCKER chain (Docker ACCEPT ngay) → Container
                                        ↑
                              UFW không bao giờ được hỏi!
```

## 2. Giải pháp: Bind vào 127.0.0.1

### File docker-compose.yml TRƯỚC (có vấn đề):
```yaml
version: '3'
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "8080:8080"    # = 0.0.0.0:8080:8080 → PUBLIC!
```

### File docker-compose.yml SAU (đã sửa):
```yaml
version: '3'
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "127.0.0.1:8080:8080"    # Chỉ localhost → AN TOÀN!
```

## 3. Tại sao "127.0.0.1:8080:8080" an toàn?

Khi bind vào `127.0.0.1`:
- Docker tạo rule iptables chỉ chấp nhận kết nối từ **loopback interface**
- Request từ internet đến VPS:8080 bị kernel **từ chối ngay tại network stack**, không cần qua UFW
- Nginx chạy trên cùng máy vẫn có thể `proxy_pass http://127.0.0.1:8080` → **hoạt động bình thường**

## 4. Luồng hoạt động sau khi fix

```
Internet → VPS:80 → UFW (allow 80) → Nginx → 127.0.0.1:8080 → Docker Container
Internet → VPS:8080 → Kernel DROP (127.0.0.1 không tiếp nhận external traffic)
```

## 5. Kiểm tra kết quả

```bash
# Kiểm tra port binding của container
docker compose ps
# Output mong muốn:
# backend   127.0.0.1:8080->8080/tcp   ← CHỈ localhost
# nginx     0.0.0.0:80->80/tcp         ← Public (đúng)

# Kiểm tra iptables rule
sudo iptables -t nat -L DOCKER -n
# Chỉ thấy rule cho 127.0.0.1, không có 0.0.0.0
```
