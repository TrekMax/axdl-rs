# AXDL 烧录协议文档

> 基于 axdl-rs 代码分析，非官方逆向工程文档。

## 1. 概述

AXDL (Axera Download) 是 Axera（爱芯元智）SoC 芯片的固件烧录协议。该协议通过 USB 或串口与处于 ROM Boot 模式的设备通信，将固件镜像文件（`.axp` 格式）写入 Flash 存储器。

### 1.1 支持的芯片

- AX630C / AX620E 系列（双级 FDL）
- AX650N 系列（单级 FDL）

### 1.2 传输层

| 传输方式 | USB VID | USB PID | 参数 |
|---------|---------|---------|------|
| USB (libusb) | `0x32C9` | `0x1000` | Bulk EP OUT: `0x01`, EP IN: `0x81` |
| 串口 (Serial) | `0x32C9` | `0x1000` | 波特率: 115200 |
| WebUSB (浏览器) | `0x32C9` | `0x1000` | Bulk EP OUT: `0x01`, EP IN: `0x01` |
| WebSerial (浏览器) | `0x32C9` | `0x1000` | 波特率: 115200, 缓冲区: 48000 字节 |

---

## 2. 帧格式

所有 AXDL 协议消息（握手请求除外）均使用以下帧格式：

```
偏移   大小    字段            说明
─────────────────────────────────────────────────────
0x00   4字节   Signature      帧签名，固定值 0x5C6D8E9F（小端序）
0x04   2字节   Length         有效载荷长度（小端序 u16）
0x06   2字节   Command        命令码/响应码（小端序 u16）
0x08   N字节   Payload        有效载荷（变长，长度由 Length 字段指定）
0x08+N 2字节   Checksum       校验和（小端序 u16）
```

**最小帧长度**：10 字节（Signature 4 + Length 2 + Command 2 + Checksum 2，无有效载荷）

### 2.1 帧结构图

```
+----------+--------+---------+------------------+----------+
| 9F 8E 6D | Length | Command |     Payload      | Checksum |
| 5C       | (u16)  | (u16)   |   (Length bytes)  | (u16)    |
+----------+--------+---------+------------------+----------+
  4 bytes   2 bytes  2 bytes    variable length    2 bytes
```

### 2.2 校验和算法

采用 **反码求和（One's Complement Addition）** 算法：

1. 将 Checksum 字段初始值设为自身
2. 依次将以下字段以 u16 小端序累加：
   - Length 字段
   - Command 字段
   - Payload 中的每对字节（每 2 字节视为一个 u16 小端序）
   - 若 Payload 长度为奇数，最后一字节与 `0x00` 组成 u16
3. 累加过程中，若结果超过 `0xFFFF`，则将高位溢出部分加回低 16 位
4. 验证时：所有字段（含 Checksum 自身）累加结果应等于 `0xFFFF`

**构造校验和**：先将 Checksum 字段置为 `0x0000`，计算累加值后，取其按位取反（`!checksum`）作为最终值。

#### 校验和计算示例

```
帧数据: 9F 8E 6D 5C 00 00 01 00 FE FF
  Length   = 0x0000
  Command  = 0x0001
  Payload  = (空)
  Checksum = 0xFFFE

验证: 0xFFFE + 0x0000 + 0x0001 = 0xFFFF ✓
```

```
帧数据: 9F 8E 6D 5C 08 00 01 00 00 00 00 03 00 68 01 00 F5 94
  Length   = 0x0008
  Command  = 0x0001
  Payload  = 00 00 00 03 00 68 01 00
  Checksum = 0x94F5

验证: 0x94F5 + 0x0008 + 0x0001 + 0x0000 + 0x0300 + 0x6800 + 0x0100 = 0xFFFF ✓
```

---

## 3. 命令列表

### 3.1 命令码

| 命令码 | 名称 | 方向 | 说明 |
|-------|------|------|------|
| `0x0000` | Start RAM Download | Host → Device | 初始化 RAM 下载模式 |
| `0x0001` | Start Partition | Host → Device | 声明分区传输的目标地址和大小 |
| `0x0002` | Start Block | Host → Device | 声明即将发送的数据块大小 |
| `0x0003` | End Partition | Host → Device | 标志分区传输完成 |
| `0x0004` | End RAM Download | Host → Device | 完成 RAM 下载阶段 |
| `0x000B` | Set Partition Table | Host → Device | 下发分区表到设备 |

### 3.2 响应码

| 响应码 | 名称 | 说明 |
|-------|------|------|
| `0x0080` | Success | 命令执行成功 |
| `0x0081` | Handshake Response | 握手响应（携带设备标识字符串） |

---

## 4. 命令详细说明

### 4.1 握手请求

