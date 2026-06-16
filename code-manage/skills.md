# 云效 DevOps
---
name: yunxiao-devops
description: 阿里云云效 DevOps 平台集成技能。提供代码管理、流水线、项目管理、组织管理、制品仓库、应用交付、测试管理等完整 DevOps 能力。当用户需要与云效平台交互时使用，包括：(1) 代码仓库操作（分支、文件、合并请求）(2) 流水线创建和运行 (3) 项目和工作项管理 (4) 组织成员管理 (5) 制品仓库查询 (6) 应用部署和发布 (7) 测试用例管理。触发词：云效、Codeup、Flow、Projex、AppStack、DevOps。
---

# 云效 DevOps 技能

## 前置条件

用户必须设置环境变量 `YUNXIAO_TOKEN`（个人访问令牌）。
获取方式：云效控制台 → 个人设置 → 个人访问令牌。
API 返回 401/403 时，提示用户检查 token 是否过期。

**验证 token 是否有效**（能列出组织即为有效）：
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations"
```

## API 基础

| 项目 | 值 |
|------|-----|
| Base URL | `https://openapi-rdc.aliyuncs.com` |
| 认证 Header | `x-yunxiao-token: $YUNXIAO_TOKEN` |
| Content-Type | `application/json`（POST/PUT 时必须） |

> **响应格式**: Codeup API 直接返回 JSON 数组/对象，**不**通过 `.data` 字段提取（与部分其他云效 API 不同）。Projex 工作项 API 同样直接返回 JSON，但 PUT 更新返回 `HTTP 204`（无响应体）。
>
> **API 路径约定**: 云效 oapi 使用 `:action` 冒号语法表示资源操作（如 `POST /workitems:search`），与 RESTful 子路径 `/resource/action` 不同。

**通用 curl 模板**：
```bash
# GET
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" "https://openapi-rdc.aliyuncs.com/oapi/v1/..."

# POST/PUT（带 JSON body）
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/..."

# DELETE
curl -s -X DELETE -H "x-yunxiao-token: $YUNXIAO_TOKEN" "https://openapi-rdc.aliyuncs.com/oapi/v1/..."
```

**错误响应格式**:
```json
{
  "errorCode": "NotFound",
  "errorMessage": "Not Found",
  "requestId": "..."
}
```
也有的错误返回：
```json
{
  "timestamp": "2026-...",
  "status": 404,
  "error": "Not Found",
  "path": "..."
}
```

---

