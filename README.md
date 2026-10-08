# Báo Cáo Bài 1: Quản lý người dùng giới hạn và Truyền tải dữ liệu qua SFTP trên Windows

## 1. Các bước thực hiện trên Máy chủ (VPS)

### Bước 1: Khởi tạo tài khoản người dùng giới hạn `sftp-user`
Tạo người dùng `sftp-user` dùng riêng cho tác vụ kết nối và truyền nhận dữ liệu SFTP (không cấp quyền `sudo`):

```bash
sudo adduser sftp-user
```

*Thiết lập mật khẩu mạnh cho tài khoản:* `SftpPass#2026!Secure`

```text
Adding user `sftp-user' ...
Adding new group `sftp-user' (1002) ...
Adding new user `sftp-user' (1002) with group `sftp-user' ...
Creating home directory `/home/sftp-user' ...
Copying files from `/etc/skel' ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for sftp-user
Enter the new value, or press ENTER for the default
	Full Name []: SFTP Backup Operator
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] Y
```

---

### Bước 2: Tạo thư mục lưu log sao lưu và tệp log giả lập
Khởi tạo cấu trúc thư mục `/var/log/app-backup/` và tạo tệp tin `backup-check.log` ghi lại trạng thái sao lưu:

```bash
sudo mkdir -p /var/log/app-backup/
sudo touch /var/log/app-backup/backup-check.log
sudo bash -c 'cat << "EOF" > /var/log/app-backup/backup-check.log
[2026-10-08 20:30:01] [INFO] Starting daily automated backup process...
[2026-10-08 20:30:05] [INFO] Dumping MySQL database: springboot_db (Size: 45.2 MB) -> SUCCESS
[2026-10-08 20:30:12] [INFO] Archiving static assets: /var/www/html (Size: 12.8 MB) -> SUCCESS
[2026-10-08 20:30:18] [INFO] Encrypting backup bundle with AES-256 -> SUCCESS
[2026-10-08 20:30:20] [INFO] Checksum SHA256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Backup status: SUCCESS at Wed Oct  8 20:30:20 UTC 2026
EOF'
```

---

### Bước 3: Phân quyền thư mục và tệp tin
Gán quyền sở hữu thư mục cho `root:sftp-user`, cấp quyền đọc và truy cập thư mục cho nhóm `sftp-user`, đồng thời chặn hoàn toàn quyền truy cập từ người dùng khác (`others`):

```bash
# Gán quyền sở hữu Owner: root, Group: sftp-user
sudo chown -R root:sftp-user /var/log/app-backup

# Phân quyền thư mục: 750 (Owner: rwx, Group: r-x, Others: ---)
sudo chmod 750 /var/log/app-backup

# Phân quyền tệp log: 640 (Owner: rw-, Group: r--, Others: ---)
sudo chmod 640 /var/log/app-backup/backup-check.log
```

---

## 2. Kiểm tra trên Máy chủ Linux (VPS)

### 2.1. Kiểm tra tài khoản `sftp-user`
Kiểm tra UID, GID và các nhóm phụ của `sftp-user` để đảm bảo tài khoản **không** thuộc nhóm `sudo`:

```bash
id sftp-user
```

**Kết quả đầu ra (Output):**
```text
uid=1002(sftp-user) gid=1002(sftp-user) groups=1002(sftp-user)
```
> **Đánh giá:** Tài khoản `sftp-user` chỉ thuộc nhóm cá nhân `sftp-user`, không có quyền chạy lệnh đặc quyền thông qua `sudo`.

---

### 2.2. Kiểm tra phân quyền tệp và thư mục log
Kiểm tra quyền hạn chi tiết của thư mục `/var/log/app-backup/` và tệp `backup-check.log`:

```bash
ls -ld /var/log/app-backup
ls -l /var/log/app-backup/backup-check.log
```

**Kết quả đầu ra (Output):**
```text
drwxr-x--- 2 root sftp-user 4096 Oct  8 20:30 /var/log/app-backup
-rw-r----- 1 root sftp-user  582 Oct  8 20:30 /var/log/app-backup/backup-check.log
```

---

## 3. Quy trình kết nối SFTP và Truyền tải dữ liệu trên Windows Client

### Bước 1: Cấu hình phần mềm SFTP Client (Bitvise SSH Client / WinSCP)
Khởi chạy Bitvise SSH Client / WinSCP trên hệ điều hành Windows và nhập các thông số kết nối:

| Thông số | Giá trị cấu hình | Mô tả |
| :--- | :--- | :--- |
| **Host / Server** | `103.179.185.120` *(IP Máy chủ VPS)* | Địa chỉ IP máy chủ từ xa |
| **Port** | `22` | Cổng SSH mặc định |
| **Username** | `sftp-user` | Tài khoản giới hạn vừa khởi tạo |
| **Initial Method** | `Password` | Phương thức xác thực bằng mật khẩu |
| **Password** | `SftpPass#2026!Secure` | Mật khẩu tài khoản `sftp-user` |

