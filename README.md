# Session 12 - Bài tập 4: Tích hợp UFW và Docker

## Mô tả bài toán

Bạn đã cấu hình UFW để chặn cổng `8080`, nhưng Docker tự động can thiệp vào iptables, khiến cổng `8080` vẫn **public ra internet** dù UFW đang chặn.

## Giải pháp

Thay đổi port binding trong `docker-compose.yml`:

```yaml
# TRƯỚC (vấn đề) - bind ra tất cả interface
ports:
  - "8080:8080"       # tương đương 0.0.0.0:8080:8080

# SAU (đã sửa) - chỉ bind vào localhost
ports:
  - "127.0.0.1:8080:8080"
```

## Tại sao Docker bypass UFW?

Docker thao tác trực tiếp với **iptables** ở chain `DOCKER`, nằm trước chain `ufw-before-input`. Do đó UFW không có cơ hội kiểm tra traffic đến container.

```
Internet → iptables PREROUTING → DOCKER chain (ACCEPT) → Container
                                      ↑
                             UFW không bao giờ được hỏi
```

## Sau khi fix

```
Internet → iptables PREROUTING → DOCKER chain
                                      ↓
                               127.0.0.1 only → kernel DROP (ngoài mạng)
                               Nginx (localhost) → proxy_pass → OK
```

## Cấu trúc project

```
QuickBite-Session12-B4/
├── docker-compose.yml              # File đã sửa (127.0.0.1 binding)
├── docker-compose.old.yml          # File gốc có vấn đề (0.0.0.0)
├── nginx/
│   └── sites-available/
│       └── app.conf                # Nginx proxy config
├── commands.txt                    # Danh sách lệnh thực hiện
├── EXPLANATION.md                  # Giải thích chi tiết
└── homework/session_12/exercise_04/
    ├── docker_compose_original_problem_proof.png
    ├── docker_compose_fixed_localhost_binding_proof.png
    └── ufw_docker_integration_explanation_proof.png
```
