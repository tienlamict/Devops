# Tạo user với non-interactive shell (User ravi)

## 1. Đề bài gốc (tiếng Anh)

> To accommodate the backup agent tool's specifications, the system admin team at xFusionCorp Industries requires the creation of a user with a non-interactive shell. Here's your task:
>
> Create a user named **ravi** with a non-interactive shell on **App Server 3**.

## 2. Dịch đề bài (tiếng Việt)

> Để đáp ứng các thông số kỹ thuật của công cụ backup agent, đội ngũ quản trị hệ thống (system admin) tại xFusionCorp Industries yêu cầu tạo một user với **non-interactive shell** (shell không tương tác). Nhiệm vụ của bạn như sau:
>
> Tạo một user có tên là **ravi** với **non-interactive shell** trên **App Server 3**.

## 3. Giải thích đề bài

### Non-interactive shell là gì?

- **Interactive shell** (shell tương tác): là loại shell mà người dùng có thể đăng nhập, gõ lệnh và nhận kết quả qua terminal (ví dụ: `/bin/bash`, `/bin/sh`).
- **Non-interactive shell** (shell không tương tác): là loại shell **không cho phép đăng nhập tương tác**. User vẫn tồn tại trong hệ thống nhưng không thể login vào shell để gõ lệnh.

### Tại sao cần non-interactive shell?

Trong tình huống này, user `ravi` được tạo ra để phục vụ cho **backup agent tool** (một tiến trình/dịch vụ chạy nền), chứ không phải cho con người đăng nhập sử dụng. Đây là một **best practice về bảo mật**:

- User dịch vụ (service account) chỉ cần tồn tại để sở hữu tiến trình/file, không cần đăng nhập.
- Nếu gán non-interactive shell, kẻ tấn công không thể dùng tài khoản này để mở phiên shell, giảm rủi ro bảo mật.

### Shell non-interactive phổ biến trên Linux

Trên hầu hết các hệ thống (CentOS/RHEL/Rocky Linux mà KodeKloud sử dụng), shell non-interactive thường là:

```
/sbin/nologin
```

hoặc

```
/usr/sbin/nologin
```

(Ngoài ra `/bin/false` cũng là một lựa chọn non-interactive, nhưng `/sbin/nologin` thân thiện hơn vì hiển thị thông báo lịch sự khi có ai cố đăng nhập.)

## 4. Lời giải

> ⚠️ **QUAN TRỌNG NHẤT:** Đề bài yêu cầu tạo user **trên App Server 3**, KHÔNG phải trên `jump-host`.
> Khi mở terminal, mặc định bạn đang đứng ở máy `jump-host` (prompt là `thor@jump-host`).
> **Bắt buộc phải SSH vào App Server 3 trước**, nếu chạy `useradd` ngay trên jump-host thì
> hệ thống chấm điểm sẽ báo lỗi: `user 'ravi' does not exist on App Server 3`.

### Bước 1: SSH vào App Server 3 (KHÔNG ĐƯỢC BỎ QUA)

Từ `jump-host`, đăng nhập vào App Server 3. Thông tin server (theo cấu hình chuẩn của KodeKloud):

| Server        | Hostname    | IP            | User   | Password   |
|---------------|-------------|---------------|--------|------------|
| App Server 3  | stapp03     | 172.16.238.12 | banner | `BigGr33n` |

```bash
ssh banner@stapp03
```

- Gõ `yes` nếu được hỏi xác nhận fingerprint (lần đầu kết nối).
- Nhập mật khẩu của user `banner` khi được yêu cầu.

**Kiểm tra bạn đã vào đúng máy:** sau khi đăng nhập thành công, prompt phải đổi từ
`thor@jump-host ~$` thành `banner@stapp03 ~$`. Chỉ khi thấy `stapp03` mới chạy bước tiếp theo.

### Bước 2: Tạo user ravi với non-interactive shell

> Chỉ chạy lệnh này khi prompt đã là `banner@stapp03 ~$`.

