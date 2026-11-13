# Repository Guidance

本仓库存放 tuuject 各项目共用的 Coding Agent Skill，经 `skills` CLI 安装到各项目。

## 结构

- 每个 skill 一个目录：`skills/<skill-name>/SKILL.md`，frontmatter 的 `name` 与目录名一致。
- 名字统一用 `tuuject-` 前缀，避免与项目自带或第三方 skill 重名。
- 只在 `SKILL.md` 里放每次都要的流程；按需读取的长步骤放 `references/`，Codex 的展示信息放 `agents/openai.yaml`。

## 写法约束

- **项目无关**：skill 会装进不同项目，不写某个项目特有的命令、路径、脚本名、CI 文件名或领域规则。需要项目事实时，写「按项目 AGENTS.md」并给出 AGENTS.md 没写时的默认做法。
- 只属于单个项目的 skill（例如依赖该项目源码路径或在 skill 目录里维护状态的）留在该项目的 `skills/`，不要搬进来。
- skill 运行在使用方项目的 checkout 里，不要假设本仓库的路径存在，也不要让 skill 写回自己的目录（安装后的副本会在下次同步时被覆盖）。
- description 写清何时触发，不只写它是什么。
- 不提交密钥、Token、本机绝对路径。

## 改动之后

- 新增、删除或改名 skill 时同步 `README.md` 的列表与安装示例。
- 改名属于破坏性变更：使用方 `package.json` 的 `skills` 字段要同步改，在提交说明里写明。
- 发布：在 `main` 上打 `vX.Y.Z` tag 并推送，使用方把 `skills` 字段的 `ref` 改成新 tag 后运行 `skills:sync`。已推送的 tag 不改写，使用方的 `skills-lock.json` 靠它校验内容哈希。
