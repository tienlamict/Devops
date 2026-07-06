# Cài đặt gói SELinux và vô hiệu hóa (disable) vĩnh viễn trên App Server 2

## 1. Đề bài gốc (tiếng Anh)

> Following a security audit, the xFusionCorp Industries security team has opted to enhance application and server security with SELinux. To initiate testing, the following requirements have been established for App server 2 in the Stratos Datacenter:
>
> - Install the required SELinux packages.
> - Permanently disable SELinux for the time being; it will be re-enabled after necessary configuration changes.
> - No need to reboot the server, as a scheduled maintenance reboot is already planned for tonight.
> - Disregard the current status of SELinux via the command line; the final status after the reboot should be disabled.

---

## 2. Dịch đề bài (tiếng Việt)

Sau một đợt kiểm tra bảo mật (security audit), đội bảo mật của xFusionCorp Industries quyết định tăng cường bảo mật cho ứng dụng và máy chủ bằng **SELinux**. Để bắt đầu quá trình kiểm thử, các yêu cầu sau được đặt ra cho **App Server 2** trong Stratos Datacenter:

- Cài đặt các **gói SELinux cần thiết**.
- **Vô hiệu hóa SELinux vĩnh viễn** (permanently disable) trong thời điểm hiện tại; nó sẽ được bật lại sau khi hoàn tất các thay đổi cấu hình cần thiết.
- **Không cần khởi động lại** server, vì đã có lịch reboot bảo trì vào tối nay.
- **Bỏ qua trạng thái hiện tại** của SELinux khi kiểm tra bằng dòng lệnh; trạng thái cuối cùng **sau khi reboot** phải là **disabled**.

---

## 3. Giải thích đề bài

**SELinux (Security-Enhanced Linux)** là một cơ chế kiểm soát truy cập bắt buộc (MAC – Mandatory Access Control) tích hợp trong nhân Linux. Nó áp đặt các chính sách (policy) hạn chế quyền của tiến trình, giúp giảm thiệt hại khi một dịch vụ bị tấn công.

SELinux có 3 chế độ (mode):

| Mode | Ý nghĩa |
|------|---------|
| `enforcing` | Bật đầy đủ, chặn và ghi log các hành vi vi phạm policy |
| `permissive` | Không chặn, chỉ ghi log cảnh báo (dùng để test) |
| `disabled` | Tắt hoàn toàn SELinux |

**Điểm mấu chốt của bài này:**

- Việc chuyển SELinux sang trạng thái **`disabled`** (tắt hoàn toàn) **bắt buộc phải reboot** mới có hiệu lực. Không thể chuyển từ `enforcing`/`permissive` sang `disabled` bằng lệnh runtime (`setenforce` chỉ chuyển được giữa enforcing ↔ permissive, không xuống được disabled).
- Do đó cách "disable vĩnh viễn" đúng chuẩn là **sửa file cấu hình `/etc/selinux/config`** đặt `SELINUX=disabled`. File này được đọc khi boot, nên sau lần reboot bảo trì tối nay trạng thái sẽ là disabled.
- Đề nói rõ: **không cần reboot bây giờ** và **bỏ qua trạng thái runtime hiện tại** (`getenforce`/`sestatus` có thể vẫn hiện enforcing/permissive) → hệ thống chấm bài chỉ kiểm tra nội dung file config, không kiểm tra trạng thái đang chạy.

Vì vậy công việc gồm **2 phần**:
1. Cài các gói SELinux (đảm bảo `libselinux`, các tiện ích quản lý policy... có mặt).
2. Sửa `/etc/selinux/config` → `SELINUX=disabled`.

---

## 4. Lời giải (các bước chi tiết)

### Bước 1 — SSH vào App Server 2 (stapp02)

> ⚠️ Terminal mặc định đang đứng ở **jump-host**. Bắt buộc phải SSH sang **App Server 2** trước khi chạy lệnh, nếu không sẽ cấu hình nhầm máy → **fail bài**.

```bash
ssh steve@stapp02
```

- `steve` : user của App Server 2.
- `stapp02` : hostname của App Server 2 (IP tham khảo `172.16.238.11`).
- Nhập mật khẩu khi được hỏi (tham khảo: `Am3ric@` — ưu tiên thông tin thực tế trong lab).
- Xác nhận prompt đã đổi thành `steve@stapp02` trước khi làm tiếp.

### Bước 2 — Chuyển sang quyền root

```bash
sudo -i
```

- Các thao tác cài gói và sửa file cấu hình hệ thống cần quyền root. Nhập lại mật khẩu của `steve` nếu được hỏi.

### Bước 3 — Cài đặt các gói SELinux cần thiết

```bash
sudo yum install -y selinux-policy selinux-policy-targeted policycoreutils policycoreutils-python-utils libselinux-utils setroubleshoot-server
```

Giải thích từng gói:

| Gói | Vai trò |
|-----|---------|
| `selinux-policy` | Bộ policy nền tảng của SELinux |
| `selinux-policy-targeted` | Policy mặc định (targeted) áp dụng cho các dịch vụ |
| `policycoreutils` | Các tiện ích lõi: `setenforce`, `restorecon`, `semodule`... |
| `policycoreutils-python-utils` | Công cụ Python: `semanage`, `audit2allow`... |
| `libselinux-utils` | Cung cấp `getenforce`, `sestatus`, `setenforce` |
| `setroubleshoot-server` | Hỗ trợ phân tích/log sự cố SELinux |

