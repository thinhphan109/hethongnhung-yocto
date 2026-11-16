Học theo **đường đi của một image**, từ lúc đọc `local.conf` đến lúc thấy dấu nhắc đăng nhập. Mỗi bước bạn nên trả lời được ba ý: **đầu vào là gì, công cụ làm gì, và lấy bằng chứng ở đâu**. Hiện tại, ưu tiên học hai phần đầu vì đó là chỗ bạn đang thực hành.

## 1. Từ những lệnh bạn vừa chạy

| Giảng viên có thể hỏi | Câu trả lời gắn với bài của bạn |
|---|---|
| **VMware và QEMU đều là máy ảo. Vì sao dùng cả hai?** | VMware chạy **Ubuntu x86-64 làm máy build**. Bên trong Ubuntu, QEMU giả lập **máy ARM64 để chạy image vừa build**. Raspberry Pi 4 là đích phần cứng thật. |
| **Lệnh `source` đã làm gì?** | `source poky/oe-init-build-env build-qemu` nạp môi trường Yocto vào terminal hiện tại, tạo cấu hình ban đầu và chuyển vào `build-qemu`. Sau đó terminal mới gọi được `bitbake`. Mở terminal mới thì nạp lại môi trường. |
| **Yocto, Poky và BitBake khác nhau thế nào?** | Yocto Project là dự án cung cấp hệ sinh thái để xây dựng Linux nhúng. Poky là bản phân phối tham chiếu và cây mã nguồn bạn clone. BitBake là công cụ đọc mô tả build, tính phụ thuộc và chạy các tác vụ. |
| **`local.conf` và `bblayers.conf` khác nhau ở đâu?** | `local.conf` chọn đích và thông số build, ví dụ `MACHINE` và vị trí cache. `bblayers.conf` liệt kê các layer mà BitBake được phép đọc. Khi làm Pi, bạn thêm BSP `meta-raspberrypi` vào cấu hình layer của đích Pi. |
| **`MACHINE`, `DISTRO` và `core-image-minimal` khác nhau thế nào?** | `MACHINE` trả lời **build cho máy nào**. `DISTRO` trả lời **áp dụng chính sách bản phân phối nào**. `core-image-minimal` trả lời **tạo image nào**, với tập gói tối thiểu để hệ thống khởi động. |
| **Máy x86-64 tạo được chương trình ARM64 bằng cách nào?** | Trình biên dịch chạy trên Ubuntu x86-64, còn mã máy nó tạo ra dành cho AArch64. Đó là **cross-compilation**. Về sau, dùng `readelf -h` để xem kiến trúc của binary do nhóm tự đóng gói. |

