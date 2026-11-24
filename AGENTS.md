# 项目规范

本仓库是 [CometixCode](https://github.com/Haleclipse/CometixCode) 的 fork。
`origin` 是自己的 fork，`upstream` 是原项目。分支管理要同时满足三件事：

1. 长期维护自定义改动；
2. 向上游提交可独立 review 的修复；
3. 持续合并上游，且冲突可控。

所有 Git 写操作由维护者执行。自动化代理默认只读；只有获得明确授权，才能执行下文的写命令。

## 1. 远端

| 远端 | 地址 | 用途 |
| --- | --- | --- |
| `origin` | `https://github.com/Arctic07/nextCode.git` | 自己的 fork，接收所有分支 |
| `upstream` | `https://github.com/Haleclipse/CometixCode.git` | 只读的上游来源 |

一次性配置（已配置则跳过）：

```bash
git remote add upstream https://github.com/Haleclipse/CometixCode.git
git fetch upstream --prune

# 冲突复现与自动暂存，减少重复手工解冲突
git config rerere.enabled true
git config rerere.autoupdate true

# master 只允许快进；push 默认回自己的 fork
git config branch.master.ff only
git config remote.pushDefault origin
git config push.default current
git config branch.master.pushRemote origin
git branch --set-upstream-to=upstream/master master
```

约定：**永远不向 `upstream` 推送**。向上游提交只通过 GitHub PR。

## 2. 分支模型

| 分支 | 起点 | 生命周期 | 用途 | 推送 |
| --- | --- | --- | --- | --- |
| `master` | `upstream/master` | 永久 | 上游镜像，**零本地提交** | `origin` |
| `custom` | `master` | 永久 | fork 的唯一集成分支，包含全部自定义 | `origin` |
| `feat/<topic>` | `custom` | 短期 | 单个自定义功能 | 可选 `origin` |
| `upstream/fix-<topic>` | `upstream/master` | 短期 | 只含可上游合并的修复，用于发 PR | `origin` |

硬性规则：

- `master` 只接受 `upstream/master` 的快进，不直接 commit，不 merge `custom`。
- `custom` 是唯一长期分支：自定义功能都合到这里，不拆多个长期分支。
- `custom` 只用 merge 前进，**不 rebase、不 force push**。
- `upstream/fix-*` 必须从 `upstream/master` 切出，不能包含任何 `Fork-Only` 提交。
- 合并方向永远是 `upstream → master → custom`，绝不把 `custom` 合进 `master`。
- 分支名小写，用 `/` 与 `-`，不加日期前缀；`feat/` 只用于 fork 功能，`upstream/` 只用于上游 PR。

建议在 fork 的 Settings 里把默认分支设为 `custom`，让 clone 和首页默认展示自定义版本；`master` 保持镜像。

## 3. 日常流程

### 3.1 首次建立 `custom`

确认 `master` 与上游一致后：

```bash
git fetch upstream --prune
git switch master
git merge --ff-only upstream/master
git push origin master

git switch -c custom
git push -u origin custom
```

### 3.2 同步上游

每个工作会话开始时先 `fetch`；有上游更新就合并，别让 `custom` 漂移超过两周。

```bash
git fetch upstream --prune

git switch master
git merge --ff-only upstream/master
git push origin master

git switch custom
git merge --no-ff master
just check
just test
git push origin custom
```

每次同步产生一个 merge commit，便于整体回退。同步后核对：

```bash
git rev-list --count upstream/master..master   # 必须是 0
git diff --quiet upstream/master master        # 必须无输出
```

`master` 无法快进时立刻停下，不要 `reset`、不要强推：

```bash
git log --oneline upstream/master..master
```

先查明本地提交来源，按第 7 节处理。

### 3.3 自定义功能

```bash
git fetch origin
git switch custom
git merge --ff-only origin/custom

git switch -c feat/<topic>
# 开发、测试
git switch custom
git merge --no-ff feat/<topic>
just check
just test
git push origin custom
git branch -d feat/<topic>
```

自定义提交统一用 `feat(fork): ...` / `fix(fork): ...`，便于 `git log master..custom` 一眼识别。需要备份或协作时可以推送 `feat/<topic>`，但它不是长期分支。

### 3.4 向上游提交修复

1. 先同步上游，确认问题仍存在、上游没有同类修复。
2. 从上游切分支，不要从 `custom` 切：

   ```bash
   git fetch upstream --prune
   git switch -c upstream/fix-<topic> upstream/master
   ```

3. 只做这一件事，补测试，跑 `just check` 和 `just test`。
4. 推送前确认分支干净：

   ```bash
   git log --oneline upstream/master..HEAD
   git diff --stat upstream/master...HEAD
   test -z "$(git log --format=%B upstream/master..HEAD | grep 'Fork-Only: true')"
   ```

   第一条只应出现本次修复的提交；第二条只应出现相关文件；第三条无输出。
5. 推送并开 PR：

   ```bash
   git push -u origin upstream/fix-<topic>
   ```

   PR 方向：`Haleclipse/CometixCode:master ← Arctic07/nextCode:upstream/fix-<topic>`。

6. 上游要求修改时，只允许对这个未合并的 PR 分支 `rebase` / `amend`，并使用 `git push --force-with-lease origin upstream/fix-<topic>`。**不得对 `master`、`custom` 这样做。**
7. 上游合并后先同步，再删分支：

   ```bash
   git fetch upstream --prune
   # 按 3.2 同步 master 与 custom
   git branch -d upstream/fix-<topic>
   git push origin --delete upstream/fix-<topic>
   ```

8. fork 急需而 PR 尚未合并时，可以 `git cherry-pick -x <sha>` 到 `custom`，保留来源记录。上游合并后正常同步；若出现重复改动，以行为一致为准，不要手工回退上游版本。

已合并到上游的提交加 trailer，便于追溯：

```text
Upstream-PR: https://github.com/Haleclipse/CometixCode/pull/123
```

仅存在于 fork 的提交加：

```text
Fork-Only: true
```

## 4. 降低冲突的代码约定

- 自定义代码放独立模块 `src/fork/`，在 `src/lib.rs` 只加一行 `pub mod fork;`。优先新增文件，而不是改写上游文件。
- 必须嵌入上游文件时，用成对标记包住每一处改动，并写明主题：

  ```rust
  // [fork:begin] custom-theme
  ...
  // [fork:end] custom-theme
  ```

  冲突时规则明确：标记外以上游为准，标记内重新落位自定义逻辑；上游重构过的代码不整段还原。
- 不改上游文件的格式、不批量重命名、不跑全文件 `cargo fmt`。只格式化实际改动的片段，避免制造无意义冲突。
- 配置、环境变量、设置项使用 fork 命名空间（如 `COMETIX_FORK_*`、`fork.*`），不要占用上游键名。
- 不为迁就自定义而修改上游测试或 fixtures；自定义测试放 `src/fork/` 内联，或新增独立测试文件。
- README 以上游版本为准。fork 说明写进独立文件（如 `FORK.md` 或 `docs/fork/`）；确需改 README 时，只在文件末尾加一个标记块，不重写上游段落。
- `Cargo.toml` 的自定义依赖集中放在标记块内，不打乱上游依赖顺序；自定义功能不随意升降上游依赖版本。
- 定期用 `git diff --stat upstream/master...custom` 检查自定义面；面越大，冲突成本越高。能收敛到扩展点或新文件就收敛。

## 5. 冲突处理

1. 先确认方向没有错：当前在 `custom`，合入的是 `master` / `upstream/master`。
2. `git rerere` 已开启时，优先复用已记录的解法；解决后照常 `git add`，由 rerere 记录。
3. merge 期间 `ours` 是当前分支（`custom`），`theirs` 是上游：

   ```bash
   git checkout --ours   <path>   # 保留自定义侧
   git checkout --theirs <path>   # 采用上游侧
   ```

   整文件取舍只用于生成物或纯上游文件；源码按标记逐段解决。
4. `Cargo.lock` 不手工合并：先取上游版本，再让 Cargo 补回自定义依赖，最后提交锁文件：

   ```bash
   git checkout --theirs Cargo.lock
   cargo check
   git add Cargo.lock
   ```

   合并完成后必须验证 `cargo build` 与 `just test`；锁文件冲突单独成一个提交，便于排查。
5. 解完冲突先跑 `just check`、`just test`，再 `git commit`。不要带着未验证的 merge 继续叠加改动。
6. 拿不准就 `git merge --abort` 回到合并前，查清后再重来；不要靠强推掩盖问题。
7. 大范围重构类上游提交（如 MCP、TUI 渲染层）不要与自定义改动混在同一次同步里；先合上游、验证、再重放自定义。

## 6. 提交与发布

- 提交信息沿用上游风格：`type(scope): summary`，如 `fix(mcp): ...`、`refactor(cli): ...`。
- fork 专属提交加 `fork` scope：`feat(fork): ...`、`fix(fork): ...`。
- 一个提交只做一件事；上游 PR 分支不允许夹带格式化、重命名或 fork 改动。
- fork 版本标签用 `fork-v<semver>` 打在 `custom` 上，避免与上游标签重名。
- 不提交 `.trellis/`、`target/`、本地配置等生成物；保持工作区干净再合并。

## 7. 事故处理

- **`master` 出现本地提交**：先建救援分支保留现场，再由维护者手动回正：

  ```bash
  git branch rescue/master-$(date +%F) master
  git switch master
  git reset --hard upstream/master
  ```

  仅维护者确认无独有提交后执行；代理不得自行 reset。

- **`custom` 合错方向或合入坏提交**：不要强推。用 `git revert -m 1 <merge-sha>` 回退合并，验证后重新合并正确来源。

- **上游改写历史导致无法快进**：停下，比较 `git log --oneline upstream/master..master` 与 `git range-diff`，确认没有独有提交后再决定；不要直接 reset `custom`。

- **冲突反复出现**：检查是否绕过了第 4 节（改上游文件、全文件格式化、多个长期分支）。优先把改动搬回 `src/fork/` 或标记块，而不是反复手工解冲突。
