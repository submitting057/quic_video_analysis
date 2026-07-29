# GitHub 仓库写入与修改操作规范

本文规定自动化 Agent 或协作者如何安全地读取、修改、提交和验证本 GitHub 仓库。流程不依赖特定主机、操作系统、用户名或本地绝对路径，适用于 Windows、Linux、macOS、容器和临时执行环境。

目标仓库：

```text
https://github.com/submitting057/quic_video_analysis.git
```

## 1. 核心原则

1. **以当前仓库和远程状态为准**：执行前必须重新检查，不依赖上一台主机或上一次会话的路径和状态。
2. **不覆盖未知修改**：工作区已有变更默认属于用户或其他协作者，不得用 `reset --hard`、强制检出等方式清除。
3. **不强制推送**：禁止使用 `git push --force` 或 `--force-with-lease`，除非用户明确授权且已解释影响。
4. **只提交任务相关文件**：使用精确路径暂存，不使用无法审计的全量提交习惯。
5. **先验证再提交，提交后再验证远程**：本地成功不等于 GitHub 已更新。
6. **不把秘密写入仓库**：Token、Cookie、私钥、账号、内部地址和敏感抓包标识不得进入提交。
7. **命令和文档保持可移植**：提交到仓库的说明不得依赖某台主机的盘符、主目录或临时路径。

## 2. 本仓库的文档组织约定

根目录 `README.md` 只承担仓库概述和日期导航，不堆放单次会话的全部正文。

每次新增一组讨论文档时：

```text
仓库根目录/
├── README.md
├── AGENT.md
└── YYYYMMDD/
    ├── README.md
    ├── 01-概括性主题标题.md
    ├── 02-概括性主题标题.md
    └── ...
```

具体规则：

- 日期目录使用八位数字 `YYYYMMDD`，例如 `20260729`。
- 日期目录的 `README.md` 概述本次内容、核心结论和阅读顺序。
- 主题文件使用 `NN-概括性标题.md`，编号从 `01` 开始。
- 文件名和一级标题都要概括正文，不能使用 `notes.md`、`misc.md` 等模糊名称。
- 一个文件集中回答一组相邻问题，避免文件过大，也避免把同一概念拆得过碎。
- 根目录 `README.md` 必须增加对应日期目录的相对链接。
- 文档内部优先使用相对链接，使仓库在不同主机和 Fork 中仍可浏览。
- 已经提交的历史日期目录视为只读归档。除非用户明确点名要求修正该历史日期，否则不得删除、移动、重命名或修改目录及其中任何文件。
- 新一天的内容必须新建对应的 `YYYYMMDD/` 目录，不得通过重命名、覆盖或复用旧日期目录来写入。
- 根目录 `README.md` 的日期归档采用追加式维护：新增日期入口时必须保留全部旧日期入口，不得删除、改写或替换旧记录。
- 如果发现内容被错误写入旧日期，先保留旧目录并向用户确认迁移方案；未经确认不得自行重组历史目录。
- 中文文档使用 UTF-8；换行符可以由 Git 统一处理，但不得产生乱码。

日期来源的优先级：

1. 用户明确指定的日期。
2. 任务或会话明确记录的日期和时区。
3. 当前执行环境日期。
4. 如果以上信息冲突且会改变归档位置，询问用户。

## 3. 执行决策流程

```text
确认目标仓库
→ 判断当前目录是否为 Git 仓库
→ 验证 origin 是否指向目标仓库
→ 检查工作区和当前分支
→ 获取远程最新状态
→ 判断是空仓库、普通更新还是分支冲突
→ 编辑并验证文件
→ 精确暂存和提交
→ 推送目标分支
→ 比较本地与远程提交哈希
→ 报告文件、分支和提交号
```

## 4. 定位或克隆仓库

### 4.1 当前目录可能已经是仓库

首先运行：

```bash
git rev-parse --show-toplevel
git status --short --branch
git remote -v
```

如果第一条命令失败，当前目录不是 Git 仓库。不要在一个已有但无关的项目目录中直接执行 `git init`；应在独立目录克隆目标仓库。

### 4.2 克隆到显式目录

```bash
git clone https://github.com/submitting057/quic_video_analysis.git quic_video_analysis
cd quic_video_analysis
```

不要假设仓库一定位于：

- 某个 Windows 盘符；
- 某个用户的 Home 目录；
- `/workspace`、`/repo` 或其他固定容器路径。

