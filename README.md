![Top Langs](https://github-readme-stats.vercel.app/api/top-langs/?username=ParrotXray&langs_count=8)
#!/bin/bash

# 系统打包脚本 - 将OH系统打包成IMG文件

# 使用方法: sudo ./pack_system.sh [输出文件名]

set -e

# 配置参数

SOURCE_DIR=“OH”
OUTPUT_IMG=${1:-“oh_system.img”}
IMG_SIZE=“12G”  # 总大小 (ESP 1G + System 5G + Vendor 5G + Data 1G)

# 分区大小 (扇区数，512字节/扇区)

ESP_SIZE=$((1024 * 1024 * 1024 / 512))      # 1GB
SYSTEM_SIZE=$((5 * 1024 * 1024 * 1024 / 512)) # 5GB  
VENDOR_SIZE=$((5 * 1024 * 1024 * 1024 / 512)) # 5GB
DATA_SIZE=$((1 * 1024 * 1024 * 1024 / 512))   # 1GB

# 检查权限

if [ “$EUID” -ne 0 ]; then
echo “错误: 请使用 sudo 运行此脚本”
exit 1
fi

# 检查源目录

if [ ! -d “$SOURCE_DIR” ]; then
echo “错误: 找不到源目录 $SOURCE_DIR”
exit 1
fi

# 检查必要文件

echo “检查必要文件…”
for file in “$SOURCE_DIR/boot/EFI/BOOT/bootx64.efi”   
“$SOURCE_DIR/boot/grub/grub.cfg”   
“$SOURCE_DIR/boot/bzImage”   
“$SOURCE_DIR/boot/ramdisk.img”   
“$SOURCE_DIR/system.img”   
“$SOURCE_DIR/vendor.img”; do
if [ ! -f “$file” ]; then
echo “错误: 找不到文件 $file”
exit 1
fi
done

echo “创建镜像文件: $OUTPUT_IMG”

# 创建空镜像文件

dd if=/dev/zero of=”$OUTPUT_IMG” bs=1G count=12 status=progress

# 设置循环设备

LOOP_DEV=$(losetup -f –show “$OUTPUT_IMG”)
echo “使用循环设备: $LOOP_DEV”

# 清理函数

cleanup() {
echo “清理资源…”
sync
umount /tmp/esp_mount 2>/dev/null || true
umount /tmp/system_mount 2>/dev/null || true
umount /tmp/vendor_mount 2>/dev/null || true
umount /tmp/data_mount 2>/dev/null || true
losetup -d “$LOOP_DEV” 2>/dev/null || true
rmdir /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount 2>/dev/null || true
}
trap cleanup EXIT

echo “创建分区表…”

# 创建GPT分区表

parted -s “$LOOP_DEV” mklabel gpt

# 创建分区

parted -s “$LOOP_DEV” mkpart ESP fat32 1MiB $((ESP_SIZE * 512 / 1024 / 1024))MiB
parted -s “$LOOP_DEV” mkpart system ext4 $((ESP_SIZE * 512 / 1024 / 1024))MiB $(((ESP_SIZE + SYSTEM_SIZE) * 512 / 1024 / 1024))MiB
parted -s “$LOOP_DEV” mkpart vendor ext4 $(((ESP_SIZE + SYSTEM_SIZE) * 512 / 1024 / 1024))MiB $(((ESP_SIZE + SYSTEM_SIZE + VENDOR_SIZE) * 512 / 1024 / 1024))MiB
parted -s “$LOOP_DEV” mkpart data ext4 $(((ESP_SIZE + SYSTEM_SIZE + VENDOR_SIZE) * 512 / 1024 / 1024))MiB 100%

# 设置ESP分区为启动分区

parted -s “$LOOP_DEV” set 1 esp on

# 通知内核重新读取分区表

partprobe “$LOOP_DEV”
sleep 2

echo “格式化分区…”

# 格式化分区

mkfs.fat -F32 -n “ESP” “${LOOP_DEV}p1”
mkfs.ext4 -F -L “system” “${LOOP_DEV}p2”
mkfs.ext4 -F -L “vendor” “${LOOP_DEV}p3”
mkfs.ext4 -F -L “data” “${LOOP_DEV}p4”

# 创建挂载点

mkdir -p /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount

echo “挂载分区…”
mount “${LOOP_DEV}p1” /tmp/esp_mount
mount “${LOOP_DEV}p2” /tmp/system_mount
mount “${LOOP_DEV}p3” /tmp/vendor_mount
mount “${LOOP_DEV}p4” /tmp/data_mount

echo “复制ESP分区文件…”

# 复制boot文件夹内容到ESP分区

cp -r “$SOURCE_DIR/boot”/* /tmp/esp_mount/

# 修改grub.cfg添加nouveau参数

if [ -f “/tmp/esp_mount/grub/grub.cfg” ]; then
echo “修改GRUB配置，添加nouveau参数…”
# 备份原文件
cp “/tmp/esp_mount/grub/grub.cfg” “/tmp/esp_mount/grub/grub.cfg.bak”

```
# 在linux命令行中添加nouveau参数
sed -i 's/linux.*bzImage/& nouveau.atomic=1 nouveau.modeset=1/' "/tmp/esp_mount/grub/grub.cfg"

echo "GRUB配置已更新"
```

fi

echo “写入system分区…”

# 将system.img写入system分区

dd if=”$SOURCE_DIR/system.img” of=”${LOOP_DEV}p2” bs=4M status=progress

echo “写入vendor分区…”

# 将vendor.img写入vendor分区

dd if=”$SOURCE_DIR/vendor.img” of=”${LOOP_DEV}p3” bs=4M status=progress

echo “创建data分区基本结构…”

# 在data分区创建基本目录结构

mkdir -p /tmp/data_mount/user_data
echo “# OH System Data Partition” > /tmp/data_mount/README.txt

# 同步数据

sync

echo “卸载分区…”
umount /tmp/esp_mount
umount /tmp/system_mount  
umount /tmp/vendor_mount
umount /tmp/data_mount

# 分离循环设备

losetup -d “$LOOP_DEV”

echo “”
echo “=========================================”
echo “镜像文件创建完成: $OUTPUT_IMG”
echo “=========================================”
echo “”
echo “分区信息:”
echo “1. ESP/Boot:  1GB  (FAT32) - 引导分区”
echo “2. System:    5GB  (EXT4)  - 系统分区”  
echo “3. Vendor:    5GB  (EXT4)  - 厂商分区”
echo “4. Data:      1GB  (EXT4)  - 数据分区”
echo “”
echo “使用方法:”
echo “1. 刷写到USB设备:”
echo “   sudo dd if=$OUTPUT_IMG of=/dev/sdX bs=4M status=progress”
echo “   (请将sdX替换为你的USB设备)”
echo “”
echo “2. 虚拟机测试:”
echo “   qemu-system-x86_64 -enable-kvm -m 2G -drive file=$OUTPUT_IMG,format=raw”
echo “”
echo “注意: 刷写前请确认目标设备，避免数据丢失！”