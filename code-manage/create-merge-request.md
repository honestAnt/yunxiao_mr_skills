---
name: create-merge-request
description: 云效 Codeup 创建合并请求（MR）。提供创建 MR 的完整流程，支持自定义源分支、目标分支、标题和评审人。
---

# 创建合并请求（MR）

通过云效 OpenAPI 创建合并请求（Merge Request / Change Request）。

## 前置条件

环境变量 `YUNXIAO_TOKEN`（个人访问令牌）。

## 快速使用

调用方只需提供 4 个参数，其余由 Skill 自动获取：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| `sourceBranch` | **是** | 源分支名称 | `feature/my-feature` |
| `targetBranch` | **是** | 目标分支名称 | `develop` |
| `title` | **是** | MR 标题 | `feat: 添加用户登录功能` |
| `description` | 否 | MR 描述（支持 Markdown） | `## 变更说明\n- 新增登录接口` |

可选参数：`repoId`（仓库 ID，若已知可跳过查询）、`orgId`（组织 ID，若已知可跳过查询）、`reviewerUserIds`（评审人用户 ID 数组）、`workItemIds`（关联工作项，逗号分隔）、`triggerAIReviewRun`（是否触发 AI 评审）。

## 执行流程

### 步骤 1：确定组织和仓库（若未提供 orgId / repoId）

```bash
# 列出所有组织
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" \
  | jq '.[] | {id, name}'
```

若未指定 `orgId`，则需要遍历所有组织查找仓库。建议直接提供 `repoId`（数字 ID）来跳过查询：

```bash
# 已知 repoId，遍历 org 找到归属（仓库 ID 全局唯一）
for ORG_ID in $(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" | jq -r '.[].id'); do
  FOUND=$(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
    "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories?perPage=100" \
    | jq -r --arg id "{repoId}" '.[] | select(.id == ($id|tonumber)) | .id')
  if [ -n "$FOUND" ]; then echo "$ORG_ID"; break; fi
done
```

或按仓库名称搜索（需指定 `orgId`）：

```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories?search={repoName}&perPage=5" \
  | jq '.[] | {id, name}'
```

### 步骤 3：创建 MR

```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceBranch": "{sourceBranch}",
    "targetBranch": "{targetBranch}",
    "sourceProjectId": {repoId},
    "targetProjectId": {repoId},
    "title": "{title}",
    "description": "{description}"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/{orgId}/repositories/{repoId}/changeRequests"
```

> **关键**: `sourceProjectId` 和 `targetProjectId` 是**必填字段**，缺少会返回无意义的 500 错误。值均为仓库 ID。

### 步骤 4：解析响应

```bash
# 提取关键信息
curl ... | jq '{localId, title, status, sourceBranch, targetBranch, detailUrl}'
```

成功响应字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `localId` | integer | MR 编号（用于后续操作） |
| `bizId` | string | MR 业务 UUID |
| `status` | string | `UNDER_REVIEW` / `MERGED` / `CLOSED` |
| `detailUrl` | string | MR 详情页链接 |
| `sourceBranch` | string | 源分支 |
| `targetBranch` | string | 目标分支 |
| `ahead` | integer | 源领先目标的 commit 数 |
| `behind` | integer | 目标领先源的 commit 数 |

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `500 INTERNAL_SERVER_ERROR` | 缺少 `sourceProjectId`/`targetProjectId` | 确保 body 中包含这两个必填字段 |
| `MERGE_REQUEST_VALIDATE_ERROR` | 源分支相对目标分支无改动 | 确认源分支有未合并的提交 |
| `InvalidToken` | Token 过期或无效 | 更新 `YUNXIAO_TOKEN` |

## 完整示例

```bash
#!/bin/bash
# 已知 repoId=7010028 (creator_guide)，自动发现 orgId 并创建 MR

set -e
YUNXIAO_TOKEN="${YUNXIAO_TOKEN:?请设置 YUNXIAO_TOKEN}"
REPO_ID=7010028
SOURCE_BRANCH="agent/agent/ee87ab09"
TARGET_BRANCH="develop"
TITLE="feat: 新功能"
DESCRIPTION="## 变更说明\n- 添加了 XXX 功能"

# 1. 遍历所有组织，找到仓库归属
ORG_ID=""
for org in $(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" | jq -r '.[].id'); do
  found=$(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
    "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$org/repositories?perPage=100" \
    | jq -r --argjson id "$REPO_ID" '.[] | select(.id == $id) | .id')
  if [ -n "$found" ]; then ORG_ID="$org"; echo "Found: org=$ORG_ID repo=$REPO_ID"; break; fi
done

if [ -z "$ORG_ID" ]; then echo "仓库 $REPO_ID 未找到"; exit 1; fi

# 2. 创建 MR
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"sourceBranch\": \"$SOURCE_BRANCH\",
    \"targetBranch\": \"$TARGET_BRANCH\",
    \"sourceProjectId\": $REPO_ID,
    \"targetProjectId\": $REPO_ID,
    \"title\": \"$TITLE\",
    \"description\": \"$DESCRIPTION\"
  }" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/codeup/organizations/$ORG_ID/repositories/$REPO_ID/changeRequests" \
  | jq '{localId, title, status, sourceBranch, targetBranch, detailUrl}'
```
