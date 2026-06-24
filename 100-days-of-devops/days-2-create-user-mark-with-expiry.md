# Tạo user tạm thời có ngày hết hạn (User mark)

## 1. Đề bài gốc (tiếng Anh)

> As part of the temporary assignment to the Nautilus project, a developer named mark requires access for a limited duration. To ensure smooth access management, a temporary user account with an expiry date is needed. Here's what you need to do:
>
> Create a user named **mark** on **App Server 1** in Stratos Datacenter. Set the expiry date to **2027-03-28**, ensuring the user is created in lowercase as per standard protocol.

## 2. Dịch đề bài (tiếng Việt)

> Là một phần của nhiệm vụ tạm thời cho dự án Nautilus, một lập trình viên tên **mark** cần được cấp quyền truy cập trong một khoảng thời gian giới hạn. Để quản lý quyền truy cập một cách trơn tru, cần tạo một tài khoản user tạm thời có **ngày hết hạn (expiry date)**. Nhiệm vụ của bạn như sau:
>
> Tạo một user tên là **mark** trên **App Server 1** trong Stratos Datacenter. Đặt ngày hết hạn là **2027-03-28**, đảm bảo user được tạo bằng chữ thường (lowercase) theo đúng quy chuẩn.

## 3. Giải thích đề bài

### Expiry date (ngày hết hạn tài khoản) là gì?

- Là ngày mà sau đó tài khoản user sẽ **bị vô hiệu hoá** — user không thể đăng nhập được nữa.
- Đây là cơ chế quản lý quyền truy cập tạm thời rất hữu ích: với nhân sự làm việc theo hợp đồng/ngắn hạn (như `mark` trong dự án Nautilus), ta đặt sẵn ngày hết hạn để hệ thống **tự động khoá tài khoản** khi hết thời hạn, không phải nhớ xoá thủ công.

### Tại sao cần đặt expiry date?

Đây là một **best practice về bảo mật và quản trị**:
- Tránh việc tài khoản tạm thời bị "bỏ quên" và tồn tại mãi mãi → giảm bề mặt tấn công.
- Tự động hoá vòng đời tài khoản (account lifecycle), giảm gánh nặng cho admin.

### Định dạng ngày & yêu cầu lowercase

- Lệnh `useradd`/`usermod` dùng tùy chọn `-e` (expire) với định dạng ngày chuẩn **`YYYY-MM-DD`** → ở đây là `2027-03-28`.
- "Created in lowercase": tên user phải là `mark` (toàn chữ thường), không phải `Mark` hay `MARK`. Đây là quy chuẩn đặt tên trên Linux (Linux phân biệt hoa/thường, và username chuẩn nên viết thường).

## 4. Lời giải

> ⚠️ Đề yêu cầu tạo user **trên App Server 1 (stapp01)**, KHÔNG phải trên jump-host.
> Mặc định terminal đứng ở jump-host, phải SSH sang stapp01 trước khi chạy lệnh.

### Bước 1: SSH vào App Server 1

Thông tin server (theo cấu hình chuẩn của KodeKloud):

| Server        | Hostname | IP            | User | Password  |
|---------------|----------|---------------|------|-----------|
| App Server 1  | stapp01  | 172.16.238.10 | tony | `Ir0nM@n` |

```bash
ssh tony@stapp01
```

- Gõ `yes` nếu được hỏi xác nhận fingerprint (lần đầu kết nối).
- Nhập mật khẩu của user `tony` khi được yêu cầu.

**Xác nhận đúng máy:** prompt phải đổi từ `thor@jump-host ~$` thành `tony@stapp01 ~$` rồi mới làm tiếp.

### Bước 2: Tạo user mark với ngày hết hạn

> Chỉ chạy lệnh này khi prompt đã là `tony@stapp01 ~$`.

```bash
sudo useradd -e 2027-03-28 mark
```

Giải thích lệnh:
- `sudo` : chạy với quyền root (tạo user cần quyền quản trị).
- `useradd` : lệnh tạo user mới.
- `-e 2027-03-28` : tùy chọn **expire date**, đặt ngày hết hạn tài khoản theo định dạng `YYYY-MM-DD`.
- `mark` : tên user (viết thường theo đúng yêu cầu).

### Bước 3: Kiểm tra (verify) kết quả

Cách 1 — Dùng `chage` để xem thông tin tuổi thọ tài khoản (rõ ràng nhất):

```bash
sudo chage -l mark
```

Kết quả mong đợi sẽ có dòng:

```
Account expires : Mar 28, 2027
```

Cách 2 — Kiểm tra nhanh trong file `/etc/shadow` (cột thứ 8 là số ngày hết hạn tính từ 1970-01-01):

```bash
sudo grep mark /etc/shadow
```

Cách 3 — Xác nhận user tồn tại trong `/etc/passwd`:

```bash
getent passwd mark
```

## 5. Giải thích chi tiết

### Tùy chọn `-e` và định dạng ngày

| Tùy chọn | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| `-e YYYY-MM-DD` | Đặt ngày tài khoản hết hạn | `-e 2027-03-28` |
| `-e ""` | Xoá (bỏ) ngày hết hạn | `usermod -e "" mark` |
| `-e -1` | Vô hiệu hoá expiry (tài khoản không hết hạn) | `useradd -e -1 mark` |

### Cách hệ thống lưu expiry date

`/etc/shadow` lưu ngày hết hạn dưới dạng **số ngày kể từ 01/01/1970** (Unix epoch), không phải dạng `YYYY-MM-DD`. Ví dụ `2027-03-28` được lưu là `20905`. Lệnh `chage -l` sẽ tự dịch ra ngày dễ đọc cho bạn.

### Phân biệt expiry vs các khái niệm liên quan

| Khái niệm | Tùy chọn | Ý nghĩa |
|-----------|----------|---------|
| Account expiry | `-e` (useradd/usermod) | Ngày tài khoản bị khoá hẳn |
| Password expiry | `-M` (chage) | Số ngày mật khẩu còn hiệu lực |

Đề bài yêu cầu **account expiry** → dùng `-e`.

## 6. Lỗi thường gặp

- **Chạy nhầm trên jump-host:** prompt vẫn là `thor@jump-host` mà đã chạy `useradd` → bài chấm báo user không tồn tại trên App Server 1. **Khắc phục:** SSH vào `stapp01` trước, kiểm tra prompt.
- **Viết hoa tên user:** tạo `Mark`/`MARK` thay vì `mark` → sai yêu cầu lowercase, bài fail. Luôn dùng đúng `mark`.
- **Sai định dạng ngày:** dùng `28-03-2027` hay `03/28/2027` → lỗi. Định dạng đúng là `YYYY-MM-DD` (`2027-03-28`).
- **User đã tồn tại:** nếu `mark` đã có sẵn, dùng `usermod` để set expiry thay vì `useradd`:

  ```bash
  sudo usermod -e 2027-03-28 mark
  ```

## 7. Tóm tắt lệnh (Quick reference)

```bash
# 1. Đăng nhập App Server 1 (BẮT BUỘC làm trước)
ssh tony@stapp01

# 2. Xác nhận prompt là tony@stapp01 rồi tạo user có ngày hết hạn
sudo useradd -e 2027-03-28 mark

# 3. Kiểm tra
sudo chage -l mark
# Account expires : Mar 28, 2027
```
