# tac-releases

`tac` (Tinnove AI Coding) CLI 的 binary 分发仓库。

> 源代码托管在内部 Gerrit (`WT590_TAC/tac`)。本仓库**只承载发布产物**,不含源码。
> 任何人都可以从这里下载安装,issue / PR / 代码贡献请走内部流程。

## 安装

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh | sh
```

会装到 `/usr/local/bin/tac`(需要 sudo)。

### 装到自定义目录(免 sudo)

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --prefix $HOME/.local/bin
```

### 锁定版本

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --version v0.2.0
```

## 前置依赖

`tac` 调外部工具完成实际工作,装 `tac` 前先装这两个:

```sh
# Microsoft APM —— skill/agent/command 包管理器
curl -sSL https://aka.ms/apm-unix | sh

# Astral uv —— spec-kit 工具链 runner
curl -LsSf https://astral.sh/uv/install.sh | sh
```

`tac doctor` 会检查这些依赖是否就位。

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
