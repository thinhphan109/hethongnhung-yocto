# Chủ đề 2 · Xây dựng Linux nhúng bằng Yocto

Nhóm xây dựng image Linux tham chiếu và bản tuỳ chỉnh MyOS bằng Yocto Project 5.0 Scarthgap. Hai đích kiểm thử là QEMU ARM64 (`qemuarm64`) và Raspberry Pi 4 (`raspberrypi4-64`).

## Tài liệu

| Tài liệu | Nội dung |
| --- | --- |
| [Hướng dẫn thực hành](HUONG-DAN-THUC-HANH.md) | Quy trình cài môi trường, build, boot, tích hợp MyOS, thu bằng chứng và tra lỗi. Đọc mục cấu hình một đĩa bên dưới nếu dùng VM trên D:. |
| [Giải thích lệnh build QEMU](GIAI-THICH-LENH-BUILD-QEMU.md) | Ý nghĩa của `MACHINE`, `DISTRO`, cache, lệnh BitBake và cách đưa vào báo cáo. |
| [Giải thích cấu trúc thư mục](GIAI-THICH-THU-MUC-YOCTO.md) | Vai trò của `poky`, `build-qemu`, `downloads`, `sstate-cache` và `evidence`. |
| [Ôn bảo vệ quy trình Yocto](ON-BAO-VE-QUY-TRINH-YOCTO.md) | Câu hỏi từ cấu hình, cross-compilation và các tác vụ BitBake đến chuỗi boot và BSP. |

## Cấu hình VM đang dùng

VMware chạy Ubuntu 22.04.5 LTS x86-64 trên một đĩa ảo đặt ở D:. Trong Ubuntu, thư mục làm việc là `~/yocto-lab` trên ext4. Poky được cố định tại tag `yocto-5.0.21`, commit `4fb7016a07507683359ebd620f1951fd5ea44bfa`.

Hướng dẫn thực hành minh hoạ thêm phương án chia ba đĩa. Khi dùng một đĩa ảo như VM của nhóm, dùng cấu hình sau trong `conf/local.conf` của mỗi đích. Nếu tệp đã có các biến này, sửa giá trị đang dùng tại đó.

```bitbake
MACHINE = "qemuarm64"
DL_DIR = "${TOPDIR}/../downloads"
SSTATE_DIR = "${TOPDIR}/../sstate-cache"
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j4"
```

Với `build-rpi`, đổi `MACHINE` thành `raspberrypi4-64` và thêm layer `meta-raspberrypi` như hướng dẫn. Dùng `TMPDIR` mặc định của từng build. Thay các bước kiểm tra điểm mount trong hướng dẫn bằng `df -hT "$HOME"`. File image Pi sẽ nằm trong `~/yocto-lab/build-rpi/tmp/deploy/images/raspberrypi4-64/`.

## Mốc thực hành

1. Build `core-image-minimal` cho QEMU và xác nhận `uname -m` là `aarch64` sau khi boot.
2. Build `core-image-base` cho Pi, ghi thẻ microSD và xác nhận đúng model bo mạch.
3. Tích hợp `meta-my-os` và `myos-info.service`, sau đó build MyOS cho hai đích.
4. Thu log build, ảnh console và kết quả kiểm thử để đối chiếu trong báo cáo.

Người phụ trách chính tích hợp các đóng góp vào nhánh `main`. Thành viên có thể gửi tệp nguồn, log hoặc ảnh để người phụ trách cập nhật. Repository này tập trung vào tài liệu thực hành. Khi đã kiểm chứng build, nhóm có thể bổ sung cấu hình và layer thực tế từ VM.
