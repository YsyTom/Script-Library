#!/bin/bash
# ==========================================
# NullClaw 安装脚本（全 Linux 发行版自适应 + 日志 + 重试 + 国内镜像）
# 支持 apt / dnf / yum / pacman / zypper / apk
# 自动编译 Zig，国内镜像加速
# ==========================================

set -euo pipefail

# ---------- 0. 日志与基础检测 ----------
LOG_DIR="$HOME"
mkdir -p "$LOG_DIR"
LOG_FILE="${LOG_DIR}/nullclaw_install_$(date +%Y%m%d_%H%M%S).log"
exec > >(tee -a "$LOG_FILE") 2>&1

trap 'ret=$?; echo "[错误] 脚本在第 $LINENO 行退出，状态码: $ret"' ERR

echo "========================================="
echo " NullClaw 安装日志开始"
echo " 时间: $(date '+%Y-%m-%d %H:%M:%S')"
echo " 日志: $LOG_FILE"
echo "========================================="

# ---------- 1. 检测 sudo ----------
if [ "$EUID" -ne 0 ]; then
    echo "⚠️ 建议使用 root 或确保用户有 sudo 权限"
fi

# ---------- 2. 检测包管理器 ----------
declare -A PKG_MAP=(
    ["apt"]="apt-get install -y"
    ["dnf"]="dnf install -y"
    ["yum"]="yum install -y"
    ["pacman"]="pacman -S --noconfirm"
    ["zypper"]="zypper install -y"
    ["apk"]="apk add --no-cache"
)
PM=""
for pm in apt dnf yum pacman zypper apk; do
    if command -v $pm &> /dev/null; then PM=$pm; break; fi
done
if [ -z "$PM" ]; then
    echo "❌ 未检测到支持的包管理器 (apt/dnf/yum/pacman/zypper/apk)"
    exit 1
fi
echo ">> 使用包管理器: $PM"
INSTALL_CMD="${PKG_MAP[$PM]}"

# ---------- 3. 安装依赖 ----------
declare -A PKG_LIST=(
    ["apt"]="cmake gcc g++ make ninja-build git wget curl tar xz-utils python3 llvm-dev clang libclang-dev lld libsqlite3-dev"
    ["dnf"]="cmake gcc gcc-c++ make ninja-build git wget curl tar xz python3 llvm-devel clang-devel lld sqlite-devel"
    ["yum"]="cmake gcc gcc-c++ make ninja-build git wget curl tar xz python3 llvm-devel clang-devel lld sqlite-devel"
    ["pacman"]="cmake gcc make ninja git wget curl tar xz python llvm clang lld sqlite"
    ["zypper"]="cmake gcc gcc-c++ make ninja git wget curl tar xz python3 llvm-devel clang-devel lld sqlite3-devel"
    ["apk"]="cmake gcc g++ make ninja git wget curl tar xz python3 llvm-dev clang-dev lld-dev sqlite-dev"
)

echo ">>> [1/5] 安装编译工具链与依赖..."
if [[ "$PM" == "apt" ]]; then sudo apt-get update; fi
for pkg in ${PKG_LIST[$PM]}; do
    echo "安装: $pkg"
    sudo $INSTALL_CMD $pkg || echo "⚠️ $pkg 安装失败，请手动检查"
done

# ---------- 4. 编译 Zig ----------
ZIG_VERSION="0.15.2"
ZIG_INSTALL_DIR="/usr/local/zig"
BUILD_DIR=$(mktemp -d)
trap 'rm -rf "$BUILD_DIR"' EXIT
cd "$BUILD_DIR"

PROXY_DOWNLOAD="https://ghproxy.com/"

echo ">>> [2/5] 编译 Zig ${ZIG_VERSION}..."
wget -q --tries=5 --timeout=30 -O zig-src.tar.xz "${PROXY_DOWNLOAD}https://ziglang.org/builds/zig-${ZIG_VERSION}.tar.xz"
tar -xJf zig-src.tar.xz
ZIG_SRC_DIR=$(find . -maxdepth 1 -type d -name "zig-${ZIG_VERSION}*" | head -n1)
mkdir -p build && cd build

LLVM_CONFIG_PATH=""
if command -v llvm-config &> /dev/null; then LLVM_CONFIG_PATH=$(command -v llvm-config); fi

cmake "../$ZIG_SRC_DIR" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX="$ZIG_INSTALL_DIR" \
    $( [[ -n $LLVM_CONFIG_PATH ]] && echo "-DLLVM_CONFIG_EXECUTABLE=$LLVM_CONFIG_PATH" ) \
    -DZIG_PREFER_CLANG_CPP_DYLIB=ON

make -j"$(nproc)" install

# 备份原 zig
[ -f /usr/local/bin/zig ] && sudo mv /usr/local/bin/zig /usr/local/bin/zig.bak
sudo ln -sf "$ZIG_INSTALL_DIR/bin/zig" /usr/local/bin/zig
hash -r
zig version

# ---------- 5. 编译 NullClaw ----------
echo ">>> [3/5] 编译 NullClaw..."
cd "$HOME"
[ -d nullclaw ] && mv nullclaw nullclaw.bak
git clone --depth 1 "https://gitclone.com/github.com/nullclaw/nullclaw.git"
cd nullclaw
zig build -Doptimize=ReleaseSmall
sudo cp -f ./zig-out/bin/nullclaw /usr/local/bin/nullclaw
sudo chmod +x /usr/local/bin/nullclaw

# ---------- 6. 验证 ----------
echo ">>> [4/5] 验证安装..."
if command -v nullclaw &> /dev/null; then
    echo "✅ NullClaw 安装成功！"
    nullclaw --version || echo "⚠️ 安装成功但执行可能有问题"
else
    echo "❌ 未找到 nullclaw 命令"
    exit 1
fi

echo "常用命令："
echo "  nullclaw onboard --interactive"
echo "  nullclaw gateway --port 3000"
echo "  nullclaw status"

echo "========================================="
echo " 安装完成: $(date '+%Y-%m-%d %H:%M:%S')"
echo " 日志: $LOG_FILE"
echo "========================================="
