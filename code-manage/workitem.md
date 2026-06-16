---
name: yunxiao-workitem
description: 云效 Projex 工作项管理技能。提供工作项的搜索、查看和更新能力，支持需求（Req）、任务（Task）、缺陷（Bug）等类型。当用户需要操作云效项目协作中的工作项时使用，包括：(1) 按条件搜索/筛选工作项 (2) 获取工作项详情 (3) 更新工作项信息（标题、描述、状态、负责人、优先级等）。触发词：工作项、Projex、需求、任务、缺陷、Bug、workitem、更新工作项、搜索工作项。
---

# 云效 Projex 工作项管理

通过云效 OpenAPI 操作项目协作（Projex）中的工作项，支持需求、任务、缺陷等类型。

## 前置条件

环境变量 `YUNXIAO_TOKEN`（个人访问令牌）。
获取方式：云效控制台 → 个人设置 → 个人访问令牌。

## API 基础

> **注意**: Projex 工作项 API 使用与 Codeup 代码管理 **不同的 Base URL**。

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

# PUT（带 JSON body）
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/..."
```

**错误响应格式**:
```json
{
  "errorCode": "Openapi.RequestError",
  "errorMessage": "错误描述",
  "requestId": "...",
  "success": false
}
```

---

## 快速开始：获取组织 ID 和项目 ID

### 1. 获取组织 ID

操作工作项需要 `organizationId`，可从云效链接获取：
`https://openapi-rdc.aliyuncs.com/organization/{organizationId}`

或者通过 API 列出所有组织：
```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/platform/organizations" | jq '.[] | {id, name}'
```

### 2. 获取项目 ID（spaceId）

工作项所属的项目（空间）ID，可从云效项目链接获取：
`https://openapi-rdc.aliyuncs.com/projex/project/{spaceId}`

或在项目设置 → 基本信息中查看。

### 3. 获取工作项类型标识

工作项类型标识（`category`），常用值：

| category | 含义 |
|----------|------|
| `Req` | 需求 |
| `Task` | 任务 |
| `Bug` | 缺陷 |
| `Risk` | 风险 |
| `Request` | 原始诉求 |
| `Topic` | 主题 |

---

## 搜索工作项 (SearchWorkitems)

按条件筛选和搜索工作项列表。

### 请求

