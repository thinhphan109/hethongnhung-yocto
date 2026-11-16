# Xây dựng MyOS bằng Yocto Project trên QEMU và Raspberry Pi 4

Quy trình này dùng Yocto 5.0.21 Scarthgap, VMware Workstation và Ubuntu 22.04.5 LTS x86-64. Hai đích ARM64 dùng chung layer `meta-my-os`, distro `myos` và recipe `myos-image`. Dự án khởi động bằng image tham chiếu trên từng đích, sau đó thêm một chương trình C và dịch vụ `systemd` để kiểm chứng phần nhóm tự làm.

**Vị trí chạy lệnh.** PowerShell chạy trên Windows. Các khối ghi **Ubuntu** chạy trong Terminal của máy ảo, bằng tài khoản thường. Các khối ghi **QEMU** hoặc **Pi** chạy sau khi đăng nhập console của hệ thống vừa build. Chạy từng khối theo thứ tự và đối chiếu kết quả ở cuối mỗi chặng.

## 0. Chuẩn bị VM và ba đĩa ảo

Máy đã có VMware Workstation 17.5.2. Lần kiểm tra WSL `Ubuntu-22.04` báo thiếu tệp `ext4.vhdx`. Phương án dưới đây dùng VMware và dung lượng trống trên các ổ hiện tại. Mức trống đã quan sát là C: khoảng 93 GiB, D: 62 GiB, G: 88 GiB. Kiểm tra lại trước khi tạo VM.

**PowerShell**

```powershell
Get-CimInstance Win32_LogicalDisk -Filter 'DriveType = 3' |
    Select-Object DeviceID, @{n='FreeGiB';e={[math]::Round($_.FreeSpace/1GB,1)}}
```

