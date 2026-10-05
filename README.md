# Bài tập 1: Thay đổi cổng kết nối SSH (SSH Port Hardening)

## Thông tin sinh viên
- **Họ và tên:** Phạm Thanh Phong
- **MSSV:** N24DTCN120
- **Môn học:** IT209 - Phát triển ứng dụng Web / Cloud Infrastructure
- **Session:** Session 03 - Bài tập 1

---

## Tóm tắt các bước thực hiện
1. **Mở cổng 2222 trên UFW & Cloud Firewall:**
   ```bash
   sudo ufw allow 2222/tcp
   ```
2. **Cấu hình OpenSSH Daemon:**
   - Chỉnh sửa `/etc/ssh/sshd_config`: Đặt `Port 2222`.
   - Kiểm tra cú pháp: `sudo sshd -t`.
   - Khởi động lại dịch vụ: `sudo systemctl restart ssh`.
3. **Kiểm tra kết nối và đóng cổng 22:**
   - Kết nối cổng mới: `ssh -p 2222 -i ~/.ssh/id_digitalocean_ed25519 devops@<IP>` (Thành công).
   - Đóng cổng cũ trên UFW: `sudo ufw delete allow 22/tcp`.
   - Kiểm tra cổng cũ: `ssh -p 22 devops@<IP>` (Bị từ chối).

Chi tiết xem tại: [IT209_Session03_BT1_PhamThanhPhong_N24DTCN120.md](./IT209_Session03_BT1_PhamThanhPhong_N24DTCN120.md)
