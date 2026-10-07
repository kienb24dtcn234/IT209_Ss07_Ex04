# Session 07 - Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

**Họ tên:** Nguyễn Thế Kiên | **Mã lớp:** K24 - DevOps PTIT

## Mục tiêu
- Nginx Server Block làm Reverse Proxy định tuyến lưu lượng.
- Path Matching: `/` phục vụ tĩnh trực tiếp, `/api/` chuyển tiếp sang Spring Boot cổng 8082.

---

### Bước 1 — Tạo trang tĩnh tại /var/www/html/
```bash
sudo mkdir -p /var/www/html
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="vi">
<head><meta charset="UTF-8"><title>Trang chủ - DevOps PTIT</title></head>
<body>
  <h1>Reverse Proxy Nginx - Spring Boot</h1>
  <p>Họ tên: Nguyễn Thế Kiên</p>
  <p>Mã lớp: K24 - DevOps PTIT</p>
</body>
</html>
EOF
```

### Bước 2 — Tạo file cấu hình spring-proxy.conf
```bash
sudo tee /etc/nginx/sites-available/spring-proxy.conf > /dev/null <<'EOF'
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8082/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

### Bước 3 — Kích hoạt bằng symlink + gỡ site default (tránh trùng cổng 80)
```bash
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
```

### Bước 4 — Kiểm tra cú pháp rồi reload
```bash
sudo nginx -t
sudo systemctl reload nginx
```
→ Phải thấy: `syntax is ok` và `test is successful`.

### Bước 5 — Chạy thử
```bash
curl -I http://localhost/
curl -i http://localhost/api/health
```

---

## Kết quả mong đợi
- `nginx -t` → `syntax is ok` + `test is successful`.
- `curl -I http://localhost/` → **HTTP/1.1 200 OK**, trang tĩnh có tên + mã lớp.
- `curl http://localhost/api/health` → mã trạng thái **từ Spring Boot** (200/404... tùy backend), **KHÔNG phải 502**.

---

## Lưu ý quan trọng về /api/ (dấu `/` cuối proxy_pass)
`proxy_pass http://127.0.0.1:8082/;` **có dấu `/` cuối** → Nginx **cắt bỏ** tiền tố `/api/` trước khi chuyển tiếp:
- Client gọi `/api/health` → backend nhận `/health`.

Nếu Spring Boot của bạn khai báo endpoint là `/api/health` (giữ nguyên tiền tố), hãy **bỏ dấu `/` cuối**: `proxy_pass http://127.0.0.1:8082;` → khi đó `/api/health` được giữ nguyên khi forward.

## Nếu bị 502 Bad Gateway
Nghĩa là Nginx chạy ổn nhưng backend chưa lên. Kiểm tra Spring Boot:
```bash
curl -i http://127.0.0.1:8082/        # backend có trả lời không?
sudo ss -tunlp | grep 8082            # có tiến trình nghe cổng 8082 không?
```
Phải bật app Spring Boot (hoặc systemd service của nó — chính là Bài 3) trước khi test `/api/`.

---
