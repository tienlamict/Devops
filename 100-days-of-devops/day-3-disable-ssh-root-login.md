# Disable direct SSH root login trên các App Server

## 1. Đề bài gốc (tiếng Anh)

> Following security audits, the xFusionCorp Industries security team has rolled out new protocols, including the restriction of direct root SSH login.
>
> Your task is to disable direct SSH root login on all app servers within the Stratos Datacenter.

---

## 2. Dịch đề bài (tiếng Việt)

> Sau các đợt kiểm tra bảo mật (security audit), đội bảo mật của xFusionCorp Industries đã triển khai các quy định mới, bao gồm việc **hạn chế đăng nhập SSH trực tiếp bằng tài khoản root**.
>
> Nhiệm vụ của bạn là **vô hiệu hóa việc đăng nhập SSH trực tiếp bằng root** trên **tất cả các App Server** trong Stratos Datacenter (stapp01, stapp02, stapp03).

---

## 3. Giải thích đề bài

- **SSH root login** nghĩa là cho phép đăng nhập trực tiếp vào máy với tài khoản `root` thông qua SSH.
- Đây là một **rủi ro bảo mật lớn** vì:
  - `root` là tài khoản có **toàn quyền** trên hệ thống. Nếu kẻ tấn công đoán/brute-force được mật khẩu root, chúng chiếm toàn bộ máy.
  - Tài khoản `root` tồn tại trên **mọi** hệ thống Linux → là mục tiêu tấn công có sẵn (không cần đoán username).
  - Không có **audit trail** (dấu vết kiểm tra): mọi người đều dùng chung `root`, không biết ai đã làm gì.
- **Best practice**: đăng nhập bằng tài khoản người dùng thường, sau đó dùng `sudo`/`su` để nâng quyền khi cần. Vì vậy cần đặt `PermitRootLogin no` trong cấu hình SSH daemon.

---

## 4. Lời giải

> ⚠️ Mặc định terminal đang đứng ở **jump-host**. Phải SSH vào **từng App Server** rồi mới chỉnh sửa. Làm lần lượt cho cả 3 server: **stapp01, stapp02, stapp03**.

### Bước 1 — SSH vào App Server 1 (stapp01)

Từ jump-host:

```bash
ssh tony@stapp01
# password: Ir0nM@n
```

- `ssh tony@stapp01`: đăng nhập SSH với user `tony` vào host `stapp01`.

### Bước 2 — Mở file cấu hình SSH server và sửa `PermitRootLogin`

Cần quyền root để sửa, dùng `sudo`:

```bash
sudo sed -i 's/^#*\s*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
```

Giải thích lệnh `sed`:
- `sudo`: chạy với quyền root (sẽ hỏi password của tony — chính là `Ir0nM@n`).
- `sed -i`: chỉnh sửa file **tại chỗ** (in-place).
- `'s/.../.../'`: lệnh thay thế (substitute).
- `^#*\s*PermitRootLogin.*`: khớp dòng `PermitRootLogin` dù đang **bị comment** (`#`) hay không, có hay không có khoảng trắng, với giá trị bất kỳ.
- `PermitRootLogin no`: thay bằng dòng tắt root login.
- `/etc/ssh/sshd_config`: file cấu hình của **SSH daemon (sshd)** — lưu ý KHÔNG phải `ssh_config` (file đó dành cho client).

> 💡 Nếu muốn làm thủ công bằng editor:
> ```bash
> sudo vi /etc/ssh/sshd_config
> ```
> Tìm dòng `#PermitRootLogin ...`, bỏ dấu `#` và sửa thành:
> ```
> PermitRootLogin no
> ```

### Bước 3 — Áp dụng cấu hình (restart dịch vụ sshd)

```bash
sudo systemctl restart sshd
```

- Khởi động lại dịch vụ `sshd` để nạp cấu hình mới.

### Bước 4 — Kiểm tra (verify)

```bash
sudo sshd -t                                  # kiểm tra cú pháp file cấu hình (không output = OK)
sudo grep -i '^PermitRootLogin' /etc/ssh/sshd_config
```

Kết quả mong đợi:
```
PermitRootLogin no
```

Và kiểm tra dịch vụ đang chạy:
```bash
sudo systemctl status sshd
```

### Bước 5 — Thoát và lặp lại cho server còn lại

```bash
exit          # quay về jump-host
```

Lặp lại Bước 1–5 cho:

```bash
ssh steve@stapp02     # password: Am3ric@
ssh banner@stapp03    # password: BigGr33n
```

---

## 5. Giải thích chi tiết

### So sánh các giá trị của `PermitRootLogin`

| Giá trị                  | Ý nghĩa                                                                 |
|--------------------------|------------------------------------------------------------------------|
| `yes`                    | Cho phép root đăng nhập bằng cả mật khẩu và khóa SSH (kém an toàn nhất).|
| `no`                     | **Cấm hoàn toàn** root đăng nhập qua SSH ✅ (đáp án của bài này).        |
| `prohibit-password`      | Cho phép root nhưng **chỉ qua SSH key**, cấm mật khẩu.                  |
| `forced-commands-only`   | Chỉ cho root chạy lệnh được chỉ định trước qua key.                     |

### Phân biệt 2 file dễ nhầm

| File                    | Dành cho ai     | Vai trò                              |
|-------------------------|-----------------|--------------------------------------|
| `/etc/ssh/sshd_config`  | **Server (sshd)** | Cấu hình máy **nhận** kết nối SSH ✅ |
| `/etc/ssh/ssh_config`   | Client (ssh)    | Cấu hình máy **gửi đi** kết nối SSH  |

> Bài này sửa **sshd_config** (có chữ `d` = daemon).

---

## 6. Lỗi thường gặp

1. **Chạy lệnh ngay trên jump-host** mà chưa SSH vào app server → sửa nhầm máy, bài fail.
   → Luôn nhìn prompt `user@hostname` để xác nhận đang ở đúng server (ví dụ `tony@stapp01`).
2. **Sửa nhầm file `ssh_config`** thay vì `sshd_config` → cấu hình không có tác dụng.
3. **Quên restart sshd** → cấu hình đã sửa nhưng dịch vụ vẫn chạy với cấu hình cũ.
4. **Chỉ làm 1 server** → đề yêu cầu "all app servers" → phải làm đủ cả 3 (stapp01/02/03).
5. **Dòng bị comment `#PermitRootLogin`**: nếu chỉ thêm dòng mới mà còn dòng comment với giá trị khác có thể gây nhầm; lệnh `sed` ở trên đã xử lý cả trường hợp comment.
6. **Quên `sudo`**: user thường không có quyền ghi `/etc/ssh/sshd_config` → "Permission denied".

---

## 7. Tóm tắt lệnh (Quick reference)

Thực hiện cho **mỗi** app server (stapp01, stapp02, stapp03):

```bash
# --- stapp01 ---
ssh tony@stapp01                       # pass: Ir0nM@n
sudo sed -i 's/^#*\s*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd
sudo grep -i '^PermitRootLogin' /etc/ssh/sshd_config   # => PermitRootLogin no
exit

# --- stapp02 ---
ssh steve@stapp02                      # pass: Am3ric@
sudo sed -i 's/^#*\s*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
sudo grep -i '^PermitRootLogin' /etc/ssh/sshd_config
exit

# --- stapp03 ---
ssh banner@stapp03                     # pass: BigGr33n
sudo sed -i 's/^#*\s*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
sudo grep -i '^PermitRootLogin' /etc/ssh/sshd_config
exit
```