```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/search?spaceId={spaceId}&category={category}"
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `organizationId` | string | **是** | 路径参数，企业标识 |
| `spaceId` | string | **是** | 项目 ID |
| `category` | string | **是** | 工作项类型，多个逗号分隔：`Req,Task,Bug` |
| `spaceType` | string | 否 | 空间类型，默认 `Project`，可选 `Program`（项目集） |
| `status` | string | 否 | 状态 ID |
| `priority` | string | 否 | 优先级 ID，多个逗号分隔 |
| `assignedTo` | string | 否 | 负责人用户 ID，`self` 表示当前用户 |
| `sprint` | string | 否 | 迭代/冲刺 ID |
| `subject` | string | 否 | 标题搜索关键词 |
| `tag` | string | 否 | 标签 ID |
| `page` | integer | 否 | 页码，默认 1 |
| `perPage` | integer | 否 | 每页数量，0-200，默认 20 |
| `orderBy` | string | 否 | 排序字段 |
| `sort` | string | 否 | 排序方向：`desc`（默认）/ `asc` |
| `createdAfter` | string | 否 | 创建时间起始，格式 `YYYY-MM-DD` |
| `createdBefore` | string | 否 | 创建时间截止，格式 `YYYY-MM-DD` |
| `updatedAfter` | string | 否 | 更新时间起始 |
| `updatedBefore` | string | 否 | 更新时间截止 |
| `finishTimeAfter` | string | 否 | 完成时间起始 |
| `finishTimeBefore` | string | 否 | 完成时间截止 |
| `updateStatusAtAfter` | string | 否 | 状态更新时间起始 |
| `updateStatusAtBefore` | string | 否 | 状态更新时间截止 |
| `includeDetails` | boolean | 否 | 是否返回描述详情，默认 `false`（设为 `true` 可减少后续 GetWorkitem 调用） |
| `advancedConditions` | string | 否 | 高级过滤条件（JSON 格式），用法参考下方说明 |

### conditions / advancedConditions 高级过滤

当基础查询参数不够用时，可通过 `advancedConditions` 传入 JSON 结构化过滤条件：

```json
{
  "conditionGroups": [
    {
      "conditions": [
        {
          "fieldIdentifier": "gmtModified",
          "operator": "BETWEEN",
          "value": [1640850318000, 1640850328000],
          "className": "date",
          "format": "timestamp"
        },
        {
          "fieldIdentifier": "status",
          "operator": "EQUALS",
          "value": ["待处理"]
        }
      ],
      "operator": "AND"
    }
  ]
}
```

常用 `fieldIdentifier`：`gmtModified`（修改时间）、`status`（状态）、`assignedTo`（负责人）、`priority`（优先级）、`tag`（标签）。

> **提示**: 获取 conditions 的最简单方法：在云效页面上手动设置筛选条件 → 浏览器开发者工具 → Network → 找到 `workitem/list` 请求 → 复制 conditions 参数值。

### 响应示例

```json
{
  "success": true,
  "requestId": "ASSDS-ASSASX-XSAXSA-XSAXSAXS",
  "totalCount": 42,
  "nextToken": "",
  "maxResults": 20,
  "workitems": [
    {
      "identifier": "e8bxxxxxxxxxxxxxxxx23",
      "subject": "用户登录功能",
      "document": "实现用户登录页面和接口",
      "documentFormat": "RICHTEXT",
      "assignedTo": "19xx7043xxxxxxx914",
      "status": "处理中",
      "statusIdentifier": 100010,
      "statusStageIdentifier": 1,
      "spaceIdentifier": "e8b26xxxxx6e76aa20xxxxx23",
      "spaceName": "需求项目",
      "spaceType": "Project",
      "logicalStatus": "NORMAL",
      "categoryIdentifier": "Req",
      "parentIdentifier": "e8bxxxxxxxxxxxxxxxx24",
      "workitemTypeIdentifier": "9uxxxxxxre573f5xxxxxx0",
      "serialNumber": "PROJ-1",
      "priority": 2,
      "gmtCreate": 1640850318000,
      "gmtModified": 1640850318000,
      "creator": "19xx7043xxxxxxx914",
      "modifier": "19xx7043xxxxxxx914",
      "sprintIdentifier": "75528f17703e92e5a568...",
      "finishTime": 1699366778000,
      "tag": [{"identifier": "tagId1", "name": "高优"}]
    }
  ]
}
```

**响应字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `identifier` | string | 工作项唯一标识（UUID，用于获取详情和更新） |
| `subject` | string | 工作项标题 |
| `document` | string | 工作项描述内容 |
| `documentFormat` | string | 描述格式：`RICHTEXT` / `MARKDOWN` |
| `assignedTo` | string | 负责人用户 ID（阿里云 PK） |
| `status` | string | 状态名称 |
| `statusIdentifier` | integer | 状态 ID |
| `spaceIdentifier` | string | 所属项目 ID |
| `spaceName` | string | 所属项目名称 |
| `categoryIdentifier` | string | 工作项类型：`Req`/`Task`/`Bug` |
| `parentIdentifier` | string | 父工作项 ID |
| `serialNumber` | string | 工作项编号（人类可读） |
| `sprintIdentifier` | string | 所属迭代 ID |
| `priority` | integer | 优先级 |
| `gmtCreate` | integer | 创建时间（毫秒时间戳） |
| `gmtModified` | integer | 修改时间（毫秒时间戳） |
| `creator` | string | 创建人用户 ID |
| `modifier` | string | 修改人用户 ID |
| `finishTime` | integer | 完成时间（毫秒时间戳） |

> **分页**: 响应中的 `totalCount` 为总数，当 `workitems` 数组长度等于 `maxResults` 且 `nextToken` 非空时，表示还有下一页。下一页请求需传入 `nextToken` 的值。

---

## 获取工作项详情 (GetWorkitem)

根据工作项 ID 获取单个工作项的完整信息。

### 请求

```bash
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `organizationId` | string | **是** | 路径参数，企业标识 |
| `workItemId` | string | **是** | 工作项 ID（`identifier`，UUID 格式），可从搜索接口或其他途径获取 |

> `workItemId` 是工作项的 UUID 标识（如 `e8bxxxxxxxxxxxxxxxx23`），不是 `serialNumber`（如 `PROJ-1`）。

### 响应示例

```json
{
  "success": true,
  "requestId": "ASSDS-ASSASX-XSAXSA-XSAXSAXS",
  "workitem": {
    "identifier": "e8bxxxxxxxxxxxxxxxx23",
    "subject": "用户登录功能",
    "document": "<p>实现用户登录页面和接口</p>",
    "documentFormat": "RICHTEXT",
    "assignedTo": "19xx7043xxxxxxx914",
    "status": "处理中",
    "statusIdentifier": 100010,
    "statusStageIdentifier": 1,
    "spaceIdentifier": "e8b26xxxxx6e76aa20xxxxx23",
    "spaceName": "需求项目",
    "spaceType": "Project",
    "logicalStatus": "NORMAL",
    "categoryIdentifier": "Req",
    "parentIdentifier": "e8bxxxxxxxxxxxxxxxx24",
    "workitemTypeIdentifier": "9uxxxxxxre573f5xxxxxx0",
    "updateStatusAt": 1640850328000,
    "serialNumber": "PROJ-1",
    "gmtCreate": 1640850318000,
    "gmtModified": 1640850318000,
    "creator": "19xx7043xxxxxxx914",
    "modifier": "19xx7043xxxxxxx914",
    "sprint": [
      {
        "identifier": "75528f17703e92e5a568...",
        "name": "Sprint 1"
      }
    ],
    "tag": [
      {
        "identifier": "tagId1",
        "name": "高优"
      }
    ],
    "customFields": [
      {
        "fieldIdentifier": "custom_field_001",
        "fieldName": "需求来源",
        "value": "产品",
        "fieldType": "select",
        "displayValue": "产品"
      }
    ],
    "versions": [
      {
        "identifier": "version001",
        "name": "v1.0.0"
      }
    ]
  }
}
```

