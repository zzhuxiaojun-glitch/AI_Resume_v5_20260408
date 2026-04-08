# HR Backend API 前端调用接口速查表

> Base URL: `https://hrapi.keiten-jp.com`  
> WebSocket: `wss://hrapi.keiten-jp.com/ws`  
> 前端函数封装：`src/lib/api.ts`

---

## Positions（职位）

| 方法 | 路径 | 前端函数 | 说明 |
|------|------|---------|------|
| GET | `/api/positions` | `getPositions()` | 获取所有职位列表（按创建时间升序） |
| GET | `/api/positions/:id` | `getPosition(id)` | 获取单个职位详情 |
| POST | `/api/positions` | `createPosition(data)` | 创建新职位 |
| PATCH | `/api/positions/:id` | `updatePosition(id, data)` | 部分更新职位 |
| DELETE | `/api/positions/:id` | `deletePosition(id)` | 删除职位 → `{ deleted: true }` |

### 创建/更新职位字段

```ts
{
  title: string           // 职位名称（必填）
  department?: string     // 部门
  description?: string    // 职位描述
  skillConfig?: {
    must: string[]        // 必须技能（缺失严重扣分）
    nice: string[]        // 加分技能
    reject: string[]      // 扣分关键词
  }
  status?: 'open' | 'closed' | 'draft'   // 默认 open
  locale?: 'zh' | 'ja'                   // AI评价语言，默认 zh
}
```

### Position 返回结构

```ts
{
  id: string
  title: string
  department: string | null
  description: string | null
  skillConfig: { must: string[]; nice: string[]; reject: string[] }
  status: 'open' | 'closed' | 'draft'
  locale: 'zh' | 'ja'
  createdAt: string   // ISO 时间戳
  updatedAt: string
}
```

---

## Candidates（候选人）

| 方法 | 路径 | 前端函数 | 说明 |
|------|------|---------|------|
| GET | `/api/candidates` | `getCandidates(filters?)` | 获取列表（含评分，按总分降序） |
| GET | `/api/candidates/:id` | `getCandidate(id)` | 获取详情 + 全部评分记录 |
| PATCH | `/api/candidates/:id` | `updateCandidate(id, data)` | 更新状态/备注/联系方式 |

### 列表筛选参数（均可选）

```ts
{
  positionId?: string
  status?: 'new' | 'screening' | 'shortlisted' | 'interviewed' | 'rejected' | 'hired'
  grade?: 'A' | 'B' | 'C' | 'D' | 'F'
  universityTier?: 'S' | 'A' | 'B' | 'C' | 'D'
  jlptLevel?: 'N1' | 'N2' | 'N3' | 'N4' | 'N5'   // N2 = N1+N2，以上皆同
}
```

### 候选人状态流转

```
new → screening → shortlisted → interviewed → hired
                                           ↘ rejected
```

### CandidateListItem（列表项，含 LEFT JOIN 评分）

```ts
{
  id, positionId, name, email, phone
  education, university
  universityTier: 'S'|'A'|'B'|'C'|'D' | null
  jlptLevel: 'N1'~'N5' | null
  skills: string[] | null
  status: CandidateStatus
  notes: string | null
  createdAt: string
  // 来自 scores 表（LEFT JOIN，可能为 null）
  totalScore: number | null
  educationScore: number | null
  grade: 'A'|'B'|'C'|'D'|'F' | null
}
```

### CandidateDetail（详情，含完整评分数组）

```ts
{
  // 同上所有字段，加上：
  updatedAt: string
  scores: Score[]   // 全部 AI 评分记录
}
```

### Score（评分记录）

```ts
{
  id, candidateId, positionId
  totalScore: number        // 综合总分 0-100
  mustScore: number         // 必须技能匹配分
  niceScore: number         // 加分项匹配分
  rejectPenalty: number     // 扣分项惩罚值
  educationScore: number    // 学历/院校分
  grade: 'A'|'B'|'C'|'D'|'F'
  matchedSkills: string[]
  missingSkills: string[]
  explanation: string | null  // AI 评价（中文或日语）
  createdAt: string
}
```

**评级标准：** A(≥80) / B(≥65) / C(≥50) / D(≥35) / F(<35)

---

## Resumes（简历上传）

| 方法 | 路径 | 前端函数 | 说明 |
|------|------|---------|------|
| POST | `/api/resumes/upload` | `uploadResume(file, positionId, name?)` | 上传简历 → 自动解析 + AI评分 |

**Content-Type：** `multipart/form-data`

```ts
// 请求字段
file: File          // PDF 或 DOCX
positionId: string  // 必填，目标职位 ID
name?: string       // 候选人姓名（可选）

// 返回
{
  candidate: CandidateDetail
  score: Score
  resumeText: string   // 提取的文本前 500 字
}
```

---

## Email（邮件）

| 方法 | 路径 | 前端函数 | 说明 |
|------|------|---------|------|
| POST | `/api/email/poll` | `pollInbox(positionId)` | 触发邮箱轮询，自动解析简历并评分 |
| GET | `/api/email/stats` | `getEmailStats()` | 获取邮件处理统计 |

```ts
// pollInbox 返回
{ candidateIds: string[]; count: number }

// getEmailStats 返回
{
  emails: {
    total: number
    byClassification: Record<string, number>  // resume / not_resume / uncertain
    byStatus: Record<string, number>          // skipped / fetched / parsed / scored / error
    breakdown: Array<{ classification, status, hasAttachment, count }>
  }
  candidates: { total, withScore, withoutScore }
  resumes: { total, withFile, withoutFile }
}
```

---

## Universities（院校）

| 方法 | 路径 | 前端函数 | 说明 |
|------|------|---------|------|
| GET | `/api/universities` | `getUniversities(filters?)` | 获取院校列表（支持 `country` / `tier` 筛选）|
| GET | `/api/universities/stats` | `getUniversityStats()` | 统计（总数/按档位/按国家）|
| GET | `/api/universities/lookup?name=清华` | `lookupUniversity(name)` | 模糊搜索院校名 |

**院校档位：** S（顶尖/985/QS前50）/ A / B / C / D

---

## WebSocket 实时事件

连接 `wss://hrapi.keiten-jp.com/ws` 后自动订阅所有事件。

```ts
// 前端使用方式
const ws = useHRWebSocket(WS_URL)
ws.on('candidate:new', (e) => { ... })
ws.on('candidate:scored', (e) => { ... })
ws.on('inbox:summary', (e) => { ... })
ws.ping()  // 发送 ping 保持连接
```

### 事件类型

| 事件 | 触发时机 | 主要字段 |
|------|---------|---------|
| `heartbeat` | 连接时 & 每次 ping 回复 | `timestamp`, `connectedClients` |
| `candidate:new` | 新候选人入库（邮件/上传） | `candidateId`, `name`, `positionId`, `positionTitle`, `source` |
| `candidate:scored` | AI评分完成 | `candidateId`, `totalScore`, `grade`, `matchedSkills`, `educationScore` |
| `inbox:summary` | 邮件轮询批次完成 | `totalProcessed`, `gradeDistribution`, `topCandidates` |
| `error` | 消息格式错误 | `message` |

---

## Health

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/health` | 健康检查 → `{ status: "ok", timestamp }` |
