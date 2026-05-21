# Git 与 Upstream 工作流指南

---

## 基本概念

- **origin** —— 你自己的远程仓库（fork）
- **upstream** —— 上游官方仓库（你要贡献代码的目标）
- **工作区 / 暂存区 / 本地仓库 / 远程仓库** —— git 的四个区域

---

## 一、基础命令

### git clone —— 克隆仓库

```bash
git clone <url>
git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git
```

### git log —— 查看提交历史

```bash
# 基本用法
git log
git log --oneline                     # 一行简洁模式
git log --graph --oneline             # 显示分支图

# 搜索 commit message
git log --grep="Bluetooth"            # 搜索关键词

# 只看某个文件的提交历史
git log -- path/to/file.c
git log --oneline -- path/to/file.c
git log -p -- path/to/file.c          # 显示每次改了什么

# 按作者/时间过滤
git log --author="jinwli"
git log --since="2024-01-01"
```

### git blame —— 逐行追溯代码是谁写的

```bash
git blame path/to/file.c
git blame -L 100,200 path/to/file.c   # 只看 100-200 行
```

### git restore —— 撤销工作区修改

```bash
# 丢弃工作区中某个文件的修改（回到上次 commit 的状态）
git restore file.c

# 丢弃工作区所有修改
git restore .
```

### git reset —— 重置 HEAD 位置

```bash
# 取消暂存（把 git add 的内容撤回工作区）
git reset HEAD file.c
git restore --staged file.c           # 等价的新写法

# 回退 commit（保留工作区修改）
git reset HEAD~1                      # 撤销最近 1 个 commit，修改回到工作区
git reset --soft HEAD~1               # 撤销 commit，修改保留在暂存区
git reset --hard HEAD~1               # 彻底丢弃最近 1 个 commit 和所有修改（⚠️ 不可恢复）
```

---

## 二、remote 与 fetch

### git remote —— 管理远程仓库

```bash
# 查看当前远程仓库
git remote -v

# 添加上游仓库
git remote add upstream <upstream-url>
# 例如：
git remote add upstream git://git.kernel.org/pub/scm/linux/kernel/git/bluetooth/bluetooth-next.git

# 验证
git remote -v
# origin    git@github.com:jinwli/linux.git (fetch)
# origin    git@github.com:jinwli/linux.git (push)
# upstream  git://git.kernel.org/... (fetch)
# upstream  git://git.kernel.org/... (push)
```

### git fetch —— 拉取远程更新（不影响本地代码）

```bash
git fetch upstream                    # 从 upstream 拉取最新信息
git fetch origin                      # 从 origin 拉取
git fetch --all                       # 从所有 remote 拉取
```

`fetch` vs `pull`：
- `fetch` 只下载远程数据，不动你的工作区和分支（**安全，推荐先用 fetch**）
- `pull` = `fetch` + `merge`，会直接修改你的代码

---

## 三、分支操作

### git checkout / git switch

```bash
# 切换分支
git checkout <branch-name>

# 创建并切换到新分支
git checkout -b <new-branch>

# 基于 upstream 分支创建本地分支
git checkout -b my-fix upstream/master
```

### 游离态（Detached HEAD）

HEAD 指向某个具体 commit（如 tag、commit hash）而不是分支时，进入游离态。

```bash
# 进入游离态
git checkout <commit-hash>
git checkout v5.10

# 查看是否游离态
git status              # 显示 "HEAD detached at ..."

# 游离态下怎么处理：
# 只是看看代码 → 直接 git checkout master 切回分支即可
# 想保留游离态的修改 → 创建一个新分支
git checkout -b new-branch
```

---

## 四、commit 操作

### git commit --amend —— 修改最后一次提交

```bash
# 修改 commit message
git commit --amend

# 追加文件到上次 commit，不改 message
git add forgotten-file.c
git commit --amend --no-edit

# ⚠️ 注意：amend 会改写 commit hash，如果已 push 到远程，amend 后需要 force push
```

### Signed-off-by —— 签名（kernel/upstream 必须）

