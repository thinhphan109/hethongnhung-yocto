**Có. Đây là phần “Cấu hình và build image tham chiếu cho QEMU” trong báo cáo.** Bạn nên trình bày đích build, các cấu hình chính, lệnh build và kết quả thực tế. Phần giải thích cú pháp Bash có thể để ở phụ lục hướng dẫn chạy lại.

## 1. Khởi tạo môi trường và ghi cấu hình

Lệnh `source poky/oe-init-build-env build-qemu` nạp môi trường Yocto vào **terminal hiện tại**, tạo `build-qemu/conf/` ở lần chạy đầu rồi chuyển terminal vào `build-qemu`. Từ đó terminal nhận được lệnh `bitbake`. Khi mở terminal mới, chạy lại `source` từ `~/yocto-lab`.

Trong khối cấu hình, `cat >> conf/local.conf <<'EOF'` **nối phần văn bản mới vào cuối** `local.conf`. Dấu nháy quanh `'EOF'` giữ nguyên `${TOPDIR}` để **BitBake** giải nghĩa khi đọc cấu hình. Dòng `EOF` cuối đánh dấu hết nội dung cần ghi.

| Cấu hình | Ý nghĩa thực tế |
|---|---|
| `MACHINE = "qemuarm64"` | Chọn máy ARM64 ảo của Yocto. Ubuntu trong VMware vẫn là máy build `x86_64`. BitBake dùng cross-toolchain để tạo phần mềm cho đích AArch64. |
| `DL_DIR = "${TOPDIR}/../downloads"` | Chứa mã nguồn đã tải. `${TOPDIR}` ở đây là `build-qemu`, nên thư mục thực tế là `~/yocto-lab/downloads`. Khi làm Pi, có thể dùng lại các nguồn phù hợp từ thư mục này. |
| `SSTATE_DIR = "${TOPDIR}/../sstate-cache"` | Chứa **kết quả của các tác vụ build** có thể tái sử dụng. Khi cấu hình và chữ ký tác vụ phù hợp, BitBake có thể lấy kết quả từ cache để giảm thời gian build lại. |
| `BB_NUMBER_THREADS = "4"` | Giới hạn số tác vụ BitBake có thể chạy song song. |
| `PARALLEL_MAKE = "-j4"` | Với recipe dùng `make`, cho phép mỗi lần gọi `make` chạy tối đa bốn job song song. Hai thông số này kiểm soát song song ở hai cấp, nên tổng số tiến trình thực tế có thể lớn hơn bốn. |

**Câu giảng viên dễ hỏi:** `DL_DIR` và `SSTATE_DIR` khác nhau thế nào? `DL_DIR` giữ **nguồn đã tải**, còn `SSTATE_DIR` giữ **kết quả tác vụ đã xử lý**. Một bên tiết kiệm tải lại, bên kia tiết kiệm làm lại.

## 2. Kiểm tra cấu hình rồi build

```bash
bitbake -e core-image-minimal | grep -E '^(MACHINE|DISTRO)='
```

`bitbake -e` in **giá trị cấu hình thực tế sau khi BitBake đã đọc metadata** cho `core-image-minimal`. Dấu `|` chuyển đầu ra sang `grep`, còn mẫu lọc chỉ lấy hai dòng bắt đầu bằng `MACHINE=` hoặc `DISTRO=`. Đây là bước xác nhận cấu hình trước khi build:

```text
MACHINE="qemuarm64"
DISTRO="poky"
```

Ba khái niệm ở đây trả lời ba câu hỏi khác nhau: **`MACHINE` là build cho máy nào, `DISTRO` là theo chính sách bản phân phối nào, và `core-image-minimal` là build image nào.** Ở lượt này, bạn dùng distro tham chiếu Poky để xác nhận quy trình build và boot trước khi tạo MyOS.

```bash
set -o pipefail
{ time bitbake core-image-minimal; } 2>&1 | tee ../evidence/qemu-baseline-build.log
```

- `bitbake core-image-minimal` chạy các tác vụ cần thiết để tạo image tối thiểu cho đích đã cấu hình.
- `time` đo thời gian chạy. Khối `{ ...; }` gom lệnh build và kết quả đo thời gian để cùng đưa vào log.
- `2>&1 | tee ...` gom đầu ra thường và thông báo lỗi, rồi vừa hiển thị trên terminal vừa lưu vào tệp log.
- `set -o pipefail` giúp cả chuỗi lệnh báo thất bại nếu BitBake lỗi, kể cả khi `tee` vẫn lưu log thành công.

## 3. Đưa gì vào báo cáo?

Bạn có thể dùng đoạn này trong phần triển khai:

> Nhóm khởi tạo môi trường build `build-qemu` từ Poky, chọn `MACHINE = "qemuarm64"` và xác nhận cấu hình hiệu lực bằng `bitbake -e`. Image tham chiếu `core-image-minimal` được xây dựng bằng `bitbake core-image-minimal`. Nhóm lưu log build để ghi nhận kết quả tác vụ và thời gian thực hiện.

Sau đoạn đó, thêm **hai bằng chứng riêng**:

1. **Build thành công:** trích phần cuối log có kết quả tác vụ và thời gian `real`. Ghi rõ đây là lượt build đầu hay lượt có cache.
2. **Boot thành công:** sau khi chạy `runqemu`, chụp console và kết quả `uname -m` là `aarch64`. Log BitBake chứng minh **đã tạo image**. Console QEMU chứng minh **image đã khởi động**.
