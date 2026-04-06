# 非常规加密 U 盘 + NAS 协同存储系统

2026 年全国大学生嵌入式芯片与系统设计竞赛 · **ST 赛道**

**作者**：ecat

---

## 一、项目背景与目标

文件泄露、存储损坏、公有云信任成本高，是日常工作与科研场景中的共性痛点。本项目以 **STM32H563** 为核心，构建一款**隐蔽式加密 U 盘**，并与 **ESP32 + NAS** 协同，实现：

1. **隐蔽接入** —— 设备插入后默认不枚举，仅在用户输入自定义「非常规键位组合」后才被主机识别。
2. **分级认证** —— 键位触发 → 口令（Key）验证 → 打开加密存储空间，逐级递进。
3. **写入即加密** —— 所有落盘数据经 AES-256 加密（软件 mbedtls，运行于 Secure World），密钥由硬件 HASH + STM32 UID + 用户 Key 派生，永不出 TrustZone。
4. **NAS 自动同步** —— 验证通过后，STM32 经 UART 将加密文件推送至 ESP32，ESP32 二次加密后上传 NAS，实现多终端安全共享。

---

## 二、系统整体架构

```text
+----------------------------------------------------+
|                    Host (PC)                       |
|                                                    |
|  [Auth Tool]                                       |
|   HID key-combo  -->  USB MSC encrypted drive      |
|   Send password  -->  Unlock R/W                   |
+------------------------+---------------------------+
                         | USB Full-Speed
+------------------------v---------------------------+
|                 STM32H563RGV6                      |
|                                                    |
|  +-- Secure World (S) -------------------------+   |
|  |  Key Mgmt: UID+Key -> HASH -> AES-256 key   |   |
|  |  GTZC: USB/OCTOSPI/USART1/CRC = Secure      | 	 |
|  |  Software AES-256 encrypt (mbedtls)         |   |
|  |  Hardware HASH (SHA-256) + Hardware RNG     |   |
|  |  OCTOSPI NAND driver (W25N02 2Gbit)         |   |
|  |  Auth state machine + NSC gateway           |   |
|  +---------------------------------------------+   |
|  +-- NonSecure World (NS) ---------------------+   |
|  |  ThreadX RTOS                               |   |
|  |  USBX Device: MSC + HID composite           |   |
|  |  FileSystem: LevelX NAND + FileX / FAT      |   |
|  |  NAS sync task (via NSC -> USART1 TX)       |   |
|  +---------------------------------------------+   |
+------------------------+---------------------------+
                         | USART1 (TX/RX, Secure)
+------------------------v---------------------------+
|              ESP32 (WIFI_AES_NAS)                  |
|                                                    |
|  UART recv encrypted blocks from STM32             |
|  Software AES re-encrypt (mbedtls, K_nas)          |
|  Wi-Fi STA -> upload to NAS (SMB/HTTP/FTP)         |
|  MD5/SHA-256 digest + access control               |
+------------------------+---------------------------+
                         | Wi-Fi / LAN
                  +------v------+
                  |  NAS Server |
                  +-------------+
```

---

## 三、硬件设计

### 3.1 核心 MCU

| 项目 | 规格 |
| --- | --- |
| 型号 | STM32H563RGV6（VFQFPN-68） |
| 内核 | Cortex-M33，TrustZone |
| 主频 | 200 MHz（CSI → PLL1 ×100） |
| Flash 分区 | Bank1 前 504 KB → Secure，后 8 KB → NSC veneer；Bank2 512 KB → NonSecure |
| SRAM | 640 KB（Secure / NonSecure 按 MPCBB 分区） |

### 3.2 外部存储

| 项目 | 规格 |
| --- | --- |
| 芯片 | W25N02KVZEIR（Winbond，2 Gbit NAND Flash） |
| 接口 | OCTOSPI1 Quad-SPI 模式（IO0-IO3 + CLK + NCS） |
| 引脚 | PC3→IO0, PB0→IO1, PC2→IO2, PA1→IO3, PA3→CLK, PB10→NCS |

