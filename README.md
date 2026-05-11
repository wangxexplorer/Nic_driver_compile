# 多驱动全量编译 Jenkins 使用说明

## 1. 概述

本 Jenkins Pipeline 用于在多个 VM（虚拟机）上并行编译内核驱动。支持 8 种驱动的可选编译，所有 VM 同时并行执行，大幅缩短编译时间。

**支持的驱动：**

| 驱动名 | 类型 | 编译命令 | 依赖 |
|--------|------|----------|------|
| mcepf | 普通 | `cd /home/mcepf-{version}/src; make` | 无 |
| mcevf | 普通 | `cd /home/mcevf-{version}/src; make` | 无 |
| rnp | 普通 | `cd /home/rnp-{version}/src; make` | 无 |
| rnpvf | 普通 | `cd /home/rnpvf-{version}/src; make` | 无 |
| rnpm | 普通 | `cd /home/rnpm-{version}/src; make` | 无 |
| rnpgbe | 普通 | `cd /home/rnpgbe-{version}/src; make` | 无 |
| rnpgbevf | 普通 | `cd /home/rnpgbevf-{version}/src; make` | 无 |
| mrdma | 特殊 | `cd /home/mrdma-{version}; ./mrdma_install.sh` | **mcepf** |

## 2. 构建参数说明

### 2.1 通用参数

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `password` | string | ✅ | SSH 密码（所有 VM 使用相同密码） |
| `driver_version` | string | ✅ | **默认**驱动版本号，所有驱动通用 |
| `DRIVER_VERSION_MAP` | text | ❌ | 差异化版本映射（详见 3.2 节） |
| `OS_CONFIG` | text | ✅ | VM 配置列表（详见 2.2 节） |
| `FAIL_FAST` | boolean | ❌ | 一个 VM 失败后是否立即终止其他 VM |

### 2.2 OS_CONFIG 格式

```
OS名称:IP地址[:端口]
```

- 每行一个，端口可选，默认 `22`
- 支持注释行（以 `#` 开头）
- 默认值已预置所有 41 个 VM，可根据需要修改

**示例：**
```
# 生产环境
AnolisOS-8.6-QU1:10.84.10.188
centos7.6:10.84.10.172:2222

# 测试环境（已注释，不会执行）
# ubuntu20.04:10.84.10.141
```

### 2.3 驱动选择参数

| 参数名 | 说明 |
|--------|------|
| `BUILD_ALL` | 🔘 一键编译所有驱动 |
| `BUILD_MCEPF` | 编译 mcepf |
| `BUILD_MCEVF` | 编译 mcevf |
| `BUILD_RNP` | 编译 rnp |
| `BUILD_RNPVF` | 编译 rnpvf |
| `BUILD_RNPM` | 编译 rnpm |
| `BUILD_RNPGBE` | 编译 rnpgbe |
| `BUILD_RNPGBEVF` | 编译 rnpgbevf |
| `BUILD_MRDMA` | 编译 mrdma（依赖 mcepf） |

**使用规则：**
- `BUILD_ALL=true` → 编译所有驱动（等价于全部勾选）
- `BUILD_ALL=false` → 仅编译勾选了的驱动
- 至少要勾选一个驱动，否则构建会报错

## 3. 典型使用场景

### 3.1 场景一：全量编译，所有驱动版本相同

**需求：** 编译 mcepf、mcevf、mrdma 三个驱动，版本号都是 `1.1.14`

| 参数 | 值 |
|------|-----|
| `password` | `your_password` |
| `driver_version` | `1.1.14` |
| `DRIVER_VERSION_MAP` | *(留空)* |
| `BUILD_ALL` | `false` |
| `BUILD_MCEPF` | `true` |
| `BUILD_MCEVF` | `true` |
| `BUILD_MRDMA` | `true` |

**执行顺序：** 每个 VM 上按顺序编译 `mcepf` → `mcevf` → `mrdma`（mrdma 自动排在最后，因为依赖 mcepf）

### 3.2 场景二：部分编译，某个驱动版本不同

**需求：** 编译 mcepf、mcevf、mrdma，其中 mrdma 版本是 `2.0.1`，其余是 `1.1.14`

| 参数 | 值 |
|------|-----|
| `password` | `your_password` |
| `driver_version` | `1.1.14` |
| `DRIVER_VERSION_MAP` | `mrdma:2.0.1` |
| `BUILD_MCEPF` | `true` |
| `BUILD_MCEVF` | `true` |
| `BUILD_MRDMA` | `true` |

**说明：** mcepf 和 mcevf 不在映射表中，自动使用默认版本 `1.1.14`；mrdma 使用映射版本 `2.0.1`。

### 3.3 场景三：全量编译，多个驱动版本不同

**需求：** 所有驱动都编译，且各自版本不同

| 参数 | 值 |
|------|-----|
| `password` | `your_password` |
| `driver_version` | `1.1.14` |
| `DRIVER_VERSION_MAP` | 见下 |
| `BUILD_ALL` | `true` |