```bash
# 提交时自动加上 Signed-off-by
git commit -s -m "Bluetooth: hci_core: fix xxx"

# 给已有的 commit 补 sign-off
git commit --amend -s --no-edit
```

---

## 五、cherry-pick —— 挑选某个 commit 到当前分支

```bash
# 基本用法
git cherry-pick <commit-hash>

# 带 sign-off
git cherry-pick -s <commit-hash>

# 只放入暂存区，不自动提交（方便再修改）
git cherry-pick -n <commit-hash>

# 发生冲突时：
#   解决冲突 → git add . → git cherry-pick --continue
#   放弃 → git cherry-pick --abort
```

---

## 六、git rebase —— 变基

```bash
# 把当前分支 rebase 到 upstream/master 最新代码上
git fetch upstream
git rebase upstream/master

# 发生冲突时：
#   解决冲突 → git add . → git rebase --continue
#   放弃 → git rebase --abort

# rebase 后需要 force push（因为历史被重写了）
git push -f origin <branch>
```

---

## 七、git push

### push 到 Gerrit（code review 用）

```bash
# 推送到 Gerrit review
git push origin HEAD:refs/for/master

# amend 后更新同一个 change：
git commit --amend
git push origin HEAD:refs/for/master     # Gerrit 识别 Change-Id，自动创建新 patchset
```

### push 到普通 remote 仓库（GitHub 等）

```bash
# 推送到同名分支
git push origin <branch-name>

# 第一次推送并设置 upstream 跟踪
git push -u origin my-branch

# 强制推送（⚠️ 谨慎）
git push -f origin <branch-name>
git push --force-with-lease origin <branch-name>   # 更安全：不会覆盖别人刚推的代码
```

---

## 八、实战例子

### 例子 1：提交到 Gerrit 后本地文件丢了，重新 clone 后 cherry-pick 找回来

**场景**：你已经推了一个 change 到 Gerrit，但本地代码不小心删了/电脑坏了/目录被 rm 了。

```bash
# 1. 重新 clone
git clone <gerrit-repo-url>
cd <repo>

# 2. 在 Gerrit 网页上找到你的 change，复制 commit hash
#    (或者用 git log --all --grep="你的commit message关键词" 搜索)

# 3. 先从 Gerrit 抓取那个 change（Gerrit 上每个 change 都是一个 ref）
git fetch origin refs/changes/XX/YYYYY/Z && git cherry-pick FETCH_HEAD
# 其中 XX = change 编号最后两位，YYYYY = change 编号，Z = patchset 号
# 例如 change 12345 patchset 1：
git fetch origin refs/changes/45/12345/1 && git cherry-pick FETCH_HEAD

# 或者直接在 Gerrit 网页上点 "download" → "cherry pick" 复制命令
```

### 例子 2：提交到 Gerrit 有冲突，用 git rebase 解决

**场景**：你的 change 在 Gerrit 上放了几天，upstream 有人合了其他代码，导致你的 change 和当前 upstream/master 有冲突。Gerrit 显示 "Merge Conflict"。

```bash
# 1. 切换到你的工作分支
git checkout my-feature-branch

# 2. 拉取最新的 upstream
git fetch upstream

# 3. rebase 到最新 upstream/master
git rebase upstream/master
# 输出：CONFLICT (content): Merge conflict in file.c

# 4. 手动编辑冲突文件，删除 <<<<<<< ======= >>>>>>> 标记，保留正确代码

# 5. 标记冲突已解决
git add file.c

# 6. 继续 rebase
git rebase --continue

# 7. 如果还有冲突，重复 4-6

# 8. rebase 完成后，amend 你的 commit（因为 rebase 后 commit hash 已经变了）
#    确保 commit message 还包含原来的 Change-Id，否则 Gerrit 会创建新的 change
git commit --amend --no-edit

# 9. 重新推送到 Gerrit 同一个 change
git push origin HEAD:refs/for/master
# Gerrit 根据 Change-Id 匹配到原来的 change，创建新的 patchset
```

**如果 rebase 过程中乱了，想放弃：**

```bash
git rebase --abort                               # 回到 rebase 前的状态
```

### 例子 3：cherry-pick upstream 的某个 commit 到本地测试