**响应字段说明**（与搜索接口相同的字段略）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `sprint` | array | 关联的迭代列表，含 `identifier` 和 `name` |
| `tag` | array | 关联的标签列表，含 `identifier` 和 `name` |
| `customFields` | array | 自定义字段列表，含 `fieldIdentifier`、`fieldName`、`value`、`displayValue` |
| `versions` | array | 关联的版本列表，含 `identifier` 和 `name` |

---

## 更新工作项 (UpdateWorkitem)

更新工作项的标题、描述、状态、负责人、优先级、迭代、标签等信息。

### 请求

```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "更新后的标题",
    "assignedTo": "userId"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `organizationId` | string | **是** | 路径参数，企业标识 |
| `workItemId` | string | **是** | 路径参数，工作项 ID（UUID `identifier`） |

### Body 字段

Body 为 JSON 对象，只需传入需要更新的字段，**不需要全量传入**。

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `subject` | string | 更新标题 | `"修复登录页面样式"` |
| `description` | string | 更新描述内容 | `"<p>新的描述</p>"` |
| `formatType` | string | 描述格式，配合 `description` 使用 | `RICHTEXT`（默认）/ `MARKDOWN` |
| `status` | string | 更新状态，传入状态 ID | `"100011"` |
| `assignedTo` | string | 更新负责人，传入用户 ID | `"19xx7043xxxxxxx914"` |
| `priority` | string | 更新优先级 ID | `"1"` |
| `labels` | string[] | 更新标签 ID 列表 | `["tagId1", "tagId2"]` |
| `sprint` | string | 更新关联迭代 ID | `"75528f17703e92e5a568..."` |
| `trackers` | string[] | 更新抄送人用户 ID 列表 | `["userId1", "userId2"]` |
| `verifier` | string | 更新验证人用户 ID | `"userId"` |
| `participants` | string[] | 更新参与人用户 ID 列表 | `["userId1"]` |
| `versions` | string[] | 更新关联版本 ID 列表 | `["version001"]` |
| `customFieldValues` | object | 更新自定义字段值 | `{"fieldId": "value"}` 或 `{"fieldId": ["v1","v2"]}` |
| `operatorId` | string | 指定操作人用户 ID（使用个人令牌时无效） | — |

> **获取可选值**: 状态 ID、优先级 ID、自定义字段 ID 等可通过 `ListWorkItemAllFields` 或「获取工作项类型字段配置」接口查询。也可以在云效页面上找到对应的字段标识。

### 常用更新示例

**更新标题和描述**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "feat: 新增用户管理模块",
    "description": "## 需求描述\n实现用户增删改查功能",
    "formatType": "MARKDOWN"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

**更新状态**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "100011"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

**更新负责人和优先级**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "assignedTo": "19xx7043xxxxxxx914",
    "priority": "1"
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

**更新自定义字段**：
```bash
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "customFieldValues": {
      "custom_field_001": "技术部",
      "custom_field_002": ["选项A", "选项B"]
    }
  }' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/{organizationId}/workitems/{workItemId}"
```

### 响应示例

```json
{
  "success": true,
  "requestId": "ASSDS-ASSASX-XSAXSA-XSAXSAXS",
  "workitem": {
    "identifier": "e8bxxxxxxxxxxxxxxxx23",
    "subject": "更新后的标题",
    "status": "已完成",
    "assignedTo": "19xx7043xxxxxxx914",
    "gmtModified": 1640850428000,
    "...": "..."
  }
}
```

成功响应中 `workitem` 对象包含更新后的完整工作项信息。

---

## 常见工作流

### 工作流 1：搜索我负责的需求

```bash
# 前提：已知 ORG_ID 和 SPACE_ID
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/search?spaceId=$SPACE_ID&category=Req&assignedTo=self&status=100010" \
  | jq '.workitems[] | {identifier, subject, status, serialNumber, gmtModified}'
