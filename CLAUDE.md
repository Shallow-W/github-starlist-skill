# GitHub Star List 管理 Skill

一个用于管理 GitHub Star List（收藏列表）的 Claude Code skill。通过 GitHub GraphQL API 实现对 star list 的读取、添加仓库到指定列表等操作。

## 功能

- 读取指定 star list 中的所有仓库
- 将仓库 star 并添加到指定 list
- 从 list 中移除仓库
- 列出用户所有 star list
- 创建新的 star list

## 前置条件

- 已安装 `gh` CLI 并登录
- GitHub token 需要 `user` scope（运行 `gh auth refresh -s user` 添加）

## 使用方式

将本目录下的 `skill.md` 文件内容加载到 Claude Code 会话中即可。Claude 会自动识别相关指令并执行。

## 常用命令

### 列出所有 star list

```bash
gh api graphql -f query='{ viewer { lists(first: 20) { nodes { id name slug } } } }'
```

### 读取指定 list 中的仓库

```bash
gh api graphql -f query='{ node(id: "<LIST_ID>") { ... on UserList { name items(first: 50) { nodes { ... on Repository { nameWithOwner } } } } } }' -q '.data.node.items.nodes[].nameWithOwner'
```

### Star 仓库

```bash
gh api -X PUT "user/starred/<OWNER>/<REPO>"
```

### 将仓库添加到指定 list

```bash
# 获取仓库 node ID
REPO_ID=$(gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") { id } }' -q '.data.repository.id')

# 添加到 list
gh api graphql -f query="mutation { updateUserListsForItem(input: { itemId: \"$REPO_ID\", listIds: [\"<LIST_ID>\"] }) { clientMutationId } }"
```

### 取消 star

```bash
gh api -X DELETE "user/starred/<OWNER>/<REPO>"
```

### 创建新 list

```bash
gh api graphql -f query='mutation { createUserList(input: { name: "<LIST_NAME>", description: "<DESCRIPTION>" }) { userList { id name slug } } }'
```

### 删除 list

```bash
gh api graphql -f query='mutation { deleteUserList(input: { id: "<LIST_ID>" }) { clientMutationId } }'
```

## 注意事项

- GitHub star list 目前没有公开的 REST API，只能通过 GraphQL API 操作
- `updateUserListsForItem` 是**覆盖式**操作：传入的 `listIds` 会替换该仓库所属的所有 list，不是追加。如果要追加到现有 list，需要先查询仓库当前所在的所有 list，然后合并后再提交
- token 必须有 `user` scope，否则会报 `INSUFFICIENT_SCOPES` 错误
