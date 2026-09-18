# AI功能实现详解

## 📋 概述

智研匹配系统集成了多种AI功能，提升用户体验和系统智能化水平。本文档详细介绍AI功能的实现原理、调用方式和技术架构。

---

## 🏗️ AI架构设计

### 1.1 系统架构

```
前端调用 (React + tRPC)
    ↓ HTTP请求
tRPC路由层 (server/routes/)
    ↓ 业务逻辑处理
AI服务层 (server/ai-match.ts)
    ↓ LLM调用
外部AI服务 (LLM Council + OpenRouter)
    ↓ 本地降级
本地AI规则 (server/ai-match-local.ts)
```

### 1.2 AI服务分层

#### 1.2.1 生产环境AI服务
- **LLM Council**: 本地FastAPI服务，管理AI模型调用
- **OpenRouter**: 外部AI API服务，支持多种模型
- **自动降级**: 当外部服务不可用时，自动切换到本地规则

#### 1.2.2 本地备用AI服务
- **规则引擎**: 基于关键词匹配和相似度计算
- **本地算法**: 无需外部依赖，保障服务可用性

---

## 🤖 AI功能详解

### 2.1 AI智能匹配

#### 2.1.1 功能概述
根据学生的简历信息和项目要求，计算匹配度并提供详细解释。

#### 2.1.2 调用方式

**前端调用**:
```typescript
// 在项目详情页调用
const { data: matchResult } = trpc.ai.calculateMatch.useQuery({
  studentId: user.id,
  projectId: projectId
});
```

**API接口**:
```typescript
// server/routes/ai.ts
calculateMatch: aiProcedure
  .input(z.object({
    studentId: z.number(),
    projectId: z.number()
  }))
  .query(async ({ input }) => {
    return await aiService.calculateMatch(input.studentId, input.projectId);
  })
```

#### 2.1.3 匹配算法

**匹配维度**:
1. **技能匹配**: 学生技能标签与项目要求技能的交集
2. **兴趣匹配**: 研究兴趣关键词相似度
3. **经验匹配**: 项目经验相关性分析
4. **基础匹配**: 专业、年级等基础信息匹配

**评分公式**:
```typescript
const matchScore = (
  skillMatch * 0.4 +
  interestMatch * 0.3 +
  experienceMatch * 0.2 +
  basicMatch * 0.1
) * 100;
```

#### 2.1.4 返回数据结构

```typescript
interface MatchResult {
  score: number;        // 匹配分数 (0-100)
  analysis: {
    skillMatch: number;     // 技能匹配度
    interestMatch: number;  // 兴趣匹配度
    experienceMatch: number; // 经验匹配度
    basicMatch: number;     // 基础匹配度
    reasoning: string;      // 详细解释
  };
}
```

### 2.2 简历自动解析

#### 2.2.1 功能概述
上传PDF简历后，系统自动解析出结构化信息，包括个人信息、技能、经验等。

#### 2.2.2 调用流程

**文件上传**:
```typescript
// 学生档案管理页
const uploadMutation = trpc.student.profile.uploadResume.useMutation({
  onSuccess: (data) => {
    // 自动触发解析
    parseMutation.mutate({ resumeUrl: data.url });
  }
});
```

**AI解析**:
```typescript
const parseMutation = trpc.ai.parseResume.useMutation({
  onSuccess: (parsedData) => {
    // 更新学生档案
    updateProfile.mutate(parsedData);
  }
});
```

#### 2.2.3 解析字段

| 字段 | 类型 | 说明 | AI提取规则 |
|------|------|------|-----------|
| `studentId` | string | 学号 | 正则匹配学号格式 |
| `name` | string | 姓名 | 简历头部信息 |
| `major` | string | 专业 | 教育背景信息 |
| `grade` | string | 年级 | 从入学时间计算 |
| `gpa` | string | 绩点 | 数字识别 |
| `skills` | string[] | 技能标签 | 技术栈关键词提取 |
| `experience` | string | 项目经验 | 经历描述段落 |

#### 2.2.4 解析质量保证

**多重验证**:
1. **格式验证**: 检查提取字段的格式正确性
2. **逻辑验证**: 验证数据间的逻辑关系
3. **人工审核**: 提供编辑界面供用户修改

### 2.3 申请文书生成

#### 2.3.1 功能概述
基于学生档案和项目信息，一键生成个性化申请文书。

#### 2.3.2 生成流程