**场景**：你想测试 upstream 刚合入的某个 commit 是否解决了你的问题，但不想把整个 upstream 都 merge 进来。

```bash
# 1. 确保 upstream 是最新的
git fetch upstream

# 2. 查看 upstream 最近的 commit，找到你想要的
git log upstream/master --oneline -20
# a1b2c3d Bluetooth: hci_core: fix use-after-free
# e4f5g6h Bluetooth: btusb: add new device ID
# ...

# 3. 基于当前工作分支，cherry-pick 那个 commit 来测试
git checkout my-work-branch
git cherry-pick -s a1b2c3d

# 4. 编译测试...

# 5. 如果测试完不需要了，撤销 cherry-pick 的 commit
git reset --hard HEAD~1                         # 丢弃刚才 cherry-pick 的 commit

# 或者只是暂时测试不想提交：
git cherry-pick -n a1b2c3d                      # -n 只应用修改不 commit
# ... 测试 ...
git restore .                                   # 丢弃测试修改，恢复原状
```

### 例子 4：同步自己 fork 的仓库到 upstream 最新

**场景**：你的 GitHub fork 落后 upstream 很多 commit 了，PR 没法干净地 merge，需要同步。

```bash
# 1. 确保 local 有 upstream 和 origin 两个 remote
git remote -v
# upstream   git://git.kernel.org/.../linux.git
# origin     git@github.com:jinwli/linux.git

# 2. 拉取 upstream 最新
git fetch upstream

# 3. 切换到本地 master（fork 的主分支）
git checkout master

# 4. 把本地 master 重置到 upstream/master, 此时你本地的修改都没了，repo也干净了
git reset --hard upstream/master

# 5. 推送（同步到你 GitHub 的 fork）
git push -f origin master
# ⚠️ 必须 -f，因为本地历史被 reset 改写了

# 现在你的 fork 就和 upstream 完全一致了
```

### 例子 5：cherry-pick upstream 已经 merge 的 commit，但不配 remote (和从gerrit上拿命令其实是一样的)

**场景**：upstream（GitHub/kernel.org）已经合入了某个 commit，你想拿到本地测试，但不想配 remote、不想 fetch 整个仓库。

```bash
# 直接从 URL fetch 那个 commit，然后 cherry-pick
git fetch <repo-url> <commit-hash>
git cherry-pick FETCH_HEAD

# 例如：
git fetch git://git.kernel.org/pub/scm/linux/kernel/git/bluetooth/bluetooth-next.git a1b2c3d4e5f6
git cherry-pick FETCH_HEAD

# GitHub 也一样：
git fetch https://github.com/torvalds/linux.git a1b2c3d4e5f6
git cherry-pick FETCH_HEAD
```

一行版：

```bash
git fetch <repo-url> <commit-hash> && git cherry-pick FETCH_HEAD
```

**为什么可以这样：** `git fetch <url> <ref>` 不需要添加 remote，直接从指定 URL 抓取 commit 存到 `FETCH_HEAD`，然后 cherry-pick 就可以用。

### 例子 6：从 kernel 邮件列表网页上拿 patch 到本地

**场景**：你在 lore.kernel.org 或 patchwork 上看到别人发的一个 patch，想把它应用到本地测试。

**方法 A：从网页直接下载 raw patch**

```bash
# 1. 在 lore.kernel.org 找到 patch，点 "raw" 链接，复制 URL
#    或者直接在 patch 页面 URL 后面加 /raw
wget 'https://lore.kernel.org/linux-bluetooth/20240101000000.12345-1-jinwli@example.com/raw' -O /tmp/fix.patch

# 2. 用 git am 应用到本地（会生成 commit）
git am /tmp/fix.patch

# 3. 查看结果
git log -1
```

**方法 B：用 b4 工具（推荐）**

```bash
# 安装
pip install b4

# 从 message-id 或 lore URL 直接下载并应用
b4 am 20240101000000.12345-1-jinwli@example.com
# 或者：
b4 am https://lore.kernel.org/linux-bluetooth/20240101000000.12345-1-jinwli@example.com/

# b4 会下载 patch series，生成 .mbx 文件，然后：
git am <downloaded>.mbx
```

