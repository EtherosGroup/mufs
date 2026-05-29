# Mount Unit File System

> **MUFS 重新定义了文件系统的组织方式：挂载点（而非目录树）是存储资源的第一级命名原语。**

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Spec](https://img.shields.io/badge/spec-v1.0-green.svg)](MUFS.md)

---

MUFS 是一个内核态文件系统框架，以**挂载点**为存储组织的基本单位。每个分区在自身的超级块中携带挂载元数据，无需外部配置文件；系统路径通过变量引擎动态解析，告别编译期硬编码。

---

## 核心创新

| 维度 | 传统文件系统 | MUFS |
|------|------------|------|
| **组织原语** | 目录树层级 | 类型化挂载点（`DMP` / `SMP` / `VMP`） |
| **根目录语义** | `/` 可容纳文件和目录 | `/` 为设备标识符，不可写 |
| **配置存储** | 外部文件（如 `/etc/fstab`） | 分区超级块自描述 |
| **系统路径** | 硬编码（`/tmp`、`/home`） | 动态变量映射（`%temp%`、`%home%`） |
| **设备枚举** | 静态节点 + 用户态守护进程 | `/devices` VMP —— 内核态实时生成 |
| **启动依赖** | initramfs → pivot_root → fstab | 内核模块单阶段初始化 |

## 架构

```
                         用户空间应用程序
                  MUFS 统一命名空间视图
             /system     /data     /devices     ...

                    ── 系统调用接口 ──

                         MUFS 内核模块
           ┌──────────────┼──────────────┐
       挂载点管理      路径解析引擎    系统变量引擎
       超级块读写      VMP 生成器      热插拔监听
           └──────────────┼──────────────┘

                     虚拟文件系统层 (VFS)

                    实际文件系统驱动
            ext4           exFAT           ...

                       块设备抽象层
```

## 挂载点类型

| 类型 | 全称 | 底层支撑 | 创建者 |
|------|------|---------|--------|
| **DMP** | 设备挂载点（Device Mount Point） | 物理块设备 | 用户（`mount DMP ...`） |
| **SMP** | 系统挂载点（System Mount Point） | 物理块设备 | 安装程序（`mount SMP ...`） |
| **VMP** | 虚拟挂载点（Virtual Mount Point） | 内核动态生成 | 系统强制（`/devices` 为必选） |

## 超级块自描述

每个分区在 ext4 超级块的 `0x200` 偏移处直接声明其角色：

```
字节  0    4    6    8                                               56
┌──────┬────┬────┬──────────────────────────────────────────────────┐
│ 魔数 │版本 │标志│              挂载名（UTF-8）                      │  ← 扩展头（56 B）
├──────┴────┴────┴──────────────────────────────────────────────────┤
│ 类型 │权限 │  变量标签   │  变量子路径   │CRC32 │      预留       │  ← 扩展体（104 B）
└──────┴────┴───────────┴──────────────┴──────┴────────────────────┘
                     总计 160 字节（ext4 预留约 500 B）
```

这意味着：
- 将硬盘移到另一台机器 → 挂载配置自动跟随
- 无需同步 `/etc/fstab`
- 分区与其角色是原子整体

## 快速上手

### 格式化系统分区

```bash
mufs.mkfs \
    --type SMP \
    --name system \
    --var temp:temp/ \
    --var home:users/ \
    --var logs:logs/ \
    /dev/sda2
```

### 挂载数据分区

```bash
mount DMP /devices/sda1 data     # → /data/
mount DMP /devices/sdb1 media    # → /media/
```

### 管理系统

```bash
mufsctl list              # 列出所有挂载点
mufsctl info /system      # 查看挂载点详情
mufsctl vars              # 查看变量映射表
mufsctl check /dev/sda1   # 验证超级块完整性
mufsctl repair /dev/sda1  # 从备份超级块恢复
```

### 变量路径解析

```
应用程序请求：   %temp%/file.log
变量引擎查询：   %temp% → /system/temp/
解析后路径：     /system/temp/file.log
委托处理：       ext4 驱动 → "system" 挂载点
```

## 文档

- **[MUFS.md](MUFS.md)** — 完整技术规范（v1.0）
- **[MUFS.md §4](MUFS.md#4-超级块扩展定义)** — 超级块扩展布局
- **[MUFS.md §7](MUFS.md#7-系统变量机制)** — 系统变量引擎
- **[MUFS.md §10](MUFS.md#10-命令行接口)** — 命令行参考

## 功能规划

| ID | 功能 | 优先级 |
|----|------|--------|
| F1 | 挂载点生命周期管理（创建、查询、卸载） | P0 |
| F2 | 路径解析与标准化 | P0 |
| F3 | DMP 完整实现 | P0 |
| F4 | SMP 完整实现 + 安装工具链 | P0 |
| F5 | VMP 框架 + `/devices` | P0 |
| F6 | 系统变量动态解析引擎 | P0 |
| F7 | ext4 超级块扩展读写 | P0 |
| F8 | 超级块备份同步机制 | P0 |
| F9 | 魔数检测与分区自动发现 | P0 |
| F10 | 热插拔监听 + `/devices` 实时更新 | P1 |
| F11 | CRC32 校验与自动备份恢复 | P0 |
| F12 | `mufsctl` 用户态管理工具 | P1 |
| F13 | `mufs.mkfs` 安装格式化工具 | P1 |

## 许可证

本项目基于 **Apache License 2.0** 许可。详见 [LICENSE](LICENSE)。

---

*MUFS 为下一代开源操作系统而设计，作为其文件系统基础设施。*
