# dji-4g-vohive-mac

> 在 Mac（Apple Silicon / Intel 通用）上，用 **UTM** 跑一个 Linux 虚拟机，把**大疆 4G 模块（1 代，本质移远 Quectel EG25-G）**的 USB 身份从大疆私有 `2ca3:4006` **永久改成移远 Quectel EC25 的 `2C7C:0125`**，并在该 Linux 里一键部署 **vohive** 短信/网络/eSIM 管理平台的全套步骤。

## 视频教程

[![大疆 4G 模块在 Mac 上部署 VoHive 视频教程](https://img.youtube.com/vi/PZRkoggXFco/hqdefault.jpg)](https://youtu.be/PZRkoggXFco)

点击上方缩略图观看 YouTube 视频教程。

## 这个仓库做什么

- 给 Mac 用户提供一条**从零到能访问 vohive 后台**的可执行路径，无需另一台 Linux 真机。
- 解决大疆 4G 模块默认 VID/PID 是大疆私有、通用驱动不认的问题——通过发 AT 指令 `AT+QCFG="usbcfg",...` 把模块内部 USB 身份永久改写为移远 EC25，改一次终身有效。
- 同时覆盖 **Apple Silicon（arm64）** 和 **Intel（x86_64）** 两种 Mac：两者只有 ISO 和 VM 架构不同，VM 内所有操作完全一致。
- 包含 USB 直通、改身份后重新枚举断直通的坑及处理、验证清单、维护命令、方案选型对比。

## VoHive 是什么

> 你的副机未必需要是把手机，可以是 VoHive。短信转发 / eSIM管理 / VoWiFi

VoHive 是一个面向移远 4G 模组的管理平台，适合拿移远 EC20 这类 USB 模组做：

- 网页/Bot 收发短信
- 多卡统一管理
- 实体 eSIM / eUICC 管理（加卡、切卡、删卡）
- 基于手机卡流量的代理池（SOCKS5 / HTTP，按设备网卡强绑定出站）
- TelegramBot / 飞书Bot / QQBot 远程控制
- 在条件满足时启用 VoWiFi
- 通过 `/vocall` 发起 VoWiFi 模拟外呼

`/vocall` 这个功能比较适合：
- CTE UK / CMLINK UK 这类需要本地拨打运营商电话才能激活的号码
- 其他需要定期拨打一通电话做保号的海外号码

**VoHive 完全免费。**

### 一、适用环境

**硬件推荐：**
- EC20CEFAG
- EC20CEFHLG
- 可以小黄鱼几十块买到
- 要求：设备具备 SIM 卡槽，或搭配带 SIM 卡槽的 USB 底板

**系统建议：**
- Debian / Ubuntu
- 树莓派
- NAS

### 二、部署前先禁用宿主机 ModemManager

这一步很重要。很多发行版会默认启动 ModemManager，它会抢占 `/dev/ttyUSB*` AT 端口，导致模组识别、短信、AT 口访问异常。

```bash
# 检查状态
systemctl status ModemManager

# 如果在运行，直接禁用
sudo systemctl stop ModemManager
sudo systemctl disable ModemManager
sudo systemctl mask ModemManager

# 再次确认
systemctl status ModemManager
```

> 即使后面使用 Docker，这一步也必须在宿主机上做。

### 三、可选：把模组切到更合适的 USBNET 模式

如果你确认模组当前模式不对，可以执行：

```bash
sudo apt update
sudo apt install -y socat

echo 'AT+QCFG="usbnet",0;+CFUN=1,1' | sudo socat - /dev/ttyUSB2,crnl
```

- `AT+QCFG="usbnet",0`：切到常见的 QMI 模式
- `AT+CFUN=1,1`：重启模组
- `/dev/ttyUSB2` 只是示例，实际 AT 口请按你的设备调整

### 四、部署方式一：一键安装

```bash
curl -fsSL https://raw.githubusercontent.com/iniwex5/vohive-release/master/install.sh | bash
```

指定版本：

```bash
curl -fsSL https://raw.githubusercontent.com/iniwex5/vohive-release/master/install.sh | bash -s -- --version v1.0.0
```

仅安装二进制（不安装 systemd）：

```bash
curl -fsSL https://raw.githubusercontent.com/iniwex5/vohive-release/master/install.sh | bash -s -- --no-systemd
```

卸载：

```bash
curl -fsSL https://raw.githubusercontent.com/iniwex5/vohive-release/master/uninstall.sh | bash
```

默认安装目录（便携部署）：
- 二进制：`/opt/vohive/bin/vohive`
- 配置：`/opt/vohive/config/config.yaml`
- 数据：`/opt/vohive/data`
- 日志：`/opt/vohive/logs`

### 五、部署方式二：Docker / Docker Compose

```bash
# 1. 创建目录
mkdir -p vohive/{config,data,logs}
cd vohive
```

创建 `config/config.yaml`：

```yaml
server:
  port: 7575
  debug: false

web:
  username: admin
  password: admin123
```

创建 `docker-compose.yml`：

```yaml
services:
  vohive:
    image: iniwex/vohive:latest
    container_name: vohive
    restart: unless-stopped
    network_mode: host
    privileged: true
    volumes:
      - ./config:/app/config
      - ./data:/app/data
      - ./logs:/app/logs
    environment:
      - TZ=Asia/Shanghai
      - CONFIG_PATH=/app/config/config.yaml
    devices:
      - /dev:/dev
```

启动：

```bash
docker compose up -d
```

访问后台：`http://你的服务器IP:7575`

> Docker 部署也要先禁用宿主机 ModemManager。这里用了 privileged、`/dev` 透传和 host network，因为程序需要直接接管模组设备。

### 六、机器人常用命令

| 命令 | 说明 |
|---|---|
| `/list` | 查看设备列表 |
| `/sms 设备ID` | 查看最近短信 |
| `/send 设备ID 号码 内容` | 发送短信 |
| `/rotate 设备ID` | 切换 IP |
| `/esim 设备ID` | 查看 eSIM profile |
| `/switch 设备ID 序号或ICCID` | 切换 eSIM profile |
| `/vocall 设备ID 号码` | 发起 VoWiFi 模拟呼叫 |

### 七、补充说明

- VoWiFi 不是只要有网就一定能用，还取决于运营商、号码状态和网络环境要求
- 如果你的需求只是短信、代理池、多模组管理，不折腾 VoWiFi 也可以先用起来
- 本程序已禁止国内运营商卡发起 VoWiFi，请遵纪守法
- 未标出的运营商不代表不兼容，只是作者没有测试

### 八、已知 VoHive 支持 VoWiFi 的运营商

| 运营商 | 国家/地区 |
|---|---|
| CTE UK | 英国 |
| CMLINK UK | 英国 |
| giffgaff UK | 英国 |
| VOXI UK | 英国 |
| Vodafone UK | 英国 |
| 3UK | 英国 |
| Vodafone DE | 德国 |
| Telekom DE | 德国 |
| O2 DE | 德国 |
| T-Mobile US | 美国 |

---

## 项目依赖

本仓库本身只是一份操作手册（README），实际起作用的是上游 **[iniwex5/vohive-release](https://github.com/iniwex5/vohive-release)** 的发布资产。上游仓库仍在，但其最新 release `v1.5.5` **已无可下载的二进制 asset**（`vohive_v1.5.5_linux_<arch>` 实测 HTTP 404，release 的 assets 列表为空），在线安装脚本会在「下载二进制」那步失败。为此本仓库内置了两份可离线使用的资产：

| 内置包 | 内容 | 适用架构 | 是否联网 |
|---|---|---|---|
| `vohive-release-1.5.5.zip` | 在线安装脚本，运行时按架构到上游 release 拉取二进制 | arm64 + amd64 自动检测 | ❗ 上游二进制已 404，**当前会失败**，留作上游修复后使用 |
| `vohive-backup.tar.gz` | **离线恢复包**：内置 vohive 二进制（sha1 `ee16a5c0cd04505df43805fc81838f3e20b16aee`，与 backup `install.sh` 注释中记录的原版 sha1 一致）+ `install.sh` + `vohive.service` + `mcc-mnc-table.json` | **x86_64（Intel / 方案 B）** | ✅ 完全离线，**当前推荐路径** |

> ⚠️ 上游二进制 404 后，**Apple Silicon（方案 A，arm64）暂无内置离线二进制**：可继续试 `vohive-release-1.5.5.zip` 在线方式（等上游修复 asset），或自行备一份 `vohive_<ver>_linux_arm64` 后参照 `vohive-backup.tar.gz` 里的 `install.sh` 离线安装。Intel Mac（方案 B）直接用 `vohive-backup.tar.gz` 即可全程离线部署。

### 上游项目

详见上方「[VoHive 是什么](#vohive-是什么)」章节，包含完整的功能介绍、部署方式、机器人命令和 VoWiFi 运营商列表。

> ⚠️ 上游 v1.5.5 二进制已 404，本仓库内置离线资产作为替代方案。

---

## 完整步骤

> 来源教程：<https://linux.do/t/topic/2486016>（标题：《大疆4G模块修改设备ID并一键部署vohive平台教程》）
>
> 本机为 M3 Pro，对应方案 A。两种芯片先看第 0.5 节选方案。

### 0. 为什么这么做（背景与约束）

- 大疆 4G 模块（1 代，约 30~40 元）**本质是移远 Quectel EG25-G**，但默认 USB VID/PID 是大疆私有的 `2ca3:4006`，通用驱动不认。
- 教程通过发 AT 指令把模块内部 USB 身份**永久改写**成移远 EC25 的 `2C7C:0125`，从而接入 vohive 平台。
- 教程全部命令是 **Linux 专用**（`modprobe option`、`/sys/bus/usb-serial/.../new_id`、`/dev/ttyUSB2`、`apt-get`、`lsusb`、systemd 服务），macOS 没有等价物，不能直接在 Mac 上照搬。
- vohive 官方 `install.sh` **只支持 Linux**（`os != linux` 直接退出），但**支持 arm64**（会下载 `vohive_<ver>_linux_arm64`，systemd 起服务）。所以在 M3 上跑 **arm64 Linux 虚拟机是原生速度**，不用 x86 模拟。
- **硬约束是 USB passthrough**：必须把大疆这个 USB 设备直通进 Linux VM。这排除了 OrbStack、Multipass、Docker Desktop（都不支持任意 USB 设备直通）。免费且支持 USB 直通的最佳选择是 **UTM**（基于 QEMU）。
- 改身份这步是**一次性的、改的是模块内部 NV，改一次终身有效**；改完模块插任何机器都是 Quectel EC25 身份。

### 0.5. 架构选择（按你的 Mac 芯片二选一）

先在 Mac 终端确认芯片：
```bash
uname -m
# 输出 arm64  → Apple Silicon，选 A
# 输出 x86_64 → Intel，选 B
```

| 项 | 方案 A：Apple Silicon | 方案 B：Intel Mac |
|---|---|---|
| VM 架构 | `aarch64`（原生 ARM 虚拟化） | `x86_64`（原生虚拟化，Hypervisor.framework） |
| Ubuntu ISO 文件名 | `ubuntu-24.04-live-server-arm64.iso` | `ubuntu-24.04-live-server-amd64.iso` |
| ISO 下载地址 | `https://cdimage.ubuntu.com/releases/24.04/release/ubuntu-24.04-live-server-arm64.iso` | `https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso` |
| vohive 二进制 | 脚本自动下 `vohive_<ver>_linux_arm64` | 脚本自动下 `vohive_<ver>_linux_amd64` |
| UTM 虚拟化方式 | Virtualize（不要选 Emulate） | Virtualize |

> 后面第 2、3 步按你选的方案 A/B 取对应值；**第 4 步起（USB 直通、改身份、装 vohive、验证）两种方案完全一致，没有任何差别。**

### 1. 装 UTM（免费）

UTM 是 macOS 上基于 QEMU 的虚拟化前端，原生支持 Apple Silicon 的 ARM 虚拟化与 USB 设备直通。

```bash
brew install --cask utm
```

打开一次 UTM，确认能启动（macOS 可能要求在「系统设置 → 隐私与安全性」里允许运行）。

### 2. 下载 Ubuntu Server 24.04 ISO（按第 0.5 节选的方案）

**方案 A（Apple Silicon / arm64）：**
```bash
curl -L -o ~/Downloads/ubuntu-24.04-live-server-arm64.iso \
  https://cdimage.ubuntu.com/releases/24.04/release/ubuntu-24.04-live-server-arm64.iso
```

**方案 B（Intel / amd64）：**
```bash
curl -L -o ~/Downloads/ubuntu-24.04-live-server-amd64.iso \
  https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso
```

下载完成后 `ls -lh ~/Downloads/ubuntu-24.04-live-server-*.iso` 确认大小约 2GB+。

### 3. 在 UTM 里创建并安装 Linux 虚拟机

1. 打开 UTM → 顶部 **＋** 新建虚拟机。
2. 选择 **Virtualize**（不要选 Emulate）——原生虚拟化，性能接近原生。
3. 架构按第 0.5 节选：**方案 A 选 `aarch64`**，**方案 B 选 `x86_64`**。
4. 系统类型选 **Debian/Ubuntu**。
5. 内存 **2 GB**、CPU **2 核**、磁盘 **20 GB**（跑 vohive 足够）。
6. 「CD/DVD」挂载第 2 步下载的 ISO。
7. 网络保留默认（NAT，UTM 会给 VM 分一个 192.168.x.x 的 DHCP 地址）。
8. 启动 VM，按 Ubuntu 安装流程走：
   - 语言/键盘默认即可
   - 安装类型选 Ubuntu Server（最小化，无桌面）
   - **务必勾选安装 OpenSSH server**（方便从 Mac 终端 ssh 进去操作）
   - 用户名/密码自行设置，记下来
9. 装完重启，拔掉 ISO（UTM 里弹出 CD）。

#### 从 Mac ssh 进 VM（推荐，后面命令都在这里跑）

在 VM 控制台里先看 IP：
```bash
ip a
```
拿到 `192.168.x.x` 后，在 Mac 终端：
```bash
ssh <ubuntu用户名>@192.168.x.x
```
后续所有命令都在这个 ssh 会话里执行。

### 4. 把大疆 4G 模块直通进 VM

1. 把大疆 4G 模块 USB 插到 Mac。
2. UTM → 选中该 VM → 设置 → **USB** 选项卡 → 勾选大疆设备（显示为 VID:PID `2ca3:4006`）做 passthrough。
3. 回到 VM（或 ssh），确认设备已进来：
```bash
lsusb
# 应能看到一行含 2ca3:4006
```
> 若 VM 里没装 `lsusb`：`sudo apt-get install usbutils -y`

### 5. 改大疆模块设备 ID（改成移远 EC25 身份）

在 VM 里依次执行（教程原样）：

```bash
# 0. 装 socat（发 AT 指令用）
sudo apt-get update && sudo apt-get install socat -y

# 1. 临时加载 option 驱动模块
sudo modprobe option

# 2. 把大疆当前识别码 2ca3:4006 写入 option 驱动，生成串口文件
echo 2ca3 4006 | sudo tee /sys/bus/usb-serial/drivers/option1/new_id

# 3. 通过 /dev/ttyUSB2 发 AT 指令，永久改 USB 身份为移远 2C7C:0125
echo 'AT+QCFG="usbcfg",0x2C7C,0x0125,1,1,1,1,1,0,0' | socat - /dev/ttyUSB2,crnl

# 4. 软重启模块使配置生效
echo 'AT+CFUN=1,1' | socat - /dev/ttyUSB2,crnl
```

等几秒，模块重新初始化后查看：
```bash
lsusb
# 应显示：2c7c:0125 Quectel Wireless Solutions Co., Ltd. EC25 LTE modem
```

#### ⚠️ 关键坑：USB 重新枚举会断开直通

`AT+CFUN=1,1` 让模块软重启，VID/PID 从 `2ca3:4006` 变成 `2c7c:0125`。如果 UTM 是按 VID/PID 绑定直通的，这一瞬间直通会断开，`lsusb` 在 VM 里可能短暂看不到设备。

处理方式：
- 在 UTM 里把直通规则改成绑定到新的 Quectel 设备 `2c7c:0125`，或绑定到**物理 USB 端口**（更稳，重新枚举不会丢）。
- 重新勾选一次直通，模块就永久留在 VM 里给 vohive 用。
- 改身份是一次性的；改完后这个模块插任何机器都是 Quectel EC25 身份。

### 6. 一键部署 vohive 平台

模块身份改完且直通稳定后，在 VM 里部署 vohive。两种方式选其一：

#### 方式一（推荐·离线）：内置 `vohive-backup.tar.gz`（仅 x86_64 / 方案 B）

适合 Intel Mac 建的 amd64 VM，**全程不联网**，规避上游二进制 404：

```bash
sudo apt-get update && sudo apt-get install -y wget
wget -O vohive-backup.tar.gz \
  https://raw.githubusercontent.com/wlzh/dji-4g-vohive-mac/main/vohive-backup.tar.gz
tar -xzf vohive-backup.tar.gz
cd vohive-backup
sudo bash install.sh
```
脚本会校验架构、把内置二进制 / 运营商表 / 默认配置 / systemd 单元一并部署到位，**不再联网下载二进制**。

#### 方式二（在线）：`vohive-release-1.5.5.zip`（arm64 / amd64 自动检测）

```bash
sudo apt-get update && sudo apt-get install -y unzip
curl -L -o vohive-release-1.5.5.zip \
  https://raw.githubusercontent.com/wlzh/dji-4g-vohive-mac/main/vohive-release-1.5.5.zip
unzip -o vohive-release-1.5.5.zip
cd vohive-release-1.5.5
bash install.sh
```
> ⚠️ 此方式在「下载二进制」那步依赖上游 `iniwex5/vohive-release` 的 release asset；**上游 v1.5.5 二进制已 404，当前大概率失败**，此时方案 B 请改用方式一，方案 A 需自备 arm64 二进制。

#### 部署结果（两方式一致）

- 二进制 → `/opt/vohive/bin/vohive`
- 配置 → `/opt/vohive/config/config.yaml`（默认 Web 账号 `admin / admin`）
- 注册 systemd 服务 `vohive.service` 并启动
- 数据/日志 → `/opt/vohive/data`、`/opt/vohive/logs`

#### 访问后台

从 Mac 浏览器打开：
```
http://<VM的IP>:7575
```
默认 `admin / admin`，**登录后立即改密码**。

### 7. 验证清单

- [ ] VM 里 `lsusb` 能看到 `2c7c:0125 Quectel ... EC25 LTE modem`
- [ ] VM 里 `systemctl status vohive` 显示 active (running)
- [ ] Mac 浏览器访问 `http://<VM-IP>:7575` 能出登录页
- [ ] 用 `admin/admin` 登录成功并改密
- [ ] vohive 后台能识别到 4G 模块、看到信号/短信等功能

### 8. 维护与可选操作

#### 更新 vohive
```bash
curl -L -o vohive-release-1.5.5.zip \
  https://raw.githubusercontent.com/wlzh/dji-4g-vohive-mac/main/vohive-release-1.5.5.zip
unzip -o vohive-release-1.5.5.zip
cd vohive-release-1.5.5
bash install.sh
```
脚本会自动备份旧二进制到 `/opt/vohive/bin/vohive.bak` 再覆盖。

#### 卸载 vohive
```bash
curl -L -o vohive-release-1.5.5.zip \
  https://raw.githubusercontent.com/wlzh/dji-4g-vohive-mac/main/vohive-release-1.5.5.zip
unzip -o vohive-release-1.5.5.zip
cd vohive-release-1.5.5
bash uninstall.sh
```

#### 不用 systemd 的环境（如容器/WSL，本方案用不到）
```bash
curl -L -o vohive-release-1.5.5.zip \
  https://raw.githubusercontent.com/wlzh/dji-4g-vohive-mac/main/vohive-release-1.5.5.zip
unzip -o vohive-release-1.5.5.zip
cd vohive-release-1.5.5
bash install.sh --no-systemd
# 手动启动：/opt/vohive/bin/vohive -c /opt/vohive/config/config.yaml
```

#### 查看日志
```bash
journalctl -u vohive -f
# 或
tail -f /opt/vohive/logs/*.log
```

#### 让 Mac 开机后自动连 VM
- UTM 设置里可勾选该 VM「开机自动启动」。
- Mac 这边可加一条 ssh config，方便 `ssh vohive` 直连。

#### 如果想把模块改回大疆身份（基本不需要）
把第 5 步的 AT 指令 VID/PID 换回原值即可：
```bash
echo 'AT+QCFG="usbcfg",0x2CA3,0x4006,1,1,1,1,1,0,0' | socat - /dev/ttyUSB2,crnl
echo 'AT+CFUN=1,1' | socat - /dev/ttyUSB2,crnl
```

### 9. 方案选型对比（为什么选 UTM）

| 方案 | 能跑对应架构 Linux | USB 直通 | 备注 |
|---|---|---|---|
| **UTM** ✅ | arm64 + amd64 均可（原生 Virtualize） | 支持 | 免费，本方案首选 |
| Parallels / VMware Fusion | 是 | 支持 | 付费（Fusion Pro 个人版免费），更省心但非必要 |
| OrbStack | 是 | ❌ 不支持 | 轻量但无 USB 直通，排除 |
| Multipass | 是 | ❌ 不支持 | 无 USB 直通，排除 |
| Docker Desktop | 是 | ❌ 困难 | 任意 USB 直通很痛，排除 |

### 10. 速查：从零到能访问 vohive 的最短路径

```bash
# Mac 上：先 uname -m 确认芯片，选方案 A(arm64) 或 B(amd64)
brew install --cask utm

# 方案 A（Apple Silicon）：
curl -L -o ~/Downloads/ubuntu-24.04-live-server-arm64.iso \
  https://cdimage.ubuntu.com/releases/24.04/release/ubuntu-24.04-live-server-arm64.iso
# 方案 B（Intel）：
curl -L -o ~/Downloads/ubuntu-24.04-live-server-amd64.iso \
  https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# → UTM 图形界面建对应架构的 Ubuntu Server VM（Virtualize），装 OpenSSH
# → UTM USB 选项卡勾选大疆 2ca3:4006 直通
# → ssh 进 VM

# VM 里：
sudo apt-get update && sudo apt-get install -y socat usbutils
sudo modprobe option
echo 2ca3 4006 | sudo tee /sys/bus/usb-serial/drivers/option1/new_id
echo 'AT+QCFG="usbcfg",0x2C7C,0x0125,1,1,1,1,1,0,0' | socat - /dev/ttyUSB2,crnl
echo 'AT+CFUN=1,1' | socat - /dev/ttyUSB2,crnl
lsusb   # → 2c7c:0125 Quectel EC25
# → UTM 把直通重新绑到 2c7c:0125 / 物理端口
# 部署 vohive（二选一）：
#  · 方案 B(Intel/amd64，离线推荐)：下 vohive-backup.tar.gz → tar -xzf → sudo bash install.sh
#  · 方案 A(Apple Silicon/arm64，在线)：下 vohive-release-1.5.5.zip → unzip → bash install.sh（上游二进制 404 时会失败）
wget -O vohive-backup.tar.gz \
  https://raw.githubusercontent.com/wlzh/dji-4g-vohive-mac/main/vohive-backup.tar.gz
tar -xzf vohive-backup.tar.gz
cd vohive-backup
sudo bash install.sh
# → Mac 浏览器开 http://<VM-IP>:7575，admin/admin
```

---

## 致谢

- 上游项目：[iniwex5/vohive-release](https://github.com/iniwex5/vohive-release)（VoHive 平台与一键安装脚本）
- 原始教程：<https://linux.do/t/topic/2486016>（iniwex 发布的大疆 4G 模块改 ID + 部署 vohive 教程）

## License

本仓库仅含文档，按 CC-BY-4.0 分享。VoHive 二进制与脚本的上游许可请见 [iniwex5/vohive-release](https://github.com/iniwex5/vohive-release)。
