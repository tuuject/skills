---
name: tuuject-dev-create-feature
description: tuuject 系列项目通用的 git worktree 开发工作流，覆盖三件事：一是开始任何新 feature 开发前的强制前置步骤——先确定集成分支（项目 AGENTS.md 指定的，否则远端有 develop 用 develop，没有就用远端默认分支），从远端拉取最新代码，再在 ../<仓库文件夹名>.worktrees/feature/<feature 分支名> 建好工作树并切过去干活；二是实现需求时做完后的交付——验证、git commit、写 devlog、git push、用 gh 开 PR；三是 review、修改、合并 PR——无论是本仓库分支还是外部贡献者 fork 的 PR，都在 ../<仓库文件夹名>.worktrees/review/pr-<N> 检出审阅，fork PR 先审执行面再装依赖跑测试，合并锁定 review 过的 head。验证命令、提交与 PR 约定以所在项目的 AGENTS.md 为准。用于 push 前的所有代码改动、修 bug、写测试、改文档；只要用户要求「开个 feature」「新开一个改动」「实现 xxx」「先拉最新 develop」「建 worktree」「review / 看看 PR #N」「合并这个 PR」「处理外部贡献」，或要动仓库里任何非只读文件，就先用这个 skill，即使用户没提 worktree 也要用。只读调查、跑测试、纯问答，以及已经位于 feature worktree 内的继续开发不需要。
---

# Worktree 工作流：Feature 开工、交付与 PR Review

