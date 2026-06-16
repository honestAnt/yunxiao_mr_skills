---
name: yunxiao-workitem
description: 云效 Projex 工作项管理技能。提供工作项的搜索、查看和更新能力，支持需求（Req）、任务（Task）、缺陷（Bug）等类型。当用户需要操作云效项目协作中的工作项时使用，包括：(1) 按条件搜索/筛选工作项 (2) 获取工作项详情 (3) 更新工作项信息（标题、描述、状态、负责人、优先级等）。触发词：工作项、Projex、需求、任务、缺陷、Bug、workitem、更新工作项、搜索工作项。
---

# 云效 Projex 工作项管理

通过云效 OpenAPI 操作项目协作（Projex）中的工作项，支持需求、任务、缺陷等类型。

> **注意**: 本文档内容基于 API 实测验证（MCP Server v0.3.47 源码 + curl 调用确认）。

## 前置条件

环境变量 `YUNXIAO_TOKEN`（个人访问令牌）。
获取方式：云效控制台 → 个人设置 → 个人访问令牌。

## API 基础

| 项目 | 值 |
|------|-----|
| Base URL | `https://openapi-rdc.aliyuncs.com` |
| 认证 Header | `x-yunxiao-token: $YUNXIAO_TOKEN` |
| Content-Type | `application/json`（POST/PUT 时必须） |

**通用 curl 模板**：
```bash
# GET
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/..."

# POST（带 JSON body）
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/..."

# PUT（带 JSON body）
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/..."
```

**版本说明**：云效 Projex API 分为中心版和 Region 版，路径略有不同：

| 版本 | 路径前缀 | 适用场景 |
|------|----------|----------|
| 中心版（默认） | `/oapi/v1/projex/organizations/{organizationId}/...` | 大多数用户 |
| Region 版 | `/oapi/v1/projex/...`（不带 organizations 前缀） | Region 站点 |

> 下文默认使用中心版路径。若当前 Base URL 不是 `openapi-rdc.aliyuncs.com`，系统会自动切换为 Region 版路径。

**错误响应格式**:
```json
{
  "errorCode": "InvaildData.Failed",
  "errorMessage": "错误描述",
  "errorMsg": "错误描述",
  "success": false,
  "traceId": "..."
}
```

---

## 快速开始：获取 ID

### 1. 获取组织 ID

```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" | jq '.[] | {id, name}'
```

### 2. 获取项目 ID（spaceId）

工作项所属的项目 ID，可从云效项目链接获取。或在项目设置中查看。

### 3. 获取用户 ID

```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/user" | jq '{id, name, email}'
```

### 4. 获取工作项类型

```bash
# 列出所有可用类型
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitemTypes" \
  | jq '.[] | {id, name, categoryId}'
```

常用 `categoryId` 值：

| categoryId | 含义 |
|------------|------|
| `Req` | 需求 |
| `Task` | 任务 |
| `Bug` | 缺陷 |
| `Risk` | 风险 |
| `Request` | 诉求 |
| `Topic` | 主题 |

---

## 搜索工作项 (SearchWorkitems)

按条件筛选和搜索工作项列表。

> **关键**: 搜索使用 `POST` 方法，路径后缀为 `:search`（**冒号**，非斜杠）。

### 请求

```bash
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "spaceId": "{spaceId}",
    "category": "Req",
    "maxResults": 20,
    "orderBy": "gmtCreate",
    "sort": "desc"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems:search"
```

### Body 参数

