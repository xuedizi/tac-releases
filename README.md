# tac-releases

`tac` (Tinnove AI Coding) CLI 的 binary 分发仓库。

> 源代码托管在内部 Gerrit (`WT590_TAC/tac`)。本仓库**只承载发布产物**,不含源码。
> 任何人都可以从这里下载安装,issue / PR / 代码贡献请走内部流程。

## 安装

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh | sh
```

会装到 `/usr/local/bin/tac`(需要 sudo)。

装完会自动探测前置依赖 `apm` / `uv`,缺则打印官方一行安装命令(见下文 [前置依赖](#前置依赖))。

### 装到自定义目录(免 sudo)

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --prefix $HOME/.local/bin
```

### 一键连带装 apm + uv

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --with-deps
```

`--with-deps` 会在装完 `tac` 后,自动调 apm/uv 官方 installer 把缺失项补齐(已装的不动)。
等价环境变量:`TAC_WITH_DEPS=1`。

### 锁定版本

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --version v0.2.1
```

## 升级

装过一次之后,后续升级不用再 curl install.sh,直接:

```sh
tac self-update              # 装到最新
tac self-update --check      # 只看当前 vs 最新,不动 binary
tac self-update --version v0.2.0   # pin 到指定版本
tac self-update --dry-run    # 打印计划,不下载不写盘
```

公开 repo zero-config,无需 token。

## 前置依赖

`tac` 调外部工具完成实际工作:

| 工具 | 用途 | 官方安装 |
|---|---|---|
| `apm` | skill / agent / command 包管理器 | `curl -sSL https://aka.ms/apm-unix \| sh` |
| `uv`  | spec-kit 工具链 runner            | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |

三种安装姿势,任选其一:

1. **自动**:安装 tac 时加 `--with-deps`,一行装齐。
2. **半自动**:走默认安装,看 installer 提示哪个缺,复制对应命令手动跑。
3. **预装**:CI 镜像或干净机器,先把 apm/uv 装好再装 tac。

装完用 `tac doctor` 验证。

## 验证

```sh
tac --version
tac doctor --no-project
```

## 支持平台

| OS | Arch |
|---|---|
| macOS | amd64, arm64 |
| Linux | amd64, arm64 |
| Windows | amd64 |

## 版本历史

完整版本列表见 [Releases](https://github.com/xuedizi/tac-releases/releases)。