```typescript
// 一键申请页面
const generateStatement = trpc.ai.generateStatement.useMutation({
  onSuccess: (statement) => {
    setApplicationText(statement);
  }
});

// 调用参数
{
  studentProfile: StudentProfile,
  project: Project,
  customRequirements?: string  // 额外要求
}
```

#### 2.3.3 文书结构

**标准结构**:
1. **引言**: 表达申请意向，介绍背景
2. **能力展示**: 突出相关技能和经验
3. **项目理解**: 展示对项目的理解
4. **贡献承诺**: 表达学习意愿和贡献计划

#### 2.3.4 个性化定制

**基于学生档案**:
- 技能匹配的项目经验
- 相关的研究兴趣
- 学术背景和成绩

**基于项目特点**:
- 项目的研究方向
- 所需技能要求
- 项目难度和时长

### 2.4 项目描述扩写

#### 2.4.1 功能概述
教师发布项目时，AI帮助扩写项目描述，使其更详细和吸引人。

#### 2.4.2 扩写流程

```typescript
// 教师发布项目页
const expandDescription = trpc.ai.expandDescription.useMutation({
  onSuccess: (expandedText) => {
    setProjectDescription(expandedText);
  }
});

// 输入参数
{
  keywords: string[],      // 项目关键词
  basicDescription: string, // 基础描述
  requirements: string[],   // 项目要求
  duration: string         // 项目时长
}
```

#### 2.4.3 扩写策略

**内容扩充**:
- 添加项目背景介绍
- 详细说明研究意义
- 阐述预期成果
- 说明学习收获

**语言优化**:
- 使用专业学术语言
- 提高描述的吸引力和可读性
- 保持客观中立的语气

### 2.5 AI助手对话

#### 2.5.1 功能概述
全局AI助手，为不同角色的用户提供智能问答服务。

#### 2.5.2 助手界面

**组件结构**:
```typescript
// AIAssistantDrawer.tsx
function AIAssistantDrawer() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isOpen, setIsOpen] = useState(false);

  // 对话逻辑
}
```

#### 2.5.3 角色特定回答

**学生端助手**:
- 解释匹配度计算
- 申请文书写作建议
- 项目选择指导
- 系统使用帮助

**教师端助手**:
- 项目描述优化建议
- 申请人筛选指导
- 实习管理建议
- 系统功能说明

**管理员端助手**:
- 系统配置指导
- 用户管理建议
- 数据分析解释
- 故障排除帮助

#### 2.5.4 对话管理

**上下文保持**:
```typescript
interface ChatContext {
  role: 'student' | 'teacher' | 'admin';
  userId: number;
  conversationId: string;
  history: Message[];
}
```

**预设问题**:
- 根据用户角色显示相关问题
- 支持快速提问和深入对话
- 提供对话历史记录

---

## 🔧 AI服务集成

### 3.1 LLM调用架构

#### 3.1.1 服务配置

**环境变量**:
```bash
# AI服务配置
OPENROUTER_API_KEY=your_api_key_here
LLM_COUNCIL_BASE_URL=http://localhost:8001

# AI模型选择
AI_MODEL=deepseek-chat
AI_TIMEOUT=30000
```

#### 3.1.2 调用封装

