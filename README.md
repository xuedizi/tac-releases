# starter-android-kotlin

Android / Kotlin 项目的 TAC starter pack。声明一组经过验证的
`adapter.yaml` 默认值 + 推荐的 skill / agent / command 组合(指向
`xuedizi/tac-skills`)+ 推荐的 policy 基线(指向 `xuedizi/apm-policy`)。

## 引用方式

### 作为 `apm.yml` 依赖(让 APM 拉到本地缓存)

```yaml
dependencies:
  apm:
    - xuedizi/starter-android-kotlin#v0.1.0
```

### 作为 `.tac/adapter.yaml` 继承源

```yaml
schema_version: "1"
name: my-android-app
extends: xuedizi/starter-android-kotlin#v0.1.0
gateway:
  url: http://nexus.tinnove.com:3000   # 子项目只 override 必要字段
```

## 内容

| 文件 | 说明 |
|---|---|
| `adapter.yaml` | 默认的 adapter 基线(paths / commands / architecture / recommended_apm / recommended_policy) |
| `VERSION` | 单一版本事实源 |
| `CHANGELOG.md` | 版本变更记录 |

## 版本与其他 repo 的组合矩阵

| repo | pinned 版本 | 在哪里固定 |
|---|---|---|
| `xuedizi/apm-policy` | `v1.0.0` | `adapter.yaml` 的 `recommended_policy` |
| `xuedizi/tac-skills` | `v0.1.0` | `adapter.yaml` 的 `recommended_apm.*` 三段 |

starter 每次 bump 都要在 `recommended_apm` / `recommended_policy` 里 pin 当时
已发布的依赖版本;CI 校验所有引用都是 `xuedizi/*` 域的合法 tag。
