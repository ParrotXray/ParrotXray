![Top Langs](https://github-readme-stats.vercel.app/api/top-langs/?username=ParrotXray&langs_count=8)


#!/bin/bash

set -e

# 颜色定义

RED=’\033[0;31m’
GREEN=’\033[0;32m’
YELLOW=’\033[1;33m’
BLUE=’\033[0;34m’
CYAN=’\033[0;36m’
NC=’\033[0m’ # No Color

CURRENT_STEP=0
ELAPSED_TIME=0
START_TIME=$(date +%s)

print() {
local color=$1
local msg=”$2”

```
echo -e "${color} $message ${NC}"
```

}

show_step() {
# 显示步骤信息
echo -e “\n$1”
}

# 步骤完成标记

step_done() {
local message=”${1:-完成}”
echo -e “  ${GREEN}[  OK  ]${NC} $message”
}

# 子步骤显示

substep() {
echo “  * $1”
}

# 错误处理

step_error() {
local error_msg=”$1”
echo -e “  ${RED}[FAILED]${NC} $error_msg”
exit 1
}

# 检查并安装pv

check_pv() {
if ! command -v pv >/dev/null 2>&1; then
echo -e “${YELLOW}[ WARN ]${NC} 未检测到pv工具，正在安装…”
if command -v apt >/dev/null 2>&1; then
apt update >/dev/null 2>&1 && apt install -y pv >/dev/null 2>&1 || step_error “无法安装pv工具 (apt)”
elif command -v yum >/dev/null 2>&1; then
yum install -y pv >/dev/null 2>&1 || step_error “无法安装pv工具 (yum)”
elif command -v pacman >/dev/null 2>&1; then
pacman -S –noconfirm pv >/dev/null 2>&1 || step_error “无法安装pv工具 (pacman)”
else
step_error “无法自动安装pv，请手动安装: sudo apt install pv”
fi
echo -e “${GREEN}[  OK  ]${NC} pv安装完成”
fi
}

# 检查命令是否存在

check_command() {
local cmd=”$1”
local desc=”$2”
if ! command -v “$cmd” >/dev/null 2>&1; then
step_error “缺少必要工具: $cmd ($desc)”
fi
}

# 配置参数

ESP_SIZE_CONFIG=1
SYSTEM_SIZE_CONFIG=5
VENDOR_SIZE_CONFIG=5
DATA_SIZE_CONFIG=1

IMG_SIZE_SET=$((ESP_SIZE_CONFIG + SYSTEM_SIZE_CONFIG + VENDOR_SIZE_CONFIG + DATA_SIZE_CONFIG))
OUTPUT_IMG=${1:-“ohos.img”}

# 分区大小 (扇区数，512字节/扇区)

ESP_SIZE=$((ESP_SIZE_CONFIG * 1024 * 1024 * 1024 / 512))      # 1GB
SYSTEM_SIZE=$((SYSTEM_SIZE_CONFIG * 1024 * 1024 * 1024 / 512)) # 5GB  
VENDOR_SIZE=$((VENDOR_SIZE_CONFIG * 1024 * 1024 * 1024 / 512)) # 5GB
DATA_SIZE=$((DATA_SIZE_CONFIG * 1024 * 1024 * 1024 / 512))   # 1GB

# 清理函数

cleanup() {
if [ -n “$LOOP_DEV” ] && [ -b “$LOOP_DEV” ]; then
echo -e “\n${YELLOW}[ INFO ]${NC} 清理资源…”
sync >/dev/null 2>&1 || true
umount /tmp/esp_mount >/dev/null 2>&1 || true
umount /tmp/system_mount >/dev/null 2>&1 || true
umount /tmp/vendor_mount >/dev/null 2>&1 || true
umount /tmp/data_mount >/dev/null 2>&1 || true
losetup -d “$LOOP_DEV” >/dev/null 2>&1 || true
rmdir /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount >/dev/null 2>&1 || true
fi
}
trap cleanup EXIT

# 检查权限

if [ “$EUID” -ne 0 ]; then
echo -e “${RED}[FAILED]${NC} 请使用 sudo 运行此脚本”
exit 1
fi

echo -e “${GREEN}OH系统高级打包工具${NC}”
echo “==========================”

# 检查必要工具

substep “检查系统工具…”
check_command “parted” “磁盘分区工具”
check_command “mkfs.fat” “FAT32文件系统创建工具”
check_command “mkfs.ext4” “EXT4文件系统创建工具”
check_command “losetup” “循环设备工具”
check_command “mount” “挂载工具”

# 检查并安装pv

check_pv

# === 步骤1: 检查必要文件 ===

show_step “检查必要文件”
file_count=0
required_files=(
“boot/EFI/BOOT/bootx64.efi”
“boot/grub/grub.cfg”
“boot/bzImage”
“boot/ramdisk.img”
“system.img”
“vendor.img”
)

for file in “${required_files[@]}”; do
if [ ! -f “$file” ]; then
step_error “找不到文件 $file”
fi
if [ ! -r “$file” ]; then
step_error “文件 $file 不可读，请检查权限”
fi
substep “检查 $(basename “$file”)”
file_count=$((file_count + 1))
done
step_done “检查了 $file_count 个必要文件”

# 检查磁盘空间

substep “检查磁盘空间…”
required_space=$((IMG_SIZE_SET * 1024 * 1024 * 1024))  # 字节
available_space=$(df . | tail -1 | awk ‘{print $4}’ | sed ‘s/[^0-9]//g’)
available_space=$((available_space * 1024))  # 转换为字节

if [ “$available_space” -lt “$required_space” ]; then
step_error “磁盘空间不足。需要: $(($required_space / 1024 / 1024 / 1024))GB, 可用: $(($available_space / 1024 / 1024 / 1024))GB”
fi
substep “磁盘空间检查通过”

# === 步骤2: 创建镜像文件 ===

show_step “创建镜像文件: $OUTPUT_IMG”
substep “分配存储空间…”

# 检查是否已存在同名文件

if [ -f “$OUTPUT_IMG” ]; then
echo -e “  ${YELLOW}[ WARN ]${NC} 文件 $OUTPUT_IMG 已存在，将覆盖”
rm -f “$OUTPUT_IMG” || step_error “无法删除现有文件 $OUTPUT_IMG”
fi

# 创建镜像文件

if ! dd if=/dev/zero bs=1M count=$((IMG_SIZE_SET * 1024)) status=none 2>/dev/null | pv -s $((IMG_SIZE_SET * 1024 * 1024 * 1024)) > “$OUTPUT_IMG”; then
step_error “创建镜像文件失败，可能是磁盘空间不足或权限问题”
fi

# 验证文件大小

actual_size=$(stat -c%s “$OUTPUT_IMG” 2>/dev/null || echo “0”)
expected_size=$((IMG_SIZE_SET * 1024 * 1024 * 1024))
if [ “$actual_size” -ne “$expected_size” ]; then
step_error “镜像文件大小不正确。期望: ${expected_size}字节, 实际: ${actual_size}字节”
fi

step_done “镜像文件创建完成”

# === 步骤3: 设置循环设备 ===

show_step “设置循环设备”
LOOP_DEV=$(losetup -f –show “$OUTPUT_IMG” 2>/dev/null)
if [ -z “$LOOP_DEV” ]; then
step_error “无法创建循环设备，可能没有可用的loop设备”
fi

if [ ! -b “$LOOP_DEV” ]; then
step_error “循环设备 $LOOP_DEV 创建失败”
fi

substep “循环设备: $LOOP_DEV”
step_done

# === 步骤4: 创建分区表和分区 ===

show_step “创建分区表和分区”
substep “创建GPT分区表…”
if ! parted -s “$LOOP_DEV” mklabel gpt >/dev/null 2>&1; then
step_error “创建GPT分区表失败”
fi

partition_names=(“ESP” “system” “vendor” “data”)

for i in “${!partition_names[@]}”; do
substep “创建${partition_names[$i]}分区…”
done

# 创建分区并检查结果

if ! parted -s “$LOOP_DEV” mkpart ESP fat32 1MiB $((ESP_SIZE * 512 / 1024 / 1024))MiB >/dev/null 2>&1; then
step_error “创建ESP分区失败”
fi

if ! parted -s “$LOOP_DEV” mkpart system ext4 $((ESP_SIZE * 512 / 1024 / 1024))MiB $(((ESP_SIZE + SYSTEM_SIZE) * 512 / 1024 / 1024))MiB >/dev/null 2>&1; then
step_error “创建system分区失败”
fi

if ! parted -s “$LOOP_DEV” mkpart vendor ext4 $(((ESP_SIZE + SYSTEM_SIZE) * 512 / 1024 / 1024))MiB $(((ESP_SIZE + SYSTEM_SIZE + VENDOR_SIZE) * 512 / 1024 / 1024))MiB >/dev/null 2>&1; then
step_error “创建vendor分区失败”
fi

if ! parted -s “$LOOP_DEV” mkpart data ext4 $(((ESP_SIZE + SYSTEM_SIZE + VENDOR_SIZE) * 512 / 1024 / 1024))MiB 100% >/dev/null 2>&1; then
step_error “创建data分区失败”
fi

substep “设置ESP启动标志…”
if ! parted -s “$LOOP_DEV” set 1 esp on >/dev/null 2>&1; then
step_error “设置ESP启动标志失败”
fi

substep “刷新分区表…”
if ! partprobe “$LOOP_DEV” >/dev/null 2>&1; then
step_error “刷新分区表失败”
fi

sleep 2

# 验证分区是否创建成功

for i in {1..4}; do
if [ ! -b “${LOOP_DEV}p$i” ]; then
step_error “分区 ${LOOP_DEV}p$i 创建失败”
fi
done

step_done “创建了${#partition_names[@]}个分区”

# === 步骤5: 格式化分区 ===

show_step “格式化分区”
format_configs=(
“ESP:FAT32:mkfs.fat -F32 -n ESP”
“system:EXT4:mkfs.ext4 -F -L system”
“vendor:EXT4:mkfs.ext4 -F -L vendor”
“data:EXT4:mkfs.ext4 -F -L data”
)

for i in “${!format_configs[@]}”; do
IFS=’:’ read -r name fs_type cmd <<< “${format_configs[$i]}”
substep “格式化${name}分区 ($fs_type)…”
if ! $cmd “${LOOP_DEV}p$((i+1))” >/dev/null 2>&1; then
step_error “格式化${name}分区失败”
fi
done
step_done “格式化了${#format_configs[@]}个分区”

# === 步骤6: 挂载分区 ===

show_step “挂载分区”

# 创建挂载点

if ! mkdir -p /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount; then
step_error “创建挂载点失败”
fi

mount_points=(”/tmp/esp_mount” “/tmp/system_mount” “/tmp/vendor_mount” “/tmp/data_mount”)
mount_names=(“ESP” “system” “vendor” “data”)

for i in “${!mount_points[@]}”; do
substep “挂载${mount_names[$i]}分区…”
if ! mount “${LOOP_DEV}p$((i+1))” “${mount_points[$i]}”; then
step_error “挂载${mount_names[$i]}分区失败”
fi

```
# 验证挂载成功
if ! mountpoint -q "${mount_points[$i]}"; then
    step_error "${mount_names[$i]}分区挂载验证失败"
fi
```

done
step_done “挂载了${#mount_points[@]}个分区”

# === 步骤7: 复制系统文件 ===

show_step “复制系统文件”
substep “复制EFI启动文件…”
if ! cp -r “boot”/* /tmp/esp_mount/; then
step_error “复制EFI启动文件失败”
fi

# 修改grub.cfg添加nouveau参数

if [ -f “/tmp/esp_mount/grub/grub.cfg” ]; then
substep “修改GRUB配置…”
if ! cp “/tmp/esp_mount/grub/grub.cfg” “/tmp/esp_mount/grub/grub.cfg.bak”; then
step_error “备份GRUB配置文件失败”
fi
if ! sed -i ‘s/linux.*bzImage/& nouveau.atomic=1 nouveau.modeset=1/’ “/tmp/esp_mount/grub/grub.cfg”; then
step_error “修改GRUB配置失败”
fi
fi

substep “写入system分区…”
system_size=$(stat -c%s “system.img” 2>/dev/null)
if [ -z “$system_size” ] || [ “$system_size” -eq 0 ]; then
step_error “无法获取system.img文件大小”
fi

if ! dd if=“system.img” bs=4M status=none 2>/dev/null | pv -s “$system_size” > “${LOOP_DEV}p2”; then
step_error “写入system分区失败”
fi

substep “写入vendor分区…”
vendor_size=$(stat -c%s “vendor.img” 2>/dev/null)
if [ -z “$vendor_size” ] || [ “$vendor_size” -eq 0 ]; then
step_error “无法获取vendor.img文件大小”
fi

if ! dd if=“vendor.img” bs=4M status=none 2>/dev/null | pv -s “$vendor_size” > “${LOOP_DEV}p3”; then
step_error “写入vendor分区失败”
fi

substep “初始化data分区…”
if ! mkdir -p /tmp/data_mount/user_data; then
step_error “创建data分区目录失败”
fi

if ! echo “# OH System Data Partition - $(date)” > /tmp/data_mount/README.txt; then
step_error “创建data分区文件失败”
fi

step_done “系统文件复制完成”

# === 步骤8: 完成打包 ===

show_step “完成打包”
substep “同步数据到磁盘…”
if ! sync; then
step_error “同步数据失败”
fi

substep “卸载分区…”
for mount_point in /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount; do
if mountpoint -q “$mount_point”; then
if ! umount “$mount_point”; then
step_error “卸载 $mount_point 失败”
fi
fi
done

substep “释放循环设备…”
if ! losetup -d “$LOOP_DEV”; then
step_error “释放循环设备失败”
fi

# 验证最终文件

if [ ! -f “$OUTPUT_IMG” ]; then
step_error “最终镜像文件不存在”
fi

final_size=$(stat -c%s “$OUTPUT_IMG” 2>/dev/null || echo “0”)
if [ “$final_size” -eq 0 ]; then
step_error “最终镜像文件大小为0”
fi

final_time=$(date +%s)
total_elapsed=$((final_time - START_TIME))
step_done “打包完成，总耗时: ${total_elapsed}秒”

echo “”
echo -e “${GREEN}=============================================${NC}”
echo -e “${GREEN}[  OK  ] 镜像文件创建完成: $OUTPUT_IMG${NC}”
echo -e “${GREEN}[ INFO ] 文件大小: $(du -h “$OUTPUT_IMG” | cut -f1)${NC}”
echo -e “${GREEN}[ INFO ] 总耗时: ${total_elapsed}秒${NC}”
echo -e “${GREEN}=============================================${NC}”
echo “”
echo -e “${BLUE}分区信息:${NC}”
echo “   1. ESP/Boot:  1GB  (FAT32) - 引导分区”
echo “   2. System:    5GB  (EXT4)  - 系统分区”  
echo “   3. Vendor:    5GB  (EXT4)  - 厂商分区”
echo “   4. Data:      1GB  (EXT4)  - 数据分区”
echo “”
echo -e “${BLUE}使用方法:${NC}”
echo “   1. 刷写到USB设备:”
echo -e “      ${YELLOW}sudo pv $OUTPUT_IMG | dd of=/dev/sdX bs=4M${NC}”
echo “      (推荐使用pv显示刷写进度)”
echo “”
echo “   2. dd命令:”
echo -e “      ${YELLOW}sudo dd if=$OUTPUT_IMG of=/dev/sdX bs=4M status=progress${NC}”
echo “”
echo “   3. 虚拟机测试 (UEFI + KVM):”
echo -e “      ${YELLOW}qemu-system-x86_64 -enable-kvm -cpu host -m 2G \${NC}”
echo -e “      ${YELLOW}  -drive if=pflash,format=raw,readonly=on,file=/usr/share/ovmf/OVMF_CODE.fd \${NC}”
echo -e “      ${YELLOW}  -drive if=pflash,format=raw,file=/tmp/OVMF_VARS.fd \${NC}”
echo -e “      ${YELLOW}  -drive file=$OUTPUT_IMG,format=raw${NC}”
echo “”
echo “   4. 虚拟机测试 (传统BIOS):”
echo -e “      ${YELLOW}qemu-system-x86_64 -enable-kvm -cpu host -m 2G -drive file=$OUTPUT_IMG,format=raw${NC}”
echo “”
echo -e “${CYAN}如果没有KVM支持，移除 -enable-kvm -cpu host 参数${NC}”
echo “”
echo -e “${RED}注意: 刷写前请确认目标设备，避免数据丢失！${NC}”/ 1024))MiB >/dev/null 2>&1; then
step_error “创建system分区失败”
fi

if ! parted -s “$LOOP_DEV” mkpart vendor ext4 $(((ESP_SIZE + SYSTEM_SIZE) * 512 / 1024 / 1024))MiB $(((ESP_SIZE + SYSTEM_SIZE + VENDOR_SIZE) * 512 / 1024 / 1024))MiB >/dev/null 2>&1; then
step_error “创建vendor分区失败”
fi

if ! parted -s “$LOOP_DEV” mkpart data ext4 $(((ESP_SIZE + SYSTEM_SIZE + VENDOR_SIZE) * 512 / 1024 / 1024))MiB 100% >/dev/null 2>&1; then
step_error “创建data分区失败”
fi

substep “设置ESP启动标志…”
if ! parted -s “$LOOP_DEV” set 1 esp on >/dev/null 2>&1; then
step_error “设置ESP启动标志失败”
fi

substep “刷新分区表…”
if ! partprobe “$LOOP_DEV” >/dev/null 2>&1; then
step_error “刷新分区表失败”
fi

sleep 2

# 验证分区是否创建成功

for i in {1..4}; do
if [ ! -b “${LOOP_DEV}p$i” ]; then
step_error “分区 ${LOOP_DEV}p$i 创建失败”
fi
done

step_done “创建了${#partition_names[@]}个分区”

# === 步骤5: 格式化分区 ===

show_step “格式化分区”
format_configs=(
“ESP:FAT32:mkfs.fat -F32 -n ESP”
“system:EXT4:mkfs.ext4 -F -L system”
“vendor:EXT4:mkfs.ext4 -F -L vendor”
“data:EXT4:mkfs.ext4 -F -L data”
)

for i in “${!format_configs[@]}”; do
IFS=’:’ read -r name fs_type cmd <<< “${format_configs[$i]}”
substep “格式化${name}分区 ($fs_type)…”
if ! $cmd “${LOOP_DEV}p$((i+1))” >/dev/null 2>&1; then
step_error “格式化${name}分区失败”
fi
done
step_done “格式化了${#format_configs[@]}个分区”

# === 步骤6: 挂载分区 ===

show_step “挂载分区”

# 创建挂载点

if ! mkdir -p /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount; then
step_error “创建挂载点失败”
fi

mount_points=(”/tmp/esp_mount” “/tmp/system_mount” “/tmp/vendor_mount” “/tmp/data_mount”)
mount_names=(“ESP” “system” “vendor” “data”)

for i in “${!mount_points[@]}”; do
substep “挂载${mount_names[$i]}分区…”
if ! mount “${LOOP_DEV}p$((i+1))” “${mount_points[$i]}”; then
step_error “挂载${mount_names[$i]}分区失败”
fi

```
# 验证挂载成功
if ! mountpoint -q "${mount_points[$i]}"; then
    step_error "${mount_names[$i]}分区挂载验证失败"
fi
```

done
step_done “挂载了${#mount_points[@]}个分区”

# === 步骤7: 复制系统文件 ===

show_step “复制系统文件”
substep “复制EFI启动文件…”
if ! cp -r “boot”/* /tmp/esp_mount/; then
step_error “复制EFI启动文件失败”
fi

# 修改grub.cfg添加nouveau参数

if [ -f “/tmp/esp_mount/grub/grub.cfg” ]; then
substep “修改GRUB配置…”
if ! cp “/tmp/esp_mount/grub/grub.cfg” “/tmp/esp_mount/grub/grub.cfg.bak”; then
step_error “备份GRUB配置文件失败”
fi
if ! sed -i ‘s/linux.*bzImage/& nouveau.atomic=1 nouveau.modeset=1/’ “/tmp/esp_mount/grub/grub.cfg”; then
step_error “修改GRUB配置失败”
fi
fi

substep “写入system分区…”
system_size=$(stat -c%s “system.img” 2>/dev/null)
if [ -z “$system_size” ] || [ “$system_size” -eq 0 ]; then
step_error “无法获取system.img文件大小”
fi

if ! dd if=“system.img” bs=4M status=none 2>/dev/null | pv -s “$system_size” > “${LOOP_DEV}p2”; then
step_error “写入system分区失败”
fi

substep “写入vendor分区…”
vendor_size=$(stat -c%s “vendor.img” 2>/dev/null)
if [ -z “$vendor_size” ] || [ “$vendor_size” -eq 0 ]; then
step_error “无法获取vendor.img文件大小”
fi

if ! dd if=“vendor.img” bs=4M status=none 2>/dev/null | pv -s “$vendor_size” > “${LOOP_DEV}p3”; then
step_error “写入vendor分区失败”
fi

substep “初始化data分区…”
if ! mkdir -p /tmp/data_mount/user_data; then
step_error “创建data分区目录失败”
fi

if ! echo “# OH System Data Partition - $(date)” > /tmp/data_mount/README.txt; then
step_error “创建data分区文件失败”
fi

step_done “系统文件复制完成”

# === 步骤8: 完成打包 ===

show_step “完成打包”
substep “同步数据到磁盘…”
if ! sync; then
step_error “同步数据失败”
fi

substep “卸载分区…”
for mount_point in /tmp/esp_mount /tmp/system_mount /tmp/vendor_mount /tmp/data_mount; do
if mountpoint -q “$mount_point”; then
if ! umount “$mount_point”; then
step_error “卸载 $mount_point 失败”
fi
fi
done

substep “释放循环设备…”
if ! losetup -d “$LOOP_DEV”; then
step_error “释放循环设备失败”
fi

# 验证最终文件

if [ ! -f “$OUTPUT_IMG” ]; then
step_error “最终镜像文件不存在”
fi

final_size=$(stat -c%s “$OUTPUT_IMG” 2>/dev/null || echo “0”)
if [ “$final_size” -eq 0 ]; then
step_error “最终镜像文件大小为0”
fi

final_time=$(date +%s)
total_elapsed=$((final_time - START_TIME))
step_done “打包完成，总耗时: ${total_elapsed}秒”

echo “”
echo -e “${GREEN}🎉 =============================================${NC}”
echo -e “${GREEN}✅ 镜像文件创建完成: $OUTPUT_IMG${NC}”
echo -e “${GREEN}📁 文件大小: $(du -h “$OUTPUT_IMG” | cut -f1)${NC}”
echo -e “${GREEN}⏱️  总耗时: ${total_elapsed}秒${NC}”
echo -e “${GREEN}🎉 =============================================${NC}”
echo “”
echo -e “${BLUE}📋 分区信息:${NC}”
echo “   1. 🔧 ESP/Boot:  1GB  (FAT32) - 引导分区”
echo “   2. 💻 System:    5GB  (EXT4)  - 系统分区”  
echo “   3. 🏭 Vendor:    5GB  (EXT4)  - 厂商分区”
echo “   4. 💾 Data:      1GB  (EXT4)  - 数据分区”
echo “”
echo -e “${BLUE}🚀 使用方法:${NC}”
echo “   1. 刷写到USB设备:”
echo -e “      ${YELLOW}sudo pv $OUTPUT_IMG | dd of=/dev/sdX bs=4M${NC}”
echo “      (推荐使用pv显示刷写进度)”
echo “”
echo “   2. dd命令:”
echo -e “      ${YELLOW}sudo dd if=$OUTPUT_IMG of=/dev/sdX bs=4M status=progress${NC}”
echo “”
echo “   3. 虚拟机测试 (UEFI + KVM):”
echo -e “      ${YELLOW}qemu-system-x86_64 -enable-kvm -cpu host -m 2G \${NC}”
echo -e “      ${YELLOW}  -drive if=pflash,format=raw,readonly=on,file=/usr/share/ovmf/OVMF_CODE.fd \${NC}”
echo -e “      ${YELLOW}  -drive if=pflash,format=raw,file=/tmp/OVMF_VARS.fd \${NC}”
echo -e “      ${YELLOW}  -drive file=$OUTPUT_IMG,format=raw${NC}”
echo “”
echo “   4. 虚拟机测试 (传统BIOS):”
echo -e “      ${YELLOW}qemu-system-x86_64 -enable-kvm -cpu host -m 2G -drive file=$OUTPUT_IMG,format=raw${NC}”
echo “”
echo -e “${CYAN}💡 如果没有KVM支持，移除 -enable-kvm -cpu host 参数${NC}”
echo “”
echo -e “${RED}⚠️  注意: 刷写前请确认目标设备，避免数据丢失！${NC}”