Sử dụng lệnh `useradd` với tùy chọn `-s` để chỉ định shell:

```bash
sudo useradd -s /sbin/nologin ravi
```

Giải thích lệnh:
- `sudo` : chạy với quyền root (vì tạo user cần quyền quản trị).
- `useradd` : lệnh tạo user mới.
- `-s /sbin/nologin` : chỉ định **login shell** cho user là `/sbin/nologin` (non-interactive shell).
- `ravi` : tên của user cần tạo.

### Bước 3: Kiểm tra (verify) kết quả

Cách 1 — Xem thông tin trong file `/etc/passwd`:

```bash
cat /etc/passwd | grep ravi
```

Kết quả mong đợi (cột cuối cùng là shell):

```
ravi:x:1001:1001::/home/ravi:/sbin/nologin
```

Cách 2 — Dùng lệnh `getent`:

```bash
getent passwd ravi
```

Cách 3 — Kiểm tra trực tiếp shell của user:

```bash
grep ravi /etc/passwd | awk -F: '{print $7}'
```

Nếu kết quả trả về `/sbin/nologin` nghĩa là user đã được tạo đúng yêu cầu.

## 5. Giải thích chi tiết các thành phần

### Cấu trúc một dòng trong /etc/passwd

```
ravi:x:1001:1001::/home/ravi:/sbin/nologin
 │   │  │    │   │     │            │
 │   │  │    │   │     │            └── Login shell (non-interactive)
 │   │  │    │   │     └── Home directory
 │   │  │    │   └── GECOS (mô tả, để trống)
 │   │  │    └── GID (Group ID)
 │   │  └── UID (User ID)
 │   └── Mật khẩu (x = lưu mã hóa trong /etc/shadow)
 └── Username
```

### So sánh các loại shell

| Shell           | Loại            | Mục đích sử dụng                          |
|-----------------|-----------------|-------------------------------------------|
| `/bin/bash`     | Interactive     | User đăng nhập bình thường                 |
| `/bin/sh`       | Interactive     | Shell cơ bản                               |
| `/sbin/nologin` | Non-interactive | Service account, hiển thị thông báo từ chối |
| `/bin/false`    | Non-interactive | Service account, thoát ngay không thông báo |

### Lưu ý

- Nếu user `ravi` đã tồn tại từ trước và bạn chỉ cần đổi shell, dùng lệnh:

  ```bash
  sudo usermod -s /sbin/nologin ravi
  ```

- Để chắc chắn đường dẫn `nologin` tồn tại trên hệ thống, có thể kiểm tra:

  ```bash
  which nologin
  ```

## 6. Lỗi thường gặp

### ❌ Lỗi: `user 'ravi' does not exist on App Server 3`

**Nguyên nhân:** Bạn chạy lệnh `useradd` ngay trên `jump-host` (prompt vẫn là
`thor@jump-host`) mà quên SSH vào `stapp03`. User `ravi` được tạo trên jump-host chứ
không phải trên App Server 3, nên hệ thống chấm điểm không tìm thấy.

**Cách khắc phục:** SSH vào App Server 3 trước (`ssh banner@stapp03`), kiểm tra prompt đã
là `banner@stapp03`, rồi mới chạy `useradd`.

> 📌 Ghi nhớ: `jump-host` chỉ là máy bàn đạp. Hầu hết task của KodeKloud yêu cầu thao tác
> trên một server cụ thể (App Server 1/2/3, DB server...). **Luôn nhìn prompt
> `user@hostname` để chắc chắn đang đứng đúng máy trước khi gõ lệnh.**

## 7. Tóm tắt lệnh (Quick reference)

```bash
# 1. Đăng nhập App Server 3 (BẮT BUỘC làm trước)
ssh banner@stapp03

# 2. Xác nhận prompt là banner@stapp03 rồi tạo user
sudo useradd -s /sbin/nologin ravi

# 3. Kiểm tra
getent passwd ravi
# ravi:x:1002:1002::/home/ravi:/sbin/nologin
```