### 3.3 USB 接口

| 项目 | 规格 |
| --- | --- |
| 模式 | USB Device Full-Speed（内置 PHY） |
| 引脚 | PA11→DM, PA12→DP |
| 时钟 | HSI48（48 MHz，CRS 自校准可选） |

### 3.4 STM32 ↔ ESP32 通信

| 项目 | 规格 |
| --- | --- |
| 外设 | USART1（已配 Secure） |
| 波特率 | 建议 921600 bps（后续可升至 DMA + 2 Mbps） |
| 协议 | 自定义帧协议（见第六章） |

### 3.5 ESP32 端

| 项目 | 规格 |
| --- | --- |
| 芯片 | ESP32（或 ESP32-S3，视实际选型） |
| 功能 | Wi-Fi STA → 局域网 NAS；UART 接收加密文件 |
| 加密库 | mbedtls（ESP-IDF 内置） |

---

## 四、软件分层方案

### 4.1 STM32 Secure World

Secure 侧在 `NonSecure_Init()` 跳转前完成所有安全关键初始化，并通过 **NSC（Non-Secure Callable）** veneer 函数向 NonSecure 暴露受控接口。

#### 4.1.1 安全启动

1. `HAL_Init()` → `SystemClock_Config()`（200 MHz）
2. `MX_GTZC_S_Init()` —— 将 USB、OCTOSPI1、USART1、CRC、TIM6、DCACHE1 标记为 Secure
3. 校验内部 Flash 完整性（可选：基于 CRC / HASH 计算固件摘要并与 OTP 区预烧值比对）
4. 跳转 NonSecure

#### 4.1.2 密钥管理（TrustZone 保护）

```text
派生流程：
  STM32_UID（96-bit，芯片唯一 ID，不可外读）
      │
      ├─ 与用户 Key（口令）拼接
      │
      ▼
  硬件 HASH / SHA-256（H563 内置 HASH 加速器）
      │
      ├─ 取前 256 bit
      ▼
  AES-256 密钥 K_flash        ← 用于 NAND 读写加解密
  AES-256 密钥 K_transport    ← 用于 UART 传输加密（可与 K_flash 相同或独立派生）
```

- `K_flash` 和 `K_transport` 仅存在于 Secure SRAM，NonSecure 代码永远无法直接访问。
- 用户 Key 由主机通过 USB HID 报文下发，在 Secure 侧接收后立即与 UID 混合、派生，明文 Key 不保留。
- 硬件 HASH 在 CubeMX 的 Security → HASH 中启用，分配至 Cortex-M33 Secure。

#### 4.1.3 软件 AES 加解密

> **重要说明**：STM32H5**63** 不含硬件 AES（CRYP）加速器，该外设仅在 STM32H5**73** 上提供。
> 因此 AES 加解密采用 **软件实现（mbedtls）**，运行于 Secure World，由 TrustZone 保护密钥安全。

- 库：**mbedtls**（STM32Cube 固件包已包含 `Middlewares/Third_Party/mbedTLS`，或使用 ARM 官方 mbedtls）
- 算法：**AES-256-CBC**（或 AES-256-GCM，GCM 可附带完整性校验标签）
- IV：每次写入时由**硬件 RNG** 生成（H563 内置 RNG），与密文一同写入 NAND
- 性能参考：Cortex-M33 @ 200 MHz 软件 AES-256-CBC 约 **30~50 MB/s**（远超 USB FS 12 Mbps 上限，不构成瓶颈）

#### 4.1.4 NAND Flash 驱动

基于已初始化的 `OCTOSPI1`，实现 W25N02KVZEIR 驱动层：

