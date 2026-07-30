# xxy plugin 维护与多机同步

本文件描述 `xxy` personal plugin 的 git 长期维护、多机部署与同步方式。

插件位于 `~/.claude/skills/xxy/`，作为 personal 级 plugin 跨所有项目自动加载（`xxy@skills-dir`）。

## 前置状态

- 路径：`~/.claude/skills/xxy/`
- 结构：`.claude-plugin/plugin.json` + `skills/`（3 个）+ `commands/`（7 个）+ `.gitignore`
- 本地已是 git 仓库（`main` 分支，已首次 commit）
- 调用名：`/xxy:<name>`（namespace `xxy`，子项无 `xxy-` 前缀）

## 一、首次 setup（本机 → 远程）

1. 在 GitHub / GitLab 建**私有空仓库**（建议名 `xxy-claude-plugin`），**不要**勾选 README / .gitignore / license（本地已有，避免冲突）
2. 关联远程并推送：
```bash
cd ~/.claude/skills/xxy
git remote add origin <你的仓库URL>
git push -u origin main
```

## 二、多机部署（每台新机器，一条命令）

```bash
git clone <仓库URL> ~/.claude/skills/xxy
```
重启 Claude Code → `claude plugin list` 应见 `xxy@skills-dir` ✔ loaded。

> 若目标机 `~/.claude/skills/xxy` 已存在且非空，git 会拒绝 clone；先清空或换临时路径再移入。

## 三、日常同步

**源机**（改完 skill / command 后）：
```bash
cd ~/.claude/skills/xxy
git add .
git commit -m "update: <改了啥>"
git push
```

**其他机器**：
```bash
cd ~/.claude/skills/xxy && git pull
# 重启 Claude Code 让新内容进 skill 清单
```

## 四、版本节点（重要变更打 tag）

1. 改 `.claude-plugin/plugin.json` 的 `"version"` → 新版本号
2. `git add . && git commit -m "release: v1.1.0" && git push`
3. 打 tag：
```bash
git tag v1.1.0
git push --tags
```

## 五、进阶：marketplace 方式（可选）

想用 `claude plugin install / update` 原生管理（像 superpowers 那样）：

1. 仓库根加 `.claude-plugin/marketplace.json`：
```json
{
  "name": "xxy",
  "owner": { "name": "xxy" },
  "plugins": [{ "name": "xxy", "source": "./", "description": "xxy 个人工作流 skill 集" }]
}
```
2. 多机改用（替代第二节 clone）：
```bash
claude plugin marketplace add <git-host>/<你>/xxy-claude-plugin
claude plugin install xxy
```
3. 更新：源机 push 后，各机 `claude plugin update xxy`

> 个人用纯 git（二、三节）就够；marketplace 适合分享给他人，或想要 `claude plugin update` 原生升级时再上。

## 六、注意事项

- `.git` 在 plugin 目录内，**不影响** Claude Code 扫描 skills/commands
- **单源修改 + push，多机 pull**，避免双向改动冲突（SKILL.md / command 是纯文本，冲突了 git merge 能解）
- 别和 `~/.claude` 的 dotfiles 仓库嵌套——xxy 保持独立仓库
- 多机 clone 前如果 `~/.claude/skills/xxy` 已存在（非空），git 会拒绝；先删空或换路径

## 撤销本地 git init（如果不想用 git 管理）

```bash
rm -rf ~/.claude/skills/xxy/.git
```
plugin 内容不受影响——`.git` 删了，`skills/` `commands/` `plugin.json` 都还在，Claude Code 照常加载。


## 七、加载说明
  
  ### 加载方案

  本机（已就绪，重启即用）

  plugin 已在正确路径，重启 Claude Code 即可。验证命令：
  claude plugin list | grep xxy                     # 期望: xxy@skills-dir ✔ loaded
  claude plugin validate ~/.claude/skills/xxy       # 期望: Validation passed
  重启后 skill 清单出现 /xxy:dev-recap-docs、/xxy:app-error 等 10 项。

  新机器

  git clone <仓库URL> ~/.claude/skills/xxy
  ### 重启 Claude Code → 自动扫描加载
  关键：clone 到 ~/.claude/skills/xxy/ 这个固定路径。放对位置即被扫描，无需任何额外 install 命令。

  验证已加载

  - claude plugin list 出现 xxy@skills-dir ✔ loaded
  - 重启后 skill 清单含 /xxy: 前缀的 10 项，且没有 /xxy-foo（旧 standalone 已全部移除，无重复）

  禁用 / 移除（如需）

  skills-dir plugin 没有 enabledPlugins 那种开关（那是 marketplace plugin 的）。控制加载靠物理位置：
  - 临时禁用：把 xxy 目录移出 ~/.claude/skills/（或改名，如 xxy.disabled），重启后不再加载
  - 彻底删除：rm -rf ~/.claude/skills/xxy
  - 恢复：移回原位 / 重新 clone

  生效时机（重要）

  skill / command 清单在会话启动时注入一次。所以——
  - 任何 plugin 改动（新建、加 skill、改位置、多机 pull 新内容）都要重启 Claude Code，新会话才生效
  - claude plugin list 的 loaded 状态是即时反映的；但实际 /xxy:foo 能否调用，看新会话的清单

  一句话：本机重启即可用；新机 clone 到 ~/.claude/skills/xxy/ 再重启即可用。两处都不需要 install 或 enable。

## 八、如何新增 skill / command

### 新增 skill（方法论类）
1. `skills/<名字>/SKILL.md`（kebab-case，**无需 `xxy-` 前缀**——namespace `xxy` 已提供归属）
2. frontmatter：
```yaml
---
name: <名字>                       # 必须与目录名一致
description: "<何时触发 + 做什么>"   # 模型靠它判断何时调用，触发词前置
---
```
3. 正文写方法论；引用其他 xxy skill 用 `/xxy:<名字>`

### 新增 command（流程类）
`commands/<名字>.md`，frontmatter 同上（`name` 与文件名一致，去 `.md`）。

### 完成后（纳入 git + 多机同步）
```bash
cd ~/.claude/skills/xxy
git add . && git commit -m "add: <名字>" && git push
# 其他机器：cd ~/.claude/skills/xxy && git pull
```
**重启 Claude Code** → 新 skill/command 进清单。先验证结构：`claude plugin validate ~/.claude/skills/xxy`

### 注意
- `name` 字段必须与目录/文件名一致，否则调用异常
- `description` 写清触发场景，避免泛泛（模型靠语义匹配）
- 新 skill 若可能与现有重叠，先走 `/xxy:skill-recap` 的去重逻辑