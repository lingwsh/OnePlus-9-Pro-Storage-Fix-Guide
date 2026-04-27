# 一加 9 Pro 存储清理指南

**找回 20-50+ GB 被"其他"吞掉的存储空间**

## 问题描述

一加 9 Pro (LE2123) 运行 OxygenOS 14 时，存储设置中的"其他"类别会膨胀到 30-50+ GB，且无法通过常规手段清理。这是一个**已知的、未修复的 bug**，影响整个一加 9 系列。一加从未发布修复工具，OOS 14 已是该设备的最终大版本。

### 根本原因

膨胀由三个来源叠加造成:

1. **基带 ramdump 文件** (`/data/vendor/ramdump/`) — 高通基带崩溃转储（每个约 269 MB），部分固件版本下每天生成数次。OxygenOS 从不自动删除。
2. **系统日志** (`/data/oplus/log/`) — 工程日志无限期累积。
3. **升级残留** — 四次大版本升级（OOS 11→12→13→14）遗留的 dalvik-cache、过期 OTA 安装包、以及 Play 商店 APK 缓存。

以上文件均受 SELinux 保护，普通用户无法在未 root 的情况下访问。

## 快速开始

### 准备工作

- 一台安装了 ADB 的电脑（[下载 Platform Tools](https://developer.android.com/tools/releases/platform-tools)）
- 一根 USB 数据线（非纯充电线）— **建议使用 USB 2.0**（详见下方 USB 问题章节）
- 已开启 USB 调试的一加 9 Pro

### 开启 USB 调试

1. **设置 → 关于手机** → 连续点击**版本号** 7 次
2. **设置 → 开发者选项** → 开启 **USB 调试**
3. **设置 → 开发者选项 → 默认 USB 配置** → 设为**文件传输**

### 分步操作

#### 第 1 步: 清空相册回收站 (0-50 GB)

在手机上: **相册 → 合集 → 最近删除 → 清空**

这可能是回收空间最大的一步。2021 年起已知的 MediaStorage bug 导致已删除照片在回收站中无限累积。

#### 第 2 步: 清理 Play 商店缓存 (0-15 GB)

```bash
adb shell du -sh /sdcard/Android/data/com.android.vending/
# 如果很大（几个 GB），清理:
adb shell rm -rf /sdcard/Android/data/com.android.vending/*
```

#### 第 3 步: LogKit 清日志 (2-30 GB)

在手机上: 打开**拨号盘** → 输入 `*#800#`

- 如果打开了 **LogKit** → 点击 **"一键清除日志"**
- 如果打开了**反馈**应用 → 你的欧版 OOS 14 已禁用 LogKit，跳过此步

#### 第 4 步: ADB 缓存清理和 Dex 重置 (2-8 GB)

```bash
# 清理所有应用缓存
adb shell pm trim-caches 999G

# 重置 ART 编译产物（应用冷启动会变慢几天）
adb shell pm compile --reset -a

# 触发后台重新优化
adb shell pm bg-dexopt-job
```

#### 第 5 步: 清理 sdcard 垃圾文件夹 (0.5-2 GB)

```bash
# 查看各文件夹大小
adb shell du -sh /sdcard/*/ 2>/dev/null | sort -rh | head -20

# 常见的可安全删除的垃圾文件夹
adb shell rm -rf /sdcard/tencent/*
adb shell rm -rf /sdcard/amap/*
adb shell rm -rf /sdcard/Qmap/*
adb shell rm -rf /sdcard/QQBrowser/*
adb shell rm -rf /sdcard/TbsReaderTemp/*
adb shell rm -rf /sdcard/MIUI/*
```

> **请勿删除**: `DCIM`、`Pictures`、`Download`、`Movies`、`Music`、`Documents`、`Android` — 这些包含用户数据。

#### 第 6 步: 在手机上清理应用数据 (5-15 GB)

在手机上: **设置 → 应用** → 清理最大的应用缓存:

- **微信** (3-5 GB): 微信 → 我 → 设置 → 通用 → 存储空间 → 清理
- **Chrome** (0.5-1.5 GB): Chrome → ⋮ → 设置 → 隐私 → 清除浏览数据
- **YouTube、Spotify、Outlook** 等: 设置 → 应用 → [应用名] → 存储 → 清除缓存
- **系统更新** (0-6 GB): 设置 → 应用 → ⋮ 显示系统应用 → 系统更新 → 存储 → 清除数据

#### 第 7 步: 查看结果

```bash
adb shell df -h /data
```

### 预期回收: 15-40 GB (不含 MSM 刷机)

---

## 剩余的约 16 GB 隐藏存储

完成以上所有步骤后，你可能仍有约 16 GB 的"其他"无法清除。这是被 SELinux 锁定在 `/data/vendor/ramdump/` 和 `/data/oplus/log/` 中的基带 ramdump 和系统日志。

### 方案 A: MSM 刷机 (推荐)

使用高通 EDL（紧急下载）模式进行工厂级刷机。这是一加售后服务中心使用的方法。

| 特性 | 普通恢复出厂 | MSM 刷机 |
|---|---|---|
| 清除应用数据 | 是 | 是 |
| 清除隐藏 ramdump/日志 | **否** | **是** |
| 重写所有分区 | 否 | 是 |
| Bootloader 状态 | 不变 | 重新锁定 |
| Widevine L1 | 保留 | **保留** |
| 银行应用 (MobilePay 等) | 正常 | **正常** |

**准备:**
- Windows 电脑
- USB 数据线
- LE2123 对应区域的固件包（欧版为 BA 后缀）
- 完整备份（Google 备份 + 云同步照片 + 微信聊天记录导出）

**操作步骤:**
1. 下载 LE2123 对应区域的 MSM 固件包
2. 在 Windows 上安装高通 USB 驱动
3. 完整备份手机所有数据
4. 完全关机
5. 同时按住**音量上 + 音量下**，插入 USB 线 → 进入 EDL 模式（黑屏）
6. 运行 MsmDownloadTool → 选择固件 → 开始
7. 等待约 5 分钟 → 手机重启进入初始设置
8. 设置时从 Google 备份恢复

### 方案 B: Root (不推荐)

通过 `fastboot oem unlock` 解锁 bootloader **会导致**:
- **Widevine L1** → 降级为 L3（Netflix 画质锁定 480p，约 30% 概率永久降级）
- **Play Integrity** → DEVICE 级别验证失败 → **MobilePay 停止工作**
- **银行应用** → 可能检测到 bootloader 已解锁

绕过模块（Magisk + PlayIntegrityFork）不稳定，需要持续更新。

---

## 已知的 USB 连接问题

一加 9 Pro 存在 USB 3.0 描述符协商 bug。症状:
- 手机充电但 `lsusb` 中不显示
- 内核日志显示: `got a wrong device descriptor, warm reset device`

**解决方法:**
1. **直接插入笔记本电脑** — 不要通过扩展坞（尤其是 Dell TB16）
2. **使用 USB 2.0 / USB-A 接口** — 此 bug 仅影响 USB 3.0 SuperSpeed
3. **使用无线 ADB** 作为备选（需要 ADB 31+）:
   ```bash
   adb pair <IP>:<配对端口> <配对码>
   adb connect <IP>:<端口>
   ```

---

## 存储诊断脚本

要详细了解存储占用情况，运行:

```bash
adb shell dumpsys diskstats > diskstats.txt
```

然后使用 [fix-phone-storage.md](fix-phone-storage.md) 中的 Python 脚本解析，获取按应用分解的详细数据。

### 理解手机界面中"其他"的构成

| 组成部分 | 典型大小 | 可清理? |
|---|---|---|
| Ramdump + 系统日志 | 约 16 GB | 仅 MSM 刷机 |
| 文件系统开销 (ext4) | 约 9 GB | 否（正常） |
| 应用缓存 | 3-5 GB | 是（逐个应用） |
| dalvik-cache | 3-5 GB | 是 (`pm compile --reset`) |
| sdcard 垃圾文件 | 0.5-2 GB | 是 (ADB rm) |
| Play 商店 APK 缓存 | 0-15 GB | 是 (ADB rm) |

手机界面显示的"其他"是以上所有项目的总和 — 不仅仅是 ramdump/日志。

---

## 社区证据

| 来源 | 发现 |
|---|---|
| XDA 4321573 | 用户删除 ramdump_modem .elf 文件后回收 **84 GB** |
| XDA 4376113 | OP9P 128GB 同样问题；仅 MSM 刷机修复 |
| XDA 4525693 | OOS 13 未修复；"OOS 存日志从不删除" |
| Android Police (2021) | Russakovskii 仅清空相册回收站就从 126 GB 恢复到 5 GB |
| Gizmochina (2024) | 一加在 OOS 15 (OP13) 缩小了 `super` 分区 — 间接承认浪费空间 |

---

## 日常维护

恢复后建议每月执行:
1. 清空相册回收站
2. `adb shell pm trim-caches 999G`
3. 清理微信/Chrome/YouTube 缓存
4. 如果 LogKit 可用: `*#800#` → 一键清除日志

---

## 本仓库文件

- `README.md` — 本指南（英文版）
- `README_CN.md` — 本指南（中文版）
- `fix-phone-storage.md` — 技术技能文件，包含所有命令和分析脚本
- `find_lost_storage.md` — 原始深度研究文档

## 免责声明

本指南按原样提供，供一加 9 Pro 社区使用。使用风险自负。MSM 刷机会擦除所有数据 — 请务必先备份。