```typescript
// server/_core/llm.ts
export async function invokeLLM(params: InvokeParams): Promise<InvokeResult> {
  try {
    // 调用LLM Council服务
    const response = await fetch(`${process.env.LLM_COUNCIL_BASE_URL}/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: process.env.AI_MODEL,
        messages: params.messages,
        temperature: 0.7,
        max_tokens: 2000
      })
    });

    if (!response.ok) {
      throw new Error(`LLM service error: ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    console.error('LLM调用失败:', error);
    throw error;
  }
}
```

### 3.2 降级策略

#### 3.2.1 自动降级机制

```typescript
// server/ai-match.ts
export async function calculateMatch(studentId: number, projectId: number) {
  try {
    // 优先使用LLM服务
    return await calculateMatchWithLLM(studentId, projectId);
  } catch (error) {
    console.warn('LLM服务不可用，使用本地规则降级');
    // 降级到本地规则计算
    return await calculateMatchLocal(studentId, projectId);
  }
}
```

#### 3.2.2 本地规则算法

**关键词匹配**:
```typescript
function calculateKeywordMatch(text1: string, text2: string): number {
  const words1 = extractKeywords(text1);
  const words2 = extractKeywords(text2);

  const intersection = words1.filter(word => words2.includes(word));
  const union = [...new Set([...words1, ...words2])];

  return intersection.length / union.length;
}
```

**相似度计算**:
```typescript
function calculateSimilarity(str1: string, str2: string): number {
  // 使用余弦相似度或其他相似度算法
  const vec1 = textToVector(str1);
  const vec2 = textToVector(str2);

  return cosineSimilarity(vec1, vec2);
}
```

### 3.3 缓存优化

#### 3.3.1 匹配结果缓存

```typescript
// 缓存匹配结果24小时
const CACHE_TTL = 24 * 60 * 60 * 1000; // 24小时

export async function getCachedMatch(studentId: number, projectId: number) {
  const cacheKey = `match:${studentId}:${projectId}`;

  // 检查缓存
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 计算新结果
  const result = await calculateMatch(studentId, projectId);

  // 写入缓存
  await redis.setex(cacheKey, CACHE_TTL, JSON.stringify(result));

  return result;
}
```

#### 3.3.2 缓存策略

**缓存键设计**:
- `match:{studentId}:{projectId}` - 匹配结果
- `parse:{resumeHash}` - 简历解析结果
- `statement:{studentId}:{projectId}` - 文书生成结果

**缓存失效**:
- 学生档案更新时清除相关缓存
- 项目信息变更时清除相关缓存
- 定期清理过期缓存

---

## 📊 AI功能监控

### 4.1 性能监控

#### 4.1.1 响应时间统计

```typescript
// AI调用性能监控
const startTime = Date.now();

try {
  const result = await invokeLLM(params);
  const duration = Date.now() - startTime;

  // 记录性能指标
  await logPerformanceMetrics({
    function: 'invokeLLM',
    duration,
    success: true,
    model: params.model
  });

  return result;
} catch (error) {
  const duration = Date.now() - startTime;

  // 记录错误指标
  await logPerformanceMetrics({
    function: 'invokeLLM',
    duration,
    success: false,
    error: error.message
  });

  throw error;
}
```

#### 4.1.2 成功率统计

**监控指标**:
- AI服务调用成功率
- 各功能模块使用情况
- 用户反馈统计
- 降级触发频率

### 4.2 质量评估

#### 4.2.1 用户反馈收集

```typescript
// AI结果质量反馈
interface AIQualityFeedback {
  functionName: string;
  resultId: string;
  rating: 1 | 2 | 3 | 4 | 5;  // 用户评分
  comments?: string;          // 用户意见
  userId: number;
  timestamp: Date;
}
```

#### 4.2.2 持续优化

**基于反馈的改进**:
1. 分析低分反馈的原因
2. 调整AI提示词和参数
3. 优化本地降级算法
4. 改进用户界面交互

---

## 🔒 安全考虑

### 5.1 数据隐私保护

#### 5.1.1 数据脱敏
- 不在AI调用中包含敏感个人信息
- 使用哈希或编码处理用户标识
- 限制AI响应的数据范围

#### 5.1.2 访问控制
- AI功能调用需要用户认证
- 根据用户角色限制AI功能访问
- 记录AI调用日志用于审计

### 5.2 服务稳定性

#### 5.2.1 限流保护
```typescript
// AI调用限流
const rateLimiter = new RateLimiter({
  windowMs: 60 * 1000,    // 1分钟
  max: 10,                // 最多10次调用
  message: 'AI调用过于频繁，请稍后再试'
});
```

#### 5.2.2 错误处理
- 优雅的错误提示
- 自动重试机制
- 服务降级保障

---

## 🚀 未来扩展

### 6.1 新功能规划

#### 6.1.1 高级AI功能
- **智能推荐算法优化**: 基于更多维度的推荐
- **自动面试评估**: AI辅助面试官决策
- **进度预测**: 基于历史数据预测实习进度
- **个性化学习路径**: 根据学生表现推荐学习内容

#### 6.1.2 多模型支持
- 支持多种AI模型选择
- 按功能选择最适合的模型
- A/B测试不同模型效果
- 模型性能对比分析

### 6.2 技术优化

#### 6.2.1 性能提升
- 实现AI结果预计算
- 优化缓存策略
- 引入AI服务集群
- 实现异步AI处理

#### 6.2.2 智能化增强
- 引入机器学习优化匹配算法
- 实现用户行为分析
- 添加情感分析功能
- 支持多语言AI交互

---

## 📚 相关文档

- [系统架构与创新亮点](../ARCHITECTURE_AND_INNOVATION.md)
- [数据库操作架构说明](../database/DATABASE_OPERATIONS.md)
- [用户手册](../../USER_MANUAL.md)
- [部署指南](../deployment/DEPLOYMENT.md)

---

**最后更新**: 2024年12月  
**版本**: v1.0  
**维护者**: AI功能开发团队