Body 为 JSON 对象：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `spaceId` | string | **是** | 项目 ID |
| `category` | string | **是** | 工作项类型，多个逗号分隔：`Req,Task,Bug` |
| `spaceType` | string | 否 | 空间类型，默认 `Project`，可选 `Program`（项目集） |
| `status` | string | 否 | 状态 ID（如 `28`=待确认, `100010`=处理中, `100011`=已完成） |
| `priority` | string | 否 | 优先级 ID |
| `assignedTo` | string | 否 | 负责人用户 ID，`self` 表示当前用户 |
| `creator` | string | 否 | 创建人用户 ID |
| `sprint` | string | 否 | 迭代 ID |
| `subject` | string | 否 | 标题关键词 |
| `tag` | string | 否 | 标签 ID |
| `workitemType` | string | 否 | 工作项子类型 ID |
| `statusStage` | string | 否 | 状态阶段 |
| `subjectDescription` | string | 否 | 描述内容搜索 |
| `page` | integer | 否 | 页码 |
| `perPage` / `maxResults` | integer | 否 | 每页数量，默认 20，最大 200 |
| `orderBy` | string | 否 | 排序字段，默认 `gmtCreate` |
| `sort` | string | 否 | 排序方向：`desc`（默认）/ `asc` |
| `createdAfter` | string | 否 | 创建时间起始，格式 `YYYY-MM-DD` |
| `createdBefore` | string | 否 | 创建时间截止 |
| `updatedAfter` | string | 否 | 更新时间起始 |
| `updatedBefore` | string | 否 | 更新时间截止 |
| `finishTimeAfter` | string | 否 | 完成时间起始 |
| `finishTimeBefore` | string | 否 | 完成时间截止 |
| `updateStatusAtAfter` | string | 否 | 状态更新时间起始 |
| `updateStatusAtBefore` | string | 否 | 状态更新时间截止 |
| `includeDetails` | boolean | 否 | 是否在搜索结果中返回描述详情，默认 `false` |
| `advancedConditions` | string | 否 | 高级过滤条件（JSON 字符串），用法参考下方说明 |

### 响应

搜索成功返回 **JSON 数组**（直接是工作项列表），HTTP 200：

```json
[
  {
    "id": "3bbc1de7f744009b40f6e36f74",
    "idPath": "3bbc1de7f744009b40f6e36f74",
    "serialNumber": "YGUB-2",
    "subject": "注册用户接口报错",
    "description": "",
    "formatType": null,
    "categoryId": "Bug",
    "parentId": "EMPTY_VALUE",
    "logicalStatus": "NORMAL",
    "statusStageId": "2",
    "gmtCreate": 1781580388000,
    "gmtModified": 1781590243000,
    "updateStatusAt": 1781590243000,
    "status": {
      "id": "100010",
      "name": "处理中",
      "nameEn": "In Progress",
      "displayName": "处理中"
    },
    "assignedTo": {
      "id": "67d7f978a175e4b5e3d7a38a",
      "name": "ant1025"
    },
    "creator": {
      "id": "67d7f978a175e4b5e3d7a38a",
      "name": "ant1025"
    },
    "modifier": {
      "id": "67d7f978a175e4b5e3d7a38a",
      "name": "ant1025"
    },
    "space": {
      "id": "d50de471daa361061fdb725d76",
      "name": "autopipeline-test"
    },
    "workitemType": {
      "id": "37da3a07df4d08aef2e3b393",
      "name": "缺陷"
    },
    "sprint": null,
    "labels": null,
    "versions": null,
    "trackers": null,
    "participants": null,
    "verifier": {
      "id": "67d7f978a175e4b5e3d7a38a",
      "name": "ant1025"
    },
    "customFieldValues": [
      {
        "fieldId": "seriousLevel",
        "fieldName": "严重程度",
        "fieldFormat": "list",
        "values": [
          {
            "identifier": "97eedaf6d6858f14a780bcd62b",
            "displayValue": "2-严重"
          }
        ]
      }
    ]
  }
]
```

