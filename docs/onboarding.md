# TAC 接入手册(Phase 1)

> 适用版本:`tac v0.2.x`(spec-kit / APM 官方接入版本)
> 适用项目类型:Android-Kotlin(Phase 0 唯一交付的 starter pack)
> 文档对应二进制行为:见 `docs/superpowers/specs/2026-05-12-speckit-official-integration-design.md` 与 `…-apm-official-integration-design.md`

本手册指导一个**已有的 Android-Kotlin 工程**接入 Tinnove AI Coding(TAC)平台,完成后即可在 Claude Code / CodeBuddy / Codex 等 AI IDE 中使用 `/speckit.specify`、`/speckit.plan`、`/speckit.implement` 等流程,以及 TAC 提供的 `tac.gate`、`tac.commit`、`tac.pr` 等命令。

---

## 一、前置条件

| 项 | 要求 | 校验命令 |
| --- | --- | --- |
| 项目本身 | 已是 git 仓库(普通仓库或 worktree 均可) | `git rev-parse --git-dir` |
| 操作系统 | macOS / Linux | `uname -s` |
| 网络 | 可访问 `nexus.tinnove.com`(内网)用于拉取 apm registry;可访问 GitHub 用于拉 spec-kit | `curl -I http://nexus.tinnove.com:3000/v1/models` |
| **apm**(Microsoft) | 官方安装,在 `$PATH` 上 | `apm --version` |
| **uv**(astral) | spec-kit 通过 `uvx` 调起,必须在 `$PATH` 上 | `uv --version` |
| AI IDE | 安装下列任一(或多个):Claude Code、CodeBuddy、Codex | `claude --version` 等 |
| `TAC_GATEWAY_URL` | 推荐先 `export` 到 shell rc | `echo $TAC_GATEWAY_URL` |

> ⚠️ **未配置 `TAC_GATEWAY_URL`** 时 tac 会回退到默认 `http://nexus.tinnove.com:3000`;
> 如果你的项目要走自建网关,务必先 `export TAC_GATEWAY_URL=...` 再 `tac init`,否则需要事后手工改 `.tac/adapter.yaml`。

### 1.1 安装 apm 与 uv

`tac` 不再内嵌 apm 二进制,改为依赖系统安装。两个二进制都是必备前置:

```bash
# apm (Microsoft):四种官方通道任选其一,详见 https://aka.ms/apm-unix
curl -fsSL https://aka.ms/install-apm.sh | bash
# 或: brew install microsoft/tap/apm

# uv (astral):
curl -LsSf https://astral.sh/uv/install.sh | sh
```

`tac init` 第一步 prereqs 会校验两者均在 `$PATH`,缺失则 fail-fast 并打印官方安装链接,**不再有"绕过"参数**(测试场景另见环境变量 `TAC_SKIP_APM` / `TAC_SKIP_SPECIFY`)。

---

## 二、安装 tac

### 2.1 二进制下载(推荐)

```bash
# macOS arm64 示例;Linux 把 darwin 换成 linux 即可
curl -fsSL http://nexus.tinnove.com:50382/repository/tac-releases/v0.2.0/tac-darwin-arm64 \
  -o /usr/local/bin/tac
chmod +x /usr/local/bin/tac
tac --version
# → tac v0.2.0 (commit <sha>, built <date>)
```

### 2.2 从源码构建(贡献者)

```bash
git clone <tac repo>
cd tac
make build          # 输出 bin/tac
./bin/tac --version
```

### 2.3 体检 tac 本身

在任意目录(无须是 TAC 项目)执行:

```bash
tac doctor --no-project
```

期望输出:

```
[tac] tac v0.2.0 (commit <sha>, built <date>)
[tac] platform: darwin/arm64
[tac] gateway env: http://nexus.tinnove.com:3000 ✓
[tac] uv: /opt/homebrew/bin/uv (uv 0.5.x)
[tac] apm: /opt/homebrew/bin/apm (apm 0.x.x)
[tac] detected IDEs (PATH): [claude]
```

