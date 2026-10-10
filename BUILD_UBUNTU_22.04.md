# FreeSWITCH 在 Ubuntu 22.04 上的源码编译安装指南

本指南记录了在 Ubuntu 22.04 (jammy) x86_64 上从源码编译安装 FreeSWITCH master
（1.11.4-dev）的完整可复现过程，已在真实环境验证通过。

## 为什么需要源码编译

| 方案 | 不可行原因 |
|------|-----------|
| SignalWire 官方 apt 仓库 | 只提供 Debian 发行版（bookworm/bullseye），没有 Ubuntu jammy 版本；bookworm 包的依赖（libjpeg62-turbo、libavcodec59、libperl5.36 等）在 jammy 上不存在 |
| 官方 Docker 镜像 | 无公开可用镜像，需要 SignalWire 认证 |
| Ubuntu 自带 apt 包 | 版本过旧（1.10.7，2021 年） |

因此采用源码编译。核心难点是 4 个依赖库在 Ubuntu 上版本不足，必须自行从源码编译。

## 环境要求

- Ubuntu 22.04 (jammy)，x86_64
- 建议 8 核以上 CPU、16GB 以上内存、30GB 以上可用磁盘
- 可访问 GitHub 的网络

## 第 1 步：安装基础依赖

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential autoconf automake libtool libtool-bin pkg-config cmake \
    git wget gawk zip unzip \
    libncurses-dev libssl-dev uuid-dev libsqlite3-dev libcurl4-openssl-dev \
    libpcre2-dev libxml2-dev zlib1g-dev libjpeg-dev libtiff-dev libpng-dev \
    libfreetype-dev libopus-dev libogg-dev libvorbis-dev libpq-dev \
    libspeex-dev libspeexdsp-dev libldns-dev libedit-dev liblua5.3-dev \
    libsndfile1-dev libavformat-dev libswscale-dev libavutil-dev yasm
```

> 说明：本机大部分基础包已预装，实际新装的是
> `libspeex-dev libspeexdsp-dev libldns-dev libedit-dev liblua5.3-dev libsndfile1-dev`。
> 以上清单为完整集合，供全新机器使用。可选模块（libvlc、ImageMagick、codec2 等）
> 缺失时 configure 会自动跳过，不影响主构建。

## 第 2 步：从源码编译 4 个依赖库

以下 4 个库版本要求超出 Ubuntu jammy 自带版本，必须自编。建议统一放在一个
目录下（本例为 `/xxx/signalwire/`），安装到系统默认前缀（`/usr` 或 `/usr/local`），
pkg-config 默认搜索路径可直接找到，无需额外设置 `PKG_CONFIG_PATH`。

### 2.1 sofia-sip（要求 >= 1.13.18）

```bash
git clone --depth 1 https://github.com/freeswitch/sofia-sip.git
cd sofia-sip
./bootstrap.sh
./configure
make -j$(nproc)
sudo make install
sudo ldconfig
pkg-config --modversion sofia-sip-ua   # 验证：应输出 >= 1.13.18
```

### 2.2 spandsp（要求 >= 3.1.1）

```bash
git clone https://github.com/freeswitch/spandsp.git
cd spandsp
git checkout v3.1.1    # 必须！master 分支的 AC_INIT 版本号仍是 3.0.0，不满足要求
./bootstrap.sh
./configure
make -j$(nproc)
sudo make install
sudo ldconfig
pkg-config --modversion spandsp   # 验证：应输出 3.1.1
```

### 2.3 libks（要求 libks2 >= 2.0.11）

```bash
git clone https://github.com/signalwire/libks.git
cd libks
git fetch --unshallow --tags   # 必须！浅克隆无 git 标签，cmake 生成 changelog 时会报错
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF .
make -j$(nproc)
sudo make install
sudo ldconfig
pkg-config --modversion libks2   # 验证：应输出 >= 2.0.11
```

### 2.4 signalwire-client-c（要求 signalwire_client2 >= 2.0.0）

```bash
git clone https://github.com/signalwire/signalwire-c.git   # 注意：仓库名是 signalwire-c
cd signalwire-c
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF .
make -j$(nproc)
sudo make install
sudo ldconfig
pkg-config --modversion signalwire_client2   # 验证：应输出 >= 2.0.0
```

## 第 3 步：编译 FreeSWITCH

```bash
cd /path/to/freeswitch        # FreeSWITCH 源码仓库（本例为 git clone 的 master）
./configure
make -j$(nproc)
sudo make install             # 默认安装到 /usr/local/freeswitch
```

configure 成功后会输出配置汇总，安装目录默认为 `/usr/local/freeswitch`。
完整构建在 12 核机器上约需 5~15 分钟。

## 第 4 步：启动与验证

FreeSWITCH 需要 root 权限运行（SCHED_FIFO 实时调度、pid 文件写入）：

```bash
# 后台启动
sudo /usr/local/freeswitch/bin/freeswitch -nc -nonat