**核心响应字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 工作项唯一标识（用于获取详情和更新） |
| `serialNumber` | string | 工作项编号（人类可读，如 `YGUB-2`） |
| `subject` | string | 工作项标题 |
| `description` | string | 描述（搜索时通常为空，除非 `includeDetails=true`） |
| `formatType` | string/null | 描述格式：`RICHTEXT` / `MARKDOWN` |
| `categoryId` | string | 类型：`Req`/`Task`/`Bug`/`Risk`/`Request`/`Topic` |
| `parentId` | string | 父工作项 ID（`EMPTY_VALUE` 表示无） |
| `status.id` | string | 状态 ID |
| `status.name` | string | 状态名称 |
| `assignedTo.id` | string | 负责人用户 ID |
| `assignedTo.name` | string | 负责人名称 |
| `space.id` | string | 所属项目 ID |
| `space.name` | string | 所属项目名称 |
| `workitemType.id` | string | 工作项子类型 ID |
| `workitemType.name` | string | 工作项子类型名称 |
| `customFieldValues` | array | 自定义字段值列表 |
| `gmtCreate` | integer | 创建时间（毫秒时间戳） |
| `gmtModified` | integer | 修改时间（毫秒时间戳） |

> **响应即数组**: 搜索响应直接返回 JSON 数组 `[{...}, ...]`，**不**需要通过 `.workitems` 或 `.data` 提取。空结果返回 `[]`。

> **分页**: 使用 `page`/`maxResults` 分页。若返回数组长度等于 `maxResults`，可能存在下一页，递增 `page` 继续请求。

---

## 获取工作项详情 (GetWorkitem)

根据工作项 `id` 获取单个工作项的完整信息（含描述内容）。

### 请求

```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `organizationId` | string | **是** | 路径参数，企业标识 |
| `workItemId` | string | **是** | 工作项 `id`（如 `3bbc1de7f744009b40f6e36f74`），不是 `serialNumber` |

### 响应

返回 **单个 JSON 对象**，HTTP 200。字段结构与搜索响应中的单个元素相同，但**额外包含以下字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `description` | string | 工作项描述（JSON 字符串，内含 `htmlValue` 和 `jsonMLValue`） |
| `formatType` | string | `RICHTEXT`（默认）或 `MARKDOWN` |
| `logicalStatus` | string | `NORMAL` / `DELETED` / `ARCHIVED` |
| `statusStageId` | string | 状态阶段 ID |
| `updateStatusAt` | integer | 状态更新时间（毫秒时间戳） |
| `idPath` | string | 工作项 ID 路径 |

示例（省略部分字段）：
```json
{
  "id": "3bbc1de7f744009b40f6e36f74",
  "serialNumber": "YGUB-2",
  "subject": "注册用户接口报错",
  "description": "{\"htmlValue\":\"<article>...\",\"jsonMLValue\":[...]}",
  "formatType": "RICHTEXT",
  "categoryId": "Bug",
  "parentId": "EMPTY_VALUE",
  "status": {"id": "100010", "name": "处理中"},
  "assignedTo": {"id": "67d7f978a175e4b5e3d7a38a", "name": "ant1025"},
  "space": {"id": "d50de471daa361061fdb725d76", "name": "autopipeline-test"},
  "workitemType": {"id": "37da3a07df4d08aef2e3b393", "name": "缺陷"},
  "customFieldValues": [
    {"fieldId": "seriousLevel", "fieldName": "严重程度", "values": [{"displayValue": "2-严重"}]}
  ],
  "gmtCreate": 1781580388000,
  "gmtModified": 1781590243000
}
```

> **响应是对象**: 详情接口直接返回工作项 JSON 对象，**不**通过 `.workitem` 提取。与 Codeup API（有 `.data` 包装）格式不同。

---

## 更新工作项 (UpdateWorkitem)

更新工作项的标题、描述、状态、负责人、优先级等信息。

### 请求

```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"subject": "更新后的标题"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

### Body 字段