任一项打 `✗` 都需要先解决——见 [§七 故障排查](#七故障排查)。`apm` 缺失会让 `tac doctor` 整体非零退出。

---

## 三、第一次接入:`tac init`

### 3.1 标准流程

```bash
cd /path/to/your-android-project        # 必须是 git 仓库
export TAC_GATEWAY_URL=http://nexus.tinnove.com:3000

tac init \
  --template android-kotlin \
  --name your-project-name \
  --gateway $TAC_GATEWAY_URL \
  --ide claude,codex                   # 可省略,见 §3.2
```

`tac init` 会顺序跑 12 个步骤,每步以 `[N/12] label ✓` 形式落屏:

| # | 步骤 | 产物 |
| --- | --- | --- |
| 1 | prereqs | 校验是 git 仓库、`.tac` 未占用、`--template` 合法、`apm` 与(必要时)`uv` 在 PATH |
| 2 | resolve IDEs | 按 `--ide` → 磁盘探测 → tty 交互 → fail-fast 顺序确定 IDE 列表 |
| 3 | skeleton | 建立 `.tac/`、`doc/memory/`、`specs/` 等目录(**不再** mkdir `.specify/*`,留给 spec-kit) |
| 4 | starter pack | 渲染并落地 `apm.yml`、`apm-policy.yml`、临时 `.gitignore.tac` |
| 5 | adapter.yaml | 写出 `.tac/adapter.yaml`(TAC 流程的核心配置) |
| 6 | specify init | 调 `uvx specify init --integration <ide>` 给每个 IDE 落 spec-kit 命令 + 模板;写 `.specify/.tac-installed-ref` |
| 7 | hook registration | 把 TAC hook 合并进每个 IDE 的 `settings.json`(路径按 IDE 拼,不再硬编码 `.claude/`) |
| 8 | hook entry script | 给每个 IDE 写出 `.<ide>/hooks-bin/tac-hook`(chmod 0755) |
| 9 | apm install/audit/compile | 调系统 `apm`,拉取 skill/agent/command 包,逐 IDE compile;`TAC_CI=1` 时 `install --frozen` |
| 10 | gitignore | 把 `.gitignore.tac` 行级合并进真正的 `.gitignore`,并删除临时文件 |
| 11 | lockfile | 写出 `.tac/lockfile.yaml`(含 `ides:` 与 `specify:`) |
| 12 | final doctor | 打印欢迎语与下一步指引 |

成功结尾:

```
✓ 项目 your-project-name 已接入 TAC v0.2.0
```

### 3.2 关键参数

| flag | 必填 | 说明 |
| --- | --- | --- |
| `--template` | ✅ | Phase 0 唯一可用值:`android-kotlin` |
| `--name` | | 项目名;默认取目录 basename |
| `--gateway` | | LLM 网关 URL;默认依次取 flag → `TAC_GATEWAY_URL` → 内置兜底 |
| `--policy-root` | | `apm-policy.yml extends:` 的目标;优先级 flag → `TAC_POLICY_ROOT` → 内置兜底 `xuedizi/apm-policy#v1.0.0`。沙箱/fork 场景可切到自定义根策略,正式项目保持默认 |
| `--root` | | 项目根目录;默认 cwd,CI 里建议显式传 |
| `--force` | | 已存在 `.tac/` 时是否覆盖,**会丢失现有 TAC 配置** |
| `--ide` | | 逗号分隔的 IDE 列表(`claude` / `codebuddy` / `codex`),省略时按下文优先级链解析 |

#### `--ide` 解析优先级

step 2 `resolve IDEs` 按下面四档依次决策,**找到任一档就停止,不会再"兜底默认 claude"**:

1. `--ide flag` 显式指定 → 用此值(每个值会做合法性校验)
2. 磁盘探测 → 项目里已经有 `.claude/` / `.codebuddy/` / `.codex/` 任一目录,就用全部检测到的
3. tty 交互 → 上面两档都没命中且 `stdin` 是 tty 时,提示用户从 `[1/2/3]` 或名字里选,逗号分隔多选
4. fail-fast → 上面都不满足(典型是 CI 非 tty)→ 报错并要求显式传 `--ide`

最终落定的 IDE 列表会写入 `.tac/lockfile.yaml` 的 `ides:` 字段,后续 `apm compile`、`tac specify upgrade` 等都从这里读,不会再 detect。

### 3.3 提交初始化结果

`tac init` **不会自动 commit**。完成后执行:

```bash
git add .tac/ .specify/ apm.yml apm.lock.yaml apm-policy.yml .gitignore .claude/ .codebuddy/ .codex/
git commit -m "chore: bootstrap TAC IDP"
```

> 💡 公司内部 Gerrit 仓库注意:`commit-msg` 钩子会强制 8 字段格式。若被拦,可在消息体中包含 `init|patch|merge|revert` 任一关键字以走 `checkRevert` 旁路。

---

## 四、命令参考

### 4.1 `tac doctor`

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

> 版本下限存在 `internal/doctor/compat.go` 的 `Defaults()`,bump 依赖期望只改一处。
> 解析使用宽容正则,容忍 `apm 0.13.0`、`uv 0.10.11 (Homebrew ...)`、`git version 2.39.5 (Apple Git-154)` 等格式;
> pre-release 尾巴(`0.13.0-beta.1`)被丢弃当作 `0.13.0`。

### 4.2 `tac apm <...>`

把所有参数透传给系统的 `apm` 二进制(由 `exec.LookPath("apm")` 找到)。常用命令:

```bash
tac apm install                    # 按 apm.yml 拉取依赖
tac apm audit                      # 审计依赖合规性
tac apm compile -t claude          # 把 skill/agent 编译到 .claude/
tac apm update xuedizi/tac-skills/skills/tac-spec-flow
tac apm self-update                # 升级 apm 自身(走官方机制)
```

> 这条命令开了 `DisableFlagParsing`,所有 flag 都原样下沉给 apm,**不要**期待 cobra 帮你提示用法,直接 `tac apm --help`。
>
> 如果 `apm` 不在 PATH,`tac apm` 会立刻 fail 并打印官方安装四通道(curl / brew / scoop / signed-archive),无 fallback。

### 4.3 `tac specify <...>`

spec-kit 的版本管理子命令,与 APM 互不重叠:

```bash
tac specify version                # 打印当前 lockfile.specify.ref + .specify/.tac-installed-ref
tac specify upgrade                # 拉 latest stable tag(git ls-remote semver 排序),重跑 specify init
tac specify upgrade --ref v1.3.0   # 升到指定 ref
tac specify upgrade --dry-run      # 仅打印将要执行的动作,不落盘
tac specify upgrade --force-reinstall  # 当前已是目标 ref 时也强制重跑
tac specify upgrade --ide claude,codex  # 仅升某几个 IDE(默认升 lockfile.ides 全部)
```

`upgrade` 会:
1. 备份 `.specify/memory/` 到 `$TMPDIR/tac-specify-memory.<pid>/`
2. 对每个目标 IDE 调一次 `uvx --from git+<RepoURL>@<ref> specify init --here --integration <ide>`(stdin 透传,`--script` / 目录非空 / git 等由 spec-kit 自行向用户询问)
3. 还原 `.specify/memory/`(spec-kit 会覆盖 templates 但 memory 是项目记忆,必须保住)
4. 更新 `.tac/lockfile.yaml` 的 `specify.previous_ref` 与 `specify.ref`,刷新 `installed_at`
5. 重写 `.specify/.tac-installed-ref` 与 lockfile 同步

中途任一 IDE 的 `uvx` 失败,memory 备份不会被删,错误信息里给出 `cp -R` 还原命令。

### 4.4 `tac init`

见 [§三](#三第一次接入tac-init)。

### 4.5 `tac uninstall <...>`

三种语义,详见 [§8.5](#85-卸载单个--全部):

```bash
tac uninstall tac-spec-flow                # 单卸 APM-managed 条目
tac uninstall specify                      # 卸 spec-kit
tac uninstall --all                        # 全卸 TAC
# 通用 flag: --root, --yes, --dry-run
```

### 4.6 `tac self-update`

升级 tac 自身,详见 [§8.1](#81-升级-tac-自身):

```bash
tac self-update                            # 拉 latest 原子替换
tac self-update --check                    # 只查不装
tac self-update --version v0.2.1           # pin 到具体 tag
tac self-update --dry-run                  # 打印计划不写盘
tac self-update --force                    # 重装当前版本
```

---

## 五、初始化后的项目结构

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
│   ├── hooks-bin/tac-hook  ← exec → `tac hook`(Phase 3 实现)
│   └── commands/speckit.*  ← spec-kit init 写入的命令(每个 IDE 一份)
├── apm.yml                 ← 此项目要拉的 skill/agent/command 清单
├── apm-policy.yml          ← apm 安全策略(deny MCP / plugin 等)
├── apm.lock.yaml           ← apm install 后生成的解析锁
├── doc/memory/
├── specs/                  ← /speckit.specify 创建的 feature 文档
└── .gitignore              ← 已合入 TAC managed 段
```

### 5.1 `adapter.yaml` 长这样

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

如果你的 gradle task 名字不同(例如使用 `ktlintMainCheck` 或多 module),修改 `commands.*` 即可,TAC 的 `tac.gate` 命令会照搬执行。

### 5.2 `apm.yml` 长这样(节选)

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

> spec-kit(`github/spec-kit`)**不再** 写入 `apm.yml`——它由 `tac specify` 接管。
> 升级某个 APM 治理的 skill 时,改这里的 `#vX.Y.Z` 后 `tac apm install && tac apm compile -t claude`。

### 5.3 `lockfile.yaml` 长这样

```yaml
schema_version: "1"
tac_version: 0.2.0
adapter_schema: "1"
starter_pack:
  type: android-kotlin
  version: 0.1.0
ides:
  - claude
  - codex
specify:
  ref: v0.8.8
  previous_ref: ""           # 第一次升级前为空
  installed_at: 2026-05-12T08:30:00Z
created_at: 2026-05-12T08:00:00Z
```

`ides:` 与 `specify:` 是 Phase 1 新增字段;旧版 lockfile 缺这两段也能读出(`omitempty`),但 `tac doctor` 会提示重 init 才能用上 `tac specify upgrade`。

---

## 六、自动安装清单(skill / agent / command / hook)

### 6.1 Hook 入口脚本(step 8)

每个 IDE 一份,文件 `.<ide>/hooks-bin/tac-hook`,mode `0755`,内容固定:

```bash
#!/usr/bin/env bash
exec tac hook "$@"
```

> `tac hook` 子命令是 Phase 3 才挂逻辑;Phase 0/1 阶段脚本与注册位都已就位但执行体为空。

### 6.2 settings.json 合并的 4 条 hook(step 7)

每个 IDE 的 `.<ide>/settings.json`,**幂等合并**(已存在条目保留,只补不存在的):

| 事件 | matcher | command |
| --- | --- | --- |
| `PreToolUse` | `Bash` | `.<ide>/hooks-bin/tac-hook pre-bash` |
| `PreToolUse` | `Edit\|Write` | `.<ide>/hooks-bin/tac-hook pre-edit` |
| `PostToolUse` | `Bash` | `.<ide>/hooks-bin/tac-hook post-bash` |
| `Stop` | (空) | `.<ide>/hooks-bin/tac-hook on-stop` |

> 来源 `tac/internal/ide/hookreg.go` `hookSuffixes` + `entriesForIDE`,路径按 IDE 动态拼,不会出现 `.codebuddy/settings.json` 引用 `.claude/hooks-bin/` 的旧 bug。

### 6.3 spec-kit 落地的命令与目录(step 6)

`uvx --from git+<RepoURL>@<ref> specify init --here --integration <ide>` 由 spec-kit 上游写出(以 `v0.8.8` 为准;`--script` 等选项由 spec-kit 交互式询问用户):

- `.specify/memory/`(spec-kit 长期记忆,`tac specify upgrade` 会备份+还原)
- `.specify/templates/`(spec/plan/tasks 模板,升级时被 spec-kit 覆盖为最新)
- `.specify/scripts/bash/*`(spec-kit 自带 shell 助手)
- `.<ide>/commands/speckit.specify.md`、`speckit.plan.md`、`speckit.tasks.md`、`speckit.implement.md`、`speckit.clarify.md`、`speckit.constitution.md`、`speckit.analyze.md`(每个已 init 的 IDE 各一份)

由 tac 自己再追加一行标记:

- `.specify/.tac-installed-ref` = 当前 spec-kit ref(单行,doctor 用来与 `lockfile.specify.ref` 对账)

> 具体清单以当时 `<ref>` 的 spec-kit 为准,tac **不复刻、不挑选**。

### 6.4 APM 拉取的 skill / agent / command(step 9)

来源 `apm.yml`(由 `tac/internal/starter/packs/android-kotlin/apm.yml.tpl` 渲染),落到每个 IDE 的 `.<ide>/{skills,agents,commands}/`。skill/agent pin `#v0.1.0`、command pin `#v0.1.1`:

**Skills(9 个,`xuedizi/tac-skills/skills/...`)**

```
tac-spec-flow         tac-clarify-flow      tac-plan-flow
tac-tasks-flow        tac-implement-flow    tac-gate-flow
tac-commit-flow       tac-pr-flow           tac-metrics-flow
```

**Agents(5 个,`xuedizi/tac-skills/agents/...`)**

```
spec-writer.agent.md         plan-author.agent.md       tdd-implementer.agent.md
gate-runner.agent.md         commit-reviewer.agent.md
```

**Commands(3 个,`xuedizi/tac-skills/commands/...`)**

```
tac.gate    tac.commit    tac.pr
```

> spec-kit 的 `speckit.*` 命令**不在** `apm.yml`——它走 `tac specify`,见 §6.3 / §4.3。这是 APM 与 spec-kit 治理边界的体现:**每个 skill 的升级路径只能有一个权威源**。

> Starter pack(android-kotlin 默认值)**内化为 TAC binary 嵌入模板**(`tac/internal/starter/packs/android-kotlin/`)——升级 starter 字段(layers / commands / linting)= TAC binary 发版,不走远程治理通道。

### 6.5 APM 安全策略(step 4 写,step 9 生效)

`apm-policy.yml` 内容:

```yaml
extends: xuedizi/apm-policy#v1.0.0
mcp:
  trust_boundary: deny_all
  allowed_transports: []
primitives:
  # plugin 交由组织根策略 `xuedizi/apm-policy` 统一治理(`tighten_only` 继承),
  # 此处不再硬拒;根策略允许后,新建项目即可声明 plugin 依赖。
  denied: [mcp_server, instruction]
```

含义:禁所有 MCP 入站、禁 mcp_server / instruction 两类高危原语;skill / agent / command / plugin 等"声明式"原语由组织根策略决定是否放行。skill / agent / command 默认放行;plugin 当前在根策略中默认 deny,需平台 SIG 评审通过后在 `xuedizi/apm-policy` 放开。

### 6.6 治理边界一图速记

| 来源 | 升级命令 | 版本来源 |
| --- | --- | --- |
| TAC 自家 skill / agent / command(`xuedizi/tac-skills/...`) | `tac apm install / compile` | `apm.yml` `#vX.Y.Z` |
| 第三方 spec-kit(`speckit.*` 命令、模板、memory) | `tac specify upgrade` | `.tac/lockfile.yaml` `specify.ref` |
| 第三方 APM(无独立升级机制时) | `tac apm install / compile` | `apm.yml` |
| apm 自身 | `tac apm self-update`(透传) | apm 官方 |

`spec-kit` 不写 `apm.yml`、APM 不复刻 spec-kit;两套 lockfile 各管一边,doctor 项目段会分别校验一致性。

---

## 七、故障排查

### 7.1 `tac init` 报 `.tac already exists`

**原因**:目录已经接入过 TAC,或上次 init 失败留下了垃圾。

**处理**:
- 想保留现有配置 → 用 `tac upgrade`(Phase 2 提供;Phase 0 暂未实现)。
- 想完全重做 → `rm -rf .tac .specify apm.yml apm-policy.yml apm.lock.yaml` 后重跑。
- CI 临时强制 → `--force`(注意会覆盖)。

### 7.2 `step 9 (apm install/audit/compile)` 失败

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

这会跳过 step 9,但**`apm.lock.yaml` 不会生成**,后续 `tac doctor` 会提示 ✗ 并要求你 `tac apm install`。

### 7.3 `apm: not found` / `prereqs ✗ apm required but not on PATH`

**原因**:tac 不再内嵌 apm 二进制,需要 Microsoft APM 官方安装。

**处理**:按 §1.1 装 apm 后,`apm --version` 自检通过即可重跑 `tac init`。仍想绕过(仅限 e2e 测试)用 `TAC_SKIP_APM=1`。

### 7.4 `step 6 (specify init)` 失败

**最常见原因**:`uv` 未安装,或网络拉不到 `https://github.com/github/spec-kit.git`。

**快速排查**:

```bash
uv --version                       # 没有就 curl -LsSf https://astral.sh/uv/install.sh | sh
# 手动跑一遍 tac 即将执行的命令,看错误细节
uvx --from git+https://github.com/github/spec-kit.git@v0.8.8 specify --version
```

**绕过(离线/e2e)**:

```bash
TAC_SKIP_SPECIFY=1 tac init --template android-kotlin --name demo --ide claude
```

跳过 step 6,但 `.specify/` 不会被创建;后续 `tac doctor` 会标 `.specify ✗`。

### 7.5 `tac doctor` 报 `specify ref mismatch`

**症状**:`.specify/.tac-installed-ref` 内容与 `lockfile.yaml` 的 `specify.ref` 不一致。

**原因**:有人手动编辑过 lockfile,或在 `tac specify upgrade` 中途崩了。

**处理**:重跑 `tac specify upgrade --ref <lockfile 里那个 ref> --force-reinstall`,让两边对齐。

### 7.6 `tac doctor` 报 gateway `✗ (unreachable)`

依次确认:
1. `echo $TAC_GATEWAY_URL` 是否就是你以为的地址;
2. `curl -sS $TAC_GATEWAY_URL/v1/models` 是否真的通(超时 5s);
3. 公司 VPN / 代理是否在?nexus 仅内网可达;
4. 若你确认网关 OK,只是 `/v1/models` 路径不对——目前 doctor 写死探测这个路径,如有变更请提 issue。

### 7.7 hook 没生效

**症状**:执行 IDE 操作时没看到 tac 拦截。

**自检**:
```bash
ls -l .claude/hooks-bin/tac-hook       # 必须存在且可执行(其他 IDE 同理 .codebuddy / .codex)
grep tac-hook .claude/settings.json    # 必须能看到至少 4 条
which tac                              # IDE 启动的 shell 必须能找到 tac
```

> 注意:多 IDE 时每个 IDE 目录都会有自己的 `hooks-bin/`,settings.json 引用的是当前 IDE 的相对路径,不是硬编码 `.claude/`。

如 `which tac` 在 GUI IDE 里失败,通常是因为 IDE 不读 `~/.zshrc` 之类的非交互 rc;把 `tac` 安装到 `/usr/local/bin/` 即可解决。

### 7.8 已有 `.gitignore` 被改乱

**事实**:`stepGitignore` 只在末尾**追加**未出现的行,不删、不重排。如果发现现有规则被改顺序,大概率是其他工具(IDE 自动格式化)所为,不是 tac。

---

## 八、升级与卸载

### 8.1 升级 tac 自身

```bash
tac self-update                         # 拉 /releases/latest 并原子替换
tac self-update --check                 # 只查不装
tac self-update --version v0.2.1        # 锁版本
tac --version                            # 验证
tac doctor                               # 验证现有项目仍能通过
```

`self-update` 直接读 `https://github.com/xuedizi/tac-releases/releases/latest/download/manifest.json`,sha256 校验通过后原子替换正在运行的二进制(同目录写 tmp → rename;Windows 下先 mv self → `*.old`)。安装目录不可写时会提示重跑 installer 加 `--prefix $HOME/.local/bin`。

兜底(网络/权限受限场景):

```bash
curl -fsSL https://github.com/xuedizi/tac-releases/releases/latest/download/install.sh | sh
```

由于 starter pack 模板编译进二进制,**升级 tac 不会自动更新已 init 的项目**——需走 `tac upgrade`(Phase 2)或手工对照新模板调整。

### 8.2 升级 spec-kit

走 tac 内置子命令(单一权威源 = `lockfile.specify.ref`):

```bash
tac specify version                       # 看当前 ref
tac specify upgrade                       # 拉 latest stable
tac specify upgrade --ref v1.3.0          # 锁定具体 ref
git add .tac/lockfile.yaml .specify .claude .codebuddy .codex \
  && git commit -m "chore: bump spec-kit to v1.3.0"
```

> `apm.yml` 里**不再** 出现 `github/spec-kit/...`——spec-kit 由这条命令独占管理,不要再加回 APM 依赖,否则双源会漂移。

### 8.3 升级 APM 治理的 skill / agent

改 `apm.yml` 里对应条目的 `#vX.Y.Z`,然后:

```bash
tac apm install
tac apm compile -t claude       # 按你实际接入的 IDE 重复 -t
git add apm.yml apm.lock.yaml .claude && git commit -m "chore: bump tac-spec-flow"
```

### 8.4 升级 apm 自身

走 apm 官方机制,tac 不介入:

```bash
tac apm self-update      # 等价于直接调系统 apm self-update
```

### 8.5 卸载(单个 / 全部)

`tac uninstall` 一个子命令覆盖三种语义。所有写盘操作都会先打印 plan 并要求 `y/N` 确认,`--yes` 跳过,`--dry-run` 仅打印不写。

#### 8.5.1 单卸 APM-managed 条目

```bash
tac uninstall tac-spec-flow                # 子串匹配 apm.yml 的某条 dependency
tac uninstall tac-spec-flow --dry-run      # 仅打印将要做的事
tac uninstall tac-spec-flow --yes          # 跳过确认
```

行为:
1. 在 `apm.yml` 中按子串匹配命中条目并删除(版本号 `#vX.Y.Z` 之前的部分,允许 `tac-spec-flow`、`skills/skills/tac-spec-flow`、全限定名 任一粒度);
2. 重跑 `apm install` 刷新 `apm.lock.yaml`;
3. 对 `lockfile.yaml` `ides:` 里每个 IDE 跑一次 `apm compile -t <ide>`。

匹配命中 0 条会直接报错并中止。

#### 8.5.2 卸 spec-kit

```bash
tac uninstall specify
```

行为:
1. `rm -rf .specify/`;
2. 对每个 IDE `rm .<ide>/commands/speckit.*.md`(其他命令如 `tac.gate.md` 不动);
3. 把 `.tac/lockfile.yaml` 的 `specify:` 段清空(ref / previous_ref / installed_at)。

> 想再装回来:`tac specify upgrade --ref vX.Y.Z --force-reinstall`。

#### 8.5.3 全卸 TAC

```bash
tac uninstall --all                        # 走 confirm 流
tac uninstall --all --dry-run              # 看 plan 不动文件
tac uninstall --all --yes                  # CI / 脚本场景
```

行为:
1. 读 `lockfile.yaml` 的 `ides:`(没有则降级到磁盘探测 `.claude/.codebuddy/.codex/`)以确定影响范围;
2. `rm -rf .tac/ .specify/`;
3. `rm -f apm.yml apm-policy.yml apm.lock.yaml`;
4. 对每个 IDE:
   - `rm -rf .<ide>/hooks-bin/`;
   - 对 `.<ide>/commands/speckit.*.md` 逐个删除;
   - 解析 `.<ide>/settings.json`,只剔除引用 `.<ide>/hooks-bin/tac-hook` 的 hook 条目;`event` 数组若空则删 event;`hooks` map 若空则删 key;整个 doc 空则删文件。**用户自有的 hook 与 settings 项保留**;
5. 清 `.gitignore`,只摘 `# TAC managed` / `.tac/metrics/` / `.tac/audit/` / `.claude/` / `.codebuddy/` / `.codex/` / `.apm/` 这几行,用户其他规则不动。

**不会** 自动 rm `.<ide>/{skills,agents,commands/tac.*}` 这些 `apm compile` 写出的产物——那是 apm 的事,如需清理走 `apm clean` 或手工 `rm -rf`。

---

## 九、从 Phase 0 迁移到 Phase 1

如果你早期用 Phase 0 的 tac 已经 init 过项目(没有 `ides:` / `specify:` 段、`apm.yml` 里还有 `github/spec-kit/...` 行),按下面步骤迁移:

1. **升级 tac 二进制** 到 v0.2.x(§8.1);
2. **装 apm 与 uv**(§1.1);旧版项目里的 `internal/apm/bin/` 与 `scripts/fetch-apm-binary.sh` 已经从仓库删除,本地无残留可忽略。
3. **从 `apm.yml` 里删 `github/spec-kit/...` 行**;`tac apm install` 重生成 `apm.lock.yaml`;
4. **首次走 spec-kit 官方接入**:
   ```bash
   tac specify upgrade --ref v0.8.8 --force-reinstall
   ```
   这会把 `.specify/` 由 spec-kit 重建,并写入 `.specify/.tac-installed-ref`;
5. **手工补 `lockfile.yaml`** 的 `ides:` 与 `specify:` 段——或干脆 `tac init --force` 重接入(更安全);
6. `tac doctor` 全绿后 commit。

---

## 十、下一步

接入完成后立刻可用:

1. 在 Claude Code 中执行 `/speckit.specify "实现 XXX 功能"` 创建第一个 feature 规约;
2. `/speckit.plan` → `/speckit.tasks` → `/speckit.implement` 走完 TDD 闭环;
3. `tac.gate` 执行项目级质量门禁(跑 `adapter.yaml` 里的 test/lint/typecheck);
4. `tac.commit` / `tac.pr` 生成符合公司 Gerrit 规范的提交。

更详细的工作流文档参见:**《企业级 AI 编程 IDP 平台方案》**(`docs/企业级AI编程IDP平台方案.md`)。

---

## 附录 A:环境变量速查

| 变量 | 作用 | 默认 |
| --- | --- | --- |
| `TAC_GATEWAY_URL` | LLM 网关 base URL | `http://nexus.tinnove.com:3000` |
| `TAC_POLICY_ROOT` | `tac init` 写入 `apm-policy.yml` 的 `extends:` 值;沙箱/fork 场景用 | `xuedizi/apm-policy#v1.0.0` |
| `TAC_SKIP_APM` | 设置为非空时,`tac init` 跳过 apm 三连;用于离线/测试 | unset |
| `TAC_SKIP_SPECIFY` | 设置为非空时,`tac init` 跳过 step 6 specify init;同时让 prereqs 不强制 uv | unset |
| `TAC_SPECIFY_REF` | 覆盖 `lockfile.specify.ref` 与 `DefaultRef`,强行让 stepSpecify / specify upgrade 用此 ref | unset |
| `TAC_SPECIFY_GITHUB_TOKEN` | 透传给 `uvx specify init` 的 `GH_TOKEN` / `GITHUB_TOKEN`,绕过 GitHub 速率限制 | unset |
| `TAC_CI` | 设置为非空时,`tac apm install` 加 `--frozen` 拒绝 lockfile drift | unset |
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
tac specify upgrade [--ref v1.3.0]           升 spec-kit
tac uninstall <name> [--dry-run] [--yes]     单卸 apm.yml 内某条目
tac uninstall specify                        卸 spec-kit
tac uninstall --all                          全卸 TAC(确认后执行)
tac self-update [--check|--version v|--dry-run|--force]  升级 tac 自身
```
