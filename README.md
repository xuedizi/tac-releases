# tac-releases

`tac` (Tinnove AI Coding) CLI 的 binary 分发仓库 + 接入手册。

> **源代码** 托管在内部 Gerrit (`WT590_TAC/tac`)。本仓库**只承载发布产物与端用户文档**,不含源码。
> **issue / PR / 代码贡献** 请走内部 Gerrit 流程。

适用版本:`tac v0.3.x`(spec-kit / APM 官方接入版本)
适用项目类型:Android-Kotlin(Phase 0 唯一交付的 starter pack)

完成接入后即可在 Claude Code / CodeBuddy / Codex 等 AI IDE 中使用 `/speckit.specify`、`/speckit.plan`、`/speckit.implement` 等流程,以及 TAC 提供的 `tac.gate`、`tac.commit`、`tac.pr` 命令。

---

## 一、安装

### 1.1 一行命令(推荐)

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh | sh
```

会装到 `/usr/local/bin/tac`(需要 sudo)。装完会自动探测前置依赖 `apm` / `uv`,缺则打印官方一行安装命令(见 [§二 前置依赖](#二前置依赖))。

> **Windows(Git Bash / MSYS)**:在 Git Bash 内执行同一条命令,默认装到 `$HOME/.local/bin/tac.exe`(即 `%USERPROFILE%\.local\bin\tac.exe`),**不走 sudo**;装完手动把该目录加入 PATH 即可 `tac --version`。

### 1.2 装到自定义目录(免 sudo)

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --prefix $HOME/.local/bin
```

### 1.3 一键连带装 apm + uv

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --with-deps
```

`--with-deps` 会在装完 `tac` 后,自动调 apm/uv 官方 installer 把缺失项补齐(已装的不动)。等价环境变量:`TAC_WITH_DEPS=1`。

### 1.4 锁定版本

```sh
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh \
  | sh -s -- --version v0.3.0
```

### 1.5 支持平台

| OS | Arch |
|---|---|
| macOS | amd64, arm64 |
| Linux | amd64, arm64 |
| Windows | amd64(在 Git Bash / WSL 内跑 install.sh,默认装到 `$HOME/.local/bin/tac.exe`;纯 PowerShell 暂不支持) |

### 1.6 验证

```sh
tac --version
tac doctor --no-project
```

期望输出:

```
[tac] tac v0.3.0 (commit <sha>, built <date>)
[tac] platform: darwin/arm64                 # Linux: linux/amd64; Windows (Git Bash): windows/amd64
[tac] apm: /opt/homebrew/bin/apm (apm 0.13.0) ✓
[tac] uv: /opt/homebrew/bin/uv (uv 0.4.18) ✓
[tac] git: /usr/bin/git (git 2.39.5) ✓
[tac] gateway env: http://nexus.tinnove.com:3000 ✓
[tac] detected IDEs (PATH): [claude]
```

---

## 二、前置依赖

`tac` 调外部工具完成实际工作:

| 工具 | 用途 | 官方安装(macOS / Linux) |
|---|---|---|
| `apm` | skill / agent / command 包管理器 | `curl -sSL https://aka.ms/apm-unix \| sh` 或 `brew install microsoft/tap/apm` |
| `uv`  | spec-kit 工具链 runner            | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| `git` | spec-kit upgrade + hook 用 | 通常自带 |

**三种安装姿势,任选其一:**

1. **自动**:`tac` install 时加 `--with-deps`(见 §1.3),一行装齐。
2. **半自动**:走默认 install,看 installer 提示哪个缺,复制对应命令手动跑。
3. **预装**:CI 镜像或干净机器,先把 apm/uv 装好再装 tac。

---

## 三、升级

### 3.1 升级 tac 自身(推荐)

装过一次之后,后续升级用内置子命令,不用再 curl install.sh:

```sh
tac self-update              # 装到 latest
tac self-update --check      # 只看当前 vs latest,不动 binary
tac self-update --version v0.3.0   # pin 到指定版本
tac self-update --dry-run    # 打印计划,不下载不写盘
tac self-update --force      # 重装当前版本
```

**目录不可写时** 会提示重跑 installer 加 `--prefix $HOME/.local/bin`。

### 3.3 升级 spec-kit

