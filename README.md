# tuuject/skills

tuuject 各项目共用的 Coding Agent Skill。这里放跨项目一致的流程骨架；每个项目自己的命令、分支和约定写在该项目的 `AGENTS.md`，skill 运行时去读。

## Skills

| Skill | 用途 |
| --- | --- |
| [`tuu-create-feature`](skills/tuu-create-feature/SKILL.md) | 新 feature 先建 worktree；实现需求后验证、提交、写 devlog、push、开 PR；PR 在 review worktree 审阅、修改与合并 |
| [`tuu-review-design-prd`](skills/tuu-review-design-prd/SKILL.md) | 审阅或新建 `doc/design/` 下的 PRD，意见写进待定区等用户裁决 |

## 安装到项目

推荐写进项目 `package.json` 的 `skills` 字段，由 [`skills`](https://www.npmjs.com/package/skills) CLI 安装到 `.agents/skills/`、`.claude/skills/`，并把内容哈希记进 `skills-lock.json`：

```jsonc
{
  "skills": [
    {
      "source": "tuuject/skills",
      "ref": "v0.2.0",
      "skills": ["tuu-create-feature", "tuu-review-design-prd"]
    }
  ]
}
```

`ref` 填本仓库的 tag。CLI 用 `git clone --branch` 拉取，只认 tag 或分支名，不认 commit hash；省略时跟随默认分支。一次性安装也可以直接：

```sh
npx skills add tuuject/skills --skill tuu-create-feature --skill tuu-review-design-prd
```

## 项目需要在 AGENTS.md 写清的事

skill 只定流程，下面这些按项目读 `AGENTS.md`，没写时用括号里的默认：

- 集成分支（远端有 `develop` 用 `develop`，否则用远端默认分支）。
- 装依赖、lint、类型检查、测试、e2e、文档构建等验证命令，以及哪些改动要追加哪些检查（按锁文件装依赖；lint、类型检查与受影响测试）。
- 提交信息格式、不入库的文件、PR 说明必须列出的条目（跟随 git 历史与最近合并的 PR）。
- devlog 目录（仓库里有 `agent-doc/devlogs/` 才写）。
- 可以直接在集成分支提交的例外（只改 `doc/design/` PRD）。
- PR 合并方式（跟随 git 历史，默认 merge commit）。

## 维护

见 [AGENTS.md](AGENTS.md)。
