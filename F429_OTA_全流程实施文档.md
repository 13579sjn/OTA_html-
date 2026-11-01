# STM32F429IGT6 OneNET OTA 全流程实施文档

> 适用工程：`HAL_F429IGT6_bootloard`（BootLoader）、`HAL_F429IGT6_App`（设备固件 1.0.0）、`HAL_F429IGT6_App_2`（升级包固件 1.0.1）
>
> 平台：OneNET 物联网平台 · 产品 `OTA_Test`（ID `FRfr9m3Ppg`）· 设备 `OTA_Test_drive`
>
> 本文档基于**已在本板实测通过**的代码与串口日志编写，所有串口片段均来自真实运行记录。
>
> 最后更新：2026-10-27（1.0.0 → 1.0.1 升级 + 首启补报 201 全流程验证通过）

---

## 目录

1. [总体架构](#第-1-章-总体架构)
2. [BootLoader 工程](#第-2-章-bootloader-工程)
3. [App / App_2 工程](#第-3-章-app--app_2-工程)
4. [代码跳转链路](#第-4-章-代码跳转链路带调用栈)
5. [一次完整升级的时序](#第-5-章-一次完整升级的时序真实日志对照)
6. [OneNET 平台 OTA 任务添加流程](#第-6-章-onenet-平台-ota-任务添加流程)
7. [关键参数速查](#第-7-章-关键参数速查)
8. [踩坑与排查表](#第-8-章-踩坑与排查表)
9. [待优化项](#第-9-章-待优化项)
10. [文件清单索引](#文件清单索引)

---

# 第 1 章 总体架构

## 1.1 一句话讲清"四件套"

> **BootLoader 负责"决定跑哪个 App 并跳过去"，App 负责"上网、从平台拉固件、写进片外 Flash、校验后置标志重启"，片外 W25Q64 负责"存放待安装固件 + 旧固件备份 + 出厂镜像"，片内参数区负责"记录启动标志和版本号，是 BL 与 App 之间唯一的持久化通信媒介"。**

四个角色各自的不可替代性：

| 角色 | 不可替代的原因 |
|---|---|
| BootLoader | App 自己没法替换自己（Flash 正被自己执行中），必须有第二段独立代码来做"擦 A 槽 → 写新固件 → 跳转" |
| App | 只有 App 有网络协议栈（ESP8266 AT 驱动 + HTTP），BL 刻意不带网络，保持极简、极小、极可靠 |
| 片外 W25Q64 | 片内 960KB 要留给 App 本体，没有第二块可回滚的空间；旧固件备份 + 出厂镜像都需要额外空间 |
| 参数区 | 复位/掉电后 BL 与 App 都要读它；且必须能"原子更新"（断电不损坏），所以做了 32 槽轮转 + CRC |

## 1.2 片内 Flash 分区表

| 区域 | 起始地址 | 大小 | 占用的扇区 | 说明 |
|---|---|---|---|---|
| BootLoader | `0x08000000` | 48 KB (`0xC000`) | S0 + S1 + S2 | 三段 16KB 扇区，BL 实际代码约 20KB |
| 参数区 PARAM | `0x0800C000` | 16 KB (`0x4000`) | **S3** | 32 槽 × 512B 轮转 |
| App A 槽 | `0x08010000` | 960 KB (`0xF0000`) | S4 ~ S11 | S4=64KB，S5~S11 各 128KB |

### 1.2.1 F429 的扇区结构如何决定这个划分

STM32F429IGT6 是 **1MB 单 Bank Flash**，扇区大小**不是均匀的**（这点和 F407 的 1MB 双/单 Bank 划分完全不同，是移植时最容易踩的坑）：

```
扇区   起始地址      大小
S0     0x08000000    16 KB
S1     0x08004000    16 KB
S2     0x08008000    16 KB
S3     0x0800C000    16 KB      ← 只有 S0~S3 是 16KB
S4     0x08010000    64 KB
S5     0x08020000   128 KB
S6     0x08040000   128 KB
S7     0x08060000   128 KB
S8     0x08080000   128 KB
S9     0x080A0000   128 KB
S10    0x080C0000   128 KB
S11    0x080E0000   128 KB
                  ─────────
              合计 1024 KB
```

由这个结构推出三条硬约束：

1. **参数区必须独占一个可擦除扇区** —— 参数区会被反复擦写，如果它和 BL 或 App 共用扇区，擦参数就会把代码擦掉。所以参数区只能放在 S3（16KB，独立）。
2. **App 必须从 S4 的边界 `0x08010000` 开始** —— 因为 S4 是 64KB 大扇区，不能跨扇区放 App 的起始。BL 占 S0~S2（48KB）+ 参数区占 S3（16KB）= 64KB，正好等于 S0~S3 的总和，所以 App 从 `0x08010000` 起步，天然对齐 S4 边界。
3. **App 槽大小 960KB = 1MB − 64KB**，覆盖 S4~S11，是"能用的最大值"。擦除 App 槽时必须按 S4(64KB) + S5~S11(各 128KB) 分别调用擦除，不能假设等大小。

## 1.3 W25Q64 片外 Flash 分区表

W25Q64 = 8MB = 8192KB，SPI2 接口，JEDEC ID `0x00EF4017`。擦除粒度：扇区 4KB、块 64KB。

| 区域 | 起始地址 | 大小 | 用途 |
|---|---|---|---|
| STAGING（暂存区） | `0x000000` | `0x000F0000`（≈960KB） | App 下载回来的新固件先落这里 |
| BACKUP（备份区） | `0x000F0000` | `0x000F0000` | 升级前把当前 A 槽固件备份到这里，用于回滚 |
| RECOVERY（恢复区/出厂镜像） | `0x001E0000` | `0x000F0000` | 首次上电时把当时的固件固化一份，最坏情况兜底 |
| RESOURCE（资源区） | `0x002D0000` | `0x000F0000` | 预留给字库/图片等 LVGL 资源 |

四个区总占用 `0x003C0000`（3.75MB），其余约 4.25MB 空闲。

**为什么每个区都是 `0x000F0000`**：因为要能放得下最大的 App 镜像（960KB）。区域大小必须 ≥ App 槽大小，否则无法完整备份。

**写片外 Flash 的效率陷阱**（重要）：`bsp/W25q64/w25q64.c` 的 `W25Q64_WriteDataInternal()` 在写入前会先读回判断目标字节是否 `0xFF`；只要有任一字节不是 `0xFF`，它就会走"读整扇区 → 改 → 擦扇区 → 写回"的慢路径。因此：

- **下载前必须先整块擦除暂存区**（15 个 64KB 块），否则第一次写的字节都是非 0xFF，每次都触发慢路径；
- **写粒度不能太小**：每次 `W25Q64_WriteData` 内部会做 page program（典型忙等 0.4ms）。若按 4 字节一写，960KB 需要 245,760 次 page program ≈ **98 秒**；按 256 字节（正好一个 page）写，只需 3,840 次 ≈ **1.5 秒**。

这一点直接决定了 `ota_config.h` 里 `OTA_DL_CHUNK` 的取值（见第 3 章）。

## 1.4 boot_flag 四态与 boot_count 语义

定义在 `OTA/ota_params.h`（BL 与 App 各有一份，**必须逐字节一致**）：

```c
#define OTA_BOOT_FLAG_NORMAL     0U   /* 正常运行 */
#define OTA_BOOT_FLAG_TRY_NEW    1U   /* 新固件试运行（BL 刚拷完，等新固件自证） */
#define OTA_BOOT_FLAG_ROLLBACK   2U   /* 要求回滚到备份版本 */
#define OTA_BOOT_FLAG_RECOVERY   3U   /* 要求恢复出厂镜像 */

#define OTA_BOOT_MAX_TRIES       3U   /* 试运行最大尝试次数 */
```

| flag | 谁写入 | BL 的动作 | 谁清除 |
|---|---|---|---|
| `NORMAL(0)` | 出厂默认 / BL 升级完成后 / 新固件确认后 | 直接校验并跳转 A 槽 | — |
| `TRY_NEW(1)` | App 在 `ota_apply_and_reboot()` 里置位 | 校验 staging → 拷贝到 A 槽 → **boot_count+1** → 跳转；若 `boot_count > 3` 则自动转 ROLLBACK | 新固件调 `ota_confirm_boot_success()` 置回 NORMAL |
| `ROLLBACK(2)` | 新固件自证失败 / 用户手动 `ota_manual_rollback()` / BL 发现试运行超次数 | 校验 backup → 拷贝到 A 槽 → 版本号改回 `backup_version` → flag 置 NORMAL → 跳转 | BL 自己 |
| `RECOVERY(3)` | 用户手动 `ota_manual_recovery()` / A 槽和 rollback 都不可用 | 校验 recovery → 拷贝到 A 槽 → flag 置 NORMAL → 跳转 | BL 自己 |

**boot_count 的语义**：只在 `TRY_NEW` 状态下有意义，表示"这块新固件已经是第几次启动"。每次 BL 以 TRY_NEW 启动就 +1；新固件确认成功后清零。所以：

> **新固件必须在启动后、且在第 3 次复位之前，把 flag 改回 NORMAL。否则 BL 会认为"这块新固件反复重启，肯定有问题"，自动回滚。**

这就是第 4 章里 `ota_boot_confirm_if_needed()` 为什么是**必须调用**的、而不是可选项。

## 1.5 双哈希设计：为什么必须两个

| 哈希 | 算法 | 在哪儿算 | 校验什么 | 失败的后果 |
|---|---|---|---|---|
| 平台 MD5 | MD5 | App 边下载边算，与平台 `check` 返回的 `md5` 比对 | **传输完整性**：从服务器到 MCU 的字节流是否一字不差 | 报 `step=205`，擦除暂存区重下 |
| 本地 SHA-256 | SHA-256 | App 边下载边算（与 MD5 同一次遍历），写入参数区 `image_sha[32]` | **落地完整性**：片外暂存区的字节是否与 App 校验过的一致 | BL 拒绝拷贝，直接转 ROLLBACK/RECOVERY |

**为什么不能只留一个？**

1. **MD5 只在 App 里有效，BL 用不了它**。BL 没有网络，无法知道平台给的 MD5 是多少。所以 BL 需要一个**它自己能验证**的摘要 —— 那就是 App 在下载时算好、写进参数区的 SHA-256。
2. **两者校验的对象不同**。MD5 校验的是"网络收下来的字节"，SHA-256 校验的是"从 W25Q64 读回来的字节"。如果 W25Q64 在下载后、拷贝前发生了位翻转（掉电、干扰、写保护误触），只有 SHA-256 能发现。
3. **BL 绝不允许"未经验证就擦 A 槽"**。这是整条链上最危险的操作 —— 一旦擦掉 A 槽而新固件又是坏的，设备就只剩恢复区一条命。所以在 BL 里做二次校验不是冗余，是**最后一道闸门**。

实测日志里两者都出现过失败并被正确处理（见第 5 章、第 8 章）。

---

# 第 2 章 BootLoader 工程

## 2.1 `Boot\` 目录逐文件职责

全部位于 `HAL_F429IGT6_bootloard\Boot\`，共 24 个文件：

| 文件名 | 职责 | 关键函数 / 内容 |
|---|---|---|
| `partition.h` | **分区与地址的唯一真源**（片内 + 片外） | `OTA_BL_ADDR`、`OTA_PARAM_ADDR`、`OTA_APP_A_ADDR`、`OTA_APP_SIZE`、`OTA_APP_IMAGE_MAX_SIZE`、`EXT_STAGING_ADDR`、`EXT_BACKUP_ADDR`、`EXT_RECOVERY_ADDR`、`EXT_RESOURCE_ADDR`、`OTA_VERSION(maj,min,pat)` 宏 |
| `ota_params.h` | **参数区结构体定义**，BL 与 App 共享布局 | `ota_param_t`（magic/seq/crc32/boot_flag/boot_count/version/image_len/image_sha[32]/resume_offset/dl_tid/ota_tid/backup_version/backup_len/backup_sha[32]/backup_status/recovery_len/recovery_sha[32]）、`OTA_BOOT_FLAG_*`、`OTA_BACKUP_*`、`OTA_BOOT_MAX_TRIES` |
| `crc32.c/.h` | 参数区槽位的 CRC32 校验 | `crc32_calc()` |
| `sha256.c/.h` | 本地落地校验哈希 | `sha256_init/update/final`、`SHA256_DIGEST_SIZE`(=32) |
| `flash_if.c/.h` | 片内 Flash 的擦/写/读封装（4 字节对齐写） | `flash_if_erase_sector()`、`flash_if_write()`、`flash_if_read()` |
| `param_area.c/.h` | 参数区 32 槽轮转读写 | `param_area_load()`、`param_area_save()`、`param_area_defaults()` |
| `boot_config.h` | BL 编译期开关 | `BOOT_WDG_ENABLE=0`、`BOOT_RESTORE_CLOCK_ON_JUMP=1`、`BOOT_LOG_ENABLE=1`、`BOOT_VERIFY_CHUNK=64`、`BOOT_COPY_CHUNK=1024` |
| `boot_log.c/.h` | 串口日志（`printf` 重定向到 USART1） | `boot_log(fmt,...)`；宏别名 `boot_printf` / `USART1_Printf`；含 `#pragma import(__use_no_semihosting)` + `_sys_exit`/`_ttywrch`/`__stdout` |
| `boot_wdg.c/.h` | 寄存器级独立看门狗 | `boot_wdg_init()`（`IWDG->KR=0xCCCC` 启动，PR 码 4 = /64，RLR 4095 ≈ 8.19s）、`boot_wdg_feed()` |
| `boot_ext.c/.h` | 片外 W25Q64 的封装（包住工程自带 `W25q64` 驱动） | `boot_ext_init()`、`boot_ext_jedec_id()`、`boot_ext_read()`、`boot_ext_write()`、`boot_ext_erase_block()`、`boot_ext_erase_sector()` |
| `boot_shutdown.c/.h` | **跳转前的 9 步外设/时钟拆除**（本工程最关键的模块） | `boot_system_shutdown()`、内部 `boot_restore_clock_to_hsi()` |
| `app_jump.c/.h` | App 有效性校验 + 跳转执行 | `app_jump_validate()`、`app_jump_execute()` |
| `boot_start.c/.h` | BL 主状态机 | `boot_run()`；内部 `boot_verify_ext_image()`、`boot_copy_ext_to_run()`、`boot_do_rollback()`、`boot_do_recovery()`、`boot_halt()`、`boot_port_init()` |
| `README.md` | BL 集成步骤、魔术棒推导、期望串口输出、风险清单 | — |

## 2.2 `boot_run()` 完整决策流程

`boot_run()` 是整个 BL 唯一的主函数，由 `Core\Src\main.c` 的 USER CODE 2 区调用。流程如下：

```
boot_run()
│
├─ 1. boot_port_init()          // USART1 + DMA + GPIO + 片外 Flash 初始化
│                               // 把 USART1/DMA2_Stream2/7 的 NVIC 优先级设成 5（低于 App）
├─ 2. 打印 BL banner           // 版本、build 时间、三个分区地址
├─ 3. boot_ext_init()           // 读 JEDEC ID 并打印（0xEF4017 = W25Q64）
├─ 4. param_area_load(&p)       // 读参数区；失败则 param_area_defaults(&p)
├─ 5. 打印 flag / count / ver / img_len / bk_st
│
├─ 6. switch (p.boot_flag)
│   │
│   ├─ OTA_BOOT_FLAG_TRY_NEW (1)  ──────────────┐
│   │    print(">>> TRY_NEW")                   │
│   │    if (boot_count >= OTA_BOOT_MAX_TRIES)  │  ← 试运行超过 3 次
│   │        → boot_do_rollback()               │     认为新固件有问题
│   │    else                                   │
│   │        p.boot_count++; param_area_save()  │
│   │        boot_verify_ext_image(STAGING, image_len, image_sha)
│   │          失败 → boot_do_rollback()
│   │          成功 → boot_copy_ext_to_run(STAGING, image_len)
│   │                  失败 → boot_halt()（喂狗死循环）
│   │                  → p.boot_flag 保持 TRY_NEW，跳转
│   │                                           │
│   ├─ OTA_BOOT_FLAG_ROLLBACK (2) ──────────────┤
│   │    print(">>> ROLLBACK")                  │
│   │    boot_verify_ext_image(BACKUP, backup_len, backup_sha)
│   │      失败 → boot_do_recovery()
│   │    boot_copy_ext_to_run(BACKUP, backup_len)
│   │    p.version = p.backup_version            ← 版本号必须跟着回去！
│   │    p.boot_flag = NORMAL; p.boot_count = 0; p.backup_status = EMPTY
│   │    param_area_save(); 跳转
│   │                                           │
│   ├─ OTA_BOOT_FLAG_RECOVERY (3) ──────────────┤
│   │    print(">>> RECOVERY")                  │
│   │    boot_verify_ext_image(RECOVERY, recovery_len, recovery_sha)
│   │      失败 → boot_halt()
│   │    boot_copy_ext_to_run(RECOVERY, recovery_len)
│   │    p.boot_flag = NORMAL; p.boot_count = 0
│   │    param_area_save(); 跳转
│   │                                           │
│   └─ OTA_BOOT_FLAG_NORMAL (0) / 其它 ─────────┘
│        print("target app @0x08010000")
│        （什么都不做，直接往下走）
│
├─ 7. app_jump_validate(OTA_APP_A_ADDR)    // 校验 A 槽
│      失败 → 若 backup 有效则 boot_do_rollback()
│             否则 boot_do_recovery()
│             再失败 → boot_halt()（死循环 + 喂狗）
│
├─ 8. app_jump_execute(OTA_APP_A_ADDR)     // 关外设 → 恢复时钟 → 跳转（不返回）
│
└─ 9. boot_halt()                          // 理论上到不了
```

**`boot_halt()` 的设计意图**：不是 `while(1);`，而是 `while(1) { boot_wdg_feed(); }`。这样如果看门狗被启用（`BOOT_WDG_ENABLE=1`），设备不会无限重启刷屏，而是安静地停在那里等人工介入 —— 保留现场比反复重启更有价值。

## 2.3 ★重点：跳转前必须关闭外设 + 恢复时钟

这是整份 BL 里**最容易翻车**的地方。原因见 2.3.2。

### 2.3.1 `boot_system_shutdown()` 的 9 步

`Boot\boot_shutdown.c` 按固定顺序做 9 件事，**顺序不能乱**：

| 步骤 | 动作 | 为什么这个顺序 |
|---|---|---|
| 1 | `__disable_irq()` | 后面所有操作期间不能被打断，否则中断里访问已关外设会 HardFault |
| 2 | 关 SysTick（`SysTick->CTRL = 0`、清零 LOAD/VAL） | App 的 `delay_init()` 会重配；留着会带着旧的重载值干扰 |
| 3 | 清 NVIC：所有 `ICER` 关使能、所有 `ICPR` 清挂起、所有 IPR 清优先级 | 不清挂起的话，`__enable_irq()` 那一刻会立刻进中断，而此时外设时钟已经关了 |
| 4 | 关全部 16 路 DMA 流（DMA1/DMA2 的 `SxCR` 清 EN，`LIFCR/LIFCR` 清标志） | DMA 流在传输中被 App 重新初始化，残留的 EN 位会让新配置不生效 |
| 5 | `HAL_UART_DeInit(&huart1)` + `HAL_SPI_DeInit(&hspi2)` | 归还 HAL 内部状态（`gState`/`State`），否则 App 里 HAL 认为"外设还开着"，init 会提前返回 |
| 6 | `HAL_GPIO_DeInit`（PA9/PA10 = USART1，PB12~PB15 = SPI2/W25Q64） | 让 App 的 `HAL_GPIO_Init` 从干净状态开始 |
| 7 | **`boot_restore_clock_to_hsi()`** | 见 2.3.2，本步最关键 |
| 8 | 清 `RCC->AHB1ENR / AHB2ENR / AHB3ENR / APB1ENR / APB2ENR` 全为 0 | 关闭所有外设时钟。App 的 `MX_xxx_Init()` 会逐个重新使能；不关的话会有"外设寄存器残留"导致的诡异现象 |
| 9 | 清 `SCB->ICSR` 的 PENDSTSET / PENDSVSET | FreeRTOS 的 PendSV 若还挂着，App 一开中断就会跳进未初始化的调度器 |

**刻意不做的三件事**（都有原因）：

| 不做 | 原因 |
|---|---|
| 不关 LSI（`RCC->CSR &= ~RCC_CSR_LSION`） | 看门狗用 LSI；关了 IWDG 就失效。而且 App 也不管 LSI，保持开着无害 |
| 不关 OverDrive（`HAL_PWREx_DisableOverDrive()`） | `HAL_PWREx_EnableOverDrive()` 是**幂等**的：已启用时它会检测到 ODEN/ODSWRDY 已置位直接返回。App 调用它不会有问题；而关掉反而可能让 180MHz 时序出错 |
| 不调用 `HAL_DeInit()` | `HAL_DeInit()` 会把 SysTick、NVIC 优先级分组全部恢复默认，还会 `__disable_irq()`，行为太重且不可控；手工逐项拆除更精确 |

### 2.3.2 为什么必须手工把 SYSCLK 切回 HSI（最核心的坑）

先看 HAL 里 `HAL_RCC_ClockConfig()` 的关键逻辑（简化）：

```c
/* STM32F4xx_HAL_Driver 里 HAL_RCC_ClockConfig() 的片段 */
if ((HAL_RCC_GetSysClockFreq() >> AHBPrescTable[...]) != (ClockType >> AHBPrescTable[...]))
{
    /* 只有当"目标频率"与"当前频率"不同时，才真正重新配置 PLL */
    __HAL_RCC_HSE_CONFIG(...);
    __HAL_RCC_PLL_CONFIG(...);
    ...
}
/* 否则直接跳过，只改 AHB/APB 分频 */
```

也就是说：**HAL 判断"要不要重配 PLL"的依据是当前 SYSCLK 与目标 SYSCLK 是否相等。**

BL 自己也是 180MHz、App 目标也是 180MHz —— 如果 BL 不做任何处理直接跳转，App 里的 `SystemClock_Config()` 一进来发现"当前已经是 180MHz 了"，就会**跳过 PLL 重配**，只改分频器。这看起来没问题，但实际会炸，因为：

1. BL 的 PLL 配置和 App 的 PLL 配置**参数未必相同**（比如 m/n/p/q、时钟源），跳过重配就等于沿用 BL 的参数；
2. 更严重的是 **Flash 等待周期（LATENCY）**。App 的 `SystemClock_Config()` 里 `HAL_RCC_ClockConfig(&clk, FLASH_LATENCY_5)` 是在"切换频率"的同时设置 latency 的。一旦跳过分支，latency 是否被正确设成 5 就取决于 HAL 的实现细节；
3. 只要时钟树有一点点不对，**串口波特率就是错的**（表现为串口全是乱码），HSE/PLL 抖动还会让 SPI 时序出错。

所以 BL 的做法是：**跳转前主动把时钟树彻底打回复位状态（HSI 16MHz、PLL 关、所有分频 /1、latency 0）**，让 App 进来时"看到一个全新上电的芯片"，`SystemClock_Config()` 必然走完整的重配分支。

`Boot\boot_shutdown.c` 里的实现（节选）：

```c
/* 把时钟树恢复到复位状态：SYSCLK = HSI = 16MHz，PLL 关闭，全部分频 /1。
   目的：使 App 的 SystemClock_Config() 必定走完整的 PLL 重配分支。
   ⚠ HAL 的 HAL_RCC_OscConfig()/HAL_RCC_ClockConfig() 在 SYSCLK 已等于目标值时会
      跳过 PLL 重配，所以不能指望 HAL，必须手工写寄存器。 */
static void boot_restore_clock_to_hsi(void)
{
    /* 1) 先把 SYSCLK 切到 HSI（必须在关 PLL 之前切，否则会掉到 0Hz 死机） */
    RCC->CFGR &= ~RCC_CFGR_SW;
    RCC->CFGR |=  RCC_CFGR_SW_HSI;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_HSI)
    {
        /* 等硬件确认切换完成 */
    }

    /* 2) 关 PLL，等 PLLRDY 清零 */
    RCC->CR &= ~RCC_CR_PLLON;
    while ((RCC->CR & RCC_CR_PLLRDY) != 0U) { }

    /* 3) 关 HSE，等 HSERDY 清零 */
    RCC->CR &= ~RCC_CR_HSEON;
    while ((RCC->CR & RCC_CR_HSERDY) != 0U) { }

    /* 4) PLLCFGR 写回复位值（0x24003010 = M=16, N=192, P=2, Q=4, 源=HSI） */
    RCC->PLLCFGR = 0x24003010UL;

    /* 5) AHB/APB1/APB2 分频全部回 /1 */
    RCC->CFGR &= ~(RCC_CFGR_HPRE | RCC_CFGR_PPRE1 | RCC_CFGR_PPRE2);

    /* 6) 主频只有 16MHz，Flash 等待周期降到 0 */
    __HAL_FLASH_SET_LATENCY(FLASH_LATENCY_0);
}
```

**顺序上最致命的一点**：必须"先切 SYSCLK 到 HSI，再关 PLL"。反过来做，在 PLL 被关掉而 SYSCLK 还指向 PLL 的那个瞬间，系统时钟会掉到 0Hz —— 芯片当场卡死，且**无法用调试器恢复**（需要断电）。

### 2.3.3 实测验证

正是因为这个改造，跳转后 App 的串口输出完全正常：

```
[BL] jump to app @0x08010000
[BL] shutdown peripherals, restore RCC to HSI...

==============================================
  STM32F429IGT6 OTA BootLoader v1.0
...
```

（App 侧）`AT+RST=1` / `AT+CWMODE=1` / `AT+CWJAP="nanGT8","20061225"` —— 115200 波特率一字不差，证明时钟树和 USART1 都正确重建了。

## 2.4 `app_jump_validate()` 的校验条件

`Boot\app_jump.c`，三道闸门：

```c
/**
 * @brief 校验 addr 处是否存在一个"看起来可信"的 App
 * @retval 0 = 有效；<0 = 无效（错误码）
 */
int app_jump_validate(uint32_t addr)
{
    uint32_t msp;
    uint32_t reset;

    /* 闸门 1：地址必须落在 App A 槽窗口内 */
    if ((addr < OTA_APP_A_ADDR) ||
        (addr >= (OTA_APP_A_ADDR + OTA_APP_A_SIZE)))
    {
        return -1;
    }

    msp   = *(volatile uint32_t *)(addr + 0U);   /* 向量表第 0 项 = 初始 MSP */
    reset = *(volatile uint32_t *)(addr + 4U);   /* 向量表第 1 项 = Reset_Handler */

    /* 闸门 2：MSP 必须落在合法 RAM 内
       —— 普通 SRAM 0x20000000~0x20030000（192KB）
       —— CCM RAM   0x10000000~0x10010000（64KB）
       ★ 这里是"范围判断"而不是"必须等于 0x20030000"：
         实测 App 加了 OTA 模块后栈顶从 0x2000DE38 一路长到 0x20015268，
         若写成等值判断就会把合法 App 误判为无效。 */
    if (!(((msp >= 0x20000000UL) && (msp <= 0x20030000UL)) ||
          ((msp >= 0x10000000UL) && (msp <= 0x10010000UL))))
    {
        return -2;
    }

    /* 闸门 3：Reset_Handler 必须是 Thumb 地址（bit0=1）且落在片内 Flash 内 */
    if ((reset < 0x08000000UL) || (reset >= 0x08100000UL) || ((reset & 1U) == 0U))
    {
        return -3;
    }

    /* 额外：全 0xFF（未烧录）或全 0x00（未初始化）直接判无效 */
    if ((msp == 0xFFFFFFFFUL) || (msp == 0x00000000UL))
    {
        return -4;
    }

    return 0;
}
```

实测的两个合法值（同一块板子、不同版本的 App）：

| 版本 | `vec[0]` (MSP) | `vec[1]` (Reset) |
|---|---|---|
| App 1.0.0（含 P1+P2） | `0x20010988` | `0x080103F9` |
| App 1.0.0（含 P1~P4） | `0x20014E38` | `0x080103F9` |
| App_2 1.0.1 | `0x20015268` | `0x080103F9` |

> 注意 `vec[1]` 一直是 `0x080103F9` —— 因为链接基址固定在 `0x08010000`，Reset_Handler 是启动文件里第一个函数，位置稳定。而 `vec[0]` 随 .data/.bss 增长而上移（每次编译都可能变），所以**绝不能把它写成常量**。

## 2.5 `app_jump_execute()` 的关键几行

```c
/* 用 static volatile 变量保存目标地址：
   —— static：保证地址在栈上，__set_MSP() 之后栈已经被换掉，局部变量会失效；
   —— volatile：防止编译器把它优化进寄存器（换栈后寄存器内容不可靠）。 */
static volatile uint32_t s_jump_addr;

void app_jump_execute(uint32_t addr)
{
    s_jump_addr = addr;

    /* 1) 取目标固件的初始 MSP 并写入 MSP —— 换栈！此后局部变量全部失效 */
    __set_MSP(*(volatile uint32_t *)s_jump_addr);

    /* 2) 重定位向量表：让中断从新固件的向量表取入口 */
    SCB->VTOR = s_jump_addr;

    /* 3) 数据/指令同步屏障 —— 保证 VTOR 写入和栈指针变更对后续取指可见 */
    __DSB();
    __ISB();

    /* 4) 开总中断（BL 在 shutdown 里关了） */
    __enable_irq();

    /* 5) 跳转到新固件的 Reset_Handler（向量表第 1 项） */
    ((void (*)(void))(*(volatile uint32_t *)(s_jump_addr + 4U)))();
}
```

**每一行都不能省**：

| 行 | 省了会怎样 |
|---|---|
| `static volatile` | 换栈后局部变量地址失效，取到垃圾值 → 跳到野地址 |
| `__set_MSP()` | 新固件用的是旧栈，`main()` 里一压栈就把 BL 的栈内容覆盖，行为不可预测 |
| `SCB->VTOR` | 中断仍然走 BL 的向量表 → 第一个 SysTick/USART 中断就跳进 BL 已拆除的代码，HardFault |
| `__DSB(); __ISB();` | Cortex-M4 是乱序执行的，不插屏障可能"VTOR 还没生效就开始取指" |
| `__enable_irq()` | BL 关了中断，不开新固件收不到任何中断，看起来像死机 |

## 2.6 `Core\Src\main.c` 只改了 2 处

BL 的 `main.c` 是 CubeMX 生成的，改动严格限制在 USER CODE 区内（`GENERATE CODE` 不会覆盖）：

```c
/* USER CODE BEGIN Includes */
#include "boot_start.h"
/* USER CODE END Includes */
```

```c
  /* USER CODE BEGIN 2 */
  boot_run();          /* 不返回：内部完成校验+跳转，或进入 boot_halt() */
  /* USER CODE END 2 */

  /* Infinite loop */
  while (1) { }
```

> 这样设计的好处：**BL 的 CubeMX 工程可以随时重新生成**，业务逻辑全部在 `Boot\` 目录里，不受生成影响。

## 2.7 魔术棒（Options for Target）配置

打开 `HAL_F429IGT6_bootloard\MDK-ARM\HAL_F429IGT6.uvprojx` → 魔术棒图标。

### 2.7.1 Target 页

| 项目 | 取值 | 理由 |
|---|---|---|
| **IROM1 Start** | `0x08000000` | BL 从片内 Flash 最开头开始（上电复位向量就在这里） |
| **IROM1 Size** | `0xC000` | 48KB = S0+S1+S2 三个 16KB 扇区 |
| **IRAM1 Start** | `0x20000000` | 普通 SRAM 起始 |
| **IRAM1 Size** | `0x30000` | 192KB（F429IGT6 的 SRAM1+SRAM2+SRAM3 连续映射区） |

> BL **不需要** CCM RAM 段，也不需要用 scatter file 自定义布局 —— 默认的 IROM1/IRAM1 就够。

### 2.7.2 C/C++ 页

| 项目 | 取值 |
|---|---|
| **Define** | `USE_HAL_DRIVER, STM32F429xx` |
| **Include Paths** | 原有路径 + **`..\Boot`**（★ 不加这一条，`#include "boot_start.h"` 会找不到） |
| **Optimization** | `-O1` 或 `-O2` 均可（BL 对体积敏感，`-O2` 更小） |
| **C99 Mode** | 建议勾上（代码里用了 `//` 注释和混合声明） |

### 2.7.3 Debug 页

| 项目 | 取值 |
|---|---|
| **Use** | `ST-Link Debugger`（按实际调试器选） |
| **Settings → Flash Download** | 勾 **`Verify`**（烧完校验，能提前发现 Flash 写坏） |
| | 勾 **`Reset and Run`**（烧完自动复位运行） |
| **Settings → Debug** | `Reset: Autodetect`，`Connect: Normal` |

### 2.7.4 BL **不需要**生成 bin

只有 **App 才需要** bin 文件（因为要上传到 OneNET）。BL 直接用调试器烧 `.axf` 即可，不需要 `fromelf --bin`。

## 2.8 上电期望的串口输出

USART1，115200-8-N-1。BL 阶段的真实日志：

```
==============================================
  STM32F429IGT6 OTA BootLoader v1.0
  build Oct 26 2026 23:54:08
  BL   : 0x08000000 + 48KB
  PARAM: 0x0800C000 + 16KB
  APP A: 0x08010000 + 960KB
==============================================
[BL] ext flash JEDEC ID: 0xEF4017 (W25Q64)
[BL] flag=0 count=0 ver=0x010000 img_len=95496 bk_st=2
[BL] target app @0x08010000
[BL] vec[0]=0x20014E38 vec[1]=0x080103F9
[BL] jump to app @0x08010000
[BL] shutdown peripherals, restore RCC to HSI...
```

各字段含义：

| 字段 | 含义 |
|---|---|
| `JEDEC ID` | `0xEF4017` = W25Q64 正常；`0x000000` / `0xFFFFFF` = SPI 通信异常（见第 8 章） |
| `flag` | boot_flag 当前值，0=NORMAL 1=TRY_NEW 2=ROLLBACK 3=RECOVERY |
| `count` | boot_count（只在 TRY_NEW 下有意义） |
| `ver` | 参数区记录的固件版本（`0x010001` = 1.0.1，编码 `OTA_VERSION(1,0,1)`） |
| `img_len` | 参数区记录的待安装镜像长度（0 = 没有待装镜像） |
| `bk_st` | backup_status：0=EMPTY 1=DOWNLOADING 2=VALID |
| `vec[0]/vec[1]` | A 槽向量表前两项（MSP / Reset） |

---

# 第 3 章 App / App_2 工程

## 3.1 两个工程是什么关系

`HAL_F429IGT6_App` 与 `HAL_F429IGT6_App_2` 是**完全同源**的两个工程 —— 目录结构、所有源码、Keil 工程文件都一模一样，**唯一的区别是 `OTA\ota_config.h` 里的两行版本号**。

| 工程 | 角色 | `OTA_APP_VERSION_STR` | 用途 |
|---|---|---|---|
| `HAL_F429IGT6_App` | 设备当前固件 | `"1.0.0"` | 被烧进板子，等待升级 |
| `HAL_F429IGT6_App_2` | 升级包固件 | `"1.0.1"` | 编译出 bin 上传到 OneNET，作为"新固件" |

**为什么要用两个工程**：不能用一个工程自己升级自己 —— 因为"当前版本"和"目标版本"必须是不同的两个值，否则 `check` 返回的判定逻辑（`target > current`）无法成立。用两个工程可以随时切版本、随时验证。

> 理论上也可以只用一个工程、改版本号后重新编译上传，但那样"板子上跑的固件"和"平台上的包"容易搞混，两个工程更直观。

## 3.2 版本号在哪改、怎么改（最常用的操作）

文件：`HAL_F429IGT6_App\OTA\ota_config.h`（App_2 同理）

```c
/* ===================== 固件版本 ===================== */

/* 内部比较编码（防降级），发布新固件时与下一行同步修改 */
#define OTA_APP_VERSION_NUM     OTA_VERSION(1, 0, 0)

/* 上报平台的版本字符串：必须与平台升级包的版本号"不同且更小"才匹配得到任务 */
#define OTA_APP_VERSION_STR     "1.0.0"
```

### 3.2.1 两条规则

1. **`OTA_APP_VERSION_STR` 必须严格小于平台升级包的目标版本**，`check` 才会返回任务。
   - 设备报 `1.0.0`，平台包 `1.0.1` → `1.0.1 > 1.0.0` → **返回任务，可以升级** ✅
   - 设备报 `1.0.1`，平台包 `1.0.1` → 相等 → 代码判定"已是最新"，**返回无任务** ⚠
   - 设备报 `1.0.2`，平台包 `1.0.1` → 更小 → 代码判定"拒绝降级"，返回 `-3` ❌

2. **两行必须同时改，不能只改一个**。
   - `OTA_APP_VERSION_STR` 决定"报给平台什么版本"（决定能否被升级）；
   - `OTA_APP_VERSION_NUM` 决定"内部防降级的比较基准"（决定愿不愿意装）。
   - 只改 STR 不改 NUM：能匹配到任务，但 `check` 里的 `cur_ver <= p.version` 会用旧 NUM 比较，行为错乱。
   - 只改 NUM 不改 STR：内部认为自己是新版，但平台看到的是旧版本，永不匹配。

### 3.2.2 版本演进实例

| 场景 | 设备 `STR` | 设备 `NUM` | 平台包版本 | 结果 |
|---|---|---|---|---|
| 首次升级 | `"1.0.0"` | `OTA_VERSION(1,0,0)` | `1.0.1` | ✅ 升级到 1.0.1 |
| 想再升级一次 | 必须改成 `"1.0.1"` | `OTA_VERSION(1,0,1)` | `1.0.2` | ✅ 升级到 1.0.2 |
| 忘了改 | `"1.0.1"` | `OTA_VERSION(1,0,1)` | `1.0.1` | ⚠ `check` 返回无任务（已是最新） |

> **所以"升完 1.0.1 再想升级"必须做 1.0.2 的包** —— 这不是平台限制，是我们代码里"防降级/防重复升级"的主动设计。

## 3.3 魔术棒配置（与 BL 的差异）

### 3.3.1 Target 页（★ 关键差异）

| 项目 | BL 取值 | **App 取值** | 说明 |
|---|---|---|---|
| IROM1 Start | `0x08000000` | **`0x08010000`** | ★ 最关键的偏移，必须跳过 BL(48KB)+参数区(16KB) |
| IROM1 Size | `0xC000` | **`0xF0000`** | 960KB = S4(64KB) + S5~S11(7×128KB) |
| IRAM1 Start | `0x20000000` | `0x20000000` | 相同 |
| IRAM1 Size | `0x30000` | `0x30000` | 相同 |

> ⚠ **如果 IROM1 Start 忘了改成 `0x08010000`**：编译能过、烧录也能过，但 App 会覆盖掉 BL 的前 48KB —— 结果就是 BL 没了、每次上电直接跑 App，OTA 彻底失效。而且因为 App 从 `0x08000000` 开始，`SCB->VTOR = 0x08010000` 会指向一片空区域，第一个中断就飞。

### 3.3.2 C/C++ 页

| 项目 | 取值 |
|---|---|
| **Define** | `USE_HAL_DRIVER, STM32F429xx` |
| **Include Paths** | 原有路径 + **`..\OTA`**（★ 必须，否则 `#include "ota_engine.h"` 找不到） |

### 3.3.3 ★ 生成 bin 文件（App 专属，BL 不需要）

上传到 OneNET 的必须是 **`.bin`**（纯二进制），不能是 `.axf`/`.hex`。

配置位置：魔术棒 → **`User`** 标签 → **`After Build/Rebuild`** 区

| 项目 | 填法 |
|---|---|
| **Run #1** | 勾选 ✓ |
| 命令 | `fromelf --bin -o "..\MDK-ARM\HAL_F429IGT6.bin" ".\HAL_F429IGT6\HAL_F429IGT6.axf"` |

说明：

- `fromelf` 是 Keil 自带的工具，无需额外安装；
- 第一个路径是**输出 bin 的位置**，第二个路径是**输入的 axf**；
- 路径里的 `.\HAL_F429IGT6\HAL_F429IGT6.axf` 中，`HAL_F429IGT6` 是 `Output` 标签里配置的输出文件夹名，按工程实际填写；
- 编译成功后 `Output` 窗口会多出一行 `Build Time Elapsed` 之后的 `fromelf` 调用记录，bin 就在 `MDK-ARM\` 下；
- **每次改代码重新编译都要重新上传 bin 到平台**，因为 md5 变了。

### 3.3.4 Debug 页

与 BL 完全相同：`ST-Link Debugger` + Flash Download 勾 `Verify` + 勾 `Reset and Run`。

## 3.4 ★ VTOR 重定位（必须有，否则中断全飞）

文件：`HAL_F429IGT6_App\Core\Src\main.c`

```c
int main(void)
{
  /* USER CODE BEGIN 1 */
  /* 把向量表基址重定位到 App 的实际链接地址。
     ★ 为什么必须有：
       链接器把代码放在 0x08010000（IROM1 配置），但 Cortex-M4 复位后
       SCB->VTOR 的默认值是 0x00000000（在 F4 上映射到 0x08000000，即 BL 的向量表）。
       如果不重定位，所有中断（SysTick/USART/DMA...）都会跳到 BL 的向量表里去，
       而 BL 的对应位置要么是它自己的代码、要么已经被擦除 —— HardFault 或随机飞。 */
  SCB->VTOR = 0x08010000UL;
  __DSB();
  __ISB();
  /* USER CODE END 1 */

  HAL_Init();
  SystemClock_Config();
  ...
}
```

三个要点：

1. **必须在 `HAL_Init()` 之前**（`HAL_Init()` 里会调 `HAL_InitTick()` 使能 SysTick 中断，那时 VTOR 必须已经是正确的）；
2. **`__DSB()` + `__ISB()` 不能省** —— 保证 VTOR 写入对后续取指立即可见；
3. **数值必须和 IROM1 Start 一致**（`0x08010000`），改了一处忘了另一处就是 HardFault。

## 3.5 `OTA\` 目录逐文件说明

`HAL_F429IGT6_App\OTA\` 与 `HAL_F429IGT6_App_2\OTA\` 内容相同（除版本号），共 25 个文件。

按**分层**归类：

### 3.5.1 第 0 层：共享契约（BL 与 App 必须逐字节一致）

| 文件 | 作用 | 谁用它 |
|---|---|---|
| `partition.h` | 片内/片外所有地址、大小、`OTA_VERSION()` 宏的唯一定义 | BL + App 全部模块 |
| `ota_params.h` | `ota_param_t` 结构体布局、boot_flag/backup_status 枚举 | BL + App 全部模块 |

> ⚠ **这两个文件是整份工程里最容易出事的地方**。BL 和 App 是两次独立编译，它们通过参数区（Flash 里的字节）交换数据。只要 `ota_param_t` 的字段顺序、类型、对齐有任何差异，两边读到的就是乱码，表现为"flag 明明是 1 但 BL 说无效"这类灵异现象。
>
> **维护规则**：改 `ota_params.h` 时必须**同时**改 `Boot\ota_params.h`，改完用文件比较工具确认两个文件字节级一致（可以用 `fc /b` 或哈希比对）。

### 3.5.2 第 1 层：基础算法与 Flash 访问

| 文件 | 作用 | 关键函数 | 谁调用它 |
|---|---|---|---|
| `crc32.c/.h` | 参数区槽位校验 | `crc32_calc()` | `param_area.c` |
| `sha256.c/.h` | 落地哈希（BL 二次校验用） | `sha256_init/update/final` | `ota_engine.c`、`ota_backup_*` |
| `md5.c/.h` | 平台哈希（传输完整性） | `md5_init/update/final`、`md5_digest_to_hex()` | `ota_engine.c` |
| `flash_if.c/.h` | 片内 Flash 擦/写/读（4 字节对齐写） | `flash_if_erase_sector()`、`flash_if_write()`、`flash_if_read()` | `param_area.c`、`ota_engine.c`（备份时读 A 槽） |
| `param_area.c/.h` | 参数区 32 槽轮转 + CRC 的原子读写 | `param_area_load()`、`param_area_save()`、`param_area_defaults()` | `ota_engine.c`、`ota_task.c`、`ota_rollback_test.c`、`boot_run()`(BL 侧) |
| `w25q16.h` | 片外 Flash 的常量与接口声明 | `W25Q_EXT_JEDEC_W25Q64`、`w25q16_init/read/write/erase_block/...` | `w25q_ext.c`、`ota_engine.c` |
| `w25q_ext.c` | 片外 Flash 实现（包住 `bsp/W25q64/w25q64.c`） | `w25q16_init()`、`w25q16_write()`（4KB 分块）、`w25q16_erase_block()`（强制 64KB 对齐） | `ota_engine.c`、`freertos.c`(自检) |

**`ota_params.h` 的完整结构体**（这是整个 OTA 的"数据库表结构"）：

```c
typedef struct
{
    uint32_t magic;             /* 固定 0xA5A5A5A5，用于识别槽位是否有效 */
    uint32_t seq;               /* 递增序号，load 时取 seq 最大的槽 */
    uint32_t crc32;             /* 本槽除 magic/seq/crc32 外全部字节的 CRC32 */

    uint32_t boot_flag;         /* OTA_BOOT_FLAG_* 四态 */
    uint32_t boot_count;        /* TRY_NEW 下的试运行次数 */
    uint32_t version;           /* 当前运行的固件版本，OTA_VERSION(maj,min,pat) */

    uint32_t image_len;         /* 暂存区里待安装镜像的长度（0 = 无） */
    uint8_t  image_sha[32];     /* 待安装镜像的 SHA-256（BL 拷贝前校验） */

    uint32_t resume_offset;     /* 断点续传偏移（1KB 对齐） */
    uint32_t dl_tid;            /* 正在下载的任务号（用于判断续传是否同一任务） */

    uint32_t ota_tid;           /* 已安装但尚未上报 201 的任务号（0 = 无需补报） */

    uint32_t backup_version;    /* 备份的旧固件版本 */
    uint32_t backup_len;        /* 备份的旧固件长度 */
    uint8_t  backup_sha[32];    /* 备份的旧固件 SHA-256 */
    uint32_t backup_status;     /* OTA_BACKUP_* */

    uint32_t recovery_len;      /* 出厂镜像长度（0 = 尚未初始化） */
    uint8_t  recovery_sha[32];  /* 出厂镜像 SHA-256 */
} ota_param_t;
```

### 3.5.3 第 2 层：网络通道（不含任何 OTA 业务逻辑）

| 文件 | 作用 | 关键函数 | 谁调用它 |
|---|---|---|---|
| `esp01s.c/.h` | ESP8266 AT 驱动（裸 TCP、`+IPD` 解析、环形缓冲） | `esp01s_init()`、`esp01s_join_ap()`、`esp01s_tcp_connect()`、`esp01s_send()`、`esp01s_read()`、`esp01s_tcp_close()`、`esp01s_flush_rx()`、`esp01s_parse()` | `http_client.c`、`ota_net_switch.c` |
| `http_client.c/.h` | 极简 HTTP/1.1 客户端（支持 Range / chunked / `Ota-Errno` / 流式回调） | `http_client_request()` | `ota_engine.c`、`ota_net_switch.c` |
| `ota_port.c` | 实现 `USART1_Printf()`（P3 原设计要用的串口打印，现引擎用 `printf`，此文件保留兼容） | `USART1_Printf()` | （预留） |

**`esp01s.h` 里的两个缓冲大小**（这两行直接决定下载会不会丢字节）：

```c
/* 串口接收环形缓冲区大小：4096 字节可容忍约 350ms 的消费停顿 */
#define ESP_RX_RING_SIZE   4096U
/* 数据缓冲区（剥离 IPD 头后的字节流）：8192 字节 */
#define ESP_DATA_BUF_SIZE  8192U
```

**`http_client.c` 里三个"必须放静态区"的缓冲**（这是栈溢出事故的直接原因）：

```c
/* ★ 这三个缓冲原来在栈上（req 512 + line 160 + chunk 1024 ≈ 1.7KB），
   在 FreeRTOS 任务里会把 2KB 的 defaultTask 栈直接打爆
   （现象：[RTOS] STACK OVERFLOW in task defaultTask）。
   本工程同一时刻只有一个任务跑 HTTP，改成静态即可，且省下 1.7KB 栈。 */
static char req[512];
static char line[160];
static uint8_t chunk[1024];
```

### 3.5.4 第 3 层：通道交接（MQTT ↔ 裸 TCP）

| 文件 | 作用 | 关键函数 | 谁调用它 |
|---|---|---|---|
| `ota_net_switch.c/.h` | 把 USART3 从"AT+MQTT 常连"切给 esp01s 独占，用完再切回并重连 MQTT | `ota_net_is_active()`、`ota_net_enter()`、`ota_net_exit()`、`ota_p2_selftest()`（自测/首启入口）、内部 `ota_net_restore()`、`ota_dma_switch_mode()` | `ota_task.c`、`bsp/UART/uart_handle.c`（中断守卫） |

**核心难点**：MQTT 一直在跑，USART3 的 DMA 配置是 `DMA_NORMAL` + IDLE 中断（`uart_handle.c` 的语义）；而 esp01s 需要 `DMA_CIRCULAR` + 轮询。切换时必须**运行时改 DMA 模式**，不能改 CubeMX：

```c
/* 运行时切换 USART3 RX 的 DMA 模式。
   ★ 不要改 Core/Src/usart.c 里的 Init.Mode —— 那在 CubeMX 生成区，会被覆盖；
     而且永久改成 CIRCULAR 会破坏 uart_handle.c 里 IDLE 回调对 Size 的语义假设。 */
static int ota_dma_switch_mode(uint32_t mode)
{
    hdma_usart3_rx.Init.Mode = mode;
    HAL_DMA_DeInit(&hdma_usart3_rx);
    return (HAL_DMA_Init(&hdma_usart3_rx) == HAL_OK) ? 0 : -1;
}
```

**进入 OTA 模式的完整动作序列**（顺序不能变）：

```
ota_net_enter()
├─ uart_printf(&huart3, "AT+MQTTDISC=0\r\n")   // 主动断开 MQTT，释放 socket
├─ HAL_Delay(800)                               // ★ 模块释放 socket 需要时间，等不够会回 busy
├─ HAL_UART_AbortReceive(&huart3)               // 停掉 MQTT 模式的 DMA 接收
├─ ota_dma_switch_mode(DMA_CIRCULAR)            // 切成 esp01s 需要的模式
├─ esp01s_init()                                // AT+RST → ATE0 → CIPMUX=0 → CIPMODE=0
│                                               // 末尾启动 HAL_UART_Receive_DMA(4096)
├─ esp01s_join_ap(wifi_name, wifi_pass, 15000)  // AT+CWJAP
└─ 任一步失败 → ota_net_restore() + 返回负值
```

**退出 OTA 模式的序列**：

```
ota_net_exit()
├─ esp01s_tcp_close()                           // AT+CIPCLOSE
├─ HAL_Delay(100)
└─ ota_net_restore()
   ├─ HAL_UART_AbortReceive(&huart3)
   ├─ ota_dma_switch_mode(DMA_NORMAL)           // 切回 uart_handle.c 期望的模式
   ├─ uart_start_receive(&huart3, uart3_Recived_buf)
   └─ esp8266_Init()                            // ★ 必须重跑！因为 esp01s_init 里的 AT+RST
                                                //   已经把 MQTT 的 USERCFG/CONN/SUB 全清了
```

> **`AT+RST` 的副作用是这份设计的最大代价**：ESP8266 复位后，之前用 `AT+MQTTUSERCFG/MQTTCONN/MQTTSUB` 建立的配置全部丢失（都在 RAM 里），所以退出时**必须**重跑一遍 `esp8266_Init()`，耗时约 14 秒。这是可接受的，因为它只在"下载失败"的路径上，成功路径是直接复位。
>
> 如果将来想优化：给 `esp01s_init()` 加一个"跳过 AT+RST"的变体，就能保住 MQTT 配置，退出时间可降到 1~2 秒。

### 3.5.5 第 4 层：OTA 业务引擎

| 文件 | 作用 | 关键函数 | 谁调用它 |
|---|---|---|---|
| `ota_config.h` | **所有可调参数的唯一入口**（平台参数、版本号、下载粒度、行为开关） | 见下文 | 全部 OTA 模块 |
| `ota_engine.h` | 引擎对外接口 + 状态快照结构 | `ota_status_t`、`ota_state_t` | `ota_task.c`、UI |
| `ota_engine.c` | **OTA 执行引擎**：上报版本 → check → Range 下载 → MD5/长度/SHA-256 校验 → 备份旧固件 → 写参数区 → 置 TRY_NEW → 复位 | `ota_run_once()`、`ota_download()`、`ota_download_try()`、`ota_backup_current_app()`、`ota_apply_and_reboot()`、`ota_init_recovery_if_empty()`、`ota_confirm_boot_success()`、`ota_report_pending_201()`、`ota_boot_confirm_if_needed()`、`ota_report_version()`、`ota_manual_rollback()`、`ota_manual_recovery()`、`ota_get_status()`、`ota_feed_watchdog()` | `ota_task.c`、`ota_net_switch.c`、`ota_rollback_test.c`、UI |

**`ota_config.h` 里的关键参数（改这些就够了，不用碰逻辑）**：

```c
/* 平台参数 */
#define OTA_PID                 "FRfr9m3Ppg"
#define OTA_DEV                 "OTA_Test_drive"
#define OTA_TOKEN               "version=2018-10-31&res=products%2F...&sign=..."
#define OTA_API_HOST            "iot-api.heclouds.com"
#define OTA_API_PORT            80U
#define OTA_TASK_TYPE           1              /* ★ 必须是 1 */

/* 固件版本（两个工程唯一差异就在这两行） */
#define OTA_APP_VERSION_NUM     OTA_VERSION(1, 0, 0)
#define OTA_APP_VERSION_STR     "1.0.0"

/* 下载参数 */
#define OTA_DL_CHUNK            256U           /* 攒够 256 字节写一次片外 Flash */
#define OTA_CHECKPOINT_INTERVAL 16384U         /* 每 16KB 存一次断点 */
#define OTA_PROGRESS_STEP       100U           /* 100 = 不中途上报进度 */

/* 行为开关 */
#define OTA_TEST_NO_REBOOT      0              /* 0=真升级  1=只下载不升级 */
#define OTA_STARTUP_CONFIRM_MS  30000U          /* 新固件首启静默窗口 */
#define OTA_SELFTEST_STAGE      0              /* 0=关闭自测 2=只check 3=下载 4=首启上报 */
#define OTA_REPORT_FAIL_TO_PLATFORM  0         /* 0=失败不上报（调试期必须 0） */
#define OTA_ROLLBACK_TEST_ENABLE     0         /* 1=升级后自动回滚（仅测试包开启） */
#define OTA_ROLLBACK_TEST_DELAY_MS   5000U
```

**`ota_engine.h` 里的状态快照**（给屏幕 UI 用）：

```c
typedef enum {
    OTA_ST_IDLE = 0,        /* 空闲 */
    OTA_ST_CHECKING,        /* 正在 check 任务 */
    OTA_ST_DOWNLOADING,     /* 正在下载 */
    OTA_ST_VERIFYING,       /* 正在校验（MD5 / 长度 / SHA-256） */
    OTA_ST_BACKUPING,       /* 正在备份旧固件 */
    OTA_ST_READY_REBOOT,    /* 已就绪，等待（或已）重启 */
    OTA_ST_NONE,            /* 平台无任务 */
    OTA_ST_ERROR            /* 出错 */
} ota_state_t;

typedef struct {
    ota_state_t state;
    uint32_t    progress;      /* 0 ~ 100 */
    uint32_t    downloaded;    /* 已写入外部 Flash 的字节数 */
    uint32_t    total;         /* 升级包总字节数 */
    uint32_t    tid;           /* 当前任务号 */
    char        target[24];    /* 目标版本字符串 */
    char        msg[64];       /* 附加信息（可直接显示） */
} ota_status_t;
```

### 3.5.6 第 5 层：触发与测试

| 文件 | 作用 | 关键函数 | 谁调用它 |
|---|---|---|---|
| `ota_task.h` | 触发任务接口 | `ota_task_on_update_ident()`、`ota_task_request()`、`ota_task_is_busy()` | `esp_handle.c`、UI |
| `ota_task.c` | **懒创建**的 OTA 任务（栈 1536 字），轮询触发标志 → 执行 `ota_net_enter/run_once/exit` | `ota_task_entry()`、`ota_task_request()`、`ota_task_on_update_ident()`、`ota_task_post_update_zero()` | 由 `esp_handle.c` 触发 |
| `ota_rollback_test.h` | 回滚自测接口 | `ota_rollback_test_hook()`、`ota_rollback_test_arm()` | `ota_net_switch.c`（钩子）、用户（手动） |
| `ota_rollback_test.c` | 新固件首启后等 N 毫秒自动置 ROLLBACK，验证回滚链路 | `ota_rollback_test_hook()`、`ota_rollback_test_arm()` | `ota_net_switch.c` |

**为什么 `ota_task` 要"懒创建"**：

```c
void ota_task_request(uint32_t value)
{
    if (value == 0U) { return; }        /* 0 = 空闲值，不触发 */
    if (s_busy != 0) {                  /* 正在升级，忽略重复触发 */
        printf("[OTA] busy, trigger ignored\r\n");
        return;
    }

    s_req = value;

    if (s_h == NULL)
    {
        /* 首次触发时懒创建：这样就不用改 CubeMX 生成的 freertos.c，
           也不需要为一块可能永远用不到的功能常驻 6KB 内存 */
        if (xTaskCreate(ota_task_entry, "otaTask", 1536, NULL,
                        tskIDLE_PRIORITY + 2U, &s_h) != pdPASS)
        {
            s_h = NULL; s_req = 0U;
            printf("[OTA] xTaskCreate FAILED (heap low?)\r\n");
        }
    }
}
```

### 3.5.7 关于 `OTA_SELFTEST_STAGE`

这个开关原本是为了"分阶段验证"而加的，现在**量产配置一律是 0**：

| 值 | 行为 | 用途 |
|---|---|---|
| `0` | 只做"新固件首启确认 + 补报"，不做任何自测 | **量产/正常运行** |
| `2` | 只做一次 `check`（验证通道交接） | P2 阶段验证用 |
| `3` | 走完整 `ota_run_once()`（下载但不升级，配合 `OTA_TEST_NO_REBOOT=1`） | P3a 阶段验证用 |
| `4` | 新固件首启：补报 201 + 上报版本（不再下载） | P3b 验证包的临时配置 |

> 注意 `OTA_SELFTEST_STAGE` 只管"自测"，**不影响"Update 触发升级"**（那是 `ota_task.c` 独立负责的），也**不影响首启确认**（代码在 `#if` 之前）。

---

# 第 4 章 代码跳转链路（带调用栈）

整条 OTA 链路一共 **5 段跳转**，每一段的"谁调用谁"如下。

## 4.1 第一段：上电 → BL

```
硬件复位
  └─ 从 0x08000000 取 MSP / 从 0x08000004 取 Reset_Handler
      └─ HAL_F429IGT6_bootloard: startup_stm32f429xx.s → Reset_Handler
          └─ SystemInit()
              └─ main()
                  └─ HAL_Init()
                  └─ SystemClock_Config()      /* BL 自己也是 180MHz */
                  └─ MX_GPIO_Init() / MX_DMA_Init() / MX_USART1_UART_Init() / MX_SPI2_Init()
                  └─ /* USER CODE BEGIN 2 */
                     boot_run()                 ★ 全部业务从这里进入
                       ├─ boot_port_init()
                       ├─ boot_ext_init()
                       ├─ param_area_load(&p)
                       └─ switch (p.boot_flag) { ... }
```

## 4.2 第二段：BL → App（含 VTOR 重定位）

```
boot_run()
  ├─ app_jump_validate(OTA_APP_A_ADDR)         /* 三道闸门全过 */
  │    └─ return 0
  │
  └─ app_jump_execute(OTA_APP_A_ADDR)
       ├─ s_jump_addr = 0x08010000
       ├─ __set_MSP(*(uint32_t*)0x08010000)     /* 换栈 → MSP = 0x20014E38 */
       ├─ SCB->VTOR = 0x08010000                /* ★ 向量表重定位 */
       ├─ __DSB(); __ISB();
       ├─ __enable_irq()
       └─ ((void(*)(void))(*(uint32_t*)0x08010004))()
            │                                    /* = 0x080103F9 */
            ▼
          HAL_F429IGT6_App: Reset_Handler
            └─ SystemInit()                      /* 里面会设置 SCB->VTOR = VECT_TAB_OFFSET，
                                                    但我们的工程用 0 偏移，所以靠 main() 里手工设 */
            └─ __main()  → 初始化 .data/.bss → main()
                 └─ /* USER CODE BEGIN 1 */
                    SCB->VTOR = 0x08010000UL;    ★ 再次确认
                    __DSB(); __ISB();
                 └─ HAL_Init() → SystemClock_Config() → MX_xxx_Init()
                 └─ MX_FREERTOS_Init()           /* 建 defaultTask 等 */
                 └─ osKernelStart()              /* 不返回 */
```

**为什么 VTOR 要设两次**（`Reset_Handler` 里一次 + `main()` 里一次）？

- 启动文件里 `SystemInit()` 的 VTOR 设置来自 `system_stm32f4xx.c` 的 `VECT_TAB_OFFSET` 宏，默认是 0。如果不在那个宏里改成 `0x10000`，`SystemInit()` 实际上把 VTOR 设成了 `0x08000000`。
- `main()` 开头再设一次是**兜底**，保证在任何中断被使能之前 VTOR 一定正确。
- 更彻底的做法是改 `system_stm32f4xx.c` 的 `VECT_TAB_OFFSET`，但那是 CubeMX 生成区，容易被覆盖，所以选择在 `main()` 里手写。

## 4.3 第三段：MQTT 属性 → OTA 触发

```
ESP8266 收到平台的属性下发
  └─ USART3 IDLE 中断 → uart_handle.c: HAL_UARTEx_RxEventCallback()
      └─ 把数据塞进 uart3_Recived_buf
          │
          ▼  （FreeRTOS defaultTask 里轮询）
      freertos.c: StartDefaultTask()
        └─ uart_process()                       /* esp_handle.c */
            ├─ 解析 +MQTTSUBRECV 里的 JSON
            ├─ 解析出 ident = "Update"、value = 1
            │
            ├─ printf("ID=%s, IDENT=%s, VALUE=%s, NUM=%ld\r\n", ...)
            │
            └─ ota_task_on_update_ident(ident)   ★ 挂钩点（esp_handle.c 第 479 行附近）
                 │
                 └─ ota_task.c: ota_task_on_update_ident()
                      ├─ if (strcmp(ident, "Update") != 0) return;
                      └─ ota_task_request(1U)
                           ├─ s_req = 1
                           └─ if (s_h == NULL) xTaskCreate(ota_task_entry, "otaTask", 1536, ...)
                                                              │
                                                              ▼  新任务开始运行
                                                        ota_task_entry()
                                                          └─ for(;;) {
                                                               if (s_req && !s_busy) { ... }
                                                               vTaskDelay(200ms);
                                                             }
```

**挂钩点的实际代码**（`App\esp_handle.c`，注意这个文件是 GBK 编码）：

```c
#include "ota_task.h"                       /* 第 1 行 */
...
            /* LED / count 处理之前，先看是不是 OTA 触发 */
            ota_task_on_update_ident(ident);   /* 第 479 行 */

			if (strcmp(ident, "LED") == 0 && parsed_type == ONENET_VALUE_BOOL)
            { ... }
			else if (strcmp(ident, "count") == 0 && parsed_type == ONENET_VALUE_INTEGER)
            { ... }
```

> **为什么钩子只传 `ident` 一个参数**：`ota_task.c` 内部自己 `strcmp("Update")`，这样 `esp_handle.c` 的改动最小（一行），而且将来想加别的触发标识符不用再改 `esp_handle.c`。

## 4.4 第四段：下载链路

```
ota_task_entry()
  ├─ ota_net_enter()                            /* 断 MQTT、切 DMA、esp01s_init、入网 */
  │    ├─ uart_printf(&huart3, "AT+MQTTDISC=0\r\n")
  │    ├─ HAL_Delay(800)
  │    ├─ HAL_UART_AbortReceive(&huart3)
  │    ├─ ota_dma_switch_mode(DMA_CIRCULAR)
  │    ├─ esp01s_init()                         /* AT+RST / ATE0 / CIPMUX=0 / CIPMODE=0
  │    │                                          + HAL_UART_Receive_DMA(&huart3, s_dma_rx, 4096) */
  │    └─ esp01s_join_ap(wifi_name, wifi_pass, 15000)
  │
  └─ ota_run_once()                             /* ota_engine.c 主入口 */
       │
       ├─ 0. if (p.recovery_len == 0) ota_init_recovery_if_empty()
       │       └─ 擦 RECOVERY 区(15×64KB) → 从 A 槽 1KB 分块拷贝 + SHA-256 → 写参数区
       │
       ├─ 1. ota_report_pending_201()
       │       └─ if (p.ota_tid != 0 && p.version == OTA_APP_VERSION_NUM)
       │            → ota_api_report_status(p.ota_tid, 201)   /* 补报上次升级成功 */
       │
       ├─ 2. ota_api_report_version()
       │       └─ http_client_request(POST, "/version", {"s_version":"1.0.0","f_version":"1.0.0"})
       │
       ├─ 3. ota_api_check_task(&t)
       │       └─ http_client_request(GET, "/check?type=1&version=1.0.0")
       │            → json_get_num("tid") / ("size") / ("type") / json_get_str("target","md5")
       │            → 无 tid 或 code!=0 → t.has_task = 0 → 返回 1（无任务）
       │
       ├─ 4. 拒绝差分包 / 降级
       │       ├─ if (t.type != 1) → 上报 206，返回 -3
       │       └─ if (target_version <= p.version)
       │             ├─ 相等 → 上报 201，返回 1（已是最新）
       │             └─ 更小 → 上报 206，返回 -3（拒绝降级）
       │
       ├─ 5. ota_download(&t, &new_len, new_sha)
       │       │
       │       ├─ ota_download_try(t, len, sha, force_restart=0)
       │       │    │
       │       │    ├─ 断点续传判定：p.dl_tid == t.tid && p.backup_status == DOWNLOADING
       │       │    │     → 从 p.resume_offset 继续，并重扫 0~offset 补哈希
       │       │    │  否则 → 擦 STAGING(15×64KB)，offset = 0
       │       │    │
       │       │    └─ for (attempt = 0; attempt < 8 && offset < t.size; attempt++)
       │       │         │
       │       │         ├─ snprintf(extra, "Authorization: %s\r\nRange: bytes=%lu-\r\n")
       │       │         │
       │       │         ├─ http_client_request(GET, "/{tid}/download", ...,
       │       │         │                        on_data=dl_on_data, arg=&c, timeout=8000)
       │       │         │    │
       │       │         │    ├─ esp01s_flush_rx()
       │       │         │    ├─ esp01s_tcp_connect(host, 80, 5000)
       │       │         │    ├─ esp01s_send(req, len, 3000)
       │       │         │    ├─ http_read_line(status_line, 3000)
       │       │         │    └─ while (remaining > 0)
       │       │         │         ├─ esp01s_read(chunk, want, timeout)
       │       │         │         │    └─ esp01s_parse()  /* DMA→ring→s_data */
       │       │         │         └─ on_data(chunk, n, arg)   ★ 回调 dl_on_data
       │       │         │              │
       │       │         │              └─ dl_on_data()          /* ota_engine.c */
       │       │         │                   ├─ sha256_update(&c.sha, data, len)
       │       │         │                   ├─ md5_update(&c.md5, data, len)
       │       │         │                   └─ while (len) {
       │       │         │                        memcpy(&c.buf[c.blen], ...)
       │       │         │                        if (c.blen == OTA_DL_CHUNK /*256*/) {
       │       │         │                            w25q16_write(EXT_STAGING + c.offset, c.buf, 256)
       │       │         │                            c.offset += 256
       │       │         │                            if (offset - last_save >= 16384) 存断点
       │       │         │                            ota_feed_watchdog()
       │       │         │                        }
       │       │         │                      }
       │       │         │
       │       │         ├─ 本轮零头落盘（保证 c.offset 与 Flash 实际一致）
       │       │         ├─ 打印 try N: ret/status/len/errno/got
       │       │         ├─ if (c.offset - before < resp.content_len)
       │       │         │      → 打印 TRUNCATED（说明中间丢字节，数据整体前移）
       │       │         └─ if (c.offset >= t.size) break;
       │       │
       │       └─ 外层兜底：if (r == -5 /* 校验失败 */)
       │             → ota_download_try(..., force_restart=1)   /* 擦除 + 从头重下 */
       │
       ├─ 6. 校验
       │       ├─ if (c.offset != t.size)              → 返回 -5
       │       ├─ md5_final + md5_hex_equal(t.md5)     → 不等则返回 -5
       │       └─ sha256_final(sha_out)                /* 存给 BL 用 */
       │
       ├─ 7. ota_backup_current_app(&bk_len, bk_sha)
       │       ├─ ota_probe_run_len()                  /* 从 A 槽末尾反扫找非 0xFF，4KB 对齐 */
       │       ├─ 擦 BACKUP 区(15×64KB)
       │       └─ 1KB 分块：flash_if_read(A) → w25q16_write(BACKUP) + sha256_update
       │
       ├─ 8. 写参数区
       │       p.image_len = new_len; p.image_sha = new_sha;
       │       p.backup_version = p.version; p.backup_len = bk_len; p.backup_sha = bk_sha;
       │       p.backup_status = OTA_BACKUP_VALID;
       │       p.resume_offset = 0; p.dl_tid = t.tid;
       │       param_area_save(&p)
       │
       ├─ 9. ota_api_report_status(t.tid, 100)          /* 下载完成 */
       │
       └─ 10. if (OTA_TEST_NO_REBOOT == 0)
                ota_apply_and_reboot(t.tid)
                  ├─ p.boot_flag = TRY_NEW; p.boot_count = 0;
                  ├─ p.ota_tid = t.tid; p.dl_tid = 0;
                  ├─ param_area_save(&p)
                  ├─ 打印 "set TRY_NEW, reboot in 3s..."（3 秒倒计时，期间喂狗）
                  └─ NVIC_SystemReset()                ★ 回到第一段
```

## 4.5 第五段：升级后首启

复位后 BL 走 `TRY_NEW` 分支（见 2.2），拷完跳转。新固件启动后：

```
HAL_F429IGT6_App_2: main()
  └─ osKernelStart()
      └─ freertos.c: StartDefaultTask()
          ├─ printf("[RTOS] StartDefaultTask running, free heap=%u\r\n", ...)
          │
          └─ ota_p2_selftest()                     /* ota_net_switch.c */
              │
              ├─ ota_rollback_test_hook()          /* 默认关闭，直接 return 0 */
              │
              ├─ if (ota_boot_confirm_if_needed() != 0)     ★ 首启专用分支
              │    │
              │    │  /* --- 函数内部 --- */
              │    ├─ param_area_load(&p)
              │    ├─ if (p.boot_flag != TRY_NEW)
              │    │      → ota_confirm_boot_success(); return 0;   /* 普通启动：
              │    │            只同步版本号，不做 30s 等待 */
              │    ├─ 打印横幅：
              │    │      [OTA] #  NEW FIRMWARE FIRST BOOT (boot_flag=TRY_NEW)
              │    │      [OTA] #  version : 1.0.1
              │    │      [OTA] #  build   : Oct 27 2026 00:15:39
              │    │      [OTA] #  silent  : 30000 ms
              │    ├─ while (waited < 30000) { ota_feed_watchdog(); HAL_Delay(100); }   /* 30s 静默 */
              │    ├─ ota_confirm_boot_success()
              │    │      ├─ p.version = OTA_APP_VERSION_NUM      /* 同步版本 */
              │    │      ├─ p.boot_flag = NORMAL
              │    │      ├─ p.boot_count = 0
              │    │      ├─ p.resume_offset = 0; p.backup_status = VALID
              │    │      ├─ p.image_len = 0; memset(p.image_sha, 0, 32)
              │    │      └─ param_area_save(&p)
              │    └─ return 1
              │
              ├─ /* 首启补报（本次新加，解决"平台不知道升级成功"的问题） */
              │   int rb = ota_net_enter()
              │   if (rb == 0) {
              │       ota_report_pending_201()          /* POST /{tid}/status {"step":201} */
              │       ota_report_version()              /* POST /version {"s_version":"1.0.1",...} */
              │       ota_net_exit()                   /* 交还串口 + 重连 MQTT */
              │       printf("[OTA] first-boot report done, back to MQTT\r\n")
              │   }
              │
              └─ /* stage==0 时到这里结束，进入正常业务循环 */
```

**为什么普通启动也要调 `ota_confirm_boot_success()`**（这是个已修的真实 bug）：

```c
if (p.boot_flag != OTA_BOOT_FLAG_TRY_NEW)
{
    /*
     * 普通启动也要同步 param.version = 当前固件版本！
     *
     * 为什么必须有这一步：
     *   板子从 1.0.1 手动烧回 1.0.0 时，参数区里残留的 version 仍是 0x010001。
     *   于是 ota_run_once() 里 "target(1.0.1) <= param.version(0x010001)" 成立，
     *   被判定为"已经是最新版本"，直接补报 201 并返回 1 ——
     *   一个字节都不下载，但平台会收到"升级成功"，造成假成功。
     */
    (void)ota_confirm_boot_success();
    return 0;
}
```

---

# 第 5 章 一次完整升级的时序（真实日志对照）

以下全部来自 **2026-10-27 00:17~00:20** 那次实测（1.0.0 → 1.0.1，含截断重试和首启补报）。

## 5.1 阶段 A：旧固件 1.0.0 启动并连接 MQTT

| 时间 | 日志 | 在干什么 | 对应函数 | 失败会怎样 |
|---|---|---|---|---|
| `00:17:27` | `STM32F429IGT6 OTA BootLoader v1.0` banner | BL 启动 | `boot_run()` | 若这里没打印 → BL 没跑起来，检查 IROM1 和烧录 |
| | `[BL] ext flash JEDEC ID: 0xFFFFFF (unknown)` | 读 W25Q64 ID | `boot_ext_init()` | ID 错不影响后续（只是 SPI 时序临界），见第 8 章 |
| | `[BL] flag=0 count=0 ver=0x010001 img_len=0 bk_st=2` | 读参数区 | `param_area_load()` | 若 magic 错 → `param_area_defaults()`，参数区被重置 |
| | `[BL] target app @0x08010000`<br>`[BL] vec[0]=0x20014E38 vec[1]=0x080103F9` | 校验 A 槽 | `app_jump_validate()` | 若失败 → 走 rollback/recovery/halt |
| | `[BL] jump to app @0x08010000`<br>`[BL] shutdown peripherals, restore RCC to HSI...` | 拆外设 + 恢复时钟 + 跳转 | `boot_system_shutdown()` + `app_jump_execute()` | 时钟没恢复 → App 串口全乱码 |
| `00:17:29` | `LCD OK id=0xFFFF 240x320` / `TP OK type=0x00` | App 启动（屏幕没接，走兜底分支） | `LCD_Init()` | 不影响 OTA |
| | `W25Q64 JEDEC ID = 0xFFFFFF FAIL` | App 自检读 ID 失败 | `w25q16_read_jedec_id()` | 不影响 OTA（后续擦写都有 HAL_OK 检查） |
| | `param load=0 magic=0xA5A5A5A5 flag=0 ver=0x010001` | App 自检读参数区 | `param_area_load()` | — |
| | `SHA256(abc)=OK  MD5(abc)=OK` | 哈希算法自检 | `sha256_*` / `md5_*` | 不过说明算法实现损坏，OTA 必然失败 |
| | `FreeRTOS Start` | 调度器启动 | `osKernelStart()` | — |
| `00:17:29~44` | `AT+RST=1` / `AT+CWMODE=1` / `AT+CWJAP=...` / `AT+MQTTUSERCFG` / `AT+MQTTCONN` / `AT+MQTTSUB` | ESP8266 初始化 + MQTT 连接 | `esp8266_Init()` | 失败则后续收不到属性 |
| `00:17:44` | `[RTOS] defaultTask created, free heap=46984` | 任务创建 | `MX_FREERTOS_Init()` | free heap 46984 = 栈 512 字（若改 1536 则约 42896） |

## 5.2 阶段 B：Update 属性触发

| 时间 | 日志 | 在干什么 | 对应函数 |
|---|---|---|---|
| `00:17:55` | `[OTA] property "Update" received -> request OTA` | 钩子命中 | `ota_task_on_update_ident("Update")` |
| | `[OTA] otaTask created (stack 1536 words)` | 懒创建 OTA 任务 | `xTaskCreate(ota_task_entry, ...)` |
| | `LED=0, count=0`<br>`ID=13, IDENT=Update, VALUE=1, NUM=1` | 原有属性解析日志 | `uart_process()` |
| `00:17:56` | `[MQTT TX] AT+MQTTPUB=...set_reply...{"id":"13","code":200}` | 回执给平台 | `uart_process()` |
| `00:17:57` | `[OTA] OTA net mode READY (USART3 owned by esp01s)` | 通道交接完成 | `ota_net_enter()` |

> **失败会怎样**：如果 `ota_net_enter()` 返回负值（`-1` esp01s_init 失败、`-2` join_ap 失败、`-3` DMA 切换失败），`ota_task_entry()` 会打印 `ota_net_enter FAILED(n), stay in MQTT mode` 并且**保持 MQTT 常连**，不影响业务。

## 5.3 阶段 C：上报版本 + check

| 时间 | 日志 | 在干什么 | 对应函数 |
|---|---|---|---|
| `00:17:58` | `[HTTP] CIPSTART iot-api.heclouds.com:80 ...`<br>`[HTTP] CIPSEND 329 bytes ...`<br>`[HTTP] sent OK, waiting response...` | **补报上次升级的 201**（320 字节的 POST） | `ota_report_pending_201()` → `ota_api_report_status(tid, 201)` |
| `00:17:58` | `[HTTP] CIPSEND 351 bytes ...` | 上报当前版本（351 字节） | `ota_api_report_version()` |
| | `[OTA] version 1.0.0 reported` | 上报成功 | — |
| `00:17:59` | `[HTTP] CIPSEND 276 bytes ...`<br>`[OTA] check resp: {"code":0,"msg":"succ","data":{"target":"1.0.1","tid":1529128,"size":94824,"md5":"41e52aea...","status":1,"type":1}}` | 查询任务 | `ota_api_check_task()` |
| | `[OTA] task found: tid=1529128 size=94824 type=1 target=1.0.1` | 找到任务 | — |

> **`data.type=1` 是关键**：如果平台返回 `type=2`（差分包），代码会拒绝并上报 206。我们的包必须是**整包升级**。

## 5.4 阶段 D：下载（含截断自愈）

| 时间 | 日志 | 在干什么 | 说明 |
|---|---|---|---|
| `00:18:02` | `[OTA] erase staging area (960 KB)...` | 擦暂存区 15×64KB 块 | 耗时约 2.9 秒（`w25q16_erase_block` × 15） |
| | `[OTA] GET (try 0) Range: bytes=0- / 94824` | 第 1 轮 Range 请求 | — |
| | `[HTTP] CIPSEND 283 bytes ... sent OK, waiting response...` | 发请求 | — |
| `00:18:27` | `[OTA] try 0: ret=-2 status=206 len=94824 errno=0 got=93511/94824` | **第 1 轮超时**，只收到 93511/94824 | `ret=-2` = `ESP_TIMEOUT` |
| | `[OTA] try 0 TRUNCATED: 93511 of 94824 bytes (data shifted!)` | **检测到中途截断** | 实收 < `Content-Length` → 数据整体前移 |
| | `[OTA] GET (try 1) Range: bytes=93511- / 94824` | 第 2 轮：从断点续传 | 只补尾部 |
| `00:18:27` | `[OTA] try 1: ret=0 status=206 len=1313 errno=0 got=94824/94824` | 补齐到 94824 | 字节数够了 |
| | `[OTA] MD5 mismatch (expect 41e52aea...)` | **MD5 失败** | 因为中间缺了 1313 字节，后面的数据整体前移 |
| | `[OTA] (skip fail report step=205)` | **不上报平台**（`OTA_REPORT_FAIL_TO_PLATFORM=0`） | ★ 保住任务不被平台关掉 |
| | `[OTA] verify FAILED, erase staging and re-download from 0 ...` | **兜底：擦除整体重下** | 哈希上下文不能回退，只能从头来 |
| `00:18:30` | `[OTA] erase staging area (960 KB)...` | 再擦一次 | — |
| | `[OTA] GET (try 0) Range: bytes=0- / 94824` | 第 2 遍下载 | — |
| `00:18:39` | `[OTA] try 0: ret=0 status=206 len=94824 errno=0 got=94824/94824` | **一次收全** | 这次没有截断 |
| | `[OTA] download OK: 94824 bytes, md5=41e52aea...` | **MD5 通过** | — |
| | `[OTA] re-download OK on 2nd pass` | 兜底生效 | — |

> **这一段是整个 OTA 最有价值的部分**：它证明了"截断 → 续传补尾 → MD5 抓住 → 擦除重下 → 成功"这条自愈链是完整可用的。虽然要多花一次下载时间，但**数据正确性有绝对保证**。

## 5.5 阶段 E：备份 + 写参数 + 复位

| 时间 | 日志 | 在干什么 | 对应函数 |
|---|---|---|---|
| `00:18:43` | `[OTA] current app length = 372736 bytes` | 从 A 槽反扫出旧固件实际长度 | `ota_probe_run_len()` |
| | `[OTA] backup done: 372736 bytes` | 旧固件备份到片外 BACKUP 区完成 | `ota_backup_current_app()` |
| | `[OTA] param saved: image_len=94824 backup_len=372736` | 参数区写成功 | `param_area_save()` |
| | `[HTTP] CIPSEND 329 bytes ...` | 上报 `step=100`（下载完成） | `ota_api_report_status(tid, 100)` |
| `00:18:44` | `[OTA] set TRY_NEW, reboot in 3s...` | 置 TRY_NEW + 3 秒倒计时 | `ota_apply_and_reboot()` |
| `00:18:47` | （复位） | `NVIC_SystemReset()` | — |

> **如果这里断电**：参数区已经是 `image_len=94824 + image_sha + backup_status=VALID`，`boot_flag` 还是 NORMAL。下次上电 BL 走 NORMAL 分支直接跑旧固件 —— **安全姿态**，不会变砖。重新下发 Update 会走断点续传（`dl_tid` 相同且 `backup_status=DOWNLOADING` 才续传，此时已 VALID 所以会全量重下）。

## 5.6 阶段 F：BootLoader 执行 TRY_NEW

| 时间 | 日志 | 在干什么 | 对应函数 |
|---|---|---|---|
| `00:18:47` | `[BL] flag=1 count=0 ver=0x010000 img_len=94824 bk_st=2` | 读到 TRY_NEW + 待装镜像 | `param_area_load()` |
| | `[BL] >>> TRY_NEW` | 进入升级分支 | `boot_run()` |
| | `[BL] verify staging SHA-256 (len=94824)...` | **二次校验暂存区** | `boot_verify_ext_image()` |
| | `[BL] staging OK, copy staging -> A` | 校验通过，开始拷贝 | `boot_copy_ext_to_run()` |
| `00:18:49` | `[BL] target app @0x08010000`<br>`[BL] vec[0]=0x20014E38 vec[1]=0x080103F9` | 校验新拷进 A 槽的镜像 | `app_jump_validate()` |
| | `[BL] jump to app @0x08010000` | 跳转 | `app_jump_execute()` |

> **如果 SHA-256 不过**：BL 直接 `boot_do_rollback()` → 用备份区覆盖 A 槽 → 跑回 1.0.0。这就是"为什么必须有本地 SHA-256"的实际价值 —— 它拦住了"App 说 OK 但 W25Q64 实际写坏了"的情况。

## 5.7 阶段 G：新固件 1.0.1 首启

| 时间 | 日志 | 在干什么 | 对应函数 |
|---|---|---|---|
| `00:18:52` | `param load=0 magic=0xA5A5A5A5 flag=1 ver=0x010000` | 新固件读到 `flag=1`（TRY_NEW） | `param_area_load()` |
| | `[OTA] #  NEW FIRMWARE FIRST BOOT  (boot_flag=TRY_NEW)`<br>`[OTA] #  version : 1.0.1`<br>`[OTA] #  build   : Oct 27 2026 00:15:39`<br>`[OTA] #  silent  : 30000 ms` | **首启横幅** | `ota_boot_confirm_if_needed()` |
| `00:19:07~37` | （30 秒静默，无日志） | 静默窗口，期间喂狗 | `while (waited < 30000) { ota_feed_watchdog(); HAL_Delay(100); }` |
| `00:19:37` | `[OTA] boot confirmed, flag=NORMAL, ver=0x010001` | **固化 NORMAL** | `ota_confirm_boot_success()` |
| | `[OTA] >>> BOOT CONFIRMED: flag=NORMAL, version=1.0.1` | 确认完成 | — |
| `00:19:39` | `[OTA] >>> enter OTA net mode`<br>`[OTA] OTA net mode READY (USART3 owned by esp01s)` | 再切一次通道（为了补报） | `ota_net_enter()` |
| | `[HTTP] CIPSEND 329 bytes ...`<br>`[OTA] reported 201 to platform` | **补报 201** | `ota_report_pending_201()` |
| | `[HTTP] CIPSEND 351 bytes ...`<br>`[OTA] first-boot: version 1.0.1 reported` | **上报新版本号** | `ota_report_version()` |
| | `[OTA] <<< exit OTA net mode, reconnect MQTT`<br>`[OTA] first-boot report done, back to MQTT` | 交还串口 + 重连 MQTT | `ota_net_exit()` |
| `00:19:55~` | `AT+RST=1` ... `AT+MQTTSUB=...`<br>`[OTA] back to MQTT mode` | MQTT 恢复 | `esp8266_Init()` |
| `00:20:09` | `LED=0, count=50`<br>`ID=17, IDENT=count, VALUE=50, NUM=50`<br>`[MQTT TX] ...set_reply...{"id":"17","code":200}` | 业务恢复 | `uart_process()` |

**至此一次完整升级结束**：平台显示任务成功、设备版本 1.0.1、业务功能正常。

## 5.8 时序总览（时间消耗）

| 阶段 | 耗时 | 主要开销 |
|---|---|---|
| BL 启动 + 校验 + 跳转 | ≈ 2.5 s | Flash 读取、串口打印 |
| App 启动 + ESP8266 初始化 + MQTT | ≈ 15 s | `esp8266_Init()` 里 6 条 AT + 固定延时 |
| Update 触发 → 通道交接 | ≈ 2 s | `AT+MQTTDISC` + 0.8s 等待 + `esp01s_init` + `join_ap` |
| 上报版本 + check | ≈ 1 s | 两次短 HTTP |
| 擦除暂存区 | ≈ 3 s | 15 × 64KB 块擦除 |
| 下载（第 1 遍，截断） | ≈ 25 s | 主要是等超时（实际数据只需 9 秒） |
| 下载（第 2 遍，成功） | ≈ 9 s | 94824 字节 @ 115200 ≈ 8.2 s |
| 备份旧固件 | ≈ 4 s | 372736 字节读写 + SHA-256 |
| 打印 + 上报 100 + 3 秒倒计时 | ≈ 4 s | — |
| BL 校验 + 拷贝 | ≈ 4.5 s | 94824 字节 SHA-256 + 擦 S4~S11 + 拷贝 |
| 新固件启动 + MQTT | ≈ 15 s | 同 App 启动 |
| **30 秒静默窗口** | ≈ 30 s | 刻意等待 |
| 补报 201 + 版本 + 重连 MQTT | ≈ 20 s | 通道切换 2 s + `esp8266_Init` 14 s |
| **合计** | **≈ 2 分 20 秒** | 其中 55 秒是"刻意的/可优化"的（30s 静默 + 25s 超时等待） |

---

# 第 6 章 OneNET 平台 OTA 任务添加流程

> 控制台界面为中文，以下按实际菜单路径写。

## 6.1 前置条件

| 项目 | 值 |
|---|---|
| 产品名称 | `OTA_Test` |
| 产品 ID | `FRfr9m3Ppg` |
| 设备名称 | `OTA_Test_drive` |
| 设备鉴权信息（token） | 见第 7 章 |

## 6.2 第 1 步：物模型里加可写属性 `Update`

`产品开发` → `物模型管理` → `属性` → `添加属性`

| 字段 | 值 | 说明 |
|---|---|---|
| 功能名称 | `Update` | **必须与代码里 `strcmp(ident, "Update")` 完全一致**（大小写敏感） |
| 标识符 | `Update` | ★ 这个是关键，就是 `ident` |
| 数据类型 | `int32` | 代码里按整数解析 |
| 取值范围 | `0 ~ 999` | 任意，够用即可 |
| 读写类型 | **可写（读写）** | ★ 必须可写，否则平台下发不了 |
| 单位/描述 | 随意 | — |

> **`Update` 是升级触发器**：平台下发 `Update=1` 时，设备收到属性 → `ota_task_on_update_ident()` → 启动升级。值本身不重要（代码只看 `ident`），但**不要用 0 触发**（`ota_task_request(0)` 会被忽略，0 被约定为"空闲/已处理"语义）。

## 6.3 第 2 步：上传升级包

左侧菜单 `远程升级` → `升级包管理` → `添加升级包`

| 字段 | 值 | 说明 |
|---|---|---|
| 升级包名称 | 随意，如 `OTA_1` | — |
| **版本号** | `1.0.1` | ★ 必须**严格大于**设备当前 `OTA_APP_VERSION_STR` |
| 所属产品 | `OTA_Test` | — |
| 升级包类型 | **整包（SOTA）** | ★ 不能选差分包（代码拒绝 `type != 1`） |
| 上传文件 | `HAL_F429IGT6_App_2\MDK-ARM\HAL_F429IGT6.bin` | ★ 必须是 bin，由 `fromelf --bin` 生成 |
| 支持硬件/模组 | 按产品配置，一般留空 | — |

上传成功后平台会计算并显示 `MD5` 和文件大小 —— 这两个值就是 `check` 接口会返给设备的值，**可以用来核对 bin 是否传对**。

## 6.4 第 3 步：创建升级任务

左侧 `远程升级` → `升级任务` → `新建升级任务`

| 字段 | 建议值 | 说明 |
|---|---|---|
| 任务名称 | 如 `Test` | — |
| 选择升级包 | 上一步创建的 `OTA_1`（1.0.1） | — |
| **升级策略** | **静态升级** | 代码不处理"动态升级"的额外流程 |
| **重试策略** | **调试期建议 10 次** | ★ 见 6.7 的警告 |
| 升级超时时间 | 1440 分钟（默认） | — |
| 目标设备 | 勾选 `OTA_Test_drive` | — |

## 6.5 第 4 步：设备联机 → 下发 Update 触发

1. 板子上电，等串口出现 `AT+MQTTSUB=...` 和 `[RTOS] StartDefaultTask running`（此时 MQTT 已连上、已订阅属性）
2. 平台 `设备详情` → `物模型` / `属性` → 找到 `Update` → `下发`（或"设置"）
3. 输入 `1` → 确定

设备侧应该立刻看到：

```
[OTA] property "Update" received -> request OTA
[OTA] otaTask created (stack 1536 words)
```

## 6.6 第 5 步：观察状态

| 观察位置 | 应该看到 |
|---|---|
| `升级任务` → 批次详情 | 待升级(1) → 升级中(1) → **升级成功(1)** |
| `升级完成设备列表` | 源版本号 `1.0.0`、当前版本号 `1.0.1`、升级状态「升级成功」 |
| `设备详情` → 设备信息 | 当前版本号变成 `1.0.1` |

> 注意：设备版本号的更新依赖**设备主动上报**（`POST /version`）。如果平台显示的还是旧版本，去串口看有没有 `[OTA] first-boot: version 1.0.1 reported`。

## 6.7 ★ 调试期最重要的注意事项

我们的代码里：

```c
#define OTA_REPORT_FAIL_TO_PLATFORM   0
```

**含义：升级失败时不上报 205/206/207。**

**为什么必须是 0**：平台会把这些失败码记为该设备"升级失败"，并**消耗任务里配置的重试次数**。实测调试几轮之后，任务的重试额度耗尽，平台把任务关掉，`check` 就开始返回：

```json
{"code":12012,"msg":"not exist","request_id":"..."}
```

**这个错误不是代码 bug，是平台侧任务已经被关掉了**（表现为串口打印 `[OTA] no task on platform`）。只能到控制台**重新创建升级任务**才能继续测试。

所以：

| 阶段 | `OTA_REPORT_FAIL_TO_PLATFORM` | 重试策略 |
|---|---|---|
| 调试期 | **0**（不上报，随便怎么失败都不打死任务） | 10 次 |
| 量产 | 1（正常上报，让平台知道真实失败原因） | 按业务定 |

## 6.8 常见平台返回码

| code | 含义 | 出现时机 / 处理 |
|---|---|---|
| `0` | 成功 | `data` 里有 `tid/size/md5/type/target` |
| `10403` | `auth failed: pid not equal` | URL 里的 pid 与 token 里的 res 不一致（常见于把产品名 `OTA_Test` 当成 pid 用） |
| `12001` | `parameter required` | `check` 少了 `type` 参数 |
| `12004` | `format error` | `type=3` 等非法值 |
| `12010` | `task type error` | **`type=2` 不被支持**（我们传的是 1） |
| `12012` | `not exist` | 任务已被消费/关闭，需重建 |

---

# 第 7 章 关键参数速查

## 7.1 平台身份

| 项 | 值 |
|---|---|
| 产品名称（显示用） | `OTA_Test` |
| **产品 ID（URL 用）** | `FRfr9m3Ppg` |
| 设备名称 | `OTA_Test_drive` |
| 设备 token | `version=2018-10-31&res=products%2FFRfr9m3Ppg%2Fdevices%2FOTA_Test_drive&et=1809960178&method=sha1&sign=kLYbgpjIMWGT9klNJKWXX15G3GI%3D` |
| token 有效期 | `et=1809960178` ≈ **2027-05-10**（到期需用设备密钥重算） |

> ⚠ **产品名 ≠ 产品 ID**。URL 里必须用 `FRfr9m3Ppg`，写成 `OTA_Test` 会返回 `10403 auth failed: pid not equal`。这个坑我在 PC 端实测时踩过。

## 7.2 接口

| 项 | 值 |
|---|---|
| 域名 | `iot-api.heclouds.com` |
| 端口 | `80`（明文 HTTP，实测可用；**不是 443**） |
| 鉴权头 | `Authorization: <token>` |
| 任务类型参数 | `type=1`（**唯一合法值**） |

## 7.3 四个接口的 URL 与 body

| 用途 | 方法 | URL | Body |
|---|---|---|---|
| 上报版本 | POST | `/fuse-ota/{pid}/{dev}/version` | `{"s_version":"1.0.0","f_version":"1.0.0"}` |
| 查询任务 | GET | `/fuse-ota/{pid}/{dev}/check?type=1&version=1.0.0` | — |
| 下载固件 | GET | `/fuse-ota/{pid}/{dev}/{tid}/download` | — |
| 上报状态 | POST | `/fuse-ota/{pid}/{dev}/{tid}/status` | `{"step":N}` |

`{pid}` = `FRfr9m3Ppg`，`{dev}` = `OTA_Test_drive`，`{tid}` = check 返回的任务号。

## 7.4 Range 请求头写法

**必须始终带 Range，即使从头下载**（`offset=0` 时写 `Range: bytes=0-`）：

```
GET /fuse-ota/FRfr9m3Ppg/OTA_Test_drive/1529128/download HTTP/1.1
Host: iot-api.heclouds.com
Authorization: version=2018-10-31&res=...&sign=...
Range: bytes=0-
Connection: close
```

服务端响应（带 Range 时）：

```
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-94823/94824
Content-Length: 94824
Ota-Errno: 0
Content-Disposition: attachment; filename="OTA1"
```

> **为什么必须带 Range**：实测不带 Range 时，MCU 侧出现过整包下载 30 秒超时（`r=-2`，服务端一个字节都不回）；带上 Range 后一切正常。怀疑服务端对"无 Range 的整包请求"走了不同路径（分块/流式）。带 Range 时 `offset=0` 语义完全等价于"从头下"。

## 7.5 状态码（step）语义

| step | 含义 | 代码位置 |
|---|---|---|
| `100` | 下载完成 | `ota_run_once()` 第 9 步 |
| `201` | **升级成功** | `ota_report_pending_201()`、首启补报、`ota_confirm_boot_success()` 后 |
| `205` | MD5 校验失败 | `ota_download()` 里 MD5 不匹配时（默认被 `OTA_REPORT_FAIL_TO_PLATFORM=0` 屏蔽） |
| `206` | 升级失败（差分包/降级/长度不符） | 多处（默认屏蔽） |
| `207` | 下载失败（`Ota-Errno>0`） | `ota_download()` 里（默认屏蔽） |

## 7.6 `Ota-Errno` 响应头

下载接口会返回 `Ota-Errno: N`：

| 值 | 含义 |
|---|---|
| `0` | 正常 |
| `>0` | 平台侧错误（如任务状态不对），App 会上报 `step=207` 并放弃本次下载 |

## 7.7 分片/擦除相关常量

| 常量 | 值 | 定义位置 |
|---|---|---|
| 片内擦除单位 | S0~S3=16KB，S4=64KB，S5~S11=128KB | F429 硬件 |
| 片外擦除单位 | 扇区 4KB、块 64KB | W25Q64 硬件 |
| `OTA_APP_IMAGE_MAX_SIZE` | `0xF0000`（960KB） | `partition.h` |
| `OTA_DL_CHUNK` | `256U` | `ota_config.h` |
| `OTA_CHECKPOINT_INTERVAL` | `16384U` | `ota_config.h` |
| `OTA_STARTUP_CONFIRM_MS` | `30000U` | `ota_config.h` |
| `OTA_BOOT_MAX_TRIES` | `3U` | `ota_params.h` |
| `ESP_RX_RING_SIZE` | `4096U` | `esp01s.h` |
| `ESP_DATA_BUF_SIZE` | `8192U` | `esp01s.h` |
| `FROMELF 输出 bin` | `MDK-ARM\HAL_F429IGT6.bin` | Keil User 标签 |

---

# 第 8 章 踩坑与排查表

按出现顺序排列，每条都是**真实遇到并解决**的。

## 8.1 `[RTOS] STACK OVERFLOW in task defaultTask`

| 项 | 内容 |
|---|---|
| **现象** | 进 OTA 流程后立刻打印 `[RTOS] STACK OVERFLOW in task defaultTask`，任务卡死 |
| **根因** | `defaultTask` 栈只有 512 字（2KB）。`defaultTask` → `ota_p2_selftest` → `ota_run_once` → `ota_download` → `http_client_request_once` → `esp01s_*` 这条链的栈峰值超过 3KB。其中 `http_client_request_once()` 里 `req[512] + line[160] + chunk[1024]` ≈ **1.7KB 全在栈上**，是主凶 |
| **解决** | ① `http_client.c` 的三个缓冲改 `static`（省 1.7KB）；② `ota_engine.c` 各函数的 `path/extra/rbuf/resp/ota_param_t/md5_digest` 改 `static`（省约 1.5KB）；③ `defaultTask` 栈 512 → **1536 字** |
| **教训** | 在 FreeRTOS 任务里跑大缓冲逻辑，**先算栈**。静态化是安全的（同一时刻只有一个任务跑 HTTP，无重入），而且比加栈更根本 |

## 8.2 下载 `r=-2`（ESP_TIMEOUT），30 秒超时

| 项 | 内容 |
|---|---|
| **现象** | `[OTA] GET .../download (offset=0)` 之后 30 秒无响应，`[OTA] http download FAILED, r=-2` |
| **根因** | 不带 `Range` 头时，服务端对该端点不返回任何数据。30 秒 = `http_client` 读状态行时"跳过最多 8 行杂讯"× 3 秒 + 建连/发送 |
| **解决** | **始终带 `Range: bytes=<offset>-`**（`offset=0` 也带），与 PC 端验证成功的请求完全一致 |
| **验证** | 改完后立刻变成 `status=206 len=94824`，说明服务端正常响应了 |

## 8.3 `TRUNCATED: x of y (data shifted!)` —— 下载必截断

| 项 | 内容 |
|---|---|
| **现象** | 每次都下到 98% 左右停住：`try 0: ret=-2 ... got=93511/94824`，`TRUNCATED: 93511 of 94824 bytes (data shifted!)` |
| **根因** | ESP8266 在 115200 批量接收时，偶发一次 UART overrun（或 DMA 环形缓冲覆盖）就会让 `+IPD,<len>:` 帧头错位，解析器进入"等一个永远收不满的长度"状态 → 后续数据不再输出 → 超时。`esp01s.c` 的 `IPD_STATE_DATA` 分支在 `s_data` 满时会**丢弃字节但仍递减 `s_ipd_len`**，所以不会卡死，但会静默丢数据 |
| **解决（分两步）** | ① 加"续传补尾"：`try 1` 用 `Range: bytes=93511-` 把尾巴补齐；② 但补尾后 **MD5 仍然错**，因为丢的字节在**中间**，后面数据整体前移（哈希上下文不能回退）→ 加**兜底：校验失败就擦除暂存区、从头完整重下** |
| **效果** | 每次升级都会"下载两遍"，但**数据正确性 100% 保证**。3 次实测全部兜住 |
| **待优化** | 窗口化下载（见第 9 章） |

## 8.4 `MD5 mismatch` 但字节数完全正确

| 项 | 内容 |
|---|---|
| **现象** | `got=94824/94824` 字节数对上了，但 `MD5 mismatch` |
| **根因** | 8.3 的直接后果：中间丢了一小段，续传把尾部从"错误的偏移"开始补，于是**长度对、内容错** |
| **判断依据** | 看 `TRUNCATED` 那行 —— 如果出现，就一定是这个问题 |
| **解决** | 同 8.3 ②，擦除重下 |

## 8.5 `{"code":12010}` / `{"code":12004}` / `{"code":12001}`

| 项 | 内容 |
|---|---|
| **现象** | `check` 返回 `12010 task type error` 或 `12004 format error` 或 `12001 parameter required` |
| **根因** | `type` 参数错误。实测：`type=2` → `12010`；`type=3` → `12004`；不传 `type` → `12001`；**只有 `type=1` 合法** |
| **解决** | `ota_config.h` 里 `#define OTA_TASK_TYPE 1` |

## 8.6 `{"code":12012,"msg":"not exist"}` —— 任务消失

| 项 | 内容 |
|---|---|
| **现象** | 之前都好好的，突然 `check` 返回 `12012`，串口打印 `[OTA] no task on platform` |
| **根因** | 调试期间多次上报 `205/206/207`，平台把该设备记为"升级失败"并**消耗任务的重试次数**；次数耗尽后任务被平台关闭 |
| **解决** | ① 到控制台**重新创建升级任务**；② 代码里 `OTA_REPORT_FAIL_TO_PLATFORM = 0`，调试期失败只打串口、不动平台状态 |
| **教训** | 调试 OTA 时**一定要先关掉失败上报**，否则每失败一次就少一条命 |

## 8.7 版本号脏值导致"假升级成功"

| 项 | 内容 |
|---|---|
| **现象** | 平台显示"升级成功"，但串口里一个字节都没下载：`[OTA] task found ... [HTTP] CIPSEND 329 bytes ... ota_run_once() = 1` |
| **根因** | 板子之前跑过 1.0.1（参数区 `version=0x010001`），后来**手动烧回 1.0.0**，参数区里的版本号没有跟着回来。于是 `ota_run_once()` 判断 `target(1.0.1) <= p.version(0x010001)` 成立 → 认为"已经是最新版本" → 直接补报 201 并返回 1 |
| **解决** | `ota_boot_confirm_if_needed()` 在 `boot_flag != TRY_NEW`（普通启动）时不再直接 return，而是**先调 `ota_confirm_boot_success()` 把 `param.version` 同步成当前固件版本**，再返回 |
| **验证** | 修复后：BL 先读到旧值 `ver=0x010001`，App 跑完后 BL 读到 `ver=0x010000` —— 证明同步生效 |

## 8.8 `W25Q64 JEDEC ID = 0x000000 / 0xFFFFFF  FAIL`

| 项 | 内容 |
|---|---|
| **现象** | 偶发，BL 和 App 都可能：`[BL] ext flash JEDEC ID: 0xFFFFFF (unknown)`、`W25Q64 JEDEC ID = 0x000000 FAIL`。重上电有时又正常 `0xEF4017` |
| **根因** | SPI2 在 `SPI_BAUDRATEPRESCALER_2` = 22.5MHz（APB1 45MHz / 2）下时序临界，与杜邦线长度/上拉/电源稳定性有关。`0xFFFFFF` 是 MISO 浮空拉高，`0x000000` 是被拉低 |
| **解决** | `Core/Src/spi.c` 里 `hspi2.Init.BaudRatePrescaler` 从 `_2` 改成 **`_4`（11.25MHz）** |
| **注意** | `spi.c` 在 CubeMX 生成区，**必须在 CubeMX 里也改成 4**，否则下次 `GENERATE CODE` 会被改回来（这个坑我们踩过一次：改了代码，重新生成后又被覆盖） |
| **影响** | 不影响 OTA 正确性（下载/备份的每一次 `w25q16_write`/`erase_block` 都有 `HAL_OK` 检查，失败会返回错误码），只是启动自检会报红 |

## 8.9 串口乱码 `(鏃х増)`

| 项 | 内容 |
|---|---|
| **现象** | `[RB] #  param version   : 0x010000  (鏃х増)` |
| **根因** | 源码文件是 UTF-8，`printf` 里的中文"旧版"输出的是 UTF-8 字节；串口助手按 GBK/ANSI 解码 → 乱码 |
| **解决** | **代码里不要用中文 printf**。已把该处改成 `(old)`，并对 `OTA\*.c` 全量扫描确认无中文 `printf` 残留 |
| **规则** | 固件源码里的所有日志一律用英文/ASCII；中文只放在注释里 |

## 8.10 CubeMX 重新生成把改动"吃掉"

| 项 | 内容 |
|---|---|
| **现象** | 明明改过 SPI 预分频和 `defaultTask` 栈，重新编译后 `free heap` 从 42888 变回 46984、JEDEC 又开始读错 |
| **根因** | 在 `GENERATE CODE` 前手工改了 `Core/Src/spi.c`（prescaler）和 `Core/Src/freertos.c`（`osThreadDef(..., 1536)`），这两个位置都在**生成区**，重新生成时被覆盖 |
| **诊断技巧** | 看 `[RTOS] defaultTask created, free heap=XXXXX`：<br>`46984` = 栈 512 字<br>`42888` = 栈 1536 字<br>差值 4096 字节 = (1536−512)×4，一眼就能判断栈被改回去了 |
| **解决** | 所有需要长期生效的配置改动，**必须落到 `.ioc` 里**：<br>① SPI2 → Prescaler = 4<br>② FreeRTOS → defaultTask → Stack Size (Words) = 1536 |
| **原则** | 代码改动只放在**不会被生成的地方**：`Boot\`、`OTA\`、`bsp\`、`App\` 目录，或 `/* USER CODE BEGIN/END */` 区块内 |

## 8.11 首个固件提交 `param_area_save` 后 `check` 仍无任务

| 项 | 内容 |
|---|---|
| **现象** | `param_area_save` 成功，但 `check` 返回 `code:0` 没有 `tid` |
| **根因** | 平台上的任务已经因为 `201` 上报而变成"升级成功"并被消费 |
| **解决** | 重建任务。这是**正常行为**，不是 bug —— 一个任务对一台设备只生效一次 |

## 8.12 排查顺序建议

遇到"升级不动"时，按这个顺序看串口：

```
1. 有没有 [BL] jump to app？            → 没有则 BL 卡住了
2. 有没有 AT+MQTTSUB=...？               → 没有则 esp8266_Init 失败
3. 有没有 [OTA] property "Update"？      → 没有则平台没下发 / Update 属性不可写 / 订阅失败
4. 有没有 [OTA] OTA net mode READY？     → 没有则通道交接失败（看返回码 -1/-2/-3）
5. 有没有 [OTA] version X reported？     → 没有则 POST /version 失败
6. check resp 的 code 是多少？           → 12012 任务没了 / 12010 type 错了
7. 有没有 task found？                   → 没有则是"已是最新"或"无任务"
8. 有没有 TRUNCATED？                    → 有则下载丢字节（会自动兜底）
9. 有没有 MD5 mismatch？                 → 有则会自动重下
10. 有没有 set TRY_NEW？                 → 没有则前面某步失败
11. 复位后 [BL] flag 是多少？            → 不是 1 则参数区没写进去
12. [BL] staging OK？                    → 不是则 SHA-256 失败（会回滚）
13. 新固件横幅有没有？                    → 没有则拷贝失败/跳转失败
14. [OTA] boot confirmed？               → 没有则 30s 内崩溃了
15. 有没有 reported 201 / version reported？ → 没有则首启补报失败
```

---

# 第 9 章 待优化项

按优先级排序。

## 9.1 【高】窗口化下载：根治"每次下载两遍"

**现状**：因为丢字节会导致数据整体前移，而哈希上下文不能回退，所以只能"整包重下"。每次升级都要下两遍，耗时翻倍。

**方案**：把整包拆成固定窗口（建议 8KB）逐窗下载。

```
窗口 = 8192 字节
for (start = 0; start < total; start += WINDOW)
{
    for (retry = 0; retry < 3; retry++)
    {
        GET Range: bytes=start-(start+WINDOW-1)
        收进 win_buf[WINDOW]
        if (收到字节数 == 本窗应有字节数)   /* 收满了 */
        {
            w25q16_write(EXT_STAGING + start, win_buf, n);   /* 落盘 */
            md5_update(&md5, win_buf, n);                    /* ★ 只有收满才更新哈希 */
            sha256_update(&sha, win_buf, n);
            break;                                          /* 本窗完成 */
        }
        /* 没收满：win_buf 丢弃，重下本窗（哈希没动过，安全） */
    }
}
```

**关键点**：

1. **哈希只在"整窗收满"后更新** → 窗口内失败可以安全重试，不污染哈希；
2. 截断最多损失一个窗口（8KB ≈ 1 秒），**永远不需要整包重下**；
3. 窗口大小要和 `ESP_RX_RING_SIZE`（4096）匹配 —— 建议窗口 ≤ 8KB，超过环形缓冲大小会更容易丢；
4. 代价：HTTP 请求次数从 1 次变成 `total/8192` 次（95KB → 12 次），每次多约 0.3 秒的 AT 往返开销，总时间增加约 3.6 秒，但**省掉一整遍重下（25 秒）**，净赚。

**改动范围**：仅 `ota_engine.c` 的 `ota_download()` / `ota_download_try()` / `dl_on_data()`，约 60 行。其它模块不动。

## 9.2 【高】在 CubeMX 里固化两项配置

| 配置 | 位置 | 值 | 原因 |
|---|---|---|---|
| SPI2 Prescaler | Connectivity → SPI2 → Parameter Settings → Clock Parameters | `4`（11.25 MBits/s） | 修复 W25Q64 JEDEC 偶发读错 |
| defaultTask Stack Size | Middleware → FREERTOS → Tasks and Queues → defaultTask | `1536` Words | 防栈溢出（虽然现在靠静态化苟住了，但余量太薄） |

**必须做**，否则每次 `GENERATE CODE` 都会把这两项改回去。

## 9.3 【中】LCD 接回后的升级进度页

屏幕目前是断开的（`LCD ID:ffff` 走 FMC 兜底分支）。接回后可以显示升级进度：

```c
/* ota_engine.h 已提供快照接口，UI 每 200ms 读一次即可 */
void ota_get_status(ota_status_t *st);

/* 典型用法 */
ota_status_t st;
ota_get_status(&st);
if (st.state == OTA_ST_DOWNLOADING) {
    LCD_ShowString(20, 100, 200, 16, 16, (uint8_t *)"OTA downloading...");
    /* 画进度条：st.progress (0~100) */
}
```

`ota_status_t` 里已包含 `state / progress / downloaded / total / tid / target[24] / msg[64]`，够画一个完整的进度页。

**建议的 UI 状态映射**：

| state | 显示文案 |
|---|---|
| `OTA_ST_IDLE` / `OTA_ST_NONE` | 正常主界面 |
| `OTA_ST_CHECKING` | "检查更新…" |
| `OTA_ST_DOWNLOADING` | "下载中 45%" + 进度条 |
| `OTA_ST_VERIFYING` | "校验中…" |
| `OTA_ST_BACKUPING` | "备份中…" |
| `OTA_ST_READY_REBOOT` | "即将重启升级" |
| `OTA_ST_ERROR` | "升级失败：" + `msg` |

## 9.4 【中】`Update` 回写 0 的报文格式待确认

代码里实现在 `ota_task.c`：

```c
uart_printf(&huart3,
            "AT+MQTTPUB=0,\"$sys/%s/%s/thing/property/post\","
            "\"{\\\"id\\\":\\\"1\\\",\\\"params\\\":{\\\"Update\\\":0}}\",0,0\r\n",
            OTA_PID, OTA_DEV);
```

**目的**：升级结束（未重启）后把 `Update` 回写 0，避免下次上电因属性值仍是 1 而产生混淆。

**待确认**：这个报文格式（`property/post` + `{"id":"1","params":{"Update":0}}`）是否被平台接受。串口里能看到 `[OTA] posted Update=0 (idempotent)`，但**是否真的改了云端属性值需要在控制台确认**。

**如果不生效**，备选方案：

1. 改用 `property/set_reply` 风格（平台可能不允许设备直接改可写属性）；
2. 干脆不回写，改为在设备侧用 RAM 标志做"本次上电已处理"（重启后仍会读到 1，但触发是**事件驱动**的，不会自动触发，所以实际影响很小）。

> 注意：**触发是"属性下发事件"驱动的，不是"读当前值"驱动的**。所以即使 `Update` 一直停在 1，也不会导致设备自己反复升级 —— 只有平台**再次下发**才会触发。这一点已实测确认（回滚后 1.0.0 稳定运行，没有自动再升级）。

## 9.5 【低】缩短退出 OTA 模式的重连时间

现在 `ota_net_exit()` 要 14 秒，因为 `esp01s_init()` 里有 `AT+RST`，复位会清掉 MQTT 配置（`USERCFG/CONN/SUB` 都在模块 RAM 里），所以退出时必须重跑 `esp8266_Init()`。

**优化方案**：给 `esp01s_init()` 加一个 `esp01s_init_no_reset()` 变体（跳过 `AT+RST`），只在**首次**进入时复位，后续进出都不复位。这样退出时间可降到 1~2 秒。

**风险**：跳过 `AT+RST` 后模块可能残留上一次的 TCP 状态，需要用 `AT+CIPSTATUS` 先确认已 `CLOSED`。

## 9.6 【低】`ota_init_recovery_if_empty()` 的耗时

首次进 OTA 时会做"出厂镜像固化"：擦 15 个 64KB 块 + 拷 372KB + SHA-256，约 10 秒。只做一次（`p.recovery_len != 0` 后跳过），但会拖慢**第一次**升级。

**可选优化**：把它挪到"设备空闲时后台做"，或者在工厂生产环节预先把镜像烧进 W25Q64。

## 9.7 【低】平台 token 有效期

`et=1809960178` ≈ 2027-05-10。到期后 MQTT 和 HTTP 都会 `10403 auth failed`。

**建议**：在 `.ioc` 之外记一个日历提醒，或者改用"设备密钥 + HMAC 动态计算 token"的方案（一劳永逸，但要改协议）。

## 9.8 【低】`TODO`：`ota_port.c` 的 `USART1_Printf()`

这个函数是 P3 阶段为兼容 f407 原设计留的，现在引擎统一用 `printf`（走 `usart.c` 的 `fputc` 重定向）。如果确认没有地方用它，可以删掉这个文件。

---

# 文件清单索引

## 一、BootLoader 工程

| 路径 | 用途 |
|---|---|
| `HAL_F429IGT6_bootloard\Boot\partition.h` | 分区地址定义（★ 与 App 必须一致） |
| `HAL_F429IGT6_bootloard\Boot\ota_params.h` | 参数区结构体（★ 与 App 必须一致） |
| `HAL_F429IGT6_bootloard\Boot\crc32.c/.h` | 参数区 CRC32 |
| `HAL_F429IGT6_bootloard\Boot\sha256.c/.h` | 落地校验哈希 |
| `HAL_F429IGT6_bootloard\Boot\flash_if.c/.h` | 片内 Flash 擦写读 |
| `HAL_F429IGT6_bootloard\Boot\param_area.c/.h` | 参数区 32 槽轮转 |
| `HAL_F429IGT6_bootloard\Boot\boot_config.h` | BL 编译开关 |
| `HAL_F429IGT6_bootloard\Boot\boot_log.c/.h` | 串口日志 |
| `HAL_F429IGT6_bootloard\Boot\boot_wdg.c/.h` | 寄存器级 IWDG |
| `HAL_F429IGT6_bootloard\Boot\boot_ext.c/.h` | 片外 W25Q64 封装 |
| `HAL_F429IGT6_bootloard\Boot\boot_shutdown.c/.h` | **跳转前 9 步拆除 + 时钟恢复** |
| `HAL_F429IGT6_bootloard\Boot\app_jump.c/.h` | App 校验 + 跳转 |
| `HAL_F429IGT6_bootloard\Boot\boot_start.c/.h` | BL 主状态机 `boot_run()` |
| `HAL_F429IGT6_bootloard\Boot\README.md` | BL 集成说明 |
| `HAL_F429IGT6_bootloard\Core\Src\main.c` | 只改 2 处：`#include "boot_start.h"` + `boot_run();` |
| `HAL_F429IGT6_bootloard\Core\Src\spi.c` | SPI2 预分频（**必须在 CubeMX 里改**） |
| `HAL_F429IGT6_bootloard\MDK-ARM\HAL_F429IGT6.uvprojx` | Keil 工程（IROM1=0x08000000/0xC000） |

## 二、App / App_2 工程（两份结构相同）

| 路径 | 用途 |
|---|---|
| `…\OTA\partition.h` | 分区定义（★ 与 BL 必须一致） |
| `…\OTA\ota_params.h` | 参数区结构（★ 与 BL 必须一致） |
| `…\OTA\crc32.c/.h` | CRC32 |
| `…\OTA\sha256.c/.h` | SHA-256 |
| `…\OTA\md5.c/.h` | MD5（平台校验用） |
| `…\OTA\flash_if.c/.h` | 片内 Flash 擦写读 |
| `…\OTA\param_area.c/.h` | 参数区读写 |
| `…\OTA\w25q16.h` | 片外 Flash 常量与声明 |
| `…\OTA\w25q_ext.c` | 片外 Flash 实现（包住 `bsp/W25q64`） |
| `…\OTA\esp01s.c/.h` | ESP8266 AT 驱动 |
| `…\OTA\http_client.c/.h` | HTTP/1.1 客户端（Range/chunked） |
| `…\OTA\ota_port.c` | `USART1_Printf()`（预留） |
| `…\OTA\ota_net_switch.c/.h` | **通道交接**（MQTT ⇄ 裸 TCP）+ `ota_p2_selftest()` 首启入口 |
| `…\OTA\ota_config.h` | **所有可调参数 + 版本号**（★ 两工程唯一差异） |
| `…\OTA\ota_engine.c/.h` | **OTA 执行引擎** |
| `…\OTA\ota_task.c/.h` | 懒创建的 OTA 触发任务 |
| `…\OTA\ota_rollback_test.c/.h` | 回滚自测 |
| `…\OTA\README.md` | OTA 模块说明（P1/P2 集成步骤等） |
| `…\Core\Src\main.c` | `SCB->VTOR = 0x08010000UL;` + `__DSB(); __ISB();` |
| `…\Core\Src\spi.c` | SPI2 预分频（**必须在 CubeMX 里改**） |
| `…\Core\Src\freertos.c` | `#include "ota_net_switch.h"` + `StartDefaultTask` 里 `ota_p2_selftest();`（**栈大小必须在 CubeMX 里改**） |
| `…\Core\Inc\usart.h` | `int USART1_Printf(...)` 声明 |
| `…\App\esp_handle.c` | `#include "ota_task.h"` + `ota_task_on_update_ident(ident);`（GBK 编码！） |
| `…\bsp\UART\uart_handle.c` | 两个回调开头加"OTA 模式让路"守卫 |
| `…\bsp\W25q64\w25q64.c` | 原有 W25Q64 驱动（被 `w25q_ext.c` 包住，不改） |
| `…\MDK-ARM\HAL_F429IGT6.uvprojx` | Keil 工程（IROM1=**0x08010000**/0xF0000） |
| `…\MDK-ARM\HAL_F429IGT6.bin` | **上传到 OneNET 的升级包** |

## 三、文档与记录

| 路径 | 用途 |
|---|---|
| `OTA升级机制与Keil配置分析.md` | f407 原版 BL/App/Keil 分析（移植依据） |
| `F429移植可行性分析与分步计划.md` | f407 → F429 可行性、差异表、P0~P6 计划 |
| `ESP8266_OTA_HTTP实测记录.md` | ESP8266 裸 TCP + OneNET HTTP 接口的 PC 端实测记录 |
| `第一步验证.txt` / `串口数据.txt` | 真实串口日志存档 |
| **`F429_OTA_全流程实施文档.md`** | **本文档** |

---

## 附录：快速上手清单（给"几个月后的自己"）

如果只想快速跑一遍，按这个顺序：

```
【一次性配置】
□ 三个 .ioc：SPI2 Prescaler=4、FreeRTOS defaultTask Stack=1536 → GENERATE CODE
□ App / App_2 魔术棒：IROM1 Start=0x08010000 Size=0xF0000，Include Path 加 ..\OTA
□ App / App_2 魔术棒 User → Run #1 勾选 fromelf --bin
□ BL 魔术棒：IROM1 Start=0x08000000 Size=0xC000，Include Path 加 ..\Boot

【烧录与验证】
□ 编译 BL → 烧录 → 串口应看到 banner + jump to app
□ 编译 App（1.0.0）→ 烧录 → 串口应看到 param load / SHA256 OK / MQTT 连上
□ 编译 App_2（1.0.1）→ 取 MDK-ARM\HAL_F429IGT6.bin

【平台】
□ 物模型：属性 Update (int32, 可写, 0~999)
□ 升级包管理：上传 bin，版本填 1.0.1，类型选整包
□ 升级任务：选包 + 选设备 + 静态升级 + 重试 10 次

【触发】
□ 设备详情 → 属性 Update → 下发 1
□ 串口应看到：[OTA] property "Update" received
                [OTA] otaTask created
                [OTA] OTA net mode READY
                [OTA] task found: tid=... target=1.0.1
                [OTA] download OK: ... md5=...
                [OTA] set TRY_NEW, reboot in 3s...
                --- 复位 ---
                [BL] >>> TRY_NEW / verify staging SHA-256 / staging OK, copy staging -> A
                [OTA] # NEW FIRMWARE FIRST BOOT / version : 1.0.1
                （30 秒后）
                [OTA] >>> BOOT CONFIRMED: flag=NORMAL, version=1.0.1
                [OTA] reported 201 to platform
                [OTA] first-boot: version 1.0.1 reported
□ 平台显示：升级成功，当前版本号 1.0.1
```

**判断成功的唯一标准**：串口里同时出现 `download OK ... md5=<与平台一致的值>` 和 `BOOT CONFIRMED: flag=NORMAL, version=1.0.1`。