**方法 C：从 Patchwork 拿**

```bash
# 在 patchwork.kernel.org 找到 patch，点 "mbox" 下载
wget 'https://patchwork.kernel.org/project/bluetooth/patch/.../mbox/' -O /tmp/patch.mbox
git am /tmp/patch.mbox
```

**方法 D：手动复制（网页格式被弄乱时用）**

```bash
# 1. 从网页复制 patch 全文（From ... 到 diff 部分）
# 2. 保存到文件 /tmp/patch.txt
# 3. 尝试 git am
git am /tmp/patch.txt

# 如果 git am 失败，降级用 git apply（只应用 diff，不生成 commit）：
git apply /tmp/patch.txt
git add .
git commit -s    # 需要自己写 commit message
```

**git am 失败怎么办：**

```bash
# 跳过有问题的 patch，继续应用
git am --skip

# 放弃整个 am
git am --abort
```

---

## 九、Upstream 提交的两种方式

### 方式一：GitHub Fork → PR → Rebase → Force Push

```bash
# === 1. Fork 并 clone ===
# 在 GitHub 上 Fork 目标仓库
git clone git@github.com:<your-username>/<repo>.git
cd <repo>
git remote add upstream <original-repo-url>

# === 2. 创建功能分支 ===
git fetch upstream
git checkout -b my-feature upstream/master

# === 3. 开发提交 ===
# ... 写代码 ...
git add .
git commit -s -m "feature: add xxx support"

# === 4. Push 到自己的 fork，去 GitHub 创建 PR ===
git push -u origin my-feature

# === 5. Review 后需要修改，rebase 并重推 ===
git fetch upstream
git checkout master
git reset --hard upstream/master           # 更新本地 master
git push -f origin master                  # 同步自己的 fork

git checkout my-feature
git rebase master                          # 变基到最新 master

git push -f origin my-feature              # 更新 PR（GitHub PR 自动刷新 diff）
```

### 方式二：Send Email 到 Kernel/BlueZ 邮件列表

```bash
# === 1. 克隆 ===
git clone git://git.kernel.org/pub/scm/linux/kernel/git/bluetooth/bluetooth-next.git
# 或 bluez:
git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git

# === 2. 开发 ===
git checkout -b my-fix
# ... 写代码 ...
git commit -s
```

**commit message 格式（必须遵守）：**

```
Bluetooth: hci_core: fix use-after-free in hci_dev_do_close

When powering off the device, hci_dev_do_close() accesses hdev->workqueue
after it has already been destroyed, causing a use-after-free.

Move the destroy_workqueue() call after the last use of hdev.

Fixes: a1b2c3d4e5f6 ("Bluetooth: hci_core: refactor close sequence")
Signed-off-by: Your Name <your.email@example.com>
```

**commit message 规则要点：**

- **标题**：`subsystem: driver: short summary` 格式，使用**祈使句**（"fix" 而不是 "fixed"），不超过 75 字符
- **正文**：解释 Why（为什么改）和 What（改了什么），不要描述 How（代码本身说明）
- **Fixes tag**：用 12 字符 commit hash + 原 commit 标题，如 `Fixes: a1b2c3d4e5f6 ("Bluetooth: hci_core: refactor close")`
- `git config --global core.abbrev 12` 确保 hash 是 12 字符

**发送 patch：**

```bash
# === 3. 找 maintainer ===
./scripts/get_maintainer.pl 0001-*.patch
# 输出：
# Marcel Holtmann <marcel@holtmann.org>
# linux-bluetooth@vger.kernel.org

# === 4. 生成 patch 文件 ===
git format-patch -1 HEAD                 # 最近 1 个 commit
git format-patch upstream/master..HEAD   # 基于 upstream 的所有 commit

# === 5. 检查 patch 格式 ===
./scripts/checkpatch.pl 0001-*.patch
# 或（如果本地没有 checkpatch.pl）：
git format-patch -1 HEAD --stdout | /path/to/linux/scripts/checkpatch.pl -

# === 6. 发送 ===
git send-email --to=linux-bluetooth@vger.kernel.org \
               --cc=marcel@holtmann.org \
               0001-*.patch
```

