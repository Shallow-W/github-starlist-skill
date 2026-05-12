---
name: github-starlist
description: 管理 GitHub Star List（收藏列表）。当用户要求查看、添加、删除 star list 或将仓库加入某个 list 时触发。
triggers:
  - star list
  - starlist
  - 收藏列表
  - 加入 list
  - 添加到 list
  - star 管理
tags:
  - github
  - star
  - list
  - management
  - graphql
---

# GitHub Star List 管理

当用户提到 star list、收藏列表、将仓库加入 list 等操作时，按以下流程执行。

## 前置检查

确保 token 有 `user` scope。如果操作报 `INSUFFICIENT_SCOPES` 错误，提示用户运行：

```bash
gh auth refresh -s user
```

## 列出所有 list

用户想看自己有哪些 list 时：

```bash
gh api graphql -f query='{ viewer { lists(first: 20) { nodes { id name slug } } } }'
```

返回每个 list 的 ID、名称和 slug。

## 读取指定 list 内容

用户想看某个 list 里有什么仓库时，使用 list 的 slug 或 ID：

```bash
# 如果知道 list ID
gh api graphql -f query='{ node(id: "<LIST_ID>") { ... on UserList { name items(first: 50) { nodes { ... on Repository { nameWithOwner } } } } } }' -q '.data.node.items.nodes[].nameWithOwner'

# 如果只知道 slug，先通过列出所有 list 获取 ID
```

## 将仓库 star 并加入指定 list

当用户给了一个仓库 URL 并要求加入某个 list 时，执行三步：

### 第一步：Star 仓库

```bash
gh api -X PUT "user/starred/<OWNER>/<REPO>"
```

### 第二步：获取仓库 node ID

```bash
gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") { id } }' -q '.data.repository.id'
```

### 第三步：添加到 list

```bash
gh api graphql -f query="mutation { updateUserListsForItem(input: { itemId: \"<REPO_ID>\", listIds: [\"<LIST_ID>\"] }) { clientMutationId } }"
```

**重要**：`updateUserListsForItem` 是覆盖操作。如果仓库已在其他 list 中，需要先查询当前所属 list，合并后再提交：

```bash
# 查询仓库当前所在的所有 list
gh api graphql -f query='{ node(id: "<REPO_ID>") { ... on Repository { starredLists(first: 20) { nodes { id name } } } } }'
```

然后将现有的 list ID 与目标 list ID 合并后一起提交。

## 从 list 中移除仓库

先查询仓库当前所在的所有 list，去掉目标 list ID 后重新提交：

```bash
# 获取当前 list
# 去掉目标 list ID 后调用 updateUserListsForItem
```

## 创建新 list

```bash
gh api graphql -f query='mutation { createUserList(input: { name: "<名称>", description: "<描述>" }) { userList { id name slug } } }'
```

## 删除 list

```bash
gh api graphql -f query='mutation { deleteUserList(input: { id: "<LIST_ID>" }) { clientMutationId } }'
```