握手请求是唯一不使用 AXDL 帧格式的消息，直接发送 3 个原始字节：

```
发送: 0x3C 0x3C 0x3C
```

设备响应一个标准 AXDL 帧，Command 为 `0x0081`，Payload 为 UTF-8 编码的设备标识字符串：

| 设备状态 | 响应 Payload 内容 |
|---------|------------------|
| ROM Boot 模式 | 包含 `"romcode"`，如 `"romcode v1.0;rawy\"` |
| FDL1 已加载 | 包含 `"fdl1"` |
| FDL2 已加载 | 包含 `"fdl2"` |

握手响应帧示例：
```
9F 8E 6D 5C 10 00 81 00 72 6F 6D 63 6F 64 65 20 76 31 2E 30 3B 72 61 77 79 5C
  Signature = 0x5C6D8E9F
  Length    = 16
  Command   = 0x0081
  Payload   = "romcode v1.0;rawy\"
  Checksum  = 0x5C79
```

### 4.2 Start RAM Download (0x0000)

开始 RAM 下载模式，用于加载 FDL（Flash Downloader）到 RAM。

- **Payload**: 空
- **预期响应**: `0x0080`

### 4.3 Start Partition (0x0001)

声明接下来要传输的分区信息。根据载荷大小有三种变体：

#### 变体 A：32 位绝对地址（8 字节载荷）

用于 FDL1 加载，目标为 RAM 地址。

```
偏移   大小    字段
───────────────────────────
0x00   4字节   起始地址（u32 小端序）
0x04   4字节   数据长度（u32 小端序）
```

#### 变体 B：64 位绝对地址（16 字节载荷）

用于 FDL2 加载，目标为 RAM 地址。

```
偏移   大小    字段
───────────────────────────
0x00   8字节   起始地址（u64 小端序）
0x08   8字节   数据长度（u64 小端序）
```

#### 变体 C：分区 ID（88 字节载荷）

用于 CODE 类型镜像的分区烧写，通过分区名标识目标。

```
偏移   大小     字段
───────────────────────────
0x00   72字节   分区名（UTF-16 LE 编码，未使用部分填零）
0x48    8字节   数据总长度（u64 小端序）
0x50    8字节   保留（全零）
```

- **预期响应**: `0x0080`

### 4.4 Start Block (0x0002)

声明即将发送的数据块大小。每个数据块发送前都需要先发送此命令。

- **Payload**: 12 字节（仅前 2 字节有效）

```
偏移   大小     字段
───────────────────────────
0x00   2字节    块大小（u16 小端序）
0x02   10字节   保留（全零）
```

- **预期响应**: `0x0080`

### 4.5 数据块传输

在 `Start Block` 命令得到确认后，直接发送裸数据（不带 AXDL 帧头），设备收到数据后返回一个 `0x0080` 成功响应帧。

```
Host → Device:  [裸数据，长度与 Start Block 声明的块大小一致]
Device → Host:  [AXDL 帧, Command = 0x0080]
```

### 4.6 End Partition (0x0003)

标志当前分区的所有数据已传输完毕。

- **Payload**: 空
- **预期响应**: `0x0080`
- **超时**: 固件镜像烧写时为 60 秒（设备可能需要执行擦写操作）

### 4.7 End RAM Download (0x0004)

标志 RAM 下载阶段完成。设备将跳转执行已加载的 FDL 程序。

- **Payload**: 空
- **预期响应**: `0x0080`

### 4.8 Set Partition Table (0x000B)

将分区表配置下发到设备。

- **Payload**: 分区表二进制数据（详见第 5 节）
- **预期响应**: `0x0080`

---

## 5. 分区表格式

分区表通过 `Set Partition Table (0x000B)` 命令发送，格式如下：

```
偏移     大小     字段
───────────────────────────────
0x00     4字节    魔术字节 "par:" (0x70 0x61 0x72 0x3A)
0x04     1字节    strategy（分区策略）
0x05     1字节    unit（分区单位）
0x06     2字节    分区数量 N（u16 小端序）
0x08     88×N字节  分区条目数组
```

### 5.1 分区条目格式（88 = 0x58 字节）

```
偏移   大小     字段
───────────────────────────────
0x00   64字节   分区名（UTF-16 LE 编码，最大 32 字符，未使用部分填零）
0x40    8字节   gap 偏移（u64 小端序）
0x48    8字节   分区大小（u64 小端序）
0x50    8字节   保留（全零）
```

---

## 6. AXP 镜像文件格式

`.axp` 文件是一个标准 **ZIP 压缩包**，包含以下内容：

### 6.1 配置文件（XML）

ZIP 包内以 `.xml` 为后缀的文件，描述项目配置、分区表和镜像列表：