## 目录
1. [快速开始：获取 IDs](#快速开始获取-ids)
2. [分支操作](#分支操作)
3. [文件操作](#文件操作)
4. [仓库操作](#仓库操作)
5. [合并请求（MR）](#合并请求mr)
6. [提交操作](#提交操作)
7. [常见工作流](#常见工作流)

> **工作项管理（Projex）**: 工作项（需求/任务/缺陷）的搜索、查看和更新操作已独立为 [yunxiao-workitem](./workitem.md) 技能，与 Codeup 共享相同的 Base URL 和 Token 认证。当用户需要操作工作项时，**必须**参考该文档。<br>
> 工作项 API 使用 `POST /workitems:search`（**冒号**语法，非斜杠 `/search`），PUT 更新返回 `HTTP 204`。

---

## 快速开始：获取 IDs

### 1. 获取组织 ID
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" | jq '.[] | {id, name}'
```
响应示例：
```json
[
  {"id": "67d7dbc7171fc8e8d8c512e6", "name": "Flux 代码管理"},
  {"id": "686f7ad8ef24fb34afabbf9e", "name": "mega-test"}
]
```
将返回的 `id` 作为下文所有请求的 `{orgId}`。

### 2. 获取仓库 ID
```bash
ORG_ID="你的组织ID"
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories?search=仓库名&perPage=20" \
  | jq '.[] | {id, name, pathWithNamespace, sshUrlToRepo}'
```
响应字段：`id`（数字，用作 `{repoId}`）, `name`, `pathWithNamespace`, `defaultBranch`, `visibility`, `webUrl`

> **注意**: `{repoId}` 是纯数字（如 `7010028`），不是路径编码。

---

## 分支操作

### 列出分支
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/branches?sort=updated_desc&perPage=30"
```
查询参数：`sort`（name_asc, name_desc, updated_asc, updated_desc）, `search`, `page`, `perPage`

响应示例（数组）：
```json
[
  {
    "name": "master",
    "commit": {
      "id": "aef36f82cf6f01b83986dd5ef6581e4b85e91b3c",
      "shortId": "aef36f82",
      "title": "提交信息",
      "authorName": "fengbao.wang",
      "authoredDate": "2026-06-14T20:54:40+08:00"
    },
    "protected": false,
    "defaultBranch": true,
    "webUrl": "https://codeup.aliyun.com/..."
  }
]
```

### 获取单个分支
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/branches/{branchName}"
```
`branchName` 含 `/` 时需要 URL 编码：`feature%2Flogin`，query string 中不需要编码。

### 创建分支
```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/branches?branch={newBranch}&ref={sourceBranch}"
```
- `branch`: 新分支名
- `ref`: 源分支名或 commit SHA（默认 master）

### 删除分支
```bash
curl -s -X DELETE \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/branches/{branchName}"
```

---

## 文件操作

### 列出文件树
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/files/tree?ref={branch}"
```
响应：
```json
[
  {"name": "README.md", "path": "README.md", "type": "blob"},
  {"name": "src", "path": "src", "type": "tree"}
]
```
`type`: `blob`（文件）或 `tree`（目录）

### 读取文件内容
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/files/{filePath}?ref={branch}"
```
> **注意**: 直接 `/files/{path}?ref=`，**不需要** `/blobs` 后缀。

响应：
```json
{
  "blobId": "dedb07b805c25751b889f07d15ff4108ac02665d",
  "commitId": "aef36f82...",
  "content": "CiMgR2l05LuT5bqTIC...",   // Base64 编码
  "encoding": "base64",
  "fileName": "README.md",
  "filePath": "README.md",
  "size": "4985",
  "ref": "master"
}
```
解码内容：
```bash
curl ... | jq -r '.content' | base64 -d
```

### 创建文件
```bash
CONTENT_B64=$(echo -n '文件内容' | base64)
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"filePath\":\"path/to/file.txt\",\"content\":\"$CONTENT_B64\",\"branch\":\"master\",\"commitMessage\":\"feat: 添加文件\"}" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/files"
```
> **重要**: `content` 必须是 **Base64 编码**后的字符串。

### 更新文件
```bash
CONTENT_B64=$(echo -n '更新后的内容' | base64)
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"filePath\":\"path/to/file.txt\",\"content\":\"$CONTENT_B64\",\"branch\":\"master\",\"commitMessage\":\"fix: 更新\"}" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/files"
```

### 删除文件
```bash
curl -s -X DELETE \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"filePath":"path/to/file.txt","branch":"master","commitMessage":"chore: 删除"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/files"
```

---

## 仓库操作

### 列出仓库
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories?perPage=50&search={关键词}"
```
查询参数：`search`, `page`, `perPage`

响应字段（数组）：`id`, `name`, `pathWithNamespace`, `description`, `defaultBranch`, `visibility`, `sshUrlToRepo`, `httpUrlToRepo`, `webUrl`, `createdAt`, `lastActivityAt`, `namespaceId`, `accessLevel`, `archived`

### 获取仓库详情
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}"
```
响应额外包含：`creatorId`, `encrypted`, `starCount`, `starred`

---

## 合并请求（MR）

### 列出 MR (组织级别，可按仓库过滤)
```bash
# 所有 open MR
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/changeRequests?state=opened&perPage=30"

# 按仓库过滤（多仓库用逗号分隔）
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/changeRequests?projectIds={repoId}&state=opened"

# 完整参数示例
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/changeRequests?projectIds={repoId1},{repoId2}&state=opened&page=1&perPage=20&orderBy=updated_at&sort=desc&search=关键词&authorIds=userId1&reviewerIds=userId2&createdAfter=2024-01-01T00:00:00Z&createdBefore=2024-12-31T23:59:59Z"
```
> **注意**: MR 列表在**组织级别**（`/organizations/{orgId}/changeRequests`），通过 `projectIds`（复数）查询参数按仓库过滤，**不是**在仓库路径下。

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `state` | string | 否 | `opened`, `merged`, `closed`，不传返回全部 |
| `projectIds` | string | 否 | 仓库 ID 列表，逗号分隔 |
| `page` | integer | 否 | 页码，默认 1 |
| `perPage` | integer | 否 | 每页大小，默认 20 |
| `search` | string | 否 | 标题关键字搜索 |
| `authorIds` | string | 否 | 创建者用户 ID，逗号分隔 |
| `reviewerIds` | string | 否 | 评审人用户 ID，逗号分隔 |
| `orderBy` | string | 否 | 排序字段：`created_at` / `updated_at`，默认 `updated_at` |
| `sort` | string | 否 | 排序方式：`asc` / `desc`，默认 `desc` |
| `createdAfter` | string | 否 | 起始创建时间，ISO 8601 格式 |
| `createdBefore` | string | 否 | 截止创建时间，ISO 8601 格式 |

响应数组字段：
- `localId`: MR 编号（整数，用于单 MR 操作）
- `mrBizId`: MR 业务 UUID
- `projectId`: 仓库数字 ID
- `title`, `description`, `state`, `sourceBranch`, `targetBranch`
- `author` (对象: `{id, name, username, email}`)
- `reviewers` (数组: `[{id, name, hasReviewed, hasCommented}]`)
- `detailUrl`: 网页链接

```bash
# 提取关键信息
curl ... | jq '.[] | {localId, title, state, sourceBranch, targetBranch, projectId, author: .author.name}'
```

### 获取单个 MR
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}"
```
> `localId` 是 MR 编号（数字），非 `mrBizId`。

### 创建 MR
```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceBranch": "feature/my-feature",
    "targetBranch": "master",
    "sourceProjectId": 2813489,
    "targetProjectId": 2813489,
    "title": "feat: 功能说明",
    "description": "详细描述"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests"
```
> **必填**: `sourceBranch`, `targetBranch`, `sourceProjectId`, `targetProjectId`, `title`
> 可选字段：`reviewerUserIds` (评审人用户 ID 数组), `triggerAIReviewRun` (是否触发 AI 评审), `workItemIds` (关联工作项 ID，逗号分隔字符串)

### 更新 MR 基本信息
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"更新后的标题","description":"更新后的描述"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}"
```

### 关闭 MR
```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/close"
```
响应：`{"result": true}`

### 重新打开 MR
```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/reopen"
```
> 注意：若源分支相对目标分支无新改动，重新打开会失败。

### 评审 MR
```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reviewOpinion":"PASS","reviewComment":"代码评审通过"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/review"
```
Body 参数：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `reviewOpinion` | string | 否 | `PASS` - 通过；`NOT_PASS` - 不通过 |
| `reviewComment` | string | 否 | 评审评论内容 |
| `submitDraftCommentIds` | array[string] | 否 | 提交的草稿评论 ID 列表 |

### 合并 MR
```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mergeType":"no-fast-forward","mergeMessage":"合并信息","removeSourceBranch":false}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/merge"
```
Body 参数：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `mergeType` | string | **是** | `ff-only` / `no-fast-forward` / `squash` / `rebase` |
| `mergeMessage` | string | 否 | 合并提交信息 |
| `removeSourceBranch` | boolean | 否 | 合并后是否删除源分支 |

### 创建 MR 评论

```bash
# 全局评论（patchset_biz_id 可选）
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "comment_type": "GLOBAL_COMMENT",
    "content": "整体看起来不错",
    "draft": false,
    "resolved": false
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/comments"

# 行内评论
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "comment_type": "INLINE_COMMENT",
    "content": "这里建议优化",
    "draft": false,
    "resolved": false,
    "file_path": "src/utils.ts",
    "line_number": 42
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/comments"

# 回复评论
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "comment_type": "GLOBAL_COMMENT",
    "content": "回复评论内容",
    "draft": false,
    "resolved": false,
    "parent_comment_biz_id": "父评论bizId"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/comments"
```

Body 参数：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `comment_type` | string | **是** | `GLOBAL_COMMENT` 或 `INLINE_COMMENT` |
| `content` | string | **是** | 评论内容（1-65535 字符） |
| `draft` | boolean | **是** | 是否草稿评论 |
| `resolved` | boolean | **是** | 是否标记已解决 |
| `file_path` | string | 否 | 文件路径（仅行内评论） |
| `line_number` | integer | 否 | 行号（仅行内评论） |
| `patchset_biz_id` | string | 否 | 关联版本 ID（官方标注必填，实测全局评论可不传） |
| `from_patchset_biz_id` | string | 否 | 起始版本 ID（INLINE_COMMENT 时与 `to_patchset_biz_id` 二选一） |
| `to_patchset_biz_id` | string | 否 | 目标版本 ID（INLINE_COMMENT 时与 `from_patchset_biz_id` 二选一） |
| `parent_comment_biz_id` | string | 否 | 父评论 bizId（回复评论时使用） |

> **响应字段使用 snake_case**：`comment_biz_id`, `comment_type`, `comment_time`, `is_deleted`

### 删除 MR 评论
```bash
curl -s -X DELETE \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/comments/{commentBizId}"
```
> `commentBizId`：评论的业务 ID，从评论创建响应或列表中获取。

### 查询 MR 变更文件树
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/diffs/changeTree?fromPatchSetId={fromPatchSetId}&toPatchSetId={toPatchSetId}"
```
> `fromPatchSetId` 和 `toPatchSetId` 为必填查询参数，可从 MR 详情中的 patchset 信息获取。

### 管理 MR 评审人
```bash
# 添加评审人
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"userIds":["userId1","userId2"]}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/person/REVIEWER"

# 添加订阅人
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"userIds":["userId1"]}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}/person/SUBSCRIBER"
```

---

## 提交操作

### 列出提交
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/commits?refName={branch}&perPage=30"
```
> **重要**: 参数名是 `refName`，**不是** `ref`。

响应数组字段：
```json
[
  {
    "id": "aef36f82cf6f01b83986dd5ef6581e4b85e91b3c",
    "shortId": "aef36f82",
    "title": "提交标题（commit message 第一行）",
    "message": "完整提交信息",
    "authorName": "fengbao.wang",
    "authorEmail": "fengbao.wang@alo7.com",
    "authoredDate": "2026-06-14T20:54:40+08:00",
    "committerName": "fengbao.wang",
    "committedDate": "2026-06-14T20:54:40+08:00",
    "parentIds": ["..."],
    "webUrl": "https://codeup.aliyun.com/..."
  }
]
```

### 获取单个提交
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/commits/{sha}"
```
使用 `shortId`（8 位）或完整 `id`（40 位）均可。

响应额外含 `stats` 对象：`{"additions": 1, "deletions": 1, "total": 2}`

---

## 常见工作流

### 工作流 1：为需求创建分支并发起 MR
```bash
# 1. 获取组织 ID
ORG_ID=$(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" | jq -r '.[0].id')

# 2. 找到仓库 ID
REPO_ID=$(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories?search=my-project" \
  | jq -r '.[0].id')

# 3. 从 master 创建功能分支
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/branches?branch=feature/my-feature&ref=master"

# 4. 创建 MR（注意：MR 路径在 org 下，包含 repo）
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceBranch": "feature/my-feature",
    "targetBranch": "master",
    "sourceProjectId": '"$REPO_ID"',
    "targetProjectId": '"$REPO_ID"',
    "title": "feat: 新功能",
    "description": "## 变更说明\n- 添加了 XXX"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/changeRequests" \
  | jq '{localId, title, detailUrl}'
```

### 工作流 2：查看待处理 MR 列表
```bash
# 列出某仓库所有 open MR
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/changeRequests?projectIds=$REPO_ID&state=opened" \
  | jq '.[] | {localId, title, sourceBranch, targetBranch, author: .author.name, detailUrl}'

# 查看 MR 变更的文件
LOCAL_ID=42
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/changeRequests/$LOCAL_ID/diffs/changeTree"
```

### 工作流 3：查看文件内容并修改
```bash
# 读取文件（Base64 会自动解码）
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/files/src/config.ts?ref=master" \
  | jq -r '.content' | base64 -d

# 修改文件
NEW_CONTENT=$(echo -n '新内容' | base64)
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"filePath\":\"src/config.ts\",\"content\":\"$NEW_CONTENT\",\"branch\":\"master\",\"commitMessage\":\"fix: 更新配置\"}" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/files"
```

### 工作流 4：查看提交历史
```bash
# 最近 30 条提交
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/commits?refName=master&perPage=30" \
  | jq '.[] | {shortId, title, authorName, authoredDate}'

# 查看某提交详情（含代码变更统计）
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/commits/aef36f82" \
  | jq '{shortId, title, stats, authorName}'
```

---

## 错误处理速查

| HTTP 状态 | errorCode | 含义 | 处理 |
|-----------|-----------|------|------|
| 200 | - | 成功 | — |
| 204 | - | PUT 更新成功（无响应体） | 需 GET 验证更新结果 |
| 400 | BAD_REQUEST | 参数缺失/错误 | 检查参数名是否正确（如 `refName` vs `ref`） |
| 401 | Unauthorized | 未认证 | 提示用户检查 `YUNXIAO_TOKEN` |
| 403 | Forbidden.* | 无权限 | 检查 orgId 是否正确、token 是否有仓库权限 |
| 404 | NotFound | 路径/资源不存在 | 检查 API 路径格式、repoId/branchName 正确性 |

**通用错误检查模板**（兼容 204）：
```bash
RESPONSE=$(curl -s -w "\n%{http_code}" -H "x-yunxiao-token: $YUNXIAO_TOKEN" "$URL")
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | sed '$d')

# 200 和 204 均为成功
if [ "$HTTP_CODE" != "200" ] && [ "$HTTP_CODE" != "204" ]; then
  echo "HTTP $HTTP_CODE 错误" >&2
  echo "$BODY" | jq '{errorCode, errorMessage, error: .error, status: .status}' >&2
  exit 1
fi
# 仅 200 时检查响应体错误码（204 无响应体）
if [ "$HTTP_CODE" = "200" ]; then
  echo "$BODY" | jq -e 'has("errorCode")' > /dev/null 2>&1 && {
    echo "API 错误: $(echo "$BODY" | jq -r '.errorCode + ": " + .errorMessage')" >&2
    exit 1
  }
fi
```

---

## 关键注意事项

1. **响应格式**: Codeup API 直接返回 JSON 数组/对象，**不需要** `.data` 提取。与部分其他云效 API（返回 `{"success":true,"data":...}`）不同。
2. **组织列表**: 路径是 `/oapi/v1/platform/organizations`，**不是** `/oapi/v1/organizations`。
3. **文件读取**: 路径是 `/files/{path}?ref=branch`，**不需要** `/blobs` 后缀。
4. **提交列表**: 参数名是 `refName`，**不是** `ref`。
5. **MR 列表**: 在组织层级 `/organizations/{orgId}/changeRequests`，通过 `projectIds`（复数）查询参数按仓库过滤，支持逗号分隔多仓库。
6. **MR 单条/创建**: 在仓库层级 `/organizations/{orgId}/repositories/{repoId}/changeRequests/{localId}`。
7. **MR 创建必填**: `sourceBranch`, `targetBranch`, `sourceProjectId`, `targetProjectId`, `title` — 缺少 `sourceProjectId`/`targetProjectId` 会返回 500 无意义错误。
8. **MR 关闭/评审/合并**: 使用 POST 到 `/close`, `/review`, `/merge` 后缀，非 PUT 修改 state。
9. **localId vs mrBizId**: MR 标识用 `localId`（数字），不用 `mrBizId`（UUID）。
10. **文件内容**: 创建/更新时 `content` 必须是 Base64 编码字符串。
11. **分支名编码**: URL 路径中的 `/` 编码为 `%2F`，query string 中不需要编码。
12. **评论字段**: 使用 snake_case（`comment_type`, `file_path`, `line_number`），必填 `draft`, `patchset_biz_id`, `resolved`。