1. Tải `ubuntu-22.04.5-desktop-amd64.iso`. Tạo VM x86-64, cấp 4 vCPU, 12 GiB RAM và chọn mạng NAT. Lưu VM cùng đĩa hệ thống tại `G:\YoctoVM`, tạo đĩa dung lượng tối đa khoảng 72 GiB. Chọn đĩa tự tăng dung lượng, bỏ chọn **Allocate all disk space now**. Cài Ubuntu lên đĩa này. Có thể chọn **Minimal installation** để giảm dung lượng cài đặt. [ISO chính thức của Ubuntu 22.04.5](https://releases.ubuntu.com/22.04.5/)
2. Tắt VM sau khi cài Ubuntu. Trong **VM Settings → Add → Hard Disk → Create a new virtual disk**, tạo hai đĩa tự tăng dung lượng. Đặt đĩa cache khoảng **45 GiB** tại `D:\YoctoVM\cache.vmdk`, đĩa build Pi khoảng **60 GiB** tại `C:\YoctoVM\pi-tmp.vmdk`. Tạo thư mục tương ứng trên Windows nếu chưa có. Chọn tạo đĩa ảo mới cho cả hai.
3. Khởi động VM. Chạy `lsblk` như bên dưới để nhận diện hai đĩa mới qua dung lượng và cột `MOUNTPOINTS`. Trong ứng dụng **Disks** của Ubuntu, chọn từng đĩa mới, tạo bảng phân vùng GPT nếu đĩa chưa có, rồi tạo một phân vùng ext4 chiếm toàn bộ đĩa. Đặt nhãn filesystem lần lượt là `yocto-cache` và `yocto-pi-tmp`. Giữ đĩa đang chứa `/` và `/boot/efi` ngoài thao tác định dạng. Nếu thiếu ứng dụng Disks, cài bằng `sudo apt install gnome-disk-utility`.

**Ubuntu**

```bash
lsblk -e7 -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS,MODEL
sudo blkid -L yocto-cache
sudo blkid -L yocto-pi-tmp
```

Nếu Disks đã tự mount một trong hai phân vùng dưới `/media`, bấm nút **Unmount** cho phân vùng đó trong Disks trước khi chạy các lệnh bên dưới. Dùng tên nhãn đã đặt để mount đúng đĩa, kể cả khi thiết bị đổi tên `/dev/sd*`.

```bash
sudo install -d /mnt/yocto-cache /mnt/yocto-pi-tmp
sudo mount -L yocto-cache /mnt/yocto-cache
sudo mount -L yocto-pi-tmp /mnt/yocto-pi-tmp
mountpoint /mnt/yocto-cache
mountpoint /mnt/yocto-pi-tmp
```

Khi cả hai dòng `mountpoint` báo `is a mountpoint`, tạo các thư mục cho tài khoản build.

```bash
sudo install -d -o "$(id -u)" -g "$(id -g)" \
  /mnt/yocto-cache/downloads /mnt/yocto-cache/sstate-cache \
  /mnt/yocto-pi-tmp/myos-tmp
df -hT "$HOME" /mnt/yocto-cache /mnt/yocto-pi-tmp
nproc
free -h
```

**Đạt khi** `df -hT` hiển thị ba filesystem ext4 riêng tương ứng với home, cache và build Pi. Sau mỗi lần khởi động lại Ubuntu, chạy lại hai lệnh `mount -L` nếu `mountpoint` báo chưa mount. Kiểm tra lại dung lượng host và guest sau mỗi lần build. Các mức cấp đĩa là phương án cho image console nhỏ, cần điều chỉnh dựa trên dung lượng đo thực tế. VMware có thể tích luỹ thêm dung lượng nếu tạo snapshot.

**Lưu cho báo cáo** một ảnh `df -hT` và cấu hình CPU/RAM của VM. Giảng viên có thể hỏi vì sao máy build là x86-64 còn image là ARM64. BitBake chạy trên host x86-64, dùng cross-toolchain sinh mã máy AArch64. QEMU chạy mã AArch64 bằng giả lập CPU trên host này. VMware chạy Ubuntu x86-64 để làm môi trường build.

## 1. Cài công cụ và chốt phiên bản mã nguồn

**Ubuntu**

```bash
sudo apt update
sudo apt install -y build-essential chrpath cpio debianutils diffstat file \
  gawk gcc git iputils-ping libacl1 liblz4-tool locales python3 python3-git \
  python3-jinja2 python3-pexpect python3-pip python3-subunit socat texinfo \
  unzip wget xz-utils zstd bmap-tools
sudo locale-gen en_US.UTF-8
locale -a | grep -i en_US
git --version
bmaptool --version
```

Nếu `apt` báo thiếu `bmap-tools`, bật kho Universe bằng `sudo add-apt-repository universe`, rồi chạy lại `sudo apt update` và lệnh cài. Yocto cần `en_US.UTF-8`. Build bằng tài khoản thường, chỉ dùng `sudo` cho thao tác quản trị VM và ghi thẻ. [Gói cần cài cho build host](https://docs.yoctoproject.org/scarthgap/ref-manual/system-requirements.html#required-packages-for-the-build-host)

```bash
mkdir -p ~/yocto-lab/project-conf ~/yocto-lab/evidence
cd ~/yocto-lab
git clone --depth 1 --branch yocto-5.0.21 https://git.yoctoproject.org/poky

git init meta-raspberrypi
git -C meta-raspberrypi remote add origin https://github.com/agherzan/meta-raspberrypi.git
git -C meta-raspberrypi fetch --depth 1 origin 018d44467c4f09eb63f8ea5e86e85d755250cf89
git -C meta-raspberrypi checkout --detach FETCH_HEAD

{
    printf 'poky '
    git -C poky rev-parse HEAD
    printf 'meta-raspberrypi '
    git -C meta-raspberrypi rev-parse HEAD
} | tee project-conf/sources.lock
```

**Đạt khi** commit Poky là `4fb7016a07507683359ebd620f1951fd5ea44bfa` và commit Raspberry Pi BSP là `018d44467c4f09eb63f8ea5e86e85d755250cf89`. Tệp `sources.lock` ghi phiên bản để đối chiếu. Lệnh clone và fetch ở trên là phần thực hiện việc cố định phiên bản.

**Giảng viên có thể hỏi branch và commit khác nhau thế nào.** Branch `scarthgap` có thể tiếp tục nhận cập nhật. Tag Poky và commit BSP xác định hai cây mã nguồn đã chọn. Build lại thành công từ Git nghĩa là lặp lại được cấu hình và quy trình. Kết luận hai image giống từng byte cần một phép thử riêng với cùng đầu vào và checksum.

## 2. Build image tham chiếu cho QEMU

**Ubuntu**

```bash
cd ~/yocto-lab
cat > project-conf/common.inc <<'EOF'
DL_DIR = "/mnt/yocto-cache/downloads"
SSTATE_DIR = "/mnt/yocto-cache/sstate-cache"
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j4"

# Console-only teaching image on the local lab machine.
EXTRA_IMAGE_FEATURES = "debug-tweaks"
EOF

cat > project-conf/qemu.inc <<'EOF'
MACHINE = "qemuarm64"
require ${TOPDIR}/../project-conf/common.inc
EOF

cat > project-conf/rpi.inc <<'EOF'
MACHINE = "raspberrypi4-64"
TMPDIR = "/mnt/yocto-pi-tmp/myos-tmp"
require ${TOPDIR}/../project-conf/common.inc
EOF

source poky/oe-init-build-env build-qemu
printf '\nrequire ${TOPDIR}/../project-conf/qemu.inc\n' >> conf/local.conf
mountpoint /mnt/yocto-cache && df -hT . /mnt/yocto-cache
bitbake-layers show-layers
bitbake -e core-image-minimal | grep -E '^(MACHINE|DISTRO|TMPDIR)='
```

Chỉ thêm dòng `require` một lần vào mỗi `local.conf`. `source` chuyển terminal vào thư mục build. Trước khi chạy BitBake, kiểm tra `MACHINE="qemuarm64"`, `DISTRO="poky"` và cache đã mount. `debug-tweaks` cho phép đăng nhập `root` không mật khẩu tại console của image thực hành. Quy trình thực hành không cài máy chủ SSH.

```bash
umask 022
set -o pipefail
{ time bitbake core-image-minimal; } 2>&1 | tee ../evidence/qemu-baseline-build.log
```

Nếu terminal báo lỗi, xem mục 9 rồi chạy lại lệnh build. Khi build thành công, xác nhận có `*.qemuboot.conf` và `*.ext4` rồi chạy QEMU. `script` ghi lại console và phần khởi động vào tệp log.

```bash
ls -lh tmp/deploy/images/qemuarm64/core-image-minimal*.{qemuboot.conf,ext4}
script -q -c 'runqemu qemuarm64 core-image-minimal nographic slirp' \
  ../evidence/qemu-baseline-console.log
```

**QEMU**, đăng nhập `root` tại `login:`.

```sh
cat /etc/os-release
uname -m
uname -r
cat /proc/cmdline
poweroff
```

**Đạt khi** thấy console, `uname -m` ra `aarch64` và `os-release` hiển thị Poky. Nếu cần thoát phiên QEMU `nographic`, nhấn `Ctrl+A`, thả phím, rồi nhấn `X`. [Cách dùng `runqemu`](https://docs.yoctoproject.org/scarthgap/dev-manual/qemu.html)

**Giảng viên có thể hỏi QEMU nạp gì.** `runqemu` đọc cấu hình `.qemuboot.conf`, nạp kernel của board ảo `virt` và gắn rootfs ext4 như một đĩa ảo. Kernel mount rootfs rồi chạy tiến trình init. `nographic` đưa console ra terminal, `slirp` dùng mạng user-mode. Máy `virt` dùng thiết bị ảo như virtio. Kết quả này kiểm thử Linux và ứng dụng chung, còn Ethernet vật lý và boot từ thẻ thuộc bài kiểm thử Pi.

## 3. Build image tham chiếu cho Raspberry Pi 4

**Ubuntu**, mở terminal mới hoặc quay về `~/yocto-lab`.

```bash
cd ~/yocto-lab
source poky/oe-init-build-env build-rpi
printf '\nrequire ${TOPDIR}/../project-conf/rpi.inc\n' >> conf/local.conf
bitbake-layers add-layer ../meta-raspberrypi
mountpoint /mnt/yocto-cache && mountpoint /mnt/yocto-pi-tmp && \
  df -hT . /mnt/yocto-cache /mnt/yocto-pi-tmp
bitbake-layers show-layers
bitbake -e core-image-base | grep -E '^(MACHINE|DISTRO|TMPDIR|IMAGE_FSTYPES|WKS_FILE)='
```

Chỉ tiếp tục khi `MACHINE="raspberrypi4-64"`, `DISTRO="poky"`, `TMPDIR` nằm dưới `/mnt/yocto-pi-tmp` và `IMAGE_FSTYPES` có `wic.bz2`. Thư mục deploy đi theo `TMPDIR`, vì vậy ảnh Pi nằm trên đĩa ảo C: theo cấu hình đã chọn. BSP Scarthgap có sẵn `.wic.bz2` và `.wic.bmap`. [Cấu hình image của BSP](https://github.com/agherzan/meta-raspberrypi/blob/018d44467c4f09eb63f8ea5e86e85d755250cf89/conf/machine/include/rpi-base.inc)

```bash
set -o pipefail
{ time bitbake core-image-base; } 2>&1 | tee ../evidence/rpi-baseline-build.log
deploy=/mnt/yocto-pi-tmp/myos-tmp/deploy/images/raspberrypi4-64
ls -lh "$deploy"/core-image-base*.wic*
```

**Giảng viên có thể hỏi tại sao còn một lượt build.** `MACHINE` quyết định cấu hình CPU, lựa chọn kernel/BSP và định dạng image. AArch64 là kiến trúc chung. QEMU có board ảo `virt` còn Pi 4 có SoC, Device Tree và chuỗi boot riêng. `DL_DIR` dùng lại mã nguồn đã tải. `SSTATE_DIR` dùng lại kết quả tác vụ khi chữ ký và kiến trúc phù hợp. Mỗi đích giữ `TMPDIR` riêng.

## 4. Ghi image tham chiếu và boot Pi thật

Chuẩn bị Raspberry Pi 4, nguồn, thẻ microSD dành cho bài tập, đầu đọc thẻ USB, màn hình với cáp micro-HDMI và bàn phím USB. Cáp Ethernet và router có DHCP dùng ở chặng 7. Nếu Pi đang cắm SSD hay USB chứa hệ điều hành đang dùng, tắt Pi đúng cách rồi tháo thiết bị đó trong lúc thử thẻ để xác nhận đang boot từ microSD.

1. Chạy `lsblk` trước khi cắm đầu đọc. Cắm đầu đọc USB chứa thẻ microSD, rồi trong VMware chọn **VM → Removable Devices → đầu đọc USB → Connect (Disconnect from Host)**. Kiểm tra lại `lsblk` và xác định đĩa xuất hiện thêm, đúng model và đúng dung lượng thẻ. Chọn đĩa nguyên dạng như `/dev/sdX` theo kết quả thực tế. Nếu hội thoại Windows đề nghị format phân vùng Linux sau khi ghi, bấm **Cancel**.
2. Chạy khối sau trên Ubuntu để xác nhận cặp image và block map. Ở lần ghi MyOS cuối cùng, đổi `image_target=core-image-base` thành `image_target=myos-image`.

**Ubuntu**

```bash
lsblk -e7 -o NAME,PATH,SIZE,TYPE,FSTYPE,MODEL,TRAN,MOUNTPOINTS
deploy=/mnt/yocto-pi-tmp/myos-tmp/deploy/images/raspberrypi4-64
image_target=core-image-base
image="$deploy/${image_target}-raspberrypi4-64.rootfs.wic.bz2"
bmap="${image%.bz2}.bmap"
ls -lh "$image" "$bmap"
```

Nếu thẻ có phân vùng đang mount, dùng nút **Unmount** trong Disks cho từng phân vùng của thẻ. Sau đó chạy đoạn sau. Công cụ sẽ hiển thị đĩa lần nữa và chỉ bắt đầu ghi sau khi nhập đúng `GHI-THE`. **Ghi WIC sẽ xoá dữ liệu trên thẻ được chọn.**

```bash
read -r -p 'Nhập đường dẫn đĩa microSD đã xác nhận: ' card
if [[ -n "$card" && -b "$card" && -f "$image" && -f "$bmap" ]] &&
   [[ "$(lsblk -dnro TYPE "$card")" == disk && "$(lsblk -dnro TRAN "$card")" == usb ]] &&
   [[ -z "$(lsblk -nro MOUNTPOINTS "$card" | tr -d '[:space:]')" ]]; then
    lsblk -o NAME,PATH,SIZE,TYPE,MODEL,TRAN,MOUNTPOINTS "$card"
    read -r -p 'Nếu đúng đầu đọc microSD, nhập GHI-THE: ' flash_confirmation
    if [[ "$flash_confirmation" == 'GHI-THE' ]]; then
        sudo bmaptool copy --bmap "$bmap" "$image" "$card" && sync
    fi
else
    printf '%s\n' 'Dừng. Kiểm tra đầu đọc USB, đĩa, mount và cặp image/bmap.'
fi
```

Script yêu cầu đầu đọc đi qua USB để nhận diện đĩa an toàn hơn. Nếu VMware không chuyển được đầu đọc cho Ubuntu, có thể dùng Raspberry Pi Imager trên Windows. Trong Ubuntu giải nén bằng `bzip2 -dc "$image" > "$HOME/${image_target}-raspberrypi4-64.img"` sau khi kiểm tra còn chỗ trống. Chuyển file `.img` sang Windows qua thư mục chia sẻ của VM, mở Imager, chọn **Use custom**, chọn đúng file và đúng thẻ. Nếu Imager có mục OS customisation, bỏ qua để giữ cấu hình do Yocto build. [Tài liệu Raspberry Pi Imager](https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-the-operating-system) · [Hướng dẫn ghi thẻ của BSP](https://github.com/agherzan/meta-raspberrypi/blob/018d44467c4f09eb63f8ea5e86e85d755250cf89/README.md#quick-start)

3. Tháo đầu đọc sau khi ghi thành công. Lắp microSD vào Pi đã tắt, cắm màn hình và bàn phím, rồi cấp nguồn. Nếu thấy boot nhưng chưa thấy login, thử `Ctrl+Alt+F1` hoặc `Ctrl+Alt+F2` trên bàn phím của Pi. Đăng nhập `root` tại console.

**Pi**

```sh
cat /etc/os-release
uname -m
uname -r
tr -d '\0' < /proc/device-tree/model
printf '\n'
cat /proc/cmdline
awk '$2 == "/" { print $1, $3 }' /proc/mounts
```

**Đạt khi** hiển thị `aarch64`, model Raspberry Pi 4 và rootfs ext4 từ phân vùng thẻ. `cat /proc/cmdline` cho thấy tham số `root=` kernel nhận khi boot. Tắt bằng `poweroff` trước khi rút thẻ.

**Giảng viên có thể hỏi `.wic.bz2`, `.bmap` và Device Tree dùng làm gì.** WIC là ảnh toàn đĩa do Yocto đóng gói theo `sdimage-raspberrypi.wks`, chứa phân vùng boot FAT và rootfs ext4. `.bz2` là lớp nén. `.wic.bmap` chứa bản đồ block và checksum để `bmaptool` ghi, kiểm tra hiệu quả. Với cấu hình boot trực tiếp của BSP, firmware Pi đọc các tệp trên phân vùng boot, nạp kernel và DTB. DTB mô tả phần cứng để kernel nhận các thiết bị và gắn driver. Kernel mount phân vùng root theo `cmdline.txt` rồi chuyển sang init. `cat /proc/device-tree/model` cho biết model từ Device Tree đang dùng. [Bản mô tả phân vùng WIC](https://github.com/agherzan/meta-raspberrypi/blob/018d44467c4f09eb63f8ea5e86e85d755250cf89/wic/sdimage-raspberrypi.wks) · [Recipe sinh `cmdline.txt`](https://github.com/agherzan/meta-raspberrypi/blob/018d44467c4f09eb63f8ea5e86e85d755250cf89/recipes-bsp/bootfiles/rpi-cmdline.bb)

## 5. Tạo distro, image recipe, chương trình C và dịch vụ

**Ubuntu**, bắt đầu ở thư mục build QEMU. Nếu các mount đã được duy trì từ trước, kiểm tra rồi chạy tiếp.

```bash
cd ~/yocto-lab
source poky/oe-init-build-env build-qemu
bitbake-layers create-layer ../meta-my-os
bitbake-layers add-layer ../meta-my-os
rm -f ../meta-my-os/recipes-example/example/example_0.1.bb
mkdir -p ../meta-my-os/conf/distro ../meta-my-os/recipes-core/images \
  ../meta-my-os/recipes-apps/myos-info/files
grep -E '^LAYERSERIES_COMPAT' ../meta-my-os/conf/layer.conf
```

Layer dùng cùng phiên bản metadata với Poky. Dòng `LAYERSERIES_COMPAT` do công cụ sinh cho nhánh hiện tại cần chứa `scarthgap`. Tạo bốn tệp sau và tệp unit. Các dấu nháy đơn quanh `'EOF'` giữ nguyên biến của BitBake như `${D}` khi ghi file từ Bash.

```bash
cat > ../meta-my-os/conf/distro/myos.conf <<'EOF'
require conf/distro/poky.conf

DISTRO = "myos"
DISTRO_NAME = "MyOS Linux"
DISTRO_VERSION = "1.0"
INIT_MANAGER = "systemd"
EOF

cat > ../meta-my-os/recipes-core/images/myos-image.bb <<'EOF'
SUMMARY = "Console image for the MyOS course project"
LICENSE = "MIT"

inherit core-image

IMAGE_INSTALL:append = " myos-info iproute2"
EOF

cat > ../meta-my-os/recipes-apps/myos-info/myos-info_1.0.bb <<'EOF'
SUMMARY = "Print MyOS runtime information"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://myos-info.c file://myos-info.service"
S = "${WORKDIR}"

inherit systemd

SYSTEMD_SERVICE:${PN} = "myos-info.service"
SYSTEMD_AUTO_ENABLE:${PN} = "enable"

do_compile() {
    ${CC} ${CFLAGS} ${CPPFLAGS} "${S}/myos-info.c" ${LDFLAGS} -o "${B}/myos-info"
}

do_install() {
    install -d "${D}${bindir}" "${D}${systemd_system_unitdir}"
    install -m 0755 "${B}/myos-info" "${D}${bindir}/myos-info"
    install -m 0644 "${WORKDIR}/myos-info.service" \
        "${D}${systemd_system_unitdir}/myos-info.service"
}
EOF

cat > ../meta-my-os/recipes-apps/myos-info/files/myos-info.c <<'EOF'
/* SPDX-License-Identifier: MIT */
#include <stdio.h>
#include <stdlib.h>
#include <sys/utsname.h>

int main(void)
{
    struct utsname system;

    if (uname(&system) == -1) {
        perror("uname");
        return EXIT_FAILURE;
    }

    printf("MYOS_INFO_OK arch=%s kernel=%s\n", system.machine, system.release);
    return EXIT_SUCCESS;
}
EOF

cat > ../meta-my-os/recipes-apps/myos-info/files/myos-info.service <<'EOF'
[Unit]
Description=Report MyOS runtime information

[Service]
Type=oneshot
ExecStart=/usr/bin/myos-info
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

**Giảng viên có thể hỏi ba tệp cấu hình khác nhau thế nào.** `myos.conf` đặt chính sách distro, phiên bản và init manager. `myos-image.bb` lựa chọn gói nằm trong rootfs. `myos-info_1.0.bb` chỉ cách lấy tệp nguồn, biên dịch và đóng gói ứng dụng. `inherit systemd` gắn unit vào vòng đời của gói. `SYSTEMD_AUTO_ENABLE` và phần `[Install]` đưa dịch vụ vào `multi-user.target`. Unit `oneshot` chạy một lần rồi thoát. Với `RemainAfterExit=yes`, trạng thái `active (exited)` là kết quả mong đợi sau khi chạy thành công. [Tạo distro riêng](https://docs.yoctoproject.org/scarthgap/dev-manual/custom-distribution.html) · [Tài liệu các lớp của Yocto](https://docs.yoctoproject.org/scarthgap/ref-manual/classes.html#systemd)

**Giảng viên có thể hỏi `${CC}` và `${D}` ở đâu.** BitBake cung cấp `${CC}`, cờ biên dịch, cờ link và sysroot của đích. `do_compile` tạo ELF AArch64. `do_install` chép kết quả vào cây staging dưới `${D}` để `do_package` tách gói. Khi build `myos-image`, `do_rootfs` đưa gói vào hệ thống tệp của image. Lúc boot, `systemd` gọi binary trong rootfs thật. [Tài liệu các tác vụ BitBake](https://docs.yoctoproject.org/scarthgap/ref-manual/tasks.html)

## 6. Build và chạy MyOS trên QEMU

**Ubuntu**, ở `~/yocto-lab/build-qemu`. Lưu lựa chọn distro cuối vào cấu hình nhóm, rồi đưa cấu hình đó vào QEMU. Mỗi dòng `require` bên dưới chỉ cần thêm một lần.

```bash
printf 'DISTRO = "myos"\n' > ../project-conf/myos.inc
printf '\nrequire ${TOPDIR}/../project-conf/myos.inc\n' >> conf/local.conf
mountpoint /mnt/yocto-cache && df -hT . /mnt/yocto-cache
bitbake-layers show-recipes myos-image myos-info
bitbake -e myos-image | grep -E '^(MACHINE|DISTRO|INIT_MANAGER|IMAGE_INSTALL)='
```

Kiểm tra `MACHINE="qemuarm64"`, `DISTRO="myos"`, `INIT_MANAGER="systemd"` và gói `myos-info` xuất hiện trong `IMAGE_INSTALL`.

```bash
set -o pipefail
{ time bitbake myos-image; } 2>&1 | tee ../evidence/qemu-myos-build.log
deploy=tmp/deploy/images/qemuarm64
ls -lh "$deploy"/myos-image*.{qemuboot.conf,ext4,manifest}
grep -E '^myos-info[[:space:]]' "$deploy/myos-image-qemuarm64.rootfs.manifest"
find tmp/work -type f -path '*/myos-info/*/image/usr/bin/myos-info' \
  -exec readelf -h {} \; | grep -E 'Class:|Machine:'
script -q -c 'runqemu qemuarm64 myos-image nographic slirp' \
  ../evidence/qemu-myos-console.log
```

`readelf` trên Ubuntu kiểm tra kiểu ELF mà không thực thi chương trình. Kết quả mong đợi là `ELF64` và `AArch64`.

**QEMU**, đăng nhập `root`.

```sh
cat /etc/os-release
uname -m
cat /proc/1/comm
systemctl is-enabled myos-info.service
systemctl --no-pager status myos-info.service
journalctl -b -u myos-info.service --no-pager
/usr/bin/myos-info
ip -br addr
```

**Đạt khi** `ID=myos`, `uname -m` là `aarch64`, PID 1 là `systemd`, dịch vụ `enabled` và journal của lượt boot có `MYOS_INFO_OK`. Chụp màn hình rồi `poweroff`. Giảng viên hỏi cách chứng minh tự chạy thì dùng `journalctl -b -u myos-info.service` ngay sau boot, trước khi chạy thủ công `/usr/bin/myos-info`. `-b` giới hạn kết quả ở lượt boot hiện tại.

## 7. Build MyOS cho Pi và kiểm tra phần cứng

**Ubuntu**

```bash
cd ~/yocto-lab
source poky/oe-init-build-env build-rpi
bitbake-layers add-layer ../meta-my-os
printf '\nrequire ${TOPDIR}/../project-conf/myos.inc\n' >> conf/local.conf
mountpoint /mnt/yocto-cache && mountpoint /mnt/yocto-pi-tmp && \
  df -hT . /mnt/yocto-cache /mnt/yocto-pi-tmp
bitbake -e myos-image | grep -E '^(MACHINE|DISTRO|INIT_MANAGER|TMPDIR|IMAGE_FSTYPES)='
```

Kiểm tra thấy `raspberrypi4-64`, `myos`, `systemd` và đúng `TMPDIR` của Pi. Tiếp tục build, rồi ghi thẻ theo mục 4 với `image_target=myos-image`.

```bash
set -o pipefail
{ time bitbake myos-image; } 2>&1 | tee ../evidence/rpi-myos-build.log
deploy=/mnt/yocto-pi-tmp/myos-tmp/deploy/images/raspberrypi4-64
ls -lh "$deploy"/myos-image*.{wic.bz2,wic.bmap,manifest}
grep -E '^myos-info[[:space:]]' "$deploy/myos-image-raspberrypi4-64.rootfs.manifest"
sha256sum "$deploy/myos-image-raspberrypi4-64.rootfs.wic.bz2" \
  | tee ../evidence/rpi-myos-image.sha256
```

**Pi**, lắp thẻ đã ghi MyOS, cắm Ethernet vào router có DHCP, boot và đăng nhập `root`.

```sh
cat /etc/os-release
uname -m
uname -r
cat /proc/1/comm
tr -d '\0' < /proc/device-tree/model
printf '\n'
cat /proc/cmdline
awk '$2 == "/" { print $1, $3 }' /proc/mounts
journalctl -b -u myos-info.service --no-pager
ip -br link
ip -br addr
ip -4 route
```

Nếu Ethernet có link nhưng chưa có IPv4, dùng `networkctl list` để tìm tên cổng đúng. Ví dụ dưới đây dùng `eth0`. Thay đúng tên nếu kết quả khác. Cấu hình này được ghi vào hệ thống đang chạy trên thẻ. Nếu dùng bước này, ghi rõ trong báo cáo DHCP được cấu hình trên thiết bị sau khi ghi image.

```sh
iface=eth0
ip link show "$iface"
mkdir -p /etc/systemd/network
printf '[Match]\nName=%s\n\n[Network]\nDHCP=ipv4\n' "$iface" \
  > /etc/systemd/network/10-myos-ethernet.network
systemctl enable --now systemd-networkd.service
networkctl reload
networkctl reconfigure "$iface"
networkctl status "$iface"
ip -4 route
```

Khi có tuyến mặc định, dùng `ping` với gateway LAN đã đọc từ bảng route. Nếu gateway chặn ICMP, thử địa chỉ một máy khác trong mạng LAN mà nhóm kiểm soát và biết có phản hồi ping.

```sh
gateway=$(ip -4 route show default | awk 'NR == 1 { print $3 }')
if [ -n "$gateway" ]; then
    ping -c 3 "$gateway"
fi
```

**Đạt khi** Pi boot đúng MyOS từ thẻ, nhận model bo mạch và journal có `MYOS_INFO_OK`. Bằng chứng Ethernet gồm link cổng thật, IP được cấp và kết quả kiểm tra tới một địa chỉ LAN. Nếu chỉ thấy interface, báo cáo ghi nhận tới bước nhận cổng. Nếu thiếu router/cáp, đánh dấu chờ kiểm thử Ethernet.

**Giảng viên có thể hỏi ảnh console chứng minh được gì.** `uname -m` xác nhận kiến trúc đang chạy, `uname -r` cho biết phiên bản kernel, `/proc/1/comm` xác nhận init, manifest chứng minh gói có trong image, còn journal chứng minh dịch vụ thực thi. `LOWER_UP` cho biết trạng thái link, địa chỉ IPv4 cho biết cấu hình mạng, còn phản hồi ping cho biết liên lạc tới một đích cụ thể.

## 8. Lưu cấu hình, ảnh và số liệu cho báo cáo

**Ubuntu**

```bash
cd ~/yocto-lab
cat > .gitignore <<'EOF'
/poky/
/meta-raspberrypi/
/build-qemu/
/build-rpi/
/evidence/
EOF
git init -b main
git add .gitignore project-conf meta-my-os
git status --short
git commit -m "Add MyOS distro and build configuration for two targets"
git rev-parse HEAD
```

Nếu Git yêu cầu danh tính, dùng `git config user.name` và `git config user.email` với thông tin Git của thành viên, rồi commit lại. Khi bàn giao, giữ riêng `evidence/` và các image. Để tái lập ở máy khác, dùng cùng commit của nhóm, lấy Poky và BSP như mục 1, chạy `oe-init-build-env` cho từng đích, rồi đưa các tệp `.inc` và layer vào theo các mục 2, 3, 6 và 7. Người dùng máy khác có thể chỉnh vị trí cache và `TMPDIR` theo filesystem ext4 của mình. Ghi lại lần build lại chạy với cache dùng chung hay cache mới.

| Tư liệu cần gửi | Cần thấy trong ảnh hoặc log | Vị trí trong báo cáo |
| --- | --- | --- |
| Ảnh VM hoặc kết quả `df -hT` | VM chạy Ubuntu, dung lượng và vị trí ba filesystem | Môi trường thực nghiệm |
| Một log build thành công cho mỗi đích | Tên image, đích build, kết quả task và thời gian thực đo | Triển khai và kết quả |
| Ảnh terminal QEMU chạy MyOS | `ID=myos`, `aarch64`, PID 1 và `MYOS_INFO_OK` | Kiểm thử mô phỏng |
| Ảnh toàn cảnh Pi khi demo | Pi, thẻ hoặc đầu đọc, cáp HDMI, màn hình console | Bố trí phần cứng |
| Ảnh cận cảnh màn hình Pi hoặc log serial | Model Pi, MyOS, kernel, mount rootfs và log dịch vụ | Kiểm thử trên kit |
| Ảnh kết quả Ethernet nếu đã làm | `ip -br addr`, route và kết quả ping LAN | Đánh giá phần cứng |

Chụp ảnh QEMU và Pi **sau khi đã có kết quả thật**. Giữ log dạng văn bản để đối chiếu khi viết báo cáo. Nếu gửi ảnh chia sẻ, che thông tin mạng riêng, đường dẫn cá nhân hoặc thông tin đăng nhập nếu xuất hiện. Sơ đồ quy trình có thể vẽ từ các đầu ra đã xác nhận dưới đây.

```text
Poky + meta-my-os                       Poky + meta-my-os + meta-raspberrypi
MACHINE=qemuarm64                        MACHINE=raspberrypi4-64
           |                                          |
           +---- BitBake, cùng recipe myos-image -----+
           |                                          |
kernel + rootfs.ext4 + qemuboot.conf      .wic.bz2 + .wic.bmap
           |                                          |
       runqemu                            ghi microSD → boot Pi 4
           |                                          |
   console và journal                   console, journal và Ethernet
```

Trong báo cáo, đưa sơ đồ này vào phần thiết kế. Phần thực nghiệm dùng bảng theo từng đích với cột **tiêu chí, lệnh kiểm tra, kết quả quan sát và bằng chứng**. Nếu trình bày dung lượng, ghi rõ đang đo rootfs ext4, WIC chưa nén hay WIC đã nén. Ghi thời gian build kèm trạng thái cache. Các số này mô tả lần build của nhóm.

## 9. Tra lỗi theo dấu hiệu

| Dấu hiệu | Kiểm tra theo thứ tự |
| --- | --- |
| `bitbake: command not found` | Quay về `~/yocto-lab`, chạy `source poky/oe-init-build-env build-qemu` hoặc `build-rpi` trong chính terminal đang dùng. |
| `Nothing PROVIDES 'myos-image'` hoặc lỗi tìm distro | Chạy `bitbake-layers show-layers`, xác nhận `meta-my-os` đã thêm ở build hiện tại và tên tệp đúng. |
| `MACHINE` sai hoặc không hỗ trợ | Dùng `bitbake -e <image>` để kiểm tra biến hiệu lực. Đối chiếu `.inc` và `bblayers.conf`. Pi cần `meta-raspberrypi` trong build Pi. |
| `do_fetch` thất bại | Lấy dòng `ERROR` đầu tiên và URL nguồn từ log, kiểm tra mạng VM hoặc chạy lại sau khi mạng ổn định. Chia sẻ phần lỗi đã che thông tin proxy nếu có. |
| `do_compile`, `do_package` hoặc `do_rootfs` lỗi | Lấy đường dẫn `Logfile of failure stored in` do BitBake in ra, rồi gửi 40–80 dòng cuối của tệp đó cùng tên recipe và task. Ưu tiên sửa lỗi đầu tiên. |
| `No space left on device` hoặc BitBake dừng vì disk monitor | Chạy `df -hT`, `df -i` trong Ubuntu và kiểm tra ổ Windows chứa VMDK tương ứng. Xác nhận cả hai mount. Dùng `du -sh ~/yocto-lab/build-qemu/tmp /mnt/yocto-cache/* /mnt/yocto-pi-tmp/myos-tmp` để tìm khu vực chiếm chỗ. VMware dùng đĩa tự tăng dung lượng nên phần trống trong guest và Windows cần được kiểm tra cùng lúc. |
| Pi có nguồn nhưng màn hình không tới console | Kiểm tra đúng image Pi, quá trình ghi thành công, thiết bị boot khác đã tháo, nguồn và cáp HDMI. Thử đúng cổng micro-HDMI gần nguồn và ghi lại màn hình/đèn khi khởi động. Nếu cần chuyển sang UART, gửi model Pi và USB–TTL trước để xác nhận chân nối và mức 3,3 V. |
| Kernel đã lên nhưng báo `VFS: Unable to mount root fs` | Đối chiếu `root=` trong boot log với phân vùng được tạo bởi WIC, kiểm tra thẻ và việc ghi toàn đĩa. Trên Pi mặc định của quy trình, rootfs nằm ở phân vùng thứ hai của microSD. |
| Boot được nhưng chưa có `MYOS_INFO_OK` | Kiểm tra đang boot đúng MyOS, image manifest có `myos-info`, rồi chạy `systemctl is-enabled myos-info.service` và `journalctl -b -u myos-info.service`. Xem cả mã thoát, đường dẫn `ExecStart` và dòng lỗi đầu tiên. |
| Ethernet có cổng nhưng thiếu IPv4 | Kiểm tra cáp, router DHCP, trạng thái `LOWER_UP`, tên interface và `networkctl status`. Phân biệt lỗi link, thiếu lease DHCP và thiếu route. |

## 10. Câu hỏi tổng hợp khi bảo vệ

| Câu hỏi | Trả lời dựa trên bài làm |
| --- | --- |
| **Yocto, Poky và BitBake khác nhau thế nào?** | Yocto Project cung cấp công cụ, metadata và tài liệu để tạo Linux nhúng. Poky là bản phân phối tham chiếu và cây nguồn khởi đầu. BitBake đọc metadata, lập đồ thị phụ thuộc và chạy các task. Minh hoạ bằng lần build Poky ban đầu và `myos-image` của nhóm. |
| **Tại sao nói đã build một hệ điều hành?** | Nhóm dùng BitBake xây dựng kernel, gói user space và rootfs, tạo image boot được. Nhóm chọn distro, gói và dịch vụ của hệ thống, sau đó kiểm chứng trên hai đích. Linux kernel và BSP lấy từ các dự án nguồn đã ghi phiên bản. |
| **Phần nào nhóm phát triển, phần nào lấy từ upstream?** | Nhóm viết layer `meta-my-os`, distro, image recipe, mã C, unit, cấu hình build và ca kiểm thử. Poky cung cấp nền build và các gói sẵn có. `meta-raspberrypi` cung cấp phần tích hợp bo mạch. Mở Git để chỉ đúng file. |
| **Vì sao hai image có thể khác phiên bản kernel?** | QEMU chọn `linux-yocto`, Pi chọn `linux-raspberrypi` theo BSP. Hai lựa chọn có thể khác bản phát hành, cấu hình và patch. Dùng `uname -r` và `bitbake -e virtual/kernel` ở từng build để kiểm tra. |
| **Nếu sao chép `rootfs.ext4` sang thẻ Pi thì sao?** | Tệp đó chỉ biểu diễn filesystem. Quy trình Pi cần layout đĩa và phân vùng boot chứa firmware, cấu hình, kernel và DTB phù hợp. WIC đóng gói layout đó cùng rootfs. |
| **Tại sao đổi code C rồi BitBake chỉ chạy một phần?** | Thay đổi file trong `SRC_URI` làm thay đổi chữ ký các task liên quan. BitBake chạy lại phần ứng dụng và các bước dựng image chịu ảnh hưởng, có thể dùng lại kết quả hợp lệ từ sstate cho phần còn lại. |
| **Dịch vụ có chứng minh chạy trên phần cứng không?** | Log dịch vụ chứng minh user space chạy. Model từ Device Tree, thiết bị root và Ethernet xác nhận các khía cạnh riêng của phần cứng. Khi bảo vệ, trình bày cả ba bằng chứng cùng kết quả boot. |
| **Nếu được yêu cầu so với Buildroot thì nói gì?** | Trình bày mục tiêu của nhóm là quản lý cấu hình distro, BSP và gói phần mềm bằng layer dùng chung. Nếu giảng viên yêu cầu số liệu đối chiếu thực tế, cần build thêm cấu hình Buildroot với cùng phạm vi phần mềm và điều kiện đo. [Tài liệu Buildroot](https://buildroot.org/downloads/manual/manual.html) |

**Tình trạng xác minh.** Cấu hình và lệnh trong hướng dẫn đã đối chiếu với tài liệu và mã nguồn Scarthgap. Kết quả build, boot và ghi thẻ cần được ghi nhận trong lúc thực hành trên VM và kit.
