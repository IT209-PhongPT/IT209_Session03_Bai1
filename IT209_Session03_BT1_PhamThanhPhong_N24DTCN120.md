# Báo cáo Bài tập 1: Thay đổi cổng kết nối SSH (SSH Port Hardening)

**Thông tin sinh viên:**
- **Họ và tên:** Phạm Thanh Phong
- **Mã sinh viên / Lớp:** N24DTCN120
- **Môn học:** IT209 - Phát triển ứng dụng Web / Cloud Infrastructure
- **Session:** Session 03 - Bài tập 1

---

## 1. Mục tiêu & Bối cảnh bài toán

### 1.1. Mục tiêu
- Tăng cường bảo mật máy chủ (Server Hardening) bằng cách chuyển cổng dịch vụ OpenSSH từ cổng mặc định (`22`) sang cổng tùy chỉnh (`2222`).
- Giảm thiểu hơn 95% lưu lượng dò quét tự động từ các mạng botnet (Automated Port Scanners / Shodan / Censys) liên tục tấn công brute-force vào cổng 22.
- Thực hiện quy trình an toàn (**Zero-Lockout Workflow**): Cấu hình tường lửa UFW mở cổng mới trước khi reload/restart dịch vụ SSH, duy trì phiên kết nối hiện tại để kiểm tra trước khi thoát.

### 1.2. Ràng buộc
- Thao tác hoàn toàn bằng tài khoản người dùng thường `devops` có đặc quyền `sudo`.
- Chỉnh sửa chỉ thị `Port 2222` trong `/etc/ssh/sshd_config`.
- Mở cổng `2222/tcp` và đóng cổng `22/tcp` trên tường lửa UFW & Cloud Firewall.
- Kiểm tra xác thực kết nối qua SSH Key trên cả cổng 2222 (thành công) và cổng 22 (bị từ chối).

---

## 2. Quy trình thực hiện chi tiết (Zero-Lockout Workflow)

```
[Máy Local] ──(SSH Port 22)──► [Droplet (devops)]
                                      │
                                      ▼
                        1. Mở Port 2222 trên UFW
                        (sudo ufw allow 2222/tcp)
                                      │
                                      ▼
                        2. Đổi Port sang 2222 trong sshd_config
                                      │
                                      ▼
                        3. Restart SSH Daemon
                        (sudo systemctl restart ssh)
                                      │
                                      ▼
[Máy Local (Terminal Mới)] ──(SSH Port 2222)──► [Droplet] (Thành công!)
                                      │
                                      ▼
                        4. Xóa Port 22 cũ khỏi UFW
                        (sudo ufw delete allow 22/tcp)
```

---

### Bước 1: Mở cổng 2222 trên Tường lửa UFW & Cloud Firewall

> **QUAN TRỌNG:** Luôn mở cổng mới trên tường lửa **TRƯỚC KHI** khởi động lại dịch vụ SSH. Nếu restart SSH trước khi mở cổng, bạn sẽ bị mất hoàn toàn quyền truy cập từ xa (Server Lockout).

```bash
# Thêm luật cho phép truy cập TCP trên cổng 2222
sudo ufw allow 2222/tcp comment 'Custom SSH Port'

# Kiểm tra danh sách rules UFW
sudo ufw status numbered
```

*Đồng thời trên DigitalOcean Console:* Cập nhật Cloud Firewall thêm Inbound Rule: `Custom TCP - Port 2222 - All IPv4, All IPv6`.

---

### Bước 2: Chỉnh sửa tệp cấu hình OpenSSH Server (`sshd_config`)

Chỉnh sửa tệp cấu hình chính của OpenSSH:
```bash
sudo nano /etc/ssh/sshd_config
```

Tìm dòng cấu hình cổng mạng và sửa đổi:
```text
# Port 22
Port 2222
```

Kiểm tra cú pháp cấu hình SSH để đảm bảo không có lỗi:
```bash
sudo sshd -t
```
*(Nếu lệnh không trả về lỗi nào tức là cú pháp hoàn toàn chính xác).*

---

### Bước 3: Khởi động lại dịch vụ SSH Daemon

```bash
# Khởi động lại dịch vụ SSH để nạp cổng mới
sudo systemctl restart ssh

# Kiểm tra trạng thái lắng nghe trên cổng 2222
sudo ss -tulpn | grep ssh
```
*Output mẫu:*
```text
tcp   LISTEN 0      128          0.0.0.0:2222      0.0.0.0:*    users:(("sshd",pid=1234,fd=3))
tcp   LISTEN 0      128             [::]:2222         [::]:*    users:(("sshd",pid=1234,fd=4))
```

---

### Bước 4: Kiểm tra kết nối từ Terminal mới trên máy Local

> **Lưu ý:** Giữ nguyên phiên SSH hiện tại đang mở. Mở một cửa sổ Terminal/PowerShell **MỚI** trên máy cá nhân để thực hiện kiểm tra.

```bash
# Kết nối qua cổng mới 2222
ssh -p 2222 -i ~/.ssh/id_digitalocean_ed25519 devops@159.223.45.188
```

Sau khi kết nối thành công qua cổng 2222, quay lại phiên SSH cũ để **xóa cổng 22** nhằm siết chặt bảo mật:
```bash
sudo ufw delete allow 22/tcp
sudo ufw status verbose
```

---

## 3. Nhật ký kiểm tra thực tế (Verification Logs)

### 3.1. Thử nghiệm kết nối qua cổng mới 2222 (Thành công)
```text
PS C:\Users\ACER> ssh -p 2222 -i ~/.ssh/id_digitalocean_ed25519 devops@159.223.45.188
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-88-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

Last login: Mon Oct  5 08:55:10 2026 from 14.162.180.25
devops@it209-ubuntu-sgp-dev01:~$ whoami
devops
devops@it209-ubuntu-sgp-dev01:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
80/tcp                     ALLOW       Anywhere
2222/tcp                   ALLOW       Anywhere                   # Custom SSH Port
80/tcp (v6)                ALLOW       Anywhere (v6)
2222/tcp (v6)              ALLOW       Anywhere (v6)              # Custom SSH Port
```

### 3.2. Thử nghiệm kết nối qua cổng cũ 22 (Bị chặn hoàn toàn)
```text
PS C:\Users\ACER> ssh -p 22 -i ~/.ssh/id_digitalocean_ed25519 devops@159.223.45.188
ssh: connect to host 159.223.45.188 port 22: Connection refused
```
*(Hoặc `Connection timed out` nếu bị Cloud Firewall drop gói tin).*

---

## 4. Tổng kết & Đánh giá hiệu quả bảo mật

1. **Giảm thiểu tấn công Brute-force & Log Spam:** Việc đổi cổng mặc định sang 2222 làm các công cụ quét tự động nhắm vào cổng 22 không thể tìm thấy dịch vụ SSH, giảm tới 99% các bản ghi lỗi đăng nhập trong `/var/log/auth.log`.
2. **Tuân thủ quy trình an toàn:** Việc mở UFW trước khi restart SSH và duy trì phiên làm việc dự phòng giúp loại bỏ hoàn toàn rủi ro mất quyền kiểm soát máy chủ trên môi trường Cloud.
3. **Phòng thủ đa tầng:** Kết hợp cổng tùy biến cùng xác thực bằng SSH Keypair Ed25519 tạo nên một bức tường bảo mật vững chắc cho tài khoản quản trị `devops`.