> Nếu server dùng `dnf` (RHEL/Rocky 8+) thì thay `yum` bằng `dnf` (thường `yum` vẫn là alias của `dnf`).
> Nếu một vài gói đã có sẵn, lệnh sẽ báo "already installed" — không sao.

### Bước 4 — Vô hiệu hóa SELinux vĩnh viễn qua file config

Mở file cấu hình:

```bash
sudo vi /etc/selinux/config
```

Sửa dòng `SELINUX=` thành:

```ini
SELINUX=disabled
```

Hoặc làm nhanh bằng `sed` (an toàn, không cần mở editor):

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

- `s/^SELINUX=.*/SELINUX=disabled/` : tìm dòng bắt đầu bằng `SELINUX=` (chú ý: **không phải** `SELINUXTYPE=`) và thay toàn bộ dòng thành `SELINUX=disabled`.
- `^` đảm bảo chỉ khớp dòng `SELINUX=`, tránh nhầm với `SELINUXTYPE=`.
- `-i` : chỉnh sửa trực tiếp trong file (in-place).

### Bước 5 — Kiểm tra (verify)

Xác nhận nội dung file config đã đúng:

```bash
sudo cat /etc/selinux/config
```

Kết quả mong đợi phải có dòng:

```ini
SELINUX=disabled
```

Kiểm tra chắc chắn bằng grep (bỏ dòng comment):

```bash
grep -E '^SELINUX=' /etc/selinux/config
# Kết quả: SELINUX=disabled
```

> **Lưu ý:** Nếu chạy `getenforce` hoặc `sestatus` lúc này, kết quả có thể vẫn là `Enforcing` hoặc `Permissive`. **Điều này bình thường và ĐÚNG với đề** — trạng thái `disabled` chỉ có hiệu lực sau khi reboot (đề đã yêu cầu bỏ qua trạng thái runtime hiện tại). **KHÔNG** cần chạy `setenforce 0` để "sửa" điều này.

---

## 5. Giải thích chi tiết

**Vì sao không dùng `setenforce 0`?**

```
setenforce 0   →  chuyển Enforcing → Permissive  (chỉ tạm thời, mất khi reboot)
setenforce 1   →  chuyển Permissive → Enforcing
setenforce     →  KHÔNG thể đưa về disabled
```

→ Chỉ có sửa `/etc/selinux/config` mới đưa được về `disabled`, và cần reboot để nhân Linux áp dụng.

**Luồng hoạt động:**

```
Sửa /etc/selinux/config (SELINUX=disabled)
                │
                ▼
   Reboot (theo lịch bảo trì tối nay)
                │
                ▼
  Kernel đọc config lúc boot → SELinux = disabled  ✅
```

**So sánh runtime vs. persistent:**

| Cách | Có hiệu lực ngay? | Sống sót sau reboot? | Đưa về disabled được? |
|------|-------------------|----------------------|------------------------|
| `setenforce 0/1` | ✅ | ❌ | ❌ |
| Sửa `/etc/selinux/config` | ❌ (cần reboot) | ✅ | ✅ |

---

## 6. Lỗi thường gặp

| Lỗi | Hậu quả | Cách khắc phục |
|-----|---------|----------------|
| Quên SSH sang stapp02, chạy ngay trên jump-host | Cấu hình sai máy → fail | Luôn kiểm tra prompt `steve@stapp02` trước khi chạy |
| Sửa nhầm dòng `SELINUXTYPE=` thay vì `SELINUX=` | Trạng thái không đổi | Dùng regex `^SELINUX=` để chỉ khớp đúng dòng |
| Chạy `setenforce 0` và tưởng đã "disable" | Chỉ permissive, mất sau reboot | Không dùng `setenforce`; phải sửa file config |
| Cố `reboot` server để kiểm tra | Đề yêu cầu **không** reboot | Chỉ sửa config, để lịch bảo trì tự reboot |
| Lo lắng vì `getenforce` vẫn hiện `Enforcing` | Tưởng làm sai | Đề bảo bỏ qua trạng thái runtime — file config đúng là đủ |
| Đặt `SELINUX=disable` (thiếu chữ `d`) | Giá trị sai, không disabled | Phải đúng chính tả `disabled` |

---

## 7. Tóm tắt lệnh (Quick reference)

```bash
# 1) SSH vào App Server 2
ssh steve@stapp02

# 2) Lên quyền root
sudo -i

# 3) Cài các gói SELinux
yum install -y selinux-policy selinux-policy-targeted policycoreutils \
  policycoreutils-python-utils libselinux-utils setroubleshoot-server

# 4) Disable SELinux vĩnh viễn (sửa file config)
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# 5) Verify
grep -E '^SELINUX=' /etc/selinux/config    # => SELINUX=disabled

# KHÔNG reboot, KHÔNG cần setenforce. Reboot bảo trì tối nay sẽ áp dụng.
```
