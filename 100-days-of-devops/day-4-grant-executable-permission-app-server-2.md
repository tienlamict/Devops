# Grant Executable Permission cho script trên App Server 2

## 1. Đề bài gốc (tiếng Anh)

> In a bid to automate backup processes, the xFusionCorp Industries sysadmin team has developed a new bash script named xfusioncorp.sh. While the script has been distributed to all necessary servers, it lacks executable permissions on App Server 2 within the Stratos Datacenter.
>
> Your task is to grant executable permissions to the /tmp/xfusioncorp.sh script on App Server 2. Additionally, ensure that all users have the capability to execute it.

---

## 2. Dịch đề bài (tiếng Việt)

Nhằm tự động hóa quy trình sao lưu (backup), đội sysadmin của xFusionCorp Industries đã phát triển một script bash mới tên là `xfusioncorp.sh`. Mặc dù script đã được phân phối tới tất cả các server cần thiết, nhưng trên **App Server 2** trong Stratos Datacenter nó vẫn **thiếu quyền thực thi (executable)**.

Nhiệm vụ của bạn là **cấp quyền thực thi** cho script `/tmp/xfusioncorp.sh` trên App Server 2. Ngoài ra, phải đảm bảo **tất cả người dùng (all users)** đều có thể chạy được nó.

---

## 3. Giải thích đề bài

- Trong Linux, một file dù là script bash vẫn cần **bit thực thi (`x`)** thì mới chạy trực tiếp được (kiểu `./xfusioncorp.sh` hoặc `/tmp/xfusioncorp.sh`).
- Quyền của file được chia cho 3 nhóm đối tượng:
  - **u** (user/owner – chủ sở hữu)
  - **g** (group – nhóm)
  - **o** (others – những người còn lại)
- Yêu cầu "all users have the capability to execute it" nghĩa là cả 3 nhóm `u`, `g`, `o` đều phải có bit `x`. Dùng ký hiệu `a` (all) để áp cho tất cả.
- Đây là thao tác cơ bản khi triển khai script tự động hóa: sau khi copy file lên server, thường phải set quyền thực thi thì cron/backup mới chạy được.

---

## 4. Lời giải

### Bước 1 — SSH vào đúng server (App Server 2 = stapp02)

Mặc định terminal đang đứng ở jump-host. Phải chuyển sang App Server 2 trước:

```bash
ssh steve@stapp02
```

- `steve` : user của App Server 2.
- `stapp02` : hostname của App Server 2 (IP `172.16.238.11`).
- Khi được hỏi password, nhập: `Am3ric@` (gõ password sẽ không hiện trên màn hình – bình thường).
- Xác nhận đã vào đúng máy bằng cách nhìn prompt đổi thành `steve@stapp02`.

### Bước 2 — Cấp quyền thực thi cho tất cả user

```bash
sudo chmod 755 /tmp/xfusioncorp.sh
```

- `chmod` : lệnh thay đổi quyền (change mode).
- `755` (quyền tuyệt đối):
  - `7` = `rwx` cho **owner** (đọc + ghi + thực thi).
  - `5` = `r-x` cho **group** (đọc + thực thi).
  - `5` = `r-x` cho **others** (đọc + thực thi).
- `/tmp/xfusioncorp.sh` : đường dẫn file cần cấp quyền.
- Dùng `sudo` để tránh lỗi "Permission denied" nếu file không thuộc sở hữu của user `steve`.

> ⚠️ **Vì sao KHÔNG dùng `chmod a+x`?**
> `chmod a+x` chỉ **thêm** bit `x` mà giữ nguyên các bit khác. Nếu file gốc có quyền `000` (không có gì), kết quả sẽ ra `---x--x--x` — **thiếu bit đọc `r`**. Với script bash, trình thông dịch cần **đọc được** nội dung file mới chạy được, nên chỉ có `x` là chưa đủ.
> Dùng `chmod 755` sẽ **set tuyệt đối** cả `r` và `x` → luôn ra `-rwxr-xr-x` chuẩn, an toàn trong mọi trường hợp.
>
> Nếu vẫn muốn dùng ký hiệu tương đối thì phải cấp cả read lẫn execute: `sudo chmod a+rx /tmp/xfusioncorp.sh`.

### Bước 3 — Kiểm tra (verify)

```bash
ls -l /tmp/xfusioncorp.sh
```

Kết quả mong đợi (chú ý các chữ `x`):

```
-rwxr-xr-x 1 root root 1024 Jul  2 10:00 /tmp/xfusioncorp.sh
```

Cả 3 nhóm đều có `x` → đạt yêu cầu.

---

## 5. Giải thích chi tiết

### Cấu trúc chuỗi quyền `-rwxr-xr-x`

| Vị trí | Ký tự | Ý nghĩa |
|--------|-------|---------|
| 1      | `-`   | Loại file (`-` = file thường, `d` = thư mục) |
| 2-4    | `rwx` | Quyền của **owner** (đọc, ghi, thực thi) |
| 5-7    | `r-x` | Quyền của **group** (đọc, thực thi) |
| 8-10   | `r-x` | Quyền của **others** (đọc, thực thi) |

### So sánh cách viết `chmod`

| Cách viết | Ý nghĩa | Có chạy được script thật không? |
|-----------|---------|---------------------------------|
| `chmod 755 file`   | Set tuyệt đối `rwxr-xr-x` (có cả `r` + `x`) | ✅ **Tốt nhất** – luôn đúng |
| `chmod a+rx file`  | Thêm `r` và `x` cho all | ✅ Có – đủ read + execute |
| `chmod a+x file`   | Chỉ thêm `x`, giữ nguyên bit khác | ⚠️ Rủi ro – nếu file gốc `000` sẽ ra `--x--x--x`, thiếu `r` → bash không đọc được |
| `chmod u+x file`   | Chỉ thêm `x` cho owner | ❌ Không – thiếu group & others |

---

## 6. Lỗi thường gặp

1. **Chạy lệnh ngay trên jump-host** thay vì SSH vào stapp02 → bài chấm fail vì file trên máy đích không đổi. Luôn nhìn prompt `steve@stapp02` trước khi chạy.
2. **Dùng `chmod a+x` khi file gốc quyền `000`** → kết quả ra `---x--x--x`, **thiếu bit đọc `r`** nên bash không đọc/chạy được script. (Lỗi thực tế đã gặp trong lab này.) → Dùng `chmod 755` để set tuyệt đối cả `r` và `x`.
3. **Chỉ dùng `chmod u+x`** → chỉ owner chạy được, không thỏa "all users". Phải áp cho all (`755` hoặc `a+rx`).
4. **Quên `sudo`** → nếu file thuộc `root` sẽ bị "Operation not permitted". Thêm `sudo` cho an toàn.
5. **Gõ sai đường dẫn** (ví dụ quên `/tmp/`) → "No such file or directory". Kiểm tra bằng `ls -l /tmp/xfusioncorp.sh`.

---

## 7. Tóm tắt lệnh (Quick reference)

```bash
# 1. SSH vào App Server 2
ssh steve@stapp02      # password: Am3ric@

# 2. Cấp quyền đọc + thực thi cho tất cả user
sudo chmod 755 /tmp/xfusioncorp.sh

# 3. Kiểm tra
ls -l /tmp/xfusioncorp.sh   # phải thấy -rwxr-xr-x
```