| 操作 | W25N 命令 | 说明 |
| --- | --- | --- |
| Read ID | 9Fh | 上电校验芯片型号 |
| Page Read | 13h → 03h / 6Bh | 先加载到缓冲区，再 Quad 读出 |
| Page Program | 02h / 32h → 10h | 先写缓冲区，再执行编程 |
| Block Erase | D8h | 128 KB 块擦除 |
| Read Status | 05h | 轮询 BUSY / ECC 状态 |

上层通过 NSC 函数向 NonSecure 提供：
- `NSC_Flash_Read(page, offset, buf, len)`
- `NSC_Flash_Write(page, offset, buf, len)` —— 内部自动 AES 加密后写入
- `NSC_Flash_Erase(block)`

#### 4.1.5 认证状态机

```text
           ┌──────────────┐
           │  STATE_IDLE  │  ← 上电默认，USB 不枚举
           └──────┬───────┘
                  │ 收到隐式键位组合（USB HID 中断）
                  ▼
           ┌──────────────┐
           │ STATE_KEYED  │  ← USB 枚举为 HID 设备（仅认证通道）
           └──────┬───────┘
                  │ 收到正确 Key 口令
                  ▼
           ┌──────────────┐
           │ STATE_UNLOCK │  ← 动态切换为 MSC 复合设备，开放加密空间
           └──────┬───────┘
                  │ 超时 / 用户主动锁定 / 拔出
                  ▼
           ┌──────────────┐
           │  STATE_IDLE  │
           └──────────────┘
```

- 状态变量存放于 Secure SRAM，NonSecure 只能通过 NSC 查询当前状态。
- TIM6 用于超时看门狗：解锁后若一定时间无操作，自动回退至 IDLE。

#### 4.1.6 NSC 接口汇总

| NSC 函数 | 方向 | 说明 |
| --- | --- | --- |
| `NSC_Auth_SubmitCombo(buf, len)` | NS→S | 提交键位组合，校验通过则迁移到 STATE_KEYED |
| `NSC_Auth_SubmitKey(buf, len)` | NS→S | 提交口令 Key，校验通过则迁移到 STATE_UNLOCK |
| `NSC_Auth_GetState()` | NS→S | 返回当前认证状态 |
| `NSC_Auth_Lock()` | NS→S | 主动回退到 STATE_IDLE |
| `NSC_Flash_Read(...)` | NS→S | 读 NAND 页（自动解密），供 LevelX `driver_read` 调用 |
| `NSC_Flash_Write(...)` | NS→S | 写 NAND 页（自动加密），供 LevelX `driver_write` 调用 |
| `NSC_Flash_Erase(...)` | NS→S | 擦除 NAND 块，供 LevelX `driver_block_erase` 调用 |
| `NSC_Flash_VerifyErased(...)` | NS→S | 验证块已擦除，供 LevelX `driver_block_erased_verify` 调用 |
| `NSC_Flash_SpareRW(...)` | NS→S | 读写 spare/OOB 区（坏块标记），供 LevelX `extra_bytes` 调用 |
| `NSC_Sync_GetEncryptedBlock(...)` | NS→S | 获取指定页的密文块（用于 UART 传输给 ESP32） |

### 4.2 STM32 NonSecure World

NonSecure 运行 **ThreadX RTOS**，划分以下线程：

| 线程 | 优先级 | 职责 |
| --- | --- | --- |
| `thread_usb` | 高 | USBX Device 栈，根据认证状态动态注册 HID / MSC |
| `thread_auth` | 中 | 监听 USB HID 报文，调用 NSC 认证接口 |
| `thread_sync` | 低 | 在 STATE_UNLOCK 期间，经 NSC 取密文块，通过 USART1 推送至 ESP32 |
| `thread_idle` | 最低 | LED 指示 / 低功耗管理 |

#### 4.2.1 USB 设备策略

1. **上电 / STATE_IDLE**：USB PCD 已初始化但**不触发枚举**（VBUS 检测/软断连）。设备对主机"不可见"。
2. **STATE_KEYED**：软连接 USB，枚举为 **HID 设备**（仅暴露一个自定义 HID 接口用于口令交互）。
3. **STATE_UNLOCK**：USB 重枚举为 **MSC 复合设备**（Mass Storage + 可选 HID 管理通道），主机看到一个加密磁盘。

