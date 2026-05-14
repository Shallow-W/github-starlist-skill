---
name: github-starlist
description: Manage GitHub Star Lists via GraphQL API. Use when the user wants to view, create, modify, or sync star lists; add/remove repos from lists; star/unstar repos; or compare local submodules against a star list.
license: MIT
triggers:
  - star list
  - starlist
  - 收藏列表
  - 加入 list
  - 添加到 list
  - star 管理
  - list 同步
  - 列表同步
  - unstar
  - 取消收藏
tags:
  - github
  - star
  - list
  - management
  - graphql
  - submodule
  - sync
---

# GitHub Star List Management

Manage GitHub Star Lists (User Lists) through the GitHub GraphQL API via `gh` CLI. Triggered when the user mentions star lists, wants to organize starred repos into lists, or sync local submodules with a remote star list.

## Prerequisites

Ensure the GitHub token has the `user` scope. If operations fail with `INSUFFICIENT_SCOPES`:

```bash
gh auth refresh -s user
```

## Key Concept: updateUserListsForItem is a REPLACE Operation

The `updateUserListsForItem` mutation **replaces** all list memberships for a repo, it does not append. Always query existing memberships first and merge before submitting.

---

## Operations

### 1. List All User Lists

List every star list belonging to the authenticated user. Supports pagination.

```bash
gh api graphql -f query='{ viewer { lists(first: 20) { pageInfo { hasNextPage endCursor } nodes { id name slug description } } } }'
```

**Pagination loop** — if `hasNextPage` is true, repeat with the cursor:

```bash
gh api graphql -f query='{ viewer { lists(first: 20, after: "<END_CURSOR>") { pageInfo { hasNextPage endCursor } nodes { id name slug description } } } }'
```

Continue until `hasNextPage` is false.

### 2. Read Repos in a List

Read all repositories belonging to a specific list. Returns enriched metadata.

```bash
gh api graphql -f query='{ node(id: "<LIST_ID>") { ... on UserList { name items(first: 50) { pageInfo { hasNextPage endCursor } nodes { ... on Repository { nameWithOwner description stargazerCount primaryLanguage { name } url updatedAt } } } } } }'
```

**Pagination** — if `hasNextPage` is true:

```bash
gh api graphql -f query='{ node(id: "<LIST_ID>") { ... on UserList { name items(first: 50, after: "<END_CURSOR>") { pageInfo { hasNextPage endCursor } nodes { ... on Repository { nameWithOwner description stargazerCount primaryLanguage { name } url updatedAt } } } } } }'
```

**Extract names only** (quick output):

```bash
gh api graphql -f query='{ node(id: "<LIST_ID>") { ... on UserList { name items(first: 50) { nodes { ... on Repository { nameWithOwner } } } } } }' -q '.data.node.items.nodes[].nameWithOwner'
```

### 3. Star a Repo

```bash
gh api -X PUT "user/starred/<OWNER>/<REPO>"
```

### 4. Add Repo to a List (with Merge)

This is a three-step process due to the REPLACE semantics of `updateUserListsForItem`.

**Step 1 — Get the repo's node ID:**

```bash
REPO_ID=$(gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") { id } }' -q '.data.repository.id')
```

**Step 2 — Query the repo's current list memberships:**

```bash
gh api graphql -f query='{ node(id: "'"$REPO_ID"'") { ... on Repository { starredLists(first: 20) { pageInfo { hasNextPage endCursor } nodes { id name } } } } }'
```

Collect all existing list IDs. If `hasNextPage` is true, paginate with `after: "<END_CURSOR>"`.

**Step 3 — Merge and submit:**

Combine the existing list IDs with the target list ID, then submit:

```bash
gh api graphql -f query="mutation { updateUserListsForItem(input: { itemId: \"$REPO_ID\", listIds: [\"<EXISTING_ID_1>\", \"<EXISTING_ID_2>\", \"<TARGET_LIST_ID>\"] }) { clientMutationId } }"
```

**Preview before executing** — show the user what will change:
- Current list memberships
- New list to add
- Final combined list of IDs

Only execute after confirmation (or when the user has explicitly requested the operation).

### 5. Remove Repo from a List

Same REPLACE pattern as adding, but excluding the target list ID.

**Step 1 — Get the repo's node ID** (same as above).

**Step 2 — Query current memberships** (same as above, with pagination).

**Step 3 — Remove the target list ID from the collected IDs and submit:**

```bash
gh api graphql -f query="mutation { updateUserListsForItem(input: { itemId: \"$REPO_ID\", listIds: [\"<REMAINING_ID_1>\", \"<REMAINING_ID_2>\"] }) { clientMutationId } }"
```

### 6. Create a New List

```bash
gh api graphql -f query='mutation { createUserList(input: { name: "<NAME>", description: "<DESCRIPTION>" }) { userList { id name slug } } }'
```

### 7. Delete a List

```bash
gh api graphql -f query='mutation { deleteUserList(input: { id: "<LIST_ID>" }) { clientMutationId } }'
```

### 8. Unstar a Repo

Remove a star from a repository:

```bash
gh api -X DELETE "user/starred/<OWNER>/<REPO>"
```

Note: Unstarring does NOT remove the repo from any star lists. To fully clean up, also remove from lists (operation 5).

### 9. Rename / Update a List

Use the `updateUserList` mutation to change a list's name or description:

```bash
gh api graphql -f query='mutation { updateUserList(input: { id: "<LIST_ID>", name: "<NEW_NAME>", description: "<NEW_DESCRIPTION>" }) { userList { id name slug description } } }'
```

Only include the fields you want to change.

### 10. Cross-List Membership Report

Check which lists contain a specific repo:

```bash
# Get repo ID first
REPO_ID=$(gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") { id } }' -q '.data.repository.id')

# Query all lists containing this repo
gh api graphql -f query='{ node(id: "'"$REPO_ID"'") { ... on Repository { starredLists(first: 20) { pageInfo { hasNextPage endCursor } nodes { id name slug } } } } }'
```

Paginate if `hasNextPage` is true.

### 11. Bulk Star + Add to List

Star multiple repos and add them all to a target list in sequence. For each repo:

```bash
# For each <OWNER>/<REPO> in the list:
gh api -X PUT "user/starred/<OWNER>/<REPO>"

REPO_ID=$(gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") { id } }' -q '.data.repository.id')

# Query current memberships, merge with target list, submit
# (follow operation 4 pattern for each repo)
```

**Optimization** — if adding many repos to the same list and none have existing list memberships, batch the mutations in sequence without re-querying each time. Only query memberships when a repo might already belong to other lists.

### 12. List Diff / Sync

Compare a local directory of git submodules against a GitHub star list to find discrepancies.

**Step 1 — Read the star list repos** (operation 2, collect all `nameWithOwner` values).

**Step 2 — Read local submodules:**

```bash
git config --file .gitmodules --get-regexp path | awk '{print $2}'
```

Or parse `.gitmodules` for submodule names.

**Step 3 — Compute the diff:**

- **In star list but not local**: repos that need `git submodule add`
- **Local but not in star list**: repos that exist locally but are missing from the list (may need to be added to the list, or removed locally)

Output the diff as two lists for the user to review before taking action.

### 13. Search Within a List

Filter repos in a list by name pattern. First fetch all repos (operation 2 with pagination), then filter client-side:

```bash
# Fetch all repos from the list, then grep for the pattern
gh api graphql -f query='{ node(id: "<LIST_ID>") { ... on UserList { name items(first: 50) { nodes { ... on Repository { nameWithOwner description } } } } } }' -q '.data.node.items.nodes[] | "\(.nameWithOwner) | \(.description)"' | grep -i "<PATTERN>"
```

For large lists, paginate through all items before filtering.

---

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `INSUFFICIENT_SCOPES` | Token missing `user` scope | Run `gh auth refresh -s user` |
| `NOT_FOUND` | Invalid list ID, repo, or slug | Verify the ID/name and retry |
| `FORBIDDEN` | No permission on target resource | Check repo/list ownership |
| `RATE_LIMITED` | Too many requests | Wait and retry; add delays between bulk operations |
| Network error | `gh` CLI connectivity | Verify `gh auth status` and network connection |

For bulk operations, add a small delay (`sleep 1`) between mutations to avoid rate limiting.

## Pagination Pattern

All GraphQL queries that return connection types support `first` (page size) and `after` (cursor) arguments. Always check `pageInfo.hasNextPage` and loop with `pageInfo.endCursor`:

```
query(first: 50) → check pageInfo.hasNextPage
  if true → query(first: 50, after: endCursor) → repeat
  if false → done
```

Default page size is 50. For lists with more items, pagination is mandatory to get complete results.
