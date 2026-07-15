# Cài đặt cronie và tạo cron job cho root trên các App Server

## 1. Đề bài gốc (tiếng Anh)

> The Nautilus system admins team has prepared scripts to automate several day-to-day tasks. They want them to be deployed on all app servers in Stratos DC on a set schedule. Before that they need to test similar functionality with a sample cron job. Therefore, perform the steps below:
>
> a. Install cronie package on all Nautilus app servers and start crond service.
>
> b. Add a cron `*/5 * * * * echo hello > /tmp/cron_text` for root user.

## 2. Dịch đề bài (tiếng Việt)

Đội system admin của Nautilus đã chuẩn bị sẵn các script để tự động hoá một số công việc hàng ngày. Họ muốn triển khai các script này trên tất cả các app server thuộc Stratos DC theo một lịch cố định. Trước khi làm điều đó, họ cần thử nghiệm chức năng tương tự bằng một cron job mẫu. Vì vậy, hãy thực hiện các bước sau:

a. Cài đặt package `cronie` trên tất cả các App Server của Nautilus và khởi động service `crond`.

b. Thêm một cron job `*/5 * * * * echo hello > /tmp/cron_text` cho user `root`.

## 3. Giải thích đề bài

- **cronie** là package cung cấp daemon `crond` (bản kế thừa của Vixie Cron) trên các distro RHEL/CentOS/Rocky Linux. Đây là công cụ tiêu chuẩn để lập lịch chạy các job định kỳ trên Linux.
- **crond** là service (daemon) chạy nền, đọc các crontab đã khai báo và thực thi lệnh đúng theo lịch đã đặt.
- **crontab cho root** được lưu tại `/var/spool/cron/root` (quản lý qua lệnh `crontab -e -u root` hoặc `crontab -e` khi đang đăng nhập bằng root).
- Biểu thức `*/5 * * * *` nghĩa là "chạy mỗi 5 phút" (5 trường theo thứ tự: phút, giờ, ngày trong tháng, tháng, thứ trong tuần).
- Đây là bước thử nghiệm (proof of concept) trước khi triển khai các script tự động hoá thật sự trên toàn bộ hạ tầng — best practice là luôn test cron job đơn giản trước để đảm bảo service hoạt động đúng và cú pháp crontab chính xác, tránh lỗi khi triển khai job phức tạp trên diện rộng.
- Đề bài yêu cầu làm trên **tất cả App Server** (App Server 1, 2, 3), không phải chỉ một server.

## 4. Lời giải

Task này cần được thực hiện lặp lại giống nhau trên cả 3 App Server: `stapp01`, `stapp02`, `stapp03`. Dưới đây là các bước cho từng server (thực hiện từ jump host).

### Bước 1: SSH vào App Server 1

```bash
ssh tony@stapp01
# password: Ir0nM@n
```

### Bước 2: Cài đặt cronie và khởi động crond trên App Server 1

```bash
sudo yum install cronie -y
```
- `sudo`: cần quyền root để cài package.
- `yum install cronie -y`: cài package `cronie` (chứa crond), `-y` để tự động xác nhận "yes".

```bash
sudo systemctl start crond
sudo systemctl enable crond
```
- `systemctl start crond`: khởi động service crond ngay lập tức.
- `systemctl enable crond`: đảm bảo crond tự khởi động lại mỗi khi server reboot (không bắt buộc theo đề nhưng là best practice).

### Bước 3: Thêm cron job cho root trên App Server 1

```bash
sudo crontab -u root -e
```
- `-u root`: chỉ định crontab của user root.
- `-e`: mở editor (mặc định là vi) để chỉnh sửa crontab.

Trong editor, thêm đúng dòng sau, lưu và thoát (`Esc` → `:wq`):

```
*/5 * * * * echo hello > /tmp/cron_text
```

### Bước 4: Kiểm tra (verify) trên App Server 1

```bash
sudo systemctl status crond
sudo crontab -u root -l
```
- `systemctl status crond`: xác nhận service đang `active (running)`.
- `crontab -u root -l`: liệt kê nội dung crontab của root, phải thấy đúng dòng vừa thêm.