```bash
tac specify version                       # 看当前 ref
tac specify upgrade                       # 拉 latest stable
tac specify upgrade --ref v0.9.0          # 锁定具体 ref
```

### 3.4 升级 APM 治理的 skill / agent / command

改 `apm.yml` 里对应条目的 `#vX.Y.Z`,然后:

```bash
tac apm install
tac apm compile -t claude       # 按你实际接入的 IDE 重复 -t
```

### 3.5 升级 apm 自身

```bash
tac apm self-update      # 透传系统 apm self-update
```

---

## 四、接入项目:`tac init`

### 4.1 一行命令

```bash
cd /path/to/your-android-project        # 必须是 git 仓库
export TAC_GATEWAY_URL=http://nexus.tinnove.com:3000

tac init \
  --template android-kotlin \
  --name your-project-name \
  --gateway $TAC_GATEWAY_URL \
  --ide claude,codex                   # 必填,逗号分隔多选
```

### 4.3 参数详解

| flag | 必填 | 说明 |
| --- | --- | --- |
| `--template` | ✅ | Phase 0 唯一可用值:`android-kotlin` |
| `--ide` | ✅ | 逗号分隔的 IDE 列表(`claude` / `codebuddy` / `codex`),**不允许省略**;空值或非法值都会 fail-fast 并打印有效集 |
| `--name` |   | 项目名;默认取目录 basename |
| `--gateway` |   | LLM 网关 URL;优先级 flag → `TAC_GATEWAY_URL` → 内置兜底 |
| `--policy-root` |   | `apm-policy.yml extends:` 目标;优先级 flag → `TAC_POLICY_ROOT` → 内置兜底 `xuedizi/apm-policy#v1.0.0`。沙箱/fork 场景可切自定义根策略,正式项目保持默认 |
| `--root` |   | 项目根目录;默认 cwd,CI 里建议显式传 |
| `--force` |   | 已存在 `.tac/` 时是否覆盖,**会丢失现有 TAC 配置** |

---

## 五、命令参考

### 5.1 `tac doctor`

健康体检,**返回非零退出码即视为不健康**,适合放进 CI:

```bash
tac doctor                  # 全量(全局 + 项目)
tac doctor --no-project     # 仅 tac 自身
tac doctor --root /repo     # 指定项目根
```

| 检查项 | 失败行为 |
| --- | --- |
| tac 版本 / 平台 | 仅展示 |
| `apm` 版本矩阵 | < required(0.10.0)打 ✗ **并让 doctor 非零退出**;< recommended(0.13.0)打 ⚠;≥ recommended 打 ✓ |
| `uv` 版本矩阵 | 缺失或 < recommended(0.4.0)打 ⚠(不致 doctor 失败,spec-kit init 时才硬依赖) |
| `git` 版本矩阵 | 缺失或 < recommended(2.25.0)打 ⚠(用于 `tac specify upgrade` 与 hook) |
| `TAC_GATEWAY_URL` | 未设置:提示;设置但不通:打 ✗ |
| 已安装 IDE(PATH) | 仅展示 |
| `.tac/adapter.yaml` | 缺失/不合法 → 返回非零 |
| `.tac/lockfile.yaml` | 缺失/不合法 → 返回非零 |
| `apm.yml` / `apm.lock.yaml` | 缺失 → 提示用户跑 `tac apm install` 或重 init |
| `.specify/.tac-installed-ref` 与 lockfile.specify.ref 一致 | 不一致 → 打 ✗ 提示 `tac specify upgrade` 重对齐 |
| 每个已 init 的 IDE 目录下有 `commands/speckit.specify.md` | 缺失 → 打 ✗(spec-kit 没正确落地) |

> 版本解析使用宽容正则,容忍 `apm 0.13.0`、`uv 0.10.11 (Homebrew ...)`、`git version 2.39.5 (Apple Git-154)` 等格式;pre-release 尾巴(`0.13.0-beta.1`)被丢弃当作 `0.13.0`。

### 5.2 `tac apm <...>`

把所有参数透传给系统的 `apm` 二进制(由 `exec.LookPath("apm")` 找到)。常用命令:

```bash
tac apm install                    # 按 apm.yml 拉取依赖
tac apm audit                      # 审计依赖合规性
tac apm compile -t claude          # 把 skill/agent 编译到 .claude/
tac apm update xuedizi/tac-skills/skills/tac-spec-flow
tac apm self-update                # 升级 apm 自身(走官方机制)
```