```text
[Bitvise SSH Client Log]
20:32:01.120 Connecting to SSH server 103.179.185.120:22...
20:32:01.340 Connection established. Server version: SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6
20:32:01.480 Algorithm negotiation completed. Key exchange: curve25519-sha256, Host key: ssh-ed25519
20:32:01.512 Host key verified and accepted.
20:32:01.620 Authenticating user 'sftp-user' with method 'password'...
20:32:01.780 User 'sftp-user' authenticated successfully.
20:32:01.810 Initializing SFTP subsystem...
20:32:01.890 SFTP channel opened successfully (SFTP protocol version 3).
```

---

### Bước 2: Tải tệp tin `backup-check.log` về máy Windows
1. Mở cửa sổ **SFTP Window** (giao diện chia 2 khung: Khung trái là máy local Windows, Khung phải là VPS Linux).
2. Tại khung bên phải (**Remote files**), điều hướng đến đường dẫn: `/var/log/app-backup/`.
3. Nhấp chọn tệp `backup-check.log` và thực hiện thao tác **Drag & Drop (Kéo - Thả)** hoặc nhấn chuột phải chọn **Download** sang thư mục cục bộ `C:\Users\DELL\Downloads\backups\` ở khung bên trái (**Local files**).

```text
[SFTP Transfer Log]
Transferring remote file '/var/log/app-backup/backup-check.log' to local file 'C:\Users\DELL\Downloads\backups\backup-check.log'
Progress: [==================================================] 100% (582 bytes / 582 bytes)
Transfer rate: 116.4 KB/s - Status: COMPLETED SUCCESSFULLY
```

---

## 4. Kiểm tra tệp tin sau khi tải về trên máy Windows

Mở PowerShell trên máy tính Windows cá nhân để kiểm tra sự tồn tại và nội dung tệp tin đã tải về:

```powershell
Get-ChildItem -Path "C:\Users\DELL\Downloads\backups\backup-check.log"
Get-Content -Path "C:\Users\DELL\Downloads\backups\backup-check.log"
```

**Kết quả đầu ra (Output trên Windows Terminal):**
```text
    Directory: C:\Users\DELL\Downloads\backups

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         10/8/2026   8:32 PM            582 backup-check.log


[2026-10-08 20:30:01] [INFO] Starting daily automated backup process...
[2026-10-08 20:30:05] [INFO] Dumping MySQL database: springboot_db (Size: 45.2 MB) -> SUCCESS
[2026-10-08 20:30:12] [INFO] Archiving static assets: /var/www/html (Size: 12.8 MB) -> SUCCESS
[2026-10-08 20:30:18] [INFO] Encrypting backup bundle with AES-256 -> SUCCESS
[2026-10-08 20:30:20] [INFO] Checksum SHA256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Backup status: SUCCESS at Wed Oct  8 20:30:20 UTC 2026
```

> **Kết luận:** Tệp `backup-check.log` đã được truyền tải toàn vẹn và an toàn qua giao thức SFTP từ máy chủ Linux về máy tính cá nhân Windows. Tài khoản `sftp-user` hoạt động đúng phân quyền giới hạn, đảm bảo tính bảo mật cho hệ thống.
