# axdl-rs 非官方 Axera 镜像下载器的 Rust 实现

这是一个非官方的 Axera 镜像下载器 Rust 实现，用于将镜像文件烧录到 Axera SoC 中。

[English](./README.md) | [日本語](./README.ja.md) | [中文](./README.zh.md)

## 目录

- [准备工作](#准备工作)
- [安装](#安装)
- [Web 浏览器版](#web-浏览器版)
- [构建](#构建)
- [使用方法](#使用方法)
- [协议文档](#协议文档)
- [许可证](#许可证)

## 准备工作

### Linux (Debian 系)

为了让普通用户能够访问设备，需要配置 udev 规则。将 `99-axdl.rules` 复制到 `/etc/udev/rules.d` 并重新加载 udev 配置。

```
sudo cp 99-axdl.rules /etc/udev/rules.d/
sudo udevadm control --reload
```

如果当前用户不在 `plugdev` 组中，需要将其添加到该组并重新登录（组成员变更需要重新登录才能生效）。

```
id
# 确认输出中包含 ...,(plugdev),...
```

```
# 将用户添加到 plugdev 组
sudo usermod -a -G plugdev $USER
```

本工具依赖 libusb 和 libudev，请提前安装：

```
sudo apt install -y libudev-dev libusb-1.0-0-dev
```

## 安装

可以通过 `cargo install` 安装 `axdl-cli`：

```
cargo install axdl-cli
```

## Web 浏览器版

可以通过 [https://www.fugafuga.org/axdl-rs/axdl-gui/latest/](https://www.fugafuga.org/axdl-rs/axdl-gui/latest/) 在线运行 Web 浏览器版。

![axdl-gui](./doc/axdl-gui.drawio.svg)

1. 点击 `Open Image` 选择要烧录的 `.axp` 文件。
2. 如果不需要烧录 rootfs，勾选 `Exclude rootfs`。
3. 点击 `Open Device` 打开 USB 设备选择界面。
4. 将 Axera SoC 以下载模式连接到主机。（对于 M5Stack Module LLM，按住 BOOT 按钮的同时插入 USB 线缆。）
5. 在 Axera SoC 处于下载模式时，点击 `Download` 按钮。（设备约 10 秒后会退出下载模式，届时需要从步骤 3 重新操作。）

## 构建

构建项目前，请先通过 rustup 安装 Rust 工具链。

```bash
# 克隆仓库
git clone https://github.com/ciniml/axdl-rs.git

# 进入目录
cd axdl-rs
```

### 构建命令行版

```
# 构建
cargo build --bin axdl-cli --package axdl-cli
```

### 构建 Web 浏览器版

构建 Web 浏览器版需要安装 `wasm-pack`：

```
cargo install wasm-pack
```

然后使用 `wasm-pack` 进行构建：

```
cd axdl-gui
wasm-pack build --target web --release
```

## 使用方法

### 命令行版

要烧录 `*.axp` 镜像，执行以下命令并将 Axera SoC 设备以下载模式连接。
对于 M5Stack Module LLM，请按住 BOOT 按钮的同时将 USB 线缆插入设备。

```shell
cargo run --bin axdl-cli --package axdl-cli --release -- --file /path/to/image.axp --wait-for-device
```

如果不需要烧录 rootfs，可以指定 `--exclude-rootfs` 选项：

```shell
cargo run --bin axdl-cli --package axdl-cli --release -- --file /path/to/image.axp --wait-for-device --exclude-rootfs
```

在 Windows 或其他已安装 Axera 官方 AXDL 驱动的平台上，可以通过 `--transport serial` 选项使用串口访问：

```shell
cargo run --bin axdl-cli --package axdl-cli --release -- --file /path/to/image.axp --wait-for-device --transport serial
```

### Web 浏览器版

构建完成后，启动本地 HTTP 服务器并通过浏览器访问。需要支持 WebUSB 的浏览器（如 Chrome）。
以下是使用 Python HTTP 模块启动服务器的示例：

```
# 构建 Web 浏览器版
cd axdl-gui
wasm-pack build --target web --release
# 启动 HTTP 服务器
python -m http.server 8000
```

访问 [http://localhost:8000](http://localhost:8000) 即可打开 Web 浏览器版。

## 协议文档

烧录协议的详细分析文档请参阅 [doc/protocol.zh.md](./doc/protocol.zh.md)。

## 许可证

本项目基于 Apache License 2.0 许可证发布，详细信息请参阅 [LICENSE](LICENSE) 文件。