```xml
<Config>
  <Project alias="AX620E" name="AX630C" version="V2.0.0_P7_...">
    <FDLLevel>2</FDLLevel>  <!-- 1=单级FDL, 2=双级FDL -->

    <Partitions strategy="1" unit="2">
      <Partition gap="0" id="spl" size="768" />
      <Partition gap="0" id="ddrinit" size="512" />
      <!-- ... -->
    </Partitions>

    <ImgList>
      <Img flag="2" name="INIT" select="1">
        <ID>INIT</ID>
        <Type>INIT</Type>           <!-- 镜像类型 -->
        <Block>
          <Base>0x0</Base>          <!-- RAM 基址 -->
          <Size>0x0</Size>
        </Block>
        <File />                     <!-- 无实际文件 -->
        <Auth algo="0" />
        <Description>Handshake with romcode</Description>
      </Img>

      <Img flag="2" name="FDL1" select="1">
        <ID>FDL1</ID>
        <Type>FDL1</Type>
        <Block>
          <Base>0x02000000</Base>   <!-- FDL1 加载到 RAM 的地址 -->
          <Size>0x8000</Size>
        </Block>
        <File>fdl1.bin</File>
        <Auth algo="0" />
        <Description>First level flash downloader</Description>
      </Img>

      <Img flag="2" name="FDL2" select="1">
        <ID>FDL2</ID>
        <Type>FDL2</Type>
        <Block>
          <Base>0x000000</Base>
          <Size>0x40000</Size>
        </Block>
        <File>fdl2.bin</File>
        <Auth algo="0" />
        <Description>Second level flash downloader</Description>
      </Img>

      <Img flag="0" name="KERNEL" select="1">
        <ID>KERNEL</ID>
        <Type>CODE</Type>            <!-- CODE 类型 = 固件分区 -->
        <Block>
          <id>kernel</id>            <!-- 分区 ID，对应分区表中的名称 -->
          <Base>0x0</Base>
          <Size>0x0</Size>
        </Block>
        <File>kernel.bin</File>
        <Auth algo="0" />
        <Description>Kernel image</Description>
      </Img>
    </ImgList>
  </Project>
</Config>
```

### 6.2 镜像类型

| 类型 | 说明 |
|------|------|
| `INIT` | 初始化/握手标记，无实际文件 |
| `EIP` | 加密镜像 |
| `FDL1` | 一级引导加载器（Flash Downloader Level 1） |
| `FDL2` | 二级引导加载器（Flash Downloader Level 2） |
| `FDL` | 单级引导加载器（AX650N 使用） |
| `ERASEFLASH` | 擦除 Flash |
| `CODE` | 固件分区镜像（kernel、rootfs 等） |

### 6.3 二进制镜像文件

ZIP 包中还包含 XML `<File>` 字段引用的二进制文件，如 `fdl1.bin`、`fdl2.bin`、`kernel.bin`、`rootfs.bin` 等。

---

## 7. 完整烧录流程

### 7.1 双级 FDL 模式（FDLLevel = 2）

适用于 AX630C / AX620E 等芯片。