**修改后重发 v2：**

```bash
# 1. 改代码，amend（不要新增 commit）
git add .
git commit --amend

# 2. 在 commit message 的 Signed-off-by 下面加 changelog：
# ---
# v2:
#   - Fixed comment style as suggested by review
#   - Added missing error handling

# 3. 重新生成 patch（带 v2）
git format-patch -1 HEAD -v2

# 4. 重新发送（用 --in-reply-to 把 v2 线程关联到 v1 的讨论）
git send-email --to=... --cc=... \
               --in-reply-to=<v1-message-id> \
               0001-v2-*.patch
# 不需要 --in-reply-to 也可以，但加了能让 maintainer 在同一个邮件线程里看到所有版本
```

**如何找到 v1 的 Message-ID：** 在你发的 v1 邮件里找 `Message-ID: <xxxxx@xxxxx>` 头，或者从 lore.kernel.org 上你的 patch 页面复制。

---

## 十、checkpatch.pl —— 检查 patch 格式

### 位置和基本用法

```bash
# 位置：kernel 源码 scripts/checkpatch.pl
# BlueZ 仓库没有，可以借用 kernel 的：
/path/to/linux/scripts/checkpatch.pl 0001-*.patch

# 检查 patch 文件
./scripts/checkpatch.pl 0001-*.patch

# 直接检查待发送的 commit（不生成文件）
git format-patch -1 HEAD --stdout | ./scripts/checkpatch.pl -

# 检查某个文件
./scripts/checkpatch.pl --file path/to/file.c

# 检查最近几个 commit
./scripts/checkpatch.pl --git HEAD~3..HEAD

# 严格模式（建议使用）
./scripts/checkpatch.pl --strict 0001-*.patch
```

### 输出解读

| 级别 | 含义 | 是否必须修 |
|------|------|-----------|
| **ERROR** | 违反强制规范 | **必须修**，否则 maintainer 直接拒绝 |
| **WARNING** | 建议修复 | **强烈建议修** |
| **CHECK** | 小建议（`--strict` 时显示） | 尽量修 |

### 常见 ERROR / WARNING 及修复

| 报错信息 | 原因 | 修复 |
|---------|------|------|
| `ERROR: Missing Signed-off-by` | commit 没签名 | `git commit --amend -s` |
| `ERROR: code indent should use tabs` | 缩进用了空格 | 换成 Tab |
| `WARNING: line length exceeds 100 columns` | 行太长 | 换行 |
| `ERROR: trailing whitespace` | 行末有空格 | 删掉 |
| `ERROR: do not use C99 // comments` | 用了 `//` 注释 | 改成 `/* */` |
| `WARNING: 'unsigned' is preferred over 'unsigned int'` | 类型写法不规范 | 改成 `unsigned int` |
| `ERROR: spaces required around that '='` | 运算符缺空格 | 加空格 |

### 格式要求文档出处

```bash
# 内核源码中（需要完整的 kernel tree 才有）：
Documentation/process/submitting-patches.rst    # 提交流程总规范
Documentation/process/coding-style.rst          # C 编码风格

# 在线版：
# https://www.kernel.org/doc/html/latest/process/submitting-patches.html
# https://www.kernel.org/doc/html/latest/process/coding-style.html
```

---

## 十一、官方文档与资源

| 资源 | 链接 |
|------|------|
| 内核 patch 提交流程 | `Documentation/process/submitting-patches.rst` |
| 内核 C 编码风格 | `Documentation/process/coding-style.rst` |
| 邮件客户端配置 | `Documentation/process/email-clients.rst` |
| 在线提交指南 | https://www.kernel.org/doc/html/latest/process/submitting-patches.html |
| git send-email 互动教程 | https://git-send-email.io |
| b4 文档 | https://b4.docs.kernel.org |
| 内核 bugzilla | https://bugzilla.kernel.org |
| BlueZ 邮件列表 | linux-bluetooth@vger.kernel.org (archive: lore.kernel.org/linux-bluetooth) |