#### 4.2.2 文件系统与坏块管理（LevelX + FileX）

采用 Azure RTOS 原生的 **LevelX + FileX** 组合，LevelX 负责 NAND 坏块管理与磨损均衡，FileX 提供 FAT 文件系统，主机可直接识别磁盘。

**完整存储栈：**

```text
USBX MSC (Host sees a FAT drive)
    |
FileX (FAT file system)
    |
LevelX NAND (bad block mgmt + wear leveling)
    |
NSC gateway (NonSecure -> Secure call)
    |
W25N02 OCTOSPI driver + AES encrypt (Secure World)
    |
W25N02KVZEIR Hardware (2Gbit NAND)
```

**LevelX 自动完成的工作：**

- 上电扫描所有 block 的 spare 区，识别出厂坏块（标记 != 0xFF）
- 在 RAM 中维护坏块表（BBT），逻辑块号自动跳过坏块
- 写入/擦除失败时自动标记坏块、数据迁移到好块
- 各块擦写次数均匀分配（磨损均衡），延长 Flash 寿命
- 掉电恢复机制，保护元数据一致性

**CubeMX 配置（LevelX）：**

| 参数 | 值 | 说明 |
| --- | --- | --- |
| Runtime context | **Cortex-M33 NonSecure** | 与 ThreadX/USBX 同侧 |
| LevelX NAND Flash Support | **勾选** | 启用 NAND 支持 |
| File System Interface | **NAND custom interface** | W25N02 为 SPI NAND，需自定义驱动回调 |
| `NAND_SECTOR_MAPPING_CACHE_SIZE_NB_BIT` | **7** | 2^7 = 128 条缓存，约 1 KB RAM |
| `LX_NAND_FLASH_MAX_METADATA_PAGES` | **4** | LevelX 内部元数据页数 |
| `LX_THREAD_SAFE_ENABLE` | **Enabled** | 配合 ThreadX 多线程 |

**CubeMX 配置（FileX）：**

| 参数 | 值 | 说明 |
| --- | --- | --- |
| Runtime context | **Cortex-M33 NonSecure** | 与 LevelX 同侧 |
| FileX Core | **勾选** | 启用 FAT 文件系统 |
| File System Interface | **LevelX NAND interface** | 对接 LevelX |
| `MAX_FAT_CACHE_NB_BIT` | **4** | 2^4 = 16 条 FAT 缓存 |
| `MAX_SECTOR_CACHE_NB_BIT` | **6**（默认 8，建议调低） | 2^6 = 64 条扇区缓存 ≈ 32 KB RAM（默认 8 → 128 KB 太大） |
| `FX_MAX_LONG_NAME_LEN` | **256** | 支持长文件名/中文 |
| `FX_UPDATE_RATE_IN_SECONDS` | **10** | FAT 元数据刷新周期 |

**LevelX 驱动回调（生成代码后需填充）：**

NonSecure 侧的 5 个回调通过 NSC 网关调用 Secure 侧的 OCTOSPI 驱动：

| 回调函数 | 作用 | NSC 对应 |
| --- | --- | --- |
| `lx_nand_driver_read` | 读一页（2048B + 64B spare） | `NSC_Flash_Read()` |
| `lx_nand_driver_write` | 写一页（自动 AES 加密） | `NSC_Flash_Write()` |
| `lx_nand_driver_block_erase` | 擦除一个块（64 页） | `NSC_Flash_Erase()` |
| `lx_nand_driver_block_erased_verify` | 验证块是否擦干净 | `NSC_Flash_VerifyErased()` |
| `lx_nand_driver_extra_bytes_get/set` | 读写 spare/OOB 区（坏块标记） | `NSC_Flash_SpareRW()` |