# 验证状态（应显示 UP）
/usr/local/freeswitch/bin/fs_cli -x status

# 查看启动日志（应包含 "FreeSWITCH is ready"）
sudo grep "FreeSWITCH is ready" /usr/local/freeswitch/log/freeswitch.log

# 交互式控制台
/usr/local/freeswitch/bin/fs_cli

# 停止
sudo /usr/local/freeswitch/bin/freeswitch -stop
```

## 常见报错对照表

| configure/make 报错 | 原因 | 解决方法 |
|---------------------|------|---------|
| `no usable spandsp; please install spandsp4 devel package` | jammy 自带 spandsp 0.0.6 太旧 | 自编 spandsp v3.1.1（见 2.2） |
| `no usable sofia-sip` | jammy 自带 1.12.11 < 1.13.18 | 自编 sofia-sip（见 2.1） |
| `Package speex was not found` | 缺 speex 开发包 | `sudo apt-get install -y libspeex-dev libspeexdsp-dev` |
| `You need to either install libldns-dev` | 缺 ldns 开发包 | `sudo apt-get install -y libldns-dev` |
| `You need to either install libks2 or libks` | libks 未装 | 自编 libks（见 2.3） |
| `You need to either install signalwire-client-c2` | signalwire-c 未装 | 自编 signalwire-c（见 2.4） |
| `You need to either install libedit-dev (>= 2.11)` | 缺 libedit | `sudo apt-get install -y libedit-dev` |
| make 报 `fatal error: lua.h` | 缺 Lua 开发包 | `sudo apt-get install -y liblua5.3-dev` 后**重新 ./configure** |
| make 报 `You must install libsndfile-dev to build mod_sndfile` | 缺 libsndfile | `sudo apt-get install -y libsndfile1-dev` 后**重新 ./configure** |
| libks cmake 报 `fatal: 选项 '--count' 必须在其他非选项参数之前` | 浅克隆无 git 标签 | `git fetch --unshallow --tags` 后重跑 cmake |
| 启动报 `Failed to set SCHED_FIFO scheduler` / `Cannot open pid file` | 非 root 运行 | 用 `sudo` 启动 |

## 可选后续

- **安装声音文件**（英文提示音，约 30MB）：在源码目录运行 `make sounds-install`
- **专用用户 + systemd 服务**（生产环境建议，避免以 root 长期运行）：
  创建 `freeswitch` 用户，`chown -R freeswitch:freeswitch /usr/local/freeswitch`，
  编写 systemd unit（可参考 `debian/freeswitch-systemd.freeswitch.service`）
- **定制模块**：编辑 `modules.conf` 增删模块后重新 `make && sudo make install`

## 安全注意事项

- SignalWire 仓库脚本（fs.sh）中的 PAT token 为明文敏感信息，使用后建议在
  SignalWire 控制台撤销；如曾配置过 apt 认证（`/etc/apt/auth.conf.d/`），
  注意清理。
