# github-starlist-skill

一个 Claude Code skill，用于通过 GitHub GraphQL API 管理 Star List（收藏列表）。

## 功能

- 列出所有 star list
- 读取指定 list 中的仓库
- Star 仓库并添加到指定 list
- 从 list 中移除仓库
- 创建/删除 list

## 安装

将本仓库作为 Claude Code skill 使用：

1. 克隆到本地
2. 将 `skill.md` 的路径加入你的 Claude Code skill 配置

或作为 git submodule 添加到你的 skill 集合仓库中。

## 前置条件

- [GitHub CLI (`gh`)](https://cli.github.com/) 已安装并登录
- Token 需要 `user` scope：
  ```bash
  gh auth refresh -s user
  ```

## 使用

在 Claude Code 中，直接用自然语言描述你想对 star list 做的操作即可，例如：

- "帮我看看我的 star list 有哪些"
- "把 https://github.com/foo/bar 加入 skill list"
- "创建一个新的 star list 叫 awesome"

## 技术细节

GitHub Star List 没有公开的 REST API，本 skill 完全通过 GraphQL API 实现。核心 mutation：

- `updateUserListsForItem` — 更新仓库所属的 list（注意是覆盖式操作）
- `createUserList` — 创建新 list
- `deleteUserList` — 删除 list

## License

MIT