本地目录名可以变化，后续操作应以 `git rev-parse --show-toplevel` 返回的仓库根目录为准。

### 4.3 验证远程身份

```bash
git remote -v
git ls-remote origin
```

`origin` 必须指向目标仓库。若当前仓库指向其他项目，停止操作，不要直接改写远程地址，除非任务明确要求。

## 5. 身份验证

可使用执行环境已经配置的任一合法方式：

- Git Credential Manager 或系统凭据存储；
- GitHub CLI 登录状态；
- SSH Key；
- CI/CD 提供的短期凭据。

安全要求：

- 不在命令、文档、提交信息或日志中明文写 Personal Access Token。
- 不把 Token 拼接到 GitHub URL 后提交。
- 不提交 `.env`、私钥或浏览器 Cookie。
- 如果只能读取不能推送，应保留本地提交并报告准确的认证错误。

可选检查：

```bash
gh auth status
```

如果环境没有 GitHub CLI，不影响使用普通 Git 凭据。

## 6. 修改前检查和同步

### 6.1 检查工作区

```bash
git status --short --branch
git log -1 --oneline
git remote -v
```

如果存在未提交修改：

- 判断是否与本次任务相关。
- 不相关修改保持原样，不纳入本次提交。
- 如果修改目标文件已被用户改动，先阅读差异并尽量合并。
- 无法安全合并时，向用户说明冲突文件和需要的决策。

禁止使用以下命令清理未知修改：

```text
git reset --hard
git checkout -- <file>
git clean -fd
```

### 6.2 获取远程状态

```bash
git fetch --prune origin
git status --short --branch
```

如果当前分支已有正常上游且工作区干净，可以执行：

```bash
git pull --ff-only
```

使用 `--ff-only` 防止 Git 在不知情的情况下生成自动合并提交。

如果远程和本地已经分叉：

- 不强制推送。
- 先检查本地提交是否全部由当前任务产生。
- 只有在不会覆盖用户工作的前提下才进行 rebase 或显式合并。
- 存在语义冲突时请用户决定。

## 7. 分支策略

分支策略服从用户任务和仓库保护规则：

- 用户明确要求直接更新 `main`，且远程允许时，可以提交并推送 `main`。
- 未明确要求直接更新默认分支时，较大或高风险修改优先使用功能分支。
- 受保护分支拒绝推送时，创建功能分支并报告其名称，不尝试绕过保护。

功能分支示例：

```bash
git switch -c docs/20260729-organize-notes
git push -u origin docs/20260729-organize-notes
```

空仓库可能没有远程默认分支。此时先检查：

```bash
git branch --show-current
git ls-remote --heads origin
```

如果需要创建初始 `main`：

```bash
git switch -c main
```

如果当前已经处于尚未提交的 `main`，不需要重复创建。

## 8. 编辑文档

### 8.1 内容要求

- 以用户问题和实际回答为依据，不补写无法验证的结论。
- 技术缩写首次出现时说明含义。
- 保留重要限制条件，例如“无密钥旁路”“一个 UDP Datagram 可能包含多个 QUIC Packet”。
- 规则方法、深度方法、数据划分和评估指标要分别组织。
- 命令示例不得包含本机绝对路径、秘密或仅对单一环境有效的账号信息。
- 外部事实尽量链接到 RFC、官方文档或原始论文。

### 8.2 编辑安全

- 编辑前阅读目标文件的现有内容和相邻文档。
- 保持用户已有写作风格和目录约定。
- 不删除不相关文档。
- 大型重组应先建立索引和新文件，再确认内容覆盖，最后缩减旧入口文件。
- 移动或拆分文件后，更新所有相对链接。

## 9. 提交前验证

最低检查项：

```bash
git status --short --branch
git diff --check
git diff --stat
```

由于 `git diff` 默认不显示未跟踪文件，暂存后还必须检查：

```bash
git add -- <明确的文件或目录>
git diff --cached --check
git diff --cached --stat
git diff --cached
```

文档仓库还应验证：

- 日期目录存在且名称正确。
- 根 `README.md` 能导航到日期目录。
- 日期目录 `README.md` 能导航到所有主题文件。
- 所有相对 Markdown 链接的目标文件存在。
- 每个主题文件有唯一且概括内容的一级标题。
- 代码块闭合，表格结构完整，没有乱码。
- 原问题中的关键主题没有在拆分中遗漏。
- Git 暂存区只包含本次任务的文件。

如果仓库提供 Markdown Linter、链接检查器或测试脚本，应优先运行仓库自带命令，并记录结果。

## 10. Git 提交身份

先检查现有身份：

```bash
git config --get user.name
git config --get user.email
```

不得为了单次任务修改全局 Git 身份。若当前仓库没有身份配置，可在任务允许的情况下只设置仓库本地身份：

```bash
git config user.name "Codex"
git config user.email "codex@openai.com"
```

如果组织要求使用特定机器人账号或签名提交，应遵守组织配置，而不是覆盖它。

## 11. 暂存和提交

只暂存明确目标：

```bash
git add -- README.md AGENT.md 20260729
```

不要在混有用户变更的工作区中无检查地执行 `git add -A`。

提交信息使用简短、可读的 Conventional Commit 风格：

```text
docs: add repository update workflow
docs: organize discussion notes by date and topic
fix: repair broken documentation links
```

提交：

```bash
git commit -m "docs: add repository update workflow"
```

提交后检查：

```bash
git log -1 --oneline
git status --short --branch
```

## 12. 推送

第一次推送当前分支：

```bash
git push -u origin <branch-name>
```

已有上游分支：

```bash
git push origin <branch-name>
```

`<branch-name>` 必须替换为实际分支名，例如 `main` 或 `docs/update-workflow`，不能原样复制占位符。

遇到非 Fast-Forward 拒绝时：

1. `git fetch origin`。
2. 检查远程新增提交。
3. 安全地 rebase 或合并当前任务提交。
4. 重新运行验证。
5. 普通推送，不使用强制推送。

## 13. 远程完成验证

推送命令返回成功后，仍需比较本地和远程哈希：

```bash
git rev-parse HEAD
git ls-remote origin refs/heads/<branch-name>
git status --short --branch
```

完成条件：

- `git push` 成功。
- 本地 `HEAD` 与远程目标分支哈希一致。
- 工作区没有遗漏的任务相关修改。
- 远程分支和提交号能够明确报告。

对于私有仓库，匿名网页访问可能返回 404。这不等于推送失败；经过认证的 `git push` 和 `git ls-remote` 是更可靠的远程证据。

## 14. 常见异常处理

### 仓库为空

- 克隆出现 “empty repository” 警告是正常情况。
- 创建文档、初始提交和 `main` 分支后使用 `git push -u origin main`。

### 认证失败

- 不反复尝试包含明文 Token 的命令。
- 检查 Git Credential Manager、SSH 或 `gh auth status`。
- 保留已经完成的本地修改和提交，报告当前分支与提交号。

### 分支受保护

- 创建功能分支并推送。
- 报告需要 Pull Request 或仓库管理员审批。
- 不尝试绕过保护策略。

### 远程发生并发更新

- 获取远程提交并检查差异。
- 不覆盖其他协作者的新提交。
- 发生内容冲突时解决后重新检查链接和文档结构。

### GitHub 网页不可访问

- 私有仓库、网络代理或匿名浏览器可能导致 404/403。
- 使用经过认证的 Git 命令验证远程分支。
- 不仅凭网页抓取失败判断仓库不存在。

## 15. 最终交付报告

向用户报告：

- 已完成的内容和目录结构。
- 目标仓库、分支和完整或短提交哈希。
- 关键验证结果，例如相对链接全部有效。
- 如果使用功能分支，提供分支名称或 Pull Request 链接。
- 仍存在的限制、认证问题或需要用户处理的仓库保护步骤。

不要只说“已上传”；应给出可以审计的提交和远程验证证据。

## 16. 完成检查清单

- [ ] 已确认正确的 Git 仓库根目录。
- [ ] 已确认 `origin` 指向目标 GitHub 仓库。
- [ ] 已检查并保护现有工作区修改。
- [ ] 已获取远程最新状态。
- [ ] 已按照 `YYYYMMDD/` 约定组织新文档。
- [ ] 已确认所有既有历史日期目录及其内容未被删除、移动、重命名或修改。
- [ ] 已确认根 `README.md` 保留全部旧日期入口，只追加本次新日期入口。
- [ ] 已更新根目录和日期目录导航。
- [ ] 已验证标题、相对链接、代码块和内容覆盖。
- [ ] 已检查暂存区只包含任务相关文件。
- [ ] 已创建清晰的提交。
- [ ] 已成功推送目标分支。
- [ ] 已确认本地 HEAD 与远程分支哈希一致。
- [ ] 已向用户报告仓库、分支、提交号和验证结果。