```

### 工作流 2：查看工作项详情并更新

```bash
# 1. 获取工作项详情
WORKITEM_ID="e8bxxxxxxxxxxxxxxxx23"
curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WORKITEM_ID" \
  | jq '.workitem | {identifier, subject, status, assignedTo, description: .document}'

# 2. 更新工作项状态
curl -s -X PUT \
  -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "100011"}' \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$WORKITEM_ID" \
  | jq '{success, workitem: {identifier, subject, status}}'
```

### 工作流 3：搜索并批量更新工作项

```bash
# 1. 搜索特定条件下的工作项
WORKITEMS=$(curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/search?spaceId=$SPACE_ID&category=Bug&status=28&perPage=50" \
  | jq -r '.workitems[] | .identifier')

# 2. 逐个更新（示例：全部改为处理中状态）
for id in $WORKITEMS; do
  echo "更新工作项: $id"
  curl -s -X PUT \
    -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"status": "100010"}' \
    "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/$id" \
    | jq '{success, workitem: {identifier: .workitem.identifier, subject: .workitem.subject, status: .workitem.status}}'
done
```

### 工作流 4：搜索项目最近更新的工作项

```bash
# 搜索最近 7 天内更新的需求
SEVEN_DAYS_AGO=$(date -v-7d +%Y-%m-%d 2>/dev/null || date -d '7 days ago' +%Y-%m-%d)

curl -s -H "x-yunxiao-token: $YUNXIAO_TOKEN" \
  "https://openapi-rdc.aliyuncs.com/oapi/v1/projex/organizations/$ORG_ID/workitems/search?spaceId=$SPACE_ID&category=Req&updatedAfter=$SEVEN_DAYS_AGO&orderBy=updated&sort=desc&perPage=20" \
  | jq '.workitems[] | {identifier, subject, status, assignedTo, gmtModified}'
```

---

## 错误处理速查

| HTTP 状态 | errorCode | 含义 | 处理 |
|-----------|-----------|------|------|
| 200 | `success: true` | 成功 | — |
| 400 | `Openapi.RequestError` | 参数缺失或格式错误 | 检查必填参数（organizationId, spaceId, category） |
| 403 | `Operate.NoPermission` | 无权限 | 检查用户是否为项目成员，token 是否有效 |
| 404 | `NotFound` | 工作项不存在 | 检查 workItemId 是否正确 |

**通用错误检查模板**：
```bash
RESPONSE=$(curl -s -w "\n%{http_code}" -H "x-yunxiao-token: $YUNXIAO_TOKEN" "$URL")
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | sed '$d')

if [ "$HTTP_CODE" != "200" ]; then
  echo "HTTP $HTTP_CODE 错误" >&2
  echo "$BODY" | jq '{errorCode, errorMessage}' >&2
  exit 1
fi

# 检查 success 字段
SUCCESS=$(echo "$BODY" | jq -r '.success')
if [ "$SUCCESS" != "true" ]; then
  echo "API 错误: $(echo "$BODY" | jq -r '.errorCode + ": " + .errorMessage')" >&2
  exit 1
fi
```

---

## 关键注意事项

1. **Base URL 不同**: Projex 工作项 API 使用 `openapi-rdc.aliyuncs.com`，而 Codeup 代码管理 API 使用 `openapi-rdc.aliyuncs.com`，两者**不同**。

2. **工作项标识**: 接口使用 `identifier`（UUID，如 `e8bxxxxxxxxxxxxxxxx23`），不是 `serialNumber`（如 `PROJ-1`）。

3. **搜索必填参数**: `spaceId`（项目 ID）和 `category`（工作项类型）是搜索接口的必填参数，至少需要指定一个类型。

4. **更新是部分更新**: `UpdateWorkitem` 只需要传入需要修改的字段，不需要传所有字段。Body 是一个 field-value 映射。

5. **用户 ID 格式**: 人员类型字段（assignedTo、trackers、verifier 等）需要传阿里云 AccountID（PK），不是用户名或邮箱。可通过云效页面地址栏获取。

6. **状态和优先级**: 更新时需使用 ID 而非名称。状态 ID 可通过 `ListWorkItemAllFields` 或云效页面 Network 抓包获取。

7. **时间参数格式**: 搜索接口的时间参数使用 `YYYY-MM-DD` 格式（如 `2024-01-01`），不同于返回的时间戳格式（毫秒）。

8. **分页方式**: 搜索接口使用 `page`/`perPage` 分页，响应中包含 `totalCount`。也支持 `nextToken` 游标分页。推荐首次请求时指定 `perPage`。

9. **includeDetails**: 搜索时设置 `includeDetails=true` 可一次性获取描述等详细信息，避免对每个工作项再调用 GetWorkitem。适合需要完整信息的场景。

10. **字段 ID 获取**: `customFieldValues` 和 `status` 等字段的可选值 ID，需要通过「获取工作项类型字段配置」接口或云效页面 Network 抓包获取。