```
Host                                  Device (ROM Boot)
 │                                        │
 │  ═══════ 阶段1: 初始握手 ═══════       │
 │                                        │
 ├──── 0x3C 0x3C 0x3C ──────────────────► │
 │◄──── [0x0081] "romcode v1.0;..." ───── │
 │                                        │
 │  ═══════ 阶段2: 加载 FDL1 ═══════      │
 │                                        │
 ├──── [0x0000] Start RAM Download ─────► │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 ├──── [0x0001] Start Partition ────────► │  (32位绝对地址 + FDL1大小)
 │      (addr=0x02000000, size=N)         │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 │  ┌── 循环传输 FDL1 数据（1000字节/块）──┐│
 │  │ ├── [0x0002] Start Block ────────►│ │
 │  │ │◄── [0x0080] OK ───────────────│ │
 │  │ ├── [裸数据] ────────────────────►│ │
 │  │ │◄── [0x0080] OK ───────────────│ │
 │  └──────────────────────────────────┘│
 │                                        │
 ├──── [0x0003] End Partition ──────────► │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 ├──── [0x0004] End RAM Download ───────► │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 │  ═══════ 阶段3: FDL1 握手 ═══════      │
 │         (设备跳转执行 FDL1)              │
 │                                        │
 ├──── 0x3C 0x3C 0x3C ──────────────────► │
 │◄──── [0x0081] "fdl1" ──────────────── │
 │                                        │
 │  ═══════ 阶段4: 加载 FDL2 ═══════      │
 │                                        │
 ├──── [0x0000] Start RAM Download ─────► │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 ├──── [0x0001] Start Partition ────────► │  (64位绝对地址 + FDL2大小)
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 │  ┌── 循环传输 FDL2 数据（1000字节/块）──┐│
 │  │    (同 FDL1 传输过程)                │
 │  └──────────────────────────────────┘│
 │                                        │
 ├──── [0x0003] End Partition ──────────► │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 ├──── [0x0004] End RAM Download ───────► │
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 │  ═══════ 阶段5: 下发分区表 ═══════      │
 │         (此时 FDL2 已运行)               │
 │                                        │
 ├──── [0x000B] Set Partition Table ────► │  (分区表二进制数据)
 │◄──── [0x0080] OK ──────────────────── │
 │                                        │
 │  ═══════ 阶段6: 烧写固件镜像 ═══════    │
 │                                        │
 │  ┌── 对每个 CODE 类型镜像重复 ──────────┐│
 │  │                                      │
 │  │ ├── [0x0001] Start Partition ────►│ │  (分区ID + 镜像大小)
 │  │ │◄── [0x0080] OK ───────────────│ │
 │  │ │                                │ │
 │  │ │ ┌─ 循环传输数据（48KB/块）─────┐│ │
 │  │ │ │ ├─ [0x0002] Start Block ──►│ │
 │  │ │ │ │◄─ [0x0080] OK ─────────│ │
 │  │ │ │ ├─ [裸数据] ─────────────►│ │
 │  │ │ │ │◄─ [0x0080] OK ─────────│ │
 │  │ │ └──────────────────────────┘│ │
 │  │ │                                │ │
 │  │ ├── [0x0003] End Partition ──────►│ │  (超时60秒)
 │  │ │◄── [0x0080] OK ───────────────│ │
 │  └──────────────────────────────────┘│
 │                                        │
 │  ═══════ 烧录完成 ═══════              │
```

### 7.2 单级 FDL 模式（FDLLevel != 2）

适用于 AX650N 等芯片。流程简化为：

1. 握手（等待 `"romcode"`）
2. 加载单级 FDL（使用 32 位绝对地址）
3. 握手（等待 `"fdl2"`，注意是 `fdl2` 而不是 `fdl1`）
4. 下发分区表
5. 烧写固件镜像

### 7.3 关键参数

| 参数 | 值 |
|------|-----|
| FDL 传输块大小 | 1000 字节 |
| 固件镜像传输块大小 | 48000 字节 (48KB) |
| 命令超时 | 10 分钟 |
| End Partition 超时（固件） | 60 秒 |

---

## 8. 状态机

```
            ┌─────────────┐
            │   INITIAL   │
            └──────┬──────┘
                   │ 发送握手 0x3C 0x3C 0x3C
                   │ 收到 "romcode"
            ┌──────▼──────┐
            │ ROMCODE_READY│
            └──────┬──────┘
                   │
        ┌──────────┴──────────┐
        │                     │
   FDLLevel=2            FDLLevel=1
        │                     │
  ┌─────▼─────┐         ┌────▼────┐
  │ LOAD_FDL1 │         │LOAD_FDL │
  │ (32-bit)  │         │(32-bit) │
  └─────┬─────┘         └────┬────┘
        │ 握手 "fdl1"         │ 握手 "fdl2"
  ┌─────▼─────┐               │
  │ LOAD_FDL2 │               │
  │ (64-bit)  │               │
  └─────┬─────┘               │
        │                     │
        └──────────┬──────────┘
            ┌──────▼──────┐
            │ SET_PART_TBL│  发送 0x000B
            └──────┬──────┘
            ┌──────▼──────┐
            │ WRITE_IMAGES│  逐个烧写 CODE 镜像
            └──────┬──────┘
            ┌──────▼──────┐
            │  COMPLETE   │
            └─────────────┘
```

---

## 9. 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| 帧签名不匹配 | `InvalidFrame` 错误 |
| 校验和验证失败 | `InvalidFrame` 错误 |
| 握手字符串不匹配 | `UnexpectedHandshake` 错误 |
| 响应码非 `0x0080` | `UnexpectedResponse` 错误 |
| 设备未找到 | `DeviceNotFound` 错误 |
| 操作超时 | `DeviceTimeout` 错误 |
| ZIP/镜像文件异常 | `ImageError` / `ImageZipError` 错误 |

---

## 10. Linux udev 规则

为允许非 root 用户访问设备，需添加 udev 规则文件 `/etc/udev/rules.d/99-axdl.rules`：

```
SUBSYSTEM=="usb", ATTRS{idVendor}=="32c9", ATTRS{idProduct}=="1000", MODE="0666"
SUBSYSTEM=="usb_device", ATTRS{idVendor}=="32c9", ATTRS{idProduct}=="1000", MODE="0666"
```