**DRIVER_VERSION_MAP 内容：**
```
mcepf:1.0.0
mrdma:2.0.1
rnpgbe:1.2.0
```

**结果：**
- mcepf → `1.0.0`
- mrdma → `2.0.1`
- rnpgbe → `1.2.0`
- 其余驱动 → `1.1.14`（默认值）

### 3.4 场景四：一键全量编译

| 参数 | 值 |
|------|-----|
| `password` | `your_password` |
| `driver_version` | `1.1.14` |
| `DRIVER_VERSION_MAP` | *(留空)* |
| `BUILD_ALL` | `true` |

所有 8 个驱动全部编译，版本统一为 `1.1.14`。

## 4. 依赖关系说明

### 4.1 mrdma → mcepf

- **mrdma 编译前必须有 mcepf**
- 如果勾选了 `BUILD_MRDMA` 但没勾选 `BUILD_MCEPF`，系统会自动在每个 VM 上**先编译 mcepf**
- 如果 mcepf 编译失败，该 VM 上的 mrdma 自动跳过

### 4.2 普通驱动之间互相独立

- mcepf、mcevf、rnp 等普通驱动之间**没有依赖关系**
- 某个驱动编译失败，不影响该 VM 上其他驱动的编译
- 所有失败信息汇总后统一输出

## 5. 执行流程

```
解析 OS_CONFIG → 获取 VM 列表
解析驱动选择参数 → 获取驱动列表
    ↓
并行执行（所有 VM 同时开始）
    ↓
每个 VM 上串行执行选中的驱动：
    驱动A: 清理 → 传输包 → 解压 → 编译 → 结果
    驱动B: 清理 → 传输包 → 解压 → 编译 → 结果
    ...
    ↓
汇总结果（成功/失败/跳过）
```

## 6. 日志说明

构建日志中会输出以下关键信息：

```
将要并行编译的OS列表（共 41 个）：
  - AnolisOS-8.6-QU1: 10.84.10.188:22
  - AnolisOS-8.8: 10.84.10.232:22
  ...

将要编译的驱动列表（共 3 个）：
  - mcepf
  - mcevf
  - mrdma

[AnolisOS-8.6-QU1] >>> 开始编译驱动: mcepf (版本: 1.1.14)
[AnolisOS-8.6-QU1] ✅ 驱动 mcepf 编译完成！
[AnolisOS-8.6-QU1] >>> 开始编译驱动: mcevf (版本: 1.1.14)
[AnolisOS-8.6-QU1] ✅ 驱动 mcevf 编译完成！
[AnolisOS-8.6-QU1] >>> 开始编译驱动: mrdma (版本: 2.0.1)
[AnolisOS-8.6-QU1] ✅ 驱动 mrdma 编译完成！
```

## 7. 常见问题

### Q1: 只编译 mcepf，版本号怎么填？

只需填 `driver_version=1.1.14`，勾选 `BUILD_MCEPF=true`，其余参数留空即可。

### Q2: mrdma 依赖 mcepf，我需要手动勾选 mcepf 吗？

不需要。勾选了 `BUILD_MRDMA` 后，系统会自动先编译 mcepf。但如果你想单独编译 mcepf，需要手动勾选 `BUILD_MCEPF`。

### Q3: 某个 VM 的 IP 变了，怎么修改？

修改 `OS_CONFIG` 参数中对应行的 IP 地址即可，格式不变。

### Q4: 想临时屏蔽某个 VM 不编译？

在该行前面加 `#` 注释掉，例如：
```
# centos7.6:10.84.10.172
```

### Q5: FAIL_FAST 什么时候用？

- `FAIL_FAST=false`（默认）：某个 VM 失败后，其他 VM 继续编译，尽可能多完成
- `FAIL_FAST=true`：某个 VM 失败后，立即终止所有 VM，快速失败

## 8. 文件依赖

本 Pipeline 依赖 Jenkins 服务器上的以下目录结构：

```
/home/kernel_driver/
├── mcepf-{version}.tar.gz
├── mcevf-{version}.tar.gz
├── rnp-{version}.tar.gz
├── rnpvf-{version}.tar.gz
├── rnpm-{version}.tar.gz
├── rnpgbe-{version}.tar.gz
├── rnpgbevf-{version}.tar.gz
└── mrdma-{version}.tar.gz
```

请确保在运行构建前，所有需要的驱动包已放置到 `/home/kernel_driver/` 目录下。

## 9. 注意事项

1. **SSH 密码**：所有 VM 使用相同的 SSH 密码，通过 `password` 参数传入
2. **root 用户**：所有操作均使用 `root` 用户执行
3. **端口默认 22**：OS_CONFIG 中端口可省略，默认 22
4. **版本号格式**：`DRIVER_VERSION_MAP` 中版本号不要包含 `.tar.gz` 后缀
5. **包名规则**：驱动包名必须是 `{驱动名}-{版本号}.tar.gz` 格式

---

*文档生成时间：2026-05-11*
