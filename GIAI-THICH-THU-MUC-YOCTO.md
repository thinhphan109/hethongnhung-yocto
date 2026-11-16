Đây là thư mục làm bài Yocto trong Ubuntu, tại **`/home/ubuntu/yocto-lab`**. Năm thư mục trong ảnh có vai trò như sau:

| Thư mục | Chứa gì? |
|---|---|
| **`poky/`** | Bộ mã nguồn Poky đã tải từ Git theo phiên bản Scarthgap. Bên trong có BitBake, các layer tham chiếu và script `oe-init-build-env` để khởi tạo môi trường build. |
| **`build-qemu/`** | Thư mục làm việc cho đích QEMU. `conf/local.conf` chọn `qemuarm64` và cấu hình build. `conf/bblayers.conf` liệt kê các layer được dùng. Sau khi build thành công, kernel và rootfs nằm trong `tmp/deploy/images/qemuarm64/` bên trong thư mục này. |
| **`downloads/`** | Mã nguồn mà BitBake tải về để build các recipe. Lượt build sau có thể dùng lại các tệp đã tải. |
| **`sstate-cache/`** | Kết quả của các tác vụ build mà BitBake có thể dùng lại khi đầu vào và cấu hình tương thích. `downloads` tiết kiệm **tải lại**, còn `sstate-cache` tiết kiệm **chạy lại tác vụ**. |
| **`evidence/`** | Thư mục bạn tạo để giữ bằng chứng cho báo cáo. Lệnh `tee` ở bước trước ghi log vào `qemu-baseline-build.log` trong thư mục này. |

Khi làm Raspberry Pi, nhóm sẽ có thêm **`build-rpi/`** với cấu hình và image riêng. Hai thư mục build có thể trỏ tới cùng `downloads/` và `sstate-cache/`.

**Đưa ảnh này vào mục “Tổ chức thư mục build” trong báo cáo được.** Chú thích nên chỉ rõ ba vị trí: cấu hình ở `build-qemu/conf/`, image ở `build-qemu/tmp/deploy/images/qemuarm64/` và log ở `evidence/`. Kết quả build sẽ được chứng minh bằng phần cuối log ở bước tiếp theo.
