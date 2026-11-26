# Excalidraw — triển khai bằng Portainer Git, cổng 18896:18896

Repo của bạn: https://github.com/hoangthang0106/excalidraw

## 1. Cấu hình được chọn

- **Host port = container port = 18896**.
- Dùng **một container duy nhất**: `excalidraw`.
- Image tác giả gốc: `excalidraw/excalidraw:latest`.
- Không thêm Caddy, gateway, FlareSolverr, Redis hoặc database.
- Image chính thức chứa frontend Excalidraw được Nginx phục vụ. Compose đổi **Nginx listen 80 → 18896 ngay khi khởi động**, đồng thời đổi healthcheck theo cổng thực.
- Cổng **18896 dự kiến cho Excalidraw**; các fork chưa xác nhận cài như Maigret có thể còn cấu hình 18896 cũ. Phải kiểm tra Docker host thực tế trước khi Deploy; không chạy hai stack trên cùng host IP:port.

## 2. Kiểm tra cổng (chạy trên chính máy Docker)

Các lệnh dưới đây chỉ đọc:

```bash
ss -ltnp '( sport = :18896 )'
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps -aq | xargs -r docker inspect --format '{{.Name}} {{json .HostConfig.PortBindings}}'
```

Nếu 18896 đã sử dụng, **không triển khai** trước khi xử lý xung đột. Việc sửa GitHub không có nghĩa Portainer đã thay đổi.

## 3. Cài từ Git repository trên Portainer

Chọn **Stacks → Add stack → Git repository**:

| Trường | Giá trị |
|---|---|
| Name | `excalidraw` |
| Repository URL | `https://github.com/hoangthang0106/excalidraw.git` |
| Repository reference | `refs/heads/master` |
| Compose path | `docker-compose.yml` |
| Authentication | None (public repository) |

**Environment variables** — chỉ bắt buộc 1 biến:

| Name | Value |
|---|---|
| `LAN_IP` | IP thực tế của Docker VM/LXC/NAS, ví dụ IP LAN mà bạn truy cập Portainer |

Cấu hình yêu cầu IP LAN có trên host, không được tự gõ IP của thiết bị khác. Không cần `PORT`, `APP_PORT`, `NODE_ENV` hay mật khẩu trong GitHub.

Sau khi kiểm tra 18896 trống, nhấn **Deploy the stack**.

## Đã sửa lỗi Nginx "chown(...client_temp...) Operation not permitted" ngày 26/11/2026

Log Portainer có thể hiện:

```
nginx: [emerg] chown("/var/cache/nginx/client_temp", 101) failed (1: Operation not permitted)
nginx: configuration file /etc/nginx/nginx.conf test failed
```

Nguyên nhân: Compose trước đó tự thêm `cap_drop: ALL`, tước `CAP_CHOWN` mà Nginx dùng khi chuẩn bị thư mục cache. Đã **gỡ riêng `cap_drop: ALL`**, giữ nguyên `security_opt: no-new-privileges:true`, số lượng container, image, cổng `18896:18896`, lệnh đổi port Nginx và healthcheck. Không yêu cầu `privileged: true`.

**Sửa trên Portainer Stack đang có:** Stacks → excalidraw → Pull and redeploy. Xác nhận Containers → excalidraw → Logs không còn lỗi `chown` và hiển thị Healthy. Không cần xóa Stack. Nếu vẫn lỗi, xem Logs mới.

## 4. Kiểm tra hoạt động

1. **Containers** → `excalidraw`: trạng thái `Running` → `Healthy`.
2. **Published Ports** phải là `LAN_IP:18896:18896/tcp`.
3. Truy cập `http://LAN_IP:18896` trên thiết bị cùng LAN.
4. Nếu có lỗi, mở `Containers → excalidraw → Logs`. Khi start, lệnh `nginx -t` phải thành công.
5. Ứng dụng khởi động đúng nhưng không tải giao diện: xem Network/browser Console và kiểm tra kết nối Internet/CDN trên điện thoại; hình/nguồn ngoài có thể chưa tải.

## 5. Dữ liệu và bảo mật — cần biết

- Excalidraw bản cơ bản là **bảng vẽ lưu tại trình duyệt**, không có DB riêng trong Compose, **không cần mount volume** cho sơ đồ. Đừng xóa dữ liệu trình duyệt nếu chưa xuất file.
- Chủ động chọn **Save to file / Export → .excalidraw** hoặc PNG/SVG để sao lưu. Mỗi thiết bị/trình duyệt có vùng lưu riêng.
- Image chính thức chỉ chứa **frontend**, không chứa server cộng tác hay kho lưu file riêng. Một số tính năng cộng tác/chia sẻ nếu hoạt động sẽ dựa vào máy chủ bên ngoài được cấu hình sẵn trong bản build. **Không coi bản này là cộng tác hoàn toàn nội bộ hoặc độc lập Internet.**
- Chưa có mật khẩu truy cập trang Excalidraw. Chỉ mở trong LAN tin cậy; **không NAT 18896 ra Internet**, muốn từ xa thì dùng VPN / HTTPS / giải pháp kiểm soát truy cập đã có sẵn nếu được yêu cầu.
- Docker image `latest` từ tác giả gốc, **không build từ source fork**. File Compose sửa trong fork quyết định cài đặt; nếu sau này muốn sửa code Excalidraw, phải chủ động build image khác.
- Không tự bật GitOps auto update, không thêm reverse proxy hay backend khi chưa yêu cầu.

**Trạng thái:** Đã sửa `docker-compose.yml` trên nhánh `master`; chưa cài Docker/Portainer và chưa xác minh 18896 trên máy đang chạy.
