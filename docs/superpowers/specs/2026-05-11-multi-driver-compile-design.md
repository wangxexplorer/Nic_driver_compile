# 多驱动并行编译 Jenkinsfile 设计文档

**日期**: 2026-05-11
**作者**: Sisyphus
**状态**: 已批准

## 1. 背景

现有 Jenkins 编译任务需要从单一驱动（mcepf）扩展到支持 8 个驱动的可选编译。其中 7 个驱动的编译流程一致（解压后进入 `src` 目录执行 `make`），1 个驱动（mrdma）解压后执行 `./mrdma_install.sh` 且依赖 mcepf。

## 2. 驱动清单

### 2.1 普通驱动（7个，流程一致）

| 驱动名 | 包名格式 | 编译命令 |
|--------|----------|----------|
| mcepf | `mcepf-${driver_version}.tar.gz` | `cd /home/mcepf-${driver_version}/src; make` |
| mcevf | `mcevf-${driver_version}.tar.gz` | `cd /home/mcevf-${driver_version}/src; make` |
| rnp | `rnp-${driver_version}.tar.gz` | `cd /home/rnp-${driver_version}/src; make` |
| rnpvf | `rnpvf-${driver_version}.tar.gz` | `cd /home/rnpvf-${driver_version}/src; make` |
| rnpm | `rnpm-${driver_version}.tar.gz` | `cd /home/rnpm-${driver_version}/src; make` |
| rnpgbe | `rnpgbe-${driver_version}.tar.gz` | `cd /home/rnpgbe-${driver_version}/src; make` |
| rnpgbevf | `rnpgbevf-${driver_version}.tar.gz` | `cd /home/rnpgbevf-${driver_version}/src; make` |

### 2.2 特殊驱动（1个）

| 驱动名 | 包名格式 | 编译命令 | 依赖 |
|--------|----------|----------|------|
| mrdma | `mrdma-${driver_version}.tar.gz` | `cd /home/mrdma-${driver_version}; ./mrdma_install.sh` | 依赖 mcepf |

## 3. 参数设计

### 3.1 原有参数（保留）

- `password`: SSH 密码
- `driver_version`: 驱动版本号（如 `1.1.14`）
- `OS_CONFIG`: OS 配置列表（`OS名称:IP` 格式）
- `FAIL_FAST`: 是否快速失败

### 3.2 新增参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `BUILD_ALL` | boolean | `false` | 一键勾选所有驱动 |
| `BUILD_MCEPF` | boolean | `false` | 编译 mcepf |
| `BUILD_MCEVF` | boolean | `false` | 编译 mcevf |
| `BUILD_RNP` | boolean | `false` | 编译 rnp |
| `BUILD_RNPVF` | boolean | `false` | 编译 rnpvf |
| `BUILD_RNPM` | boolean | `false` | 编译 rnpm |
| `BUILD_RNPGBE` | boolean | `false` | 编译 rnpgbe |
| `BUILD_RNPGBEVF` | boolean | `false` | 编译 rnpgbevf |
| `BUILD_MRDMA` | boolean | `false` | 编译 mrdma |

### 3.3 参数交互逻辑

- 若 `BUILD_ALL=true`，则等价于所有驱动都被勾选
- 若 `BUILD_ALL=false`，则按各独立布尔参数决定

## 4. 执行架构

采用 **VM 级并行 + 驱动级串行** 模式：

```
外层 Parallel: 所有 VM 同时运行
└── 每个 VM Stage: "编译-${osName}"
    └── 串行循环: 按顺序编译选中的驱动列表
        ├── 清理: rm -rf /home/${driver_name}*
        ├── 传包: scp .../${driver_name}-${driver_version}.tar.gz
        ├── 解压: tar xvf ...
        └── 编译: make 或 ./mrdma_install.sh
```

## 5. 依赖逻辑（MRDMA → MCEPF）

在每个 VM 上的执行逻辑：

1. 根据用户勾选构建初始驱动列表
2. **依赖注入**：若勾选了 `BUILD_MRDMA` 但未勾选 `BUILD_MCEPF`，自动在列表头部插入 `mcepf`
3. 按列表顺序串行执行
4. **失败传播**：若 mcepf 编译失败，则该 VM 上 mrdma 自动跳过并标记失败
5. 其他 VM 不受影响，继续各自执行

## 6. 清理策略

每个驱动编译前独立清理：

```bash
rm -rf /home/${driver_name}*
```

避免不同驱动之间的文件冲突。

## 7. 错误处理

- 单个 VM 上的单个驱动编译失败：
  - 标记该 VM 的 currentBuild.result = 'UNSTABLE'
  - 如果 `FAIL_FAST=true`，立即终止所有 VM
  - 如果 `FAIL_FAST=false`，该 VM 停止继续编译后续驱动，其他 VM 继续
- mrdma 依赖 mcepf：若 mcepf 失败，mrdma 自动跳过

## 8. 日志输出

每个驱动编译前后输出明确的分隔日志：

```
============================
[${osName}] 开始编译 ${driver_name}...
============================
...
============================
[${osName}] ✅ ${driver_name} 编译完成！
============================
```