本文件的步骤是**新 feature 开工**；如果任务是实现一个需求，做完后接着走[实现需求：做完即交付](#实现需求做完即交付)，一直到开出 PR。review、修改或合并 PR（本仓库分支与外部 fork 都算）读 [references/review-pr.md](references/review-pr.md)，它沿用这里的路径约定、集成分支与查重规则。

## 项目约定从哪来

这是 tuuject 各项目共用的流程骨架。项目自己的事实以**仓库根的 AGENTS.md**（及它链接的文档）为准，与本文件冲突时听项目的：

- 集成分支（feature 从哪开、PR 合回哪）。
- 装依赖、lint、类型检查、测试、e2e、文档构建等验证命令。
- 提交信息格式、不入库的文件、PR 说明必须列出的条目。
- 是否写 devlog、写在哪。
- 允许不开 worktree、直接在集成分支提交的例外。

AGENTS.md 没写的按本文件的默认做法；默认做法也套不上时停下来问，不要凭通用习惯猜。

## 为什么

主目录 `~/Projects/<repo>` 是长期存在的基线 checkout；多个 feature 在同一个 checkout 上轮流切分支会互相踩脏、切分支丢状态。所以**每个 feature 一个独立 worktree**，每个被 review 的 PR 也一个，主目录不动：

```text
~/Projects/<repo>                                  # 主 checkout
~/Projects/<repo>.worktrees/feature/<feature 名>
~/Projects/<repo>.worktrees/review/pr-51           # PR review，detached 在 PR head
```

新 feature 必须基于远端集成分支的最新提交 `origin/$BASE`，而不是本地可能落后的同名分支，避免一开始就落后、合并时才炸。

## 不适用

- 只读调查、跑测试、看代码、纯问答：不动文件，不开 worktree。（review PR 虽然不改文件，但要检出 PR 的代码，走 review 流程。）
- 当前已经在某个 feature worktree 里开发（`git worktree list` 的输出中有当前分支）：那是本 skill 的产物，直接继续；做完仍走「实现需求：做完即交付」。
- 一次性的 hotfix、rebase、清理分支、发布准备：不属于「新 feature 开工」。

## 步骤

### 1. 确定 feature 名、仓库根与集成分支

feature 名从用户的任务描述归纳（kebab-case，例如 `edit-active-time-in-admin-panel`）；用户已给名字就用用户的，`feature/` 前缀统一由本 skill 加，不要重复叠加。名字还没定就问一句。

并处理两种边界：

- 当前目录不是仓库根（`git rev-parse --show-toplevel` 报错或更深）时，先定位仓库根再动手。
- 仓库根自身是个 worktree（`.git` 是文件，`git rev-parse --git-common-dir` 指向主目录的 `.git/worktrees/...`）时，说明主目录已被别的 feature 占用；把工作基准切回主目录（`cd <主目录>` 或后续命令 `git -C <主目录>`），worktree 都挂在主目录旁边。

仓库文件夹名 `$NAME` 取仓库根的 basename；worktree 路径 = 仓库根父目录 + `$NAME.worktrees/feature/<feature 名>`（即 `../<仓库文件夹名>.worktrees/feature/<feature 名>`）。仓库根是主 checkout 时这就是字面路径；仓库根是 worktree 时兄弟目录名本来就等于 `$NAME.worktrees`，公式不变。

集成分支 `$BASE`：AGENTS.md 指定了就用它；否则远端有 `develop` 就是 `develop`，没有就是远端默认分支：

```bash
if git -C "$REPO" ls-remote --exit-code --heads origin develop >/dev/null; then
  BASE=develop
else
  BASE=$(git -C "$REPO" ls-remote --symref origin HEAD | sed -n 's|^ref: refs/heads/\(.*\)[[:space:]]HEAD$|\1|p')
fi
```

`$BASE` 为空（网络、鉴权失败）时整个流程中止并报告，不要拿本地分支名凑。`$BASE` 之外的长期分支（例如只接收发布合并的 `main`）不在上面开 feature、不当 PR 的 base。

### 2. 先查重，再创建

功能开发中 skill 被反复触发是正常的，所以先判断，再决定建还是复用：

- `git worktree list` 里已经有这个分支 → `cd` 到它所在路径继续开发，别动该 worktree 的现有状态，也别重复 fetch。
- 分支不存在、且 worktree 路径为空 → 走第 3 步创建。
- 目标路径被非 worktree 的目录占用（用户自己建的、旧残留）→ 停下来报告路径与冲突，绝不删目录、绝不 `--force`。
- 分支已被别处签出（`git worktree list --porcelain` 能在某条 `branch refs/heads/...` 上找到它）→ 报告它属于哪个 worktree、让用户决定是继续用那个还是换个 feature 名，不要抢。

### 3. 新建 worktree

```bash
git -C "$REPO" fetch origin "$BASE"
git -C "$REPO" worktree add -b "feature/<name>" "$WORKTREE_PATH" "origin/$BASE"
```

这两条只读远端、只建新目录，不会碰主目录当前 checkout、不用 stash、也不用先切到 `$BASE`。

`git fetch` 失败（网络、鉴权）时**整个步骤中止**：报告失败，不要退回用可能落后的本地分支开工，也绝不静默继续。

如果本地 `$BASE` 领先 `origin/$BASE`（`git rev-list --count "origin/$BASE..$BASE"` > 0，或有未推送提交），明确告诉用户「这些提交不在此 worktree 里」——基于远端开分会丢东西，不要让它悄悄发生。

### 4. 切过去并准备

- `cd` 到 worktree，`git rev-parse --abbrev-ref HEAD` 确认分支正确。
- 装依赖。每个 worktree 有独立的依赖目录，不装就跑测试会得到假失败。AGENTS.md 写了安装命令就用它；没写就按锁文件选，锁文件不许被改写：

  | 锁文件 | 命令 |
  | --- | --- |
  | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
  | `bun.lock` / `bun.lockb` | `bun install --frozen-lockfile` |
  | `package-lock.json` | `npm ci` |
  | `yarn.lock` | `yarn install --immutable` |

- 之后所有编辑、测试、提交都在 worktree 内进行，**不要碰主目录 checkout 的工作区**。

### 5. 汇报后继续

告诉用户：worktree 路径、分支名、集成分支 `$BASE`、base commit（`git rev-parse --short HEAD`），以及本地 `$BASE` 是否领先这类需要人知道的风险，然后直接开始用户的 feature 工作，不要停下来等确认。

## 实现需求：做完即交付

用户要的是「实现某个需求」（新功能、修 bug、按 PRD 落地等，产出要合回 `$BASE`）时，worktree 只是开头：改完后按下面的顺序一路走到开 PR。「实现 xxx」这句话本身就授权了本节的 commit、push 与开 PR，不必逐步再问；**合并 PR 不在授权内**，仍等用户明确说。

不走本节：试验或探索性改动、用户说了先别提交、[例外情况](#例外情况)里可以直接在 `$BASE` 上做的改动，以及 PR review（按 [references/review-pr.md](references/review-pr.md)）。

### 1. 验证

按 AGENTS.md 的验证要求跑（常见于「Testing Guidelines」「Build, Test, and Development Commands」「Commit & Pull Request Guidelines」这几节，以及它链接的 verification 类文档），包括它按改动范围追加的项（例如动了前端就跑 e2e、动了文档站就跑文档构建）。AGENTS.md 没写时至少跑：`git diff --check`、`package.json` scripts 里的 lint 与类型检查、受影响的测试；跨模块改动跑完整测试。

行为变了而项目有 `agent-doc/` 之类的 agent 文档时，同步对应主题文档。失败先修；修不好就停在这里报告，不要带着失败往下提交。

### 2. 提交代码

- 只 `git add` 本次改动的路径，不用 `git add -A`；`.gitignore` 与 AGENTS.md 列出的本地数据、密钥、数据库、媒体缓存、备份不入库。
- 提交信息按 AGENTS.md；没写就跟随 `git log --oneline -20` 的风格。结尾带当前会话要求的 commit attribution。
- 依赖有变化时锁文件一起提交。

一个需求通常一个代码提交；明显独立的几块可以拆开，devlog 仍只写一篇。

### 3. 对齐最新集成分支

`git fetch origin "$BASE"`；`origin/$BASE` 有新提交就 `git rebase "origin/$BASE"`，然后重跑受影响的验证。冲突拿不准怎么解就停下报告。这一步必须在写 devlog 之前：devlog 记录的是最终提交 hash，rebase 会改掉它。

### 4. 写 devlog

项目有 devlog 约定（AGENTS.md 提到，或仓库里已有 `agent-doc/devlogs/`）时，用 `write-devlog` skill 写 `<devlog 目录>/<YYYYMMDD>-<feature 名>.md`（日期取当天），记录背景、主要变更、验证结果（含未执行的检查）与第 2 步的提交 hash，单独提交为 `docs: add <改动简述> devlog`。没装 `write-devlog` 时照该目录已有 devlog 的结构手写。不回改旧 devlog。

项目没有 devlog 约定就跳过本步，汇报时说一句。

### 5. Push

```bash
git push -u origin "feature/<name>"
```

只有分支此前推送过、且第 3 步 rebase 过时，才用 `--force-with-lease` 推这个 feature 分支；其他被拒的情况（远端有别人的提交等）报告给用户，不要 `--force`。任何情况都不直接推 `$BASE` 或发布分支。

### 6. 开 PR

先查重：`gh pr list --head "feature/<name>" --state open`。已有 PR 就只靠 push 更新，必要时 `gh pr edit` 补充说明，不再新开。否则把正文写进 scratchpad 文件再创建：

```bash
gh pr create --base "$BASE" --head "feature/<name>" --title "<代码提交标题>" --body-file <scratchpad>/pr-body.md
```

正文的语言和结构沿用仓库最近合并的 PR（`gh pr list --state merged --limit 3` 后 `gh pr view <N>`），并满足 AGENTS.md 对 PR 说明的要求。仓库还没有先例时，默认用中文和这个结构：

```markdown
## 行为变化

## 影响面

（配置、数据库、公开 API、发布产物等，按项目实际列；没有就写「无」）

## 验证

（实际跑过的命令与结果；最后一行写「未执行：……」，如真实环境验收）

## 提交

- `<hash>` <代码提交标题>
- `<hash>` docs: add <改动简述> devlog
```

结尾带当前会话要求的 PR attribution。正文不要写裸 `@名字`。

### 7. 汇报

告诉用户 PR 链接、提交列表、验证结论与未执行项。不合并、不删 worktree、不删分支。

## PR Review 速览

完整步骤见 [references/review-pr.md](references/review-pr.md)。最容易出错的几条先记住：

- 检出：`git fetch origin "$BASE" "+pull/<N>/head:refs/remotes/origin/pr/<N>"`，核对它等于 `headRefOid` 后在 `../$NAME.worktrees/review/pr-<N>` 建 detached worktree，钉在 `headRefOid`；不在主 checkout 上 `gh pr checkout`，不放 `/tmp`。
- 外部 fork 的代码不可信：先读依赖清单与锁文件、包管理器配置、`scripts/`、git hooks、`.github/`、测试配置、`.claude/` 等 agent 配置、`skills/` 与 AGENTS.md 的 diff，确认无害后才装依赖和跑测试。
- PR 上的 CI 往往不跑全量测试：先读 `.github/workflows/` 弄清实际跑了什么，没覆盖的合并前在本地跑。
- 发评论、approve、推送到 fork、合并都是对外动作，用户明确授权后才做；合并带 `--match-head-commit <headRefOid>`。

## 反例（常犯）

- 跳过 fetch 直接 `git worktree add -b ... "origin/$BASE"`：用的是上次 fetch 的陈旧 ref，开头就落后别人几天的提交。
- 基于 `HEAD` / 本地 `$BASE` 建 worktree：用户要的是「最新」，本地 ref 不一定最新。
- 不探测就假定集成分支是 `develop`（或 `main`）：有的项目没有 `develop`，有的项目 `main` 只收发布合并。
- 在主 checkout 上切分支、stash、直接改代码：违背「一个 feature 一个 worktree」的约定，也会打断主目录上其它进行中的事。
- 绕过 worktree 在 `/tmp` 里 clone 一份或建 worktree：路径约定不一致，后续审计/清理都会漏掉它。
- 已有 worktree 或分支占用时直接报错放弃：应该复用并继续，或者停下来问，而不是换个名字硬建。
- 实现需求后只汇报改动就收工，留下未提交的 worktree：用户要的是一个能审的 PR。
- 先写 devlog 再 rebase：devlog 里的提交 hash 随即失效。

## 例外情况

纯粹增加和修改 `doc/design/` PRD 的需求，可以直接在 `$BASE` 上做，不开 worktree、不开 PR；但不包括只接收发布合并的分支。AGENTS.md 另有规定的从其规定。