初始化时传入的 W25N02 几何参数：

| 参数 | 值 |
| --- | --- |
| Total blocks | 2048 |
| Pages per block | 64 |
| Page data bytes | 2048 |
| Page spare bytes | 64 |

#### 4.2.3 NAS 同步任务

`thread_sync` 工作流程：

1. 检查 `NSC_Auth_GetState() == STATE_UNLOCK`
2. 遍历待同步文件列表（可用简单位图标记"脏页"）
3. 调用 `NSC_Sync_GetEncryptedBlock()` 获取密文
4. 按帧协议打包，通过 USART1 DMA 发送至 ESP32
5. 等待 ESP32 ACK，更新同步标记

### 4.3 ESP32（WIFI_AES_NAS）

#### 4.3.1 总体流程

```text
UART 接收密文帧
    │
    ▼
帧校验（CRC-16）
    │
    ▼
AES-256 二次加密（mbedtls，独立密钥 K_nas）
    │
    ▼
Wi-Fi STA → 上传至 NAS
    │
    ▼
返回 ACK 给 STM32
```

#### 4.3.2 NAS 通信方式（可选）

| 协议 | 优势 | 说明 |
| --- | --- | --- |
| HTTP PUT / WebDAV | 实现简单 | 适配多数 NAS（群晖、威联通） |
| SMB / CIFS | 原生文件共享 | 需要轻量 SMB 客户端库 |
| FTP / SFTP | 传统方案 | ESP-IDF 可用 lwIP FTP |

#### 4.3.3 安全考虑

- `K_nas` 密钥在 ESP32 端 NVS 加密分区存储，首次配对时由用户通过 BLE / AP 配网写入。
- MD5（或 SHA-256）摘要随文件一同上传，NAS 端或取回端可校验完整性。
- 权限隔离：不同用户/终端分配不同 NAS 目录 + 不同 `K_nas`。

### 4.4 PC 端认证工具

轻量级命令行 / GUI 工具（Python + pyusb 或 C# / Electron），功能：

1. 监听特定 VID/PID 的 HID 设备出现（STATE_KEYED 后才可见）
2. 提示用户输入口令 Key
3. 通过 USB HID SET_REPORT 下发 Key
4. 显示认证结果（成功 → MSC 盘符自动弹出）

---

## 五、安全设计总结

| 威胁模型 | 防护手段 |
| --- | --- |
| 物理拾取 U 盘 | 隐式键位 + 口令双因子，不知道组合无法枚举 |
| 固件逆向 | TrustZone 隔离，密钥仅在 Secure SRAM；可配合 RDP Level 2 |
| NAND 芯片拆焊读取 | 数据全部 AES-256 加密存储，无密钥无法解密 |
| UART 总线嗅探 | 传输数据已经过 AES 加密（K_flash），ESP32 端再加一层 |
| NAS 端被入侵 | 文件二次加密（K_nas），NAS 上仅有密文 |
| 重放攻击 | 每次传输帧包含递增序号 + CRC，ESP32 校验序号单调递增 |

---

## 六、与 BitLocker 对比分析

### 6.1 本质差异

> **BitLocker 隐藏数据内容，本系统隐藏设备存在。**
>
> BitLocker 加密后，攻击者仍然能看到"这里有一个加密分区"，只是无法解密内容。
> 本系统在未输入正确键位组合前，主机操作系统**完全看不到该 USB 设备**——不仅内容不可见，连设备的存在都不可见。

### 6.2 逐项对比