### 5.3 `tac specify <...>`

spec-kit 的版本管理子命令,与 APM 互不重叠:

```bash
tac specify version                # 打印当前 lockfile.specify.ref + .specify/.tac-installed-ref
tac specify upgrade                # 拉 latest stable tag(git ls-remote semver 排序),重跑 specify init
tac specify upgrade --ref v0.9.0   # 升到指定 ref
tac specify upgrade --dry-run      # 仅打印将要执行的动作,不落盘
tac specify upgrade --force-reinstall  # 当前已是目标 ref 时也强制重跑
tac specify upgrade --ide claude,codex  # 仅升某几个 IDE(默认升 lockfile.ides 全部)
```

### 5.4 `tac uninstall <...>`

三种语义,详见 [§九 卸载](#九卸载)。

```bash
tac uninstall tac-spec-flow                # 单卸 APM-managed 条目
tac uninstall specify                      # 卸 spec-kit
tac uninstall --all                        # 全卸 TAC
# 通用 flag: --root, --yes, --dry-run
```

### 5.5 `tac self-update`

升级 tac 自身,详见 [§3.1](#31-升级-tac-自身推荐)。

### 5.6 `tac init`

见 [§四 接入项目](#四接入项目tac-init)。

---

## 六、初始化后的项目结构

```
your-android-project/
├── .tac/
│   ├── adapter.yaml        ← TAC 流程读取的核心配置(命令、目录、网关)
│   ├── lockfile.yaml       ← 本次 init 的版本快照,含 ides + specify
│   ├── metrics/            ← (Phase 2) 度量数据落盘
│   ├── audit/              ← (Phase 2) apm 日志、hook 审计
│   └── template-overrides/ ← 项目级模板覆盖
├── .specify/                ← 由 spec-kit `specify init` 创建,tac 不写其内容
│   ├── memory/             ← spec-kit 长期记忆(升级时会被 tac 备份+还原)
│   ├── templates/          ← spec 模板(升级时会被 spec-kit 覆盖为最新)
│   └── .tac-installed-ref  ← 单行 = 当前 spec-kit ref,doctor 用来对账 lockfile
├── .claude/ /.codebuddy/ /.codex/
│   ├── settings.json       ← 已并入 TAC 的 hook
│   ├── hooks-bin/tac-hook  ← exec → `tac hook`
│   └── commands/speckit.*  ← spec-kit init 写入的命令(每个 IDE 一份)
├── apm.yml                 ← 此项目要拉的 skill/agent/command 清单
├── apm-policy.yml          ← apm 安全策略(deny MCP / plugin 等)
├── apm.lock.yaml           ← apm install 后生成的解析锁
├── doc/memory/
├── specs/                  ← /speckit.specify 创建的 feature 文档
└── .gitignore              ← 已合入 TAC managed 段
```

### 6.1 `adapter.yaml`

```yaml
schema_version: "1"
name: your-project-name
project_type: android-kotlin
paths:
  spec_dir: specs
commands:
  test: "./gradlew test"
  lint: "./gradlew ktlintCheck"
  typecheck: "./gradlew compileDebugKotlin"
  build: "./gradlew assembleDebug"
architecture:
  layers:
    - ui
    - domain
    - data
gateway:
  url: http://nexus.tinnove.com:3000
```

### 6.2 `apm.yml` 

```yaml
name: your-project-name
version: 0.1.0

dependencies:
  apm:
    - xuedizi/tac-skills/skills/tac-spec-flow#v0.1.0
    - xuedizi/tac-skills/skills/tac-plan-flow#v0.1.0
    - xuedizi/tac-skills/skills/tac-implement-flow#v0.1.0
    - xuedizi/tac-skills/agents/spec-writer.agent.md#v0.1.0
    - xuedizi/tac-skills/agents/plan-author.agent.md#v0.1.0
    - xuedizi/tac-skills/commands/tac.gate.prompt.md#v0.1.1
```
---

## 七、自动安装清单(skill / agent / command / hook)

### 7.1 Hook 入口脚本(step 8)

每个 IDE 一份,文件 `.<ide>/hooks-bin/tac-hook`,mode `0755`,内容固定:

```bash
#!/usr/bin/env bash
exec tac hook "$@"
```

> `tac hook` 子命令是 Phase 3 才挂逻辑;Phase 0/1 阶段脚本与注册位都已就位但执行体为空。

### 7.2 settings.json 合并的 4 条 hook(step 7)

每个 IDE 的 `.<ide>/settings.json`,**幂等合并**(已存在条目保留,只补不存在的):

| 事件 | matcher | command |
| --- | --- | --- |
| `PreToolUse` | `Bash` | `.<ide>/hooks-bin/tac-hook pre-bash` |
| `PreToolUse` | `Edit\|Write` | `.<ide>/hooks-bin/tac-hook pre-edit` |
| `PostToolUse` | `Bash` | `.<ide>/hooks-bin/tac-hook post-bash` |
| `Stop` | (空) | `.<ide>/hooks-bin/tac-hook on-stop` |


---

## 八、故障排查

### 8.1 `tac init` 报 `.tac already exists`

**原因**:目录已经接入过 TAC,或上次 init 失败留下了垃圾。

**处理**:
- 想保留现有配置 → 用 `tac upgrade`(Phase 2 提供;Phase 0/1 暂未实现)。
- 想完全重做 → `rm -rf .tac .specify apm.yml apm-policy.yml apm.lock.yaml` 后重跑。
- CI 临时强制 → `--force`(注意会覆盖)。

### 8.2 `apm install/audit/compile` 失败

**最常见原因**:apm registry 不可达 / 网关地址错。

**快速排查**:

```bash
curl -I $TAC_GATEWAY_URL/v1/models     # 应返回 2xx/4xx,不是 timeout
tac apm install -v                     # 看 apm 自身的错误
```

**绕过(开发/CI)**:

```bash
TAC_SKIP_APM=1 tac init --template android-kotlin --name demo --ide claude
```

这会跳过 step 9,但 **`apm.lock.yaml` 不会生成**,后续 `tac doctor` 会提示 ✗ 并要求你 `tac apm install`。

### 8.3 `apm: not found` / `prereqs ✗ apm required but not on PATH`

**原因**:tac 不内嵌 apm 二进制,需要 Microsoft APM 官方安装。

**处理**:按 §二 装 apm 后,`apm --version` 自检通过即可重跑 `tac init`。仅 e2e 测试想绕过用 `TAC_SKIP_APM=1`。

### 8.4 `specify init` 失败

**最常见原因**:`uv` 未安装,或网络拉不到 `https://github.com/github/spec-kit.git`。

**快速排查**:

```bash
uv --version                       # 没有就 curl -LsSf https://astral.sh/uv/install.sh | sh
uvx --from git+https://github.com/github/spec-kit.git@v0.8.8 specify --version
```

**绕过(离线/e2e)**:

```bash
TAC_SKIP_SPECIFY=1 tac init --template android-kotlin --name demo --ide claude
```

跳过 step 6,但 `.specify/` 不会被创建;后续 `tac doctor` 会标 `.specify ✗`。

### 8.5 `tac doctor` 报 `specify ref mismatch`

**症状**:`.specify/.tac-installed-ref` 内容与 `lockfile.yaml` 的 `specify.ref` 不一致。

**原因**:有人手动编辑过 lockfile,或在 `tac specify upgrade` 中途崩了。

**处理**:重跑 `tac specify upgrade --ref <lockfile 里那个 ref> --force-reinstall`,让两边对齐。

### 8.6 `tac doctor` 报 gateway `✗ (unreachable)`

依次确认:
1. `echo $TAC_GATEWAY_URL` 是否就是你以为的地址;
2. `curl -sS $TAC_GATEWAY_URL/v1/models` 是否真的通(超时 5s);
3. 公司 VPN / 代理是否在?nexus 仅内网可达;
4. 若确认网关 OK 只是 `/v1/models` 路径不对 — 目前 doctor 写死探测这个路径,如有变更请提 issue。

### 8.7 hook 没生效

**症状**:执行 IDE 操作时没看到 tac 拦截。

**自检**:
```bash
ls -l .claude/hooks-bin/tac-hook       # 必须存在且可执行(其他 IDE 同理)
grep tac-hook .claude/settings.json    # 必须能看到至少 4 条
which tac                              # IDE 启动的 shell 必须能找到 tac
```

> 多 IDE 时每个 IDE 目录都会有自己的 `hooks-bin/`,settings.json 引用的是当前 IDE 的相对路径,不是硬编码 `.claude/`。

如 `which tac` 在 GUI IDE 里失败,通常是因为 IDE 不读 `~/.zshrc` 之类的非交互 rc;把 `tac` 安装到 `/usr/local/bin/` 即可解决。

### 8.8 已有 `.gitignore` 被改乱

**事实**:`stepGitignore` 只在末尾**追加**未出现的行,不删、不重排。发现现有规则被改顺序,大概率是其他工具(IDE 自动格式化)所为,不是 tac。

---

## 九、卸载

`tac uninstall` 一个子命令覆盖三种语义。所有写盘操作都会先打印 plan 并要求 `y/N` 确认,`--yes` 跳过,`--dry-run` 仅打印不写。

### 9.1 单卸 APM-managed 条目

```bash
tac uninstall tac-spec-flow                # 子串匹配 apm.yml 的某条 dependency
tac uninstall tac-spec-flow --dry-run      # 仅打印将要做的事
tac uninstall tac-spec-flow --yes          # 跳过确认
```


匹配命中 0 条会直接报错并中止。

### 9.2 卸 spec-kit

```bash
tac uninstall specify
```

> 想再装回来:`tac specify upgrade --ref vX.Y.Z --force-reinstall`。

### 9.3 全卸 TAC

```bash
tac uninstall --all                        # 走 confirm 流
tac uninstall --all --dry-run              # 看 plan 不动文件
tac uninstall --all --yes                  # CI / 脚本场景
```
---

## 十一、下一步

接入完成后立刻可用:

1. 在 Claude Code 中执行 `/speckit.specify "实现 XXX 功能"` 创建第一个 feature 规约;
2. `/speckit.plan` → `/speckit.tasks` → `/speckit.implement` 走完 TDD 闭环;

---

## 附录 A:环境变量速查

| 变量 | 作用 | 默认 |
| --- | --- | --- |
| `TAC_GATEWAY_URL` | LLM 网关 base URL | `http://nexus.tinnove.com:3000` |
| `TAC_POLICY_ROOT` | `tac init` 写入 `apm-policy.yml` 的 `extends:` 值;沙箱/fork 场景用 | `xuedizi/apm-policy#v1.0.0` |
| `TAC_WITH_DEPS` | install.sh 自动装缺失的 apm/uv(同 `--with-deps`) | unset |
| `TAC_SKIP_APM` | 非空时 `tac init` 跳过 apm 三连;用于离线/测试 | unset |
| `TAC_SKIP_SPECIFY` | 非空时 `tac init` 跳过 step 6 specify init;同时让 prereqs 不强制 uv | unset |
| `TAC_SPECIFY_REF` | 覆盖 `lockfile.specify.ref` 与 `DefaultRef`,强行让 stepSpecify / specify upgrade 用此 ref | unset |
| `TAC_SPECIFY_GITHUB_TOKEN` | 透传给 `uvx specify init` 的 `GH_TOKEN` / `GITHUB_TOKEN`,绕过 GitHub 速率限制 | unset |
| `TAC_CI` | 非空时 `tac apm install` 加 `--frozen` 拒绝 lockfile drift | unset |
| `TAC_INTEGRATION` | tac 仓库内部端到端测试触发开关,**业务项目无须关心** | unset |

## 附录 B:命令一句话速查

```
tac --version                                查看版本
tac doctor                                   全量体检(全局 + 项目)
tac doctor --no-project                      仅 tac 自身
tac init --template android-kotlin --ide ... 接入项目
tac apm install                              按 apm.yml 拉依赖
tac apm compile -t claude                    编译 skill 到 .claude/
tac specify version                          看当前 spec-kit ref
tac specify upgrade [--ref v0.9.0]           升 spec-kit
tac uninstall <name> [--dry-run] [--yes]     单卸 apm.yml 内某条目
tac uninstall specify                        卸 spec-kit
tac uninstall --all                          全卸 TAC(确认后执行)
tac self-update [--check|--version v|--dry-run|--force]  升级 tac 自身
```

---

## 版本历史

完整版本列表见 [Releases](https://github.com/xuedizi/tac-releases/releases)。