Body 为 JSON 对象，**只需传入需要更新的字段**（部分更新）：

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `subject` | string | 工作项标题 | `"修复登录页面样式"` |
| `description` | string | 描述内容 | `"<p>新描述</p>"` 或 Markdown |
| `formatType` | string | 描述格式，配合 `description` 使用 | `RICHTEXT`（默认）/ `MARKDOWN` |
| `status` | string | 状态 ID | `"100011"`（已完成） |
| `assignedTo` | string | 负责人用户 ID | `"67d7f978a175e4b5e3d7a38a"` |
| `priority` | string | 优先级 ID | `"1"` |
| `labels` | string[] | 标签 ID 列表 | `["tagId1", "tagId2"]` |
| `sprint` | string | 迭代 ID | `"sprintId..."` |
| `trackers` | string[] | 抄送人用户 ID 列表 | `["userId1"]` |
| `verifier` | string | 验证人用户 ID | `"userId"` |
| `participants` | string[] | 参与人用户 ID 列表 | `["userId1"]` |
| `versions` | string[] | 版本 ID 列表 | `["version001"]` |
| `customFieldValues` | object | 自定义字段值 | `{"fieldId": "value"}` 或 `{"fieldId": ["v1","v2"]}` |
| `workitemTypeId` | string | 工作项子类型 ID | `"37da3a07df4d08aef2e3b393"` |

> **获取可选值**: 状态 ID、自定义字段 ID 等可在云效页面中抓包获取，或通过「获取工作项类型字段配置」接口查询。

### 响应

更新成功返回 **HTTP 204 No Content**（无响应体）。可用 GET 验证更新结果。

```bash
# 验证更新
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}" \
  | jq '{id, subject, status: .status.name, gmtModified}'
```

### 常用更新示例

**更新标题**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"subject": "feat: 新增用户管理模块"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID"
```

**更新状态**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "100011"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID"
```

**更新负责人和优先级**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"assignedTo": "67d7f978a175e4b5e3d7a38a", "priority": "1"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID"
```

**更新自定义字段**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "customFieldValues": {
      "seriousLevel": "97eedaf6d6858f14a780bcd62b",
      "priority": "90487dc3c566c7b0ac831c0f16"
    }
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID"
```

---

## 常见工作流

### 工作流 1：搜索我的待处理缺陷

```bash
# 前提：已知 ORG_ID, SPACE_ID, USER_ID
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"spaceId\":\"$SPACE_ID\",\"category\":\"Bug\",\"assignedTo\":\"$USER_ID\",\"status\":\"28\",\"orderBy\":\"gmtCreate\",\"sort\":\"desc\"}" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems:search" \
  | jq '.[] | {id, serialNumber, subject, status: .status.name, gmtCreate}'
```

### 工作流 2：查看工作项详情并更新

```bash
# 1. 获取详情（含描述）
WI_ID="3bbc1de7f744009b40f6e36f74"
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID" \
  | jq '{id, subject, status: .status.name, assignedTo: .assignedTo.name, description}'

# 2. 更新状态
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "100010"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID"

# 3. 验证（PUT 返回 204，需 GET 验证）
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WI_ID" \
  | jq '{id, subject, status: .status.name}'
```

### 工作流 3：搜索并批量更新工作项

```bash
# 1. 搜索特定条件下的工作项
IDS=$(curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"spaceId\":\"$SPACE_ID\",\"category\":\"Bug\",\"status\":\"28\",\"maxResults\":50}" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems:search" \
  | jq -r '.[].id')

# 2. 逐个更新（示例：全部改为处理中）
for id in $IDS; do
  echo "更新工作项: $id"
  curl -s -X PUT \
    -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"status": "100010"}' \
    "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$id" \
    -o /dev/null -w "HTTP %{http_code}\n"
done
```

### 工作流 4：搜索项目最近更新的工作项

```bash
SEVEN_DAYS_AGO=$(date -v-7d +%Y-%m-%d 2>/dev/null || date -d '7 days ago' +%Y-%m-%d)

curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"spaceId\":\"$SPACE_ID\",\"category\":\"Req,Task,Bug\",\"updatedAfter\":\"$SEVEN_DAYS_AGO\",\"orderBy\":\"gmtModified\",\"sort\":\"desc\",\"maxResults\":20}" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems:search" \
  | jq '.[] | {id, subject, status: .status.name, assignedTo: .assignedTo.name, gmtModified}'
```

### 工作流 5：创建测试工作项