| 对比维度 | Microsoft BitLocker | 本系统（非常规加密 U 盘） |
| --- | --- | --- |
| **设备可见性** | 加密分区始终可见，设备管理器/磁盘管理可识别 | 未认证前 USB 不枚举，**操作系统无法感知设备存在** |
| **反取证能力** | 取证工具可检测到加密卷，知道"有东西被加密了" | 取证工具看到的是一个没有响应的 USB 口，**可否认性（deniability）** |
| **安全边界** | 依赖 Windows OS 内核 + TPM 芯片；OS 被攻破则密钥可提取 | 密钥在 **TrustZone Secure World** 硬件隔离区，与运行的操作系统完全无关 |
| **TPM 依赖** | 强依赖 TPM；无 TPM 需降级为 U 盘启动密钥或纯密码模式 | **不依赖 TPM**，使用 STM32 芯片唯一 UID + 口令派生密钥，不可克隆 |
| **操作系统** | 仅限 Windows Pro / Enterprise / Education | **跨平台**：任何支持 USB MSC 的系统（Windows / Linux / macOS） |
| **认证方式** | TPM 自动解锁 / PIN / 启动 U 盘（标准输入方式） | **隐式键位组合 + 口令**双因子；键位组合为非标准交互，无法被常规键盘记录器捕获 |
| **密钥存储** | TPM 硬件或恢复密钥存于 Microsoft 账户 / AD | **纯本地**：密钥仅在 Secure SRAM 中派生和存在，不依赖任何云服务 |
| **物理拆解攻击** | TPM 通信总线（LPC/SPI）可被嗅探提取密钥（已有公开攻击案例） | NAND 与 MCU 一体化设计，密钥由芯片 UID 派生，**拆焊 NAND 无法解密、更换 MCU 密钥丢失** |
| **数据备份** | 无内置远程备份机制，需额外配置 | 内置 **NAS 自动同步**，文件双重加密后上传，兼顾安全与可用性 |
| **加密层数** | 单层（AES-XTS） | **双层**：STM32 侧 AES-256 + ESP32 侧 AES-256，NAS 上仅存二次密文 |
| **成本** | 需 Windows Pro 以上授权（约 ¥1000+）+ TPM 硬件 | 独立硬件设备，**无操作系统授权费** |
| **便携性** | 绑定特定电脑的内置硬盘 | **便携 U 盘形态**，可在任意电脑上使用 |

### 6.3 竞赛答辩话术建议

> "BitLocker 的安全模型是'看得见但打不开'，我们的安全模型是'看不见也摸不着'。
> 在反取证场景下，BitLocker 加密卷的存在本身就是一条线索；
> 而我们的设备在未认证前，对主机操作系统和取证工具完全透明——它不存在。
> 此外，BitLocker 的信任根在 OS 内核和 TPM 总线上，而我们的信任根在 Cortex-M33 TrustZone 硬件隔离中，攻击面更小。"

---

## 七、STM32 ↔ ESP32 帧协议（USART1）

```text
┌──────┬───────┬────────┬──────────┬───────────────┬────────┬──────┐
│ HEAD │  SEQ  │  CMD   │  LEN     │   PAYLOAD     │ CRC-16 │ TAIL │
│ 0xAA │ 2B    │ 1B     │ 2B(LE)   │  0~1024B      │ 2B     │ 0x55 │
└──────┴───────┴────────┴──────────┴───────────────┴────────┴──────┘
```

| CMD | 方向 | 说明 |
| --- | --- | --- |
| 0x01 | STM32→ESP32 | 文件数据帧（密文） |
| 0x02 | STM32→ESP32 | 文件传输完成 |
| 0x03 | STM32→ESP32 | 同步请求（文件列表） |
| 0x81 | ESP32→STM32 | ACK |
| 0x82 | ESP32→STM32 | NAK / 错误码 |
| 0x83 | ESP32→STM32 | NAS 状态上报 |

---

## 八、CubeMX 待启用模块与加密方案说明

### 8.1 芯片加密能力澄清

| 外设 | STM32H563（本项目） | STM32H573 |
| --- | --- | --- |
| **HASH**（SHA-1/256） | **有** | 有 |
| **RNG**（硬件随机数） | **有** | 有 |
| **CRYP / AES**（硬件加密） | **无** | 有 |
| **PKA**（公钥加速） | **无** | 有 |
| **SAES**（安全 AES） | **无** | 有 |