(Tuỳ chọn) Đợi tối đa 5 phút rồi kiểm tra file kết quả:

```bash
cat /tmp/cron_text
```
Nếu thấy chữ `hello` thì cron job đã chạy thành công.

### Bước 5: Thoát khỏi App Server 1, lặp lại cho App Server 2 và App Server 3

```bash
exit
ssh steve@stapp02
# password: Am3ric@
```
Lặp lại nguyên các Bước 2–4 ở trên (cài `cronie`, start/enable `crond`, thêm crontab root, verify).

```bash
exit
ssh banner@stapp03
# password: BigGr33n
```
Lặp lại nguyên các Bước 2–4 ở trên cho App Server 3.

## 5. Giải thích chi tiết

**Cấu trúc biểu thức cron `*/5 * * * *`:**

| Trường       | Giá trị | Ý nghĩa                              |
|--------------|---------|----------------------------------------|
| Phút         | `*/5`   | Mỗi 5 phút (0, 5, 10, 15, ...)          |
| Giờ          | `*`     | Mọi giờ                                 |
| Ngày (tháng) | `*`     | Mọi ngày trong tháng                    |
| Tháng        | `*`     | Mọi tháng                               |
| Thứ (tuần)   | `*`     | Mọi ngày trong tuần                     |

**Lệnh thực thi:** `echo hello > /tmp/cron_text`
- `echo hello`: in ra chuỗi `hello`.
- `> /tmp/cron_text`: ghi đè (overwrite) nội dung đó vào file `/tmp/cron_text` mỗi lần chạy (không dùng `>>` nên file không bị nối dài, luôn chỉ có nội dung của lần chạy gần nhất).

**So sánh các cách thêm crontab:**

| Cách                          | Ghi chú                                                             |
|-------------------------------|----------------------------------------------------------------------|
| `crontab -e` (khi login root)  | Sửa trực tiếp crontab của user hiện tại                              |
| `sudo crontab -u root -e`      | Sửa crontab của root khi đang login bằng user khác (tony/steve/banner)|
| Sửa trực tiếp `/etc/crontab`   | Không khuyến khích cho cron job của user thông thường; dùng cho system-wide job có thêm trường username |

## 6. Lỗi thường gặp

- **Chạy nhầm lệnh trên jump host thay vì App Server**: luôn kiểm tra prompt (`user@stapp01`, `user@stapp02`, `user@stapp03`) trước khi thao tác. Đây là lỗi phổ biến nhất.
- **Chỉ làm trên 1 App Server rồi dừng**: đề yêu cầu "all Nautilus app servers" — phải lặp lại đủ trên cả `stapp01`, `stapp02`, `stapp03`.
- **Thêm cron job vào crontab của user hiện tại (tony/steve/banner) thay vì root**: phải dùng `sudo crontab -u root -e`, không phải `crontab -e` (nếu không sudo thì sẽ sửa nhầm crontab của user đang login).
- **Quên start/enable crond**: nếu chỉ cài `cronie` mà không start service, cron job sẽ không bao giờ chạy dù đã khai báo đúng crontab.
- **Sai cú pháp cron (thiếu trường, thừa dấu cách)**: `*/5 * * * * echo hello > /tmp/cron_text` phải đúng y hệt 5 trường thời gian rồi mới đến lệnh, không thêm/bớt khoảng trắng gây lỗi parse.
- **Dùng `>>` thay vì `>`**: đề yêu cầu chính xác `>` (ghi đè), không phải append.

## 7. Tóm tắt lệnh (Quick reference)

Thực hiện lần lượt trên **cả 3 server**: `stapp01` (tony/Ir0nM@n), `stapp02` (steve/Am3ric@), `stapp03` (banner/BigGr33n).

```bash
# 1. SSH vào server (đổi user/host tương ứng)
ssh tony@stapp01

# 2. Cài cronie và khởi động crond
sudo yum install cronie -y
sudo systemctl start crond
sudo systemctl enable crond

# 3. Thêm cron job cho root
sudo crontab -u root -e
# thêm dòng:
# */5 * * * * echo hello > /tmp/cron_text

# 4. Kiểm tra
sudo systemctl status crond
sudo crontab -u root -l
```