Để tự kiểm, mở `build-qemu/conf/local.conf` và `build-qemu/conf/bblayers.conf`, rồi thử chỉ bằng tay dòng nào chọn máy, dòng nào chọn layer. Sau đó nhìn kết quả `bitbake -e` để giải thích vì sao giá trị **BitBake thực sự dùng** quan trọng hơn một dòng bạn thấy trong tệp. [Các khái niệm của Yocto Scarthgap](https://docs.yoctoproject.org/scarthgap/overview-manual/concepts.html)

## 2. BitBake đang làm gì khi terminal chạy rất lâu?

BitBake đọc các **recipe** và lập quan hệ phụ thuộc. Ví dụ, muốn dựng rootfs thì trước đó phải có các gói để đưa vào rootfs. Các recipe có thể xử lý song song khi những tác vụ cần thiết đã sẵn sàng.

| Tên tác vụ thấy trong log | Hiểu đơn giản |
|---|---|
| `do_fetch` | Tải hoặc tìm lại mã nguồn. Đây là lúc `downloads/` có tác dụng. |
| `do_unpack`, `do_patch` | Giải nén nguồn và áp dụng patch nếu recipe khai báo. |
| `do_configure`, `do_compile` | Chuẩn bị cấu hình rồi biên dịch phần mềm bằng công cụ phù hợp với đích. |
| `do_install`, `do_package` | Chép kết quả vào thư mục staging của recipe, sau đó xác định tệp nào thuộc gói nào. |
| `do_rootfs`, `do_image` | Dựng hệ thống tệp gốc từ các gói đã chọn, rồi tạo tệp image theo định dạng của đích. |

**Câu dễ bị hỏi sâu:** Nếu lần build sau có nhiều dòng `didn't need to be rerun`, vì sao vẫn hợp lệ? BitBake dùng chữ ký tác vụ để kiểm tra đầu vào và cấu hình. Kết quả còn phù hợp có thể được dùng lại. Khi đổi mã C của nhóm, phần build ứng dụng và các bước dựng image liên quan được cập nhật, trong khi nhiều phần khác vẫn tái sử dụng được. [Tài liệu tác vụ BitBake](https://docs.yoctoproject.org/scarthgap/ref-manual/tasks.html) · [Cơ chế shared state của Yocto](https://docs.yoctoproject.org/scarthgap/overview-manual/concepts.html#shared-state-cache)

## 3. Từ image đến màn hình đăng nhập

```text
QEMU nạp kernel và gắn đĩa ảo chứa rootfs
      → kernel mount rootfs
      → kernel khởi chạy tiến trình init (PID 1)
      → hệ thống mở console đăng nhập
```

- **Kernel là gì?** Phần quản lý CPU, bộ nhớ, tiến trình và thiết bị. Dùng `uname -r` để xem phiên bản kernel đang chạy.
- **Rootfs là gì?** Hệ thống tệp gốc `/`, chứa chương trình, thư viện và cấu hình như `/etc/os-release`. Kernel cần truy cập được rootfs để chạy tiến trình đầu tiên của user space.
- **PID 1 là gì?** Tiến trình init, phụ trách khởi tạo các phần còn lại của hệ thống. Dùng `cat /proc/1/comm` để xem. Poky ở phiên bản bạn đang dùng mặc định chọn SysVinit. Khi làm MyOS, nhóm mới chủ động chọn `systemd`. [Cấu hình Poky tại commit đang dùng](https://github.com/yoctoproject/poky/blob/4fb7016a07507683359ebd620f1951fd5ea44bfa/meta-poky/conf/distro/poky.conf)
- **`uname -m` ra `aarch64` chứng minh điều gì?** Console đang trả lời từ hệ thống ARM64 đã khởi động. Dùng thêm `cat /etc/os-release` để xác nhận hệ thống đó là Poky ở lần đầu hay MyOS ở lần sau.

## 4. Hai câu lớn khi chuyển sang MyOS và Raspberry Pi

**“Cùng là ARM64, vì sao phải build hai image?”** ARM64 mô tả kiến trúc CPU. `qemuarm64` dùng board ảo `virt`, còn Pi 4 có phần cứng và chuỗi khởi động riêng. Pi cần BSP, firmware và **Device Tree** phù hợp. Device Tree mô tả phần cứng để kernel nhận diện thiết bị và gắn driver. Image WIC của Pi chứa cả phân vùng boot và phân vùng rootfs. Phần dùng chung giữa hai đích là distro, image recipe, ứng dụng và dịch vụ của nhóm. [Cấu hình QEMU ARM64](https://github.com/yoctoproject/poky/blob/scarthgap/meta/conf/machine/qemuarm64.conf) · [Bố cục WIC của Raspberry Pi](https://github.com/agherzan/meta-raspberrypi/blob/scarthgap/wic/sdimage-raspberrypi.wks)

**“Phần nào nhóm thực sự làm?”** Mở `meta-my-os` và chỉ ra ba thứ. `myos.conf` đặt chính sách và tên hệ thống. `myos-image.bb` chọn các gói đưa vào image. Recipe `myos-info` đóng gói chương trình C và unit `systemd`. Khi demo, **manifest** cho thấy gói đã có trong image, còn **journal của lượt boot hiện tại** cho thấy dịch vụ đã chạy.

### Bài tập ngay ở chặng hiện tại

Mở **terminal Ubuntu thứ hai** và xem log mà lượt build đang ghi:

```bash
tail -n 20 ~/yocto-lab/evidence/qemu-baseline-build.log
```

Chọn một dòng có tên tác vụ như `do_fetch`, `do_compile` hoặc `do_rootfs` rồi tự trả lời:

1. **BitBake đang xử lý recipe hoặc image nào?**
2. **Tác vụ đó nhận gì và tạo ra gì?**
3. **Dòng log này chứng minh được tới bước nào?**

Ví dụ, thấy `do_compile` hoàn thành thì bạn có thể giải thích **một thành phần đã biên dịch xong**. Để kết luận **cả image đã build xong**, cần xem kết quả cuối của BitBake. Để kết luận **hệ điều hành đã chạy**, cần nhìn console sau khi boot. Đây cũng là cách tốt nhất để trình bày bằng chứng trong báo cáo.