> H563 没有硬件 AES 加速器。AES 加解密通过 **mbedtls 软件库**在 Secure World 中执行，Cortex-M33 @ 200 MHz 的软件 AES 吞吐量远超 USB Full-Speed（12 Mbps）瓶颈，不影响实际体验。

### 8.2 CubeMX 需要启用的模块

在 CubeMX → Security 类别中，将以下外设勾选至 **Cortex-M33 Secure**：

| 模块 | CubeMX 位置 | HAL 宏定义 | 用途 |
| --- | --- | --- | --- |
| HASH | Security → HASH → Cortex-M33 Secure | `HAL_HASH_MODULE_ENABLED` | SHA-256 密钥派生 + 固件完整性 |
| RNG | Security → RNG → Cortex-M33 Secure | `HAL_RNG_MODULE_ENABLED` | AES IV 生成、随机数 |
| CRC | Computing → CRC（已在 GTZC 标记 Secure） | `HAL_CRC_MODULE_ENABLED` | UART 帧校验 |

### 8.3 软件依赖

| 库 | 来源 | 用途 |
| --- | --- | --- |
| **mbedtls** | STM32Cube FW_H5 包内 `Middlewares/Third_Party/mbedTLS`，或独立引入 | Secure 侧 AES-256-CBC/GCM 加解密 |
| **LevelX** | Azure RTOS 中间件（CubeMX 可勾选） | NAND 磨损均衡 |
| **FileX** | Azure RTOS 中间件 | FAT 文件系统 |

---

## 九、仓库结构

```text
ecat_usb_key/
├── readme.md                              ← 本文档
├── software/
│   ├── ST/USB_KEY/                        ← STM32CubeMX 工程（CMake）
│   │   ├── USB_KEY.ioc                    ← CubeMX 配置
│   │   ├── Secure/                        ← Secure World 固件
│   │   │   └── Core/
│   │   │       ├── Inc/                   ← 头文件（octospi.h, usb.h, flash.h, ...）
│   │   │       └── Src/                   ← 源码（main.c, gtzc_s.c, octospi.c, ...）
│   │   ├── NonSecure/                     ← NonSecure World 固件
│   │   │   ├── Core/                      ← main.c, app_threadx.c, ...
│   │   │   ├── USBX/                     ← USBX 配置
│   │   │   └── AZURE_RTOS/               ← ThreadX 应用层
│   │   ├── Middlewares/                   ← ThreadX / USBX / LevelX / FileX 中间件
│   │   └── Drivers/                       ← HAL / CMSIS
│   └── ESP32/WIFI_AES_NAS/               ← ESP-IDF 工程
│       └── main/main.c                   ← ESP32 主程序（待实现）
└── tools/                                 ← PC 端认证工具（待创建）
```

---

## 十、开发路线图

| 阶段 | 里程碑 | 关键任务 |
| --- | --- | --- |
| **P0 - 基础验证** | NAND 读写通 | 完成 W25N02 Quad-SPI 驱动，Secure 侧读写页/擦除块 |
| **P1 - 加密链路** | AES 加解密通 | 启用 HASH/RNG，引入 mbedtls 软件 AES，实现密钥派生 + 读写自动加解密 |
| **P2 - USB 设备** | 主机可见加密盘 | USBX MSC + LevelX/FileX，主机可格式化并读写文件 |
| **P3 - 隐式认证** | 完整认证链 | 实现 STATE 状态机 + USB 软断连/重枚举 + PC 认证工具 |
| **P4 - NAS 同步** | 文件自动上传 | USART 帧协议 + ESP32 UART 接收 + Wi-Fi 上传 NAS |
| **P5 - 安全加固** | 安全闭环 | RDP Level 2、完整性校验、防重放、异常回退 |
| **P6 - 联调优化** | 竞赛提交 | 全链路压测、功耗优化、文档与答辩准备 |