```bash
# 需要: spaceId, 用户ID, workitemTypeId（从 workitemTypes 接口获取）
curl -s -X POST \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "spaceId": "'$SPACE_ID'",
    "categoryId": "Bug",
    "subject": "API 测试工作项",
    "assignedTo": "'$USER_ID'",
    "workitemTypeId": "37da3a07df4d08aef2e3b393"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems" \
  | jq '{id, subject, serialNumber}'
```

---

## 错误处理速查

| HTTP 状态 | errorCode | 含义 | 处理 |
|-----------|-----------|------|------|
| 200 | — | GET/搜索成功 | — |
| 204 | — | PUT 更新成功 | 响应为空，需 GET 验证 |
| 400 | `InvaildData.Failed` | 参数缺失/错误 | 检查必填参数名是否正确 |
| 400 | `Openapi.RequestError` | 请求数据错误 | 检查 JSON 格式和参数类型 |
| 403 | `Operate.NoPermission` | 无权限 | 检查用户是否为项目成员 |
| 404 | `InvalidWorkitem.NotFound` | 工作项不存在 | 检查 `id` 是否正确 |
| 404 | `NotFound` | 路径不存在 | 检查 URL 路径格式 |

**通用错误检查模板**：
```bash
RESPONSE=$(curl -s -w "\n%{http_code}" -H "x-yunxiao-token: $YUNXIAO_TOKEN" "$URL")
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | sed '$d')

if [ "$HTTP_CODE" != "200" ] && [ "$HTTP_CODE" != "204" ]; then
  echo "HTTP $HTTP_CODE 错误" >&2
  echo "$BODY" | jq '{errorCode, errorMessage}' 2>/dev/null >&2
  exit 1
fi
```

---

## 关键注意事项

1. **搜索用 POST + 冒号路径**: 搜索端点路径是 `POST /workitems:search`（`:search` 用**冒号**，不是 `/search` 斜杠）。Body 为 JSON 对象。

2. **搜索响应即数组**: 搜索接口直接返回 `[{...}, ...]`，**没有**外层 `{success, workitems}` 包装。空结果返回 `[]`。用 `jq '.[]'` 遍历。

3. **详情响应即对象**: GET 详情直接返回 `{...}` 单个对象，**没有** `.workitem` 或 `.data` 包装层。

4. **PUT 返回 204**: 更新接口成功返回 `HTTP 204 No Content`（无响应体），非 200。需用 GET 验证更新结果。

5. **字段名**: 工作项唯一标识是 `id`（不是 `identifier`），类型是 `categoryId`（不是 `categoryIdentifier`），项目是 `space.id`（不是 `spaceIdentifier`）。

6. **嵌套对象**: `status`、`assignedTo`、`creator`、`modifier`、`space`、`workitemType` 都是嵌套对象，含 `id` 和 `name`，不是扁平字符串。

7. **更新是部分更新**: PUT 只传需要修改的字段即可，不需要全量。Body 是 field-value 映射。

8. **用户 ID 格式**: 人员字段传用户 ID（如 `67d7f978a175e4b5e3d7a38a`），不是用户名。可通过 `GET /oapi/v1/platform/user` 获取当前用户。

9. **时间参数**: 搜索接口的时间筛选参数使用 `YYYY-MM-DD` 格式，返回的时间字段是毫秒时间戳。

10. **中心版 vs Region 版**: 中心版路径含 `organizations/{orgId}`，Region 版不含。如果 API Base URL 不是 `openapi-rdc.aliyuncs.com`，需使用 Region 版路径。

11. **描述格式**: `description` 字段是 JSON 字符串（非普通文本），内含 `htmlValue`（HTML）和 `jsonMLValue`（结构化数组）。编辑时为 RICHTEXT 格式须传 HTML，MARKDOWN 格式传原始文本。

12. **工作项类型**: 创建前需确认项目启用了对应类型。可通过 `GET /workitemTypes` 获取所有类型，但项目可能只启用了其中一部分（通过项目的字段配置接口可查）。
