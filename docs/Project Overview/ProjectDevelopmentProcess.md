## （基于实际使用问题）修复与优化计划

### 1. 解决 404 与路由问题

- [x] 为学生端项目详情页新增页面组件（如：`client/src/pages/student/ProjectDetail.tsx`），使用 `trpc.project.getById` 拉取项目详情，并接入 AI 匹配接口 `trpc.ai.calculateMatch` / `trpc.ai.generateStatement`
- [x] 为学生端一键申请新增申请页面或弹窗逻辑（如：`client/src/pages/student/ProjectApply.tsx` 或在详情页内处理），调用 `trpc.application.create` 完成申请
- [x] 在路由配置 `client/src/App.tsx` 中新增动态路由：
  - [x] `/projects/:id` 绑定到项目详情组件（仅 `student` 角色可访问）
  - [x] `/projects/:id/apply` 绑定到一键申请组件（仅 `student` 角色可访问）
- [x] 确认 `ProjectCard` 中的链接（`/projects/${id}` 与 `/projects/${id}/apply`）与新建路由完全对应，避免再跳转到 404
- [x] 为教师端发布新项目新增页面组件（如：`client/src/pages/teacher/ProjectNew.tsx`），表单字段与后端 `project.create` 输入模型保持一致
- [x] 在 `client/src/App.tsx` 中新增路由 `/teacher/projects/new`，并通过 `ProtectedRoute` 限制为 `teacher` 角色
- [x] 在教师端「我的项目」页面 `client/src/pages/teacher/Projects.tsx` 中确认「发布新项目」按钮链接为 `/teacher/projects/new`

### 2. 数据库检索与登录权限逻辑梳理

- [x] 明确角色模型需求：一个账号对应一个固定角色，不允许通过前端随意切换学生/教师/管理员身份
- [x] 如不允许自由切换角色，则：
  - [x] 限制或移除后端 `auth.updateRole` 接口（`server/routers.ts`），改为仅管理员可在「用户管理」中修改角色
  - [x] 调整前端角色选择页面 `client/src/pages/RoleSelect.tsx`，仅作为角色说明页和当前角色首页入口，不再提供自行切换功能
- [x] 删除教师端单独首页入口（仅保留学生端 `/dashboard` 作为“首页”），确保顶部/侧边「首页」链接不会误导到错误角色或触发角色切换
- [x] 在本地认证模块 `server/_core/local-auth.ts` 中，登录时校验账号在数据库中的 `user.role` 与登录界面选择的角色是否一致，不一致则拒绝登录，防止通过登录界面“伪装”成其他角色
- [x] 对关键 `protectedProcedure` / `studentProcedure` / `teacherProcedure` 接口补充权限校验：
  - [x] `application.getMyById` 仅允许学生查看自己的申请
  - [x] `internship.getProgress` 根据 `ctx.user.role` 校验学生/教师只能查看与自己相关的申请进度
  - [x] `internship.submitWeeklyReport` 仅允许对应申请的学生提交周报
- [x] 检查并使用 `server/db.ts` 中以当前用户 id/teacherId 过滤的数据访问方法（如 `getApplicationsByStudentId`、`getInternshipsByTeacherId` 等），保证三端不会串号
- [x] 在前端 `useAuth`（`client/src/_core/hooks/useAuth.ts`）中，根据 `auth.me` 的结果写入/清理 `localStorage`（`research-match-user-info`），防止退出后仍残留旧用户与角色信息
- [x] 在登录 / 退出登录逻辑中（`Login` 页面与 `SystemLayout`、`DashboardLayout`），通过刷新 `auth.me` 与清理缓存，保证当前标签页中的身份和数据与后端一致

### 3. AI 功能前后端联通与体验完善

- [x] 保持本地规则版 AI 匹配实现 `server/ai-match-local.ts` 可用，作为 llm-council / OpenRouter 异常时的兜底实现（`calculateMatch`、`generateApplicationStatement`、`expandProjectDescription`、`chatWithAI`）
- [x] 在项目根目录 `.env` 中增加并配置：
  - [x] `OPENROUTER_API_KEY`（供 llm-council 使用）
  - [x] `LLM_COUNCIL_BASE_URL`（如 `http://localhost:8001`，指向 llm-council FastAPI 后端）
- [x] 在 `server/_core/llm.ts` 中实现真正的 `invokeLLM`：
  - [x] 从 `process.env.LLM_COUNCIL_BASE_URL` 读取 llm-council 地址
  - [x] 封装一次 HTTP 请求，将 `InvokeParams` 转换为 llm-council 所需的请求体格式
  - [x] 将 llm-council 返回的最终回答映射为当前项目使用的 `InvokeResult` 结构
- [x] 在 `server/ai-match.ts` 中完成与 llm-council 的集成：
  - [x] 使用 `invokeLLM` 实现 `calculateMatch` / `generateApplicationStatement` / `expandProjectDescription` / `chatWithAI`
  - [x] 在 LLM 调用失败时，自动退回到 `ai-match-local` 中对应的本地规则算法
- [x] 在 `server/routers.ts` 中，将 `ai` 路由从 `./ai-match-local` 切换为 `./ai-match`，让 `trpc.ai.*` 默认走 llm-council + OpenRouter
- [x] 在学生项目详情页中：
  - [x] 使用 `trpc.ai.calculateMatch` 根据当前学生档案和项目内容展示「匹配度」与解释（真实 LLM 结果）
  - [x] 明确在 LLM 不可用时的回退提示（例如提示“当前使用本地规则匹配”）
- [x] 在学生一键申请页中：
  - [x] 使用 `trpc.ai.generateStatement` 一键生成申请陈述，并支持学生编辑后再提交到 `application.create`
- [x] 在教师发布/编辑项目页中：
  - [x] 将项目关键词输入对接 `trpc.ai.expandDescription`，生成更丰富、结构化的项目描述文案
- [x] 在系统顶部 AI 按钮（`SystemLayout` 中的 `AIAssistantDrawer`）：
  - [x] 确认 `trpc.ai.chat` 接口可用，按当前实现展示对话历史与不同角色的预设问题
  - [x] 为 AI 回复失败的情况增加更友好的错误提示和重试按钮（例如提示“LLM 服务暂不可用，已切换为本地规则说明”）
- [x] 补充前端接口错误处理与 loading 状态，确保 AI 相关操作在网络较慢或 AI 不可用时有清晰反馈

### 4. 通知（邮件/消息提醒）点击无响应问题

- [x] 在 `SystemLayout` 中为通知按钮（Bell 图标）增加点击交互：
  - [x] 设计通知列表展示方式（顶部下拉、右侧抽屉或独立通知中心页面）
  - [x] 使用 `trpc.notification.list` 拉取当前用户通知列表，并通过 `trpc.notification.markAsRead` 标记已读
- [x] 在个人设置页 `client/src/pages/Settings.tsx` 中，将「通知设置」与后端 `notificationEnabled` 字段完全对齐，并确保更新后能影响通知推送逻辑
- [x] 为通知相关接口增加最小前端测试或手动测试用例：有未读时小红点出现，点击后能查看详情并正确清除未读数

### 5. 验收与回归测试

- [x] 编写一份手工测试清单，覆盖学生/教师/管理员三端的主要流程（登录、查看/发布项目、申请流程、通知查看、AI 使用）
- [x] 重点验证：任意一个账号在三端访问时看到的数据是否始终与自身角色和权限匹配
- [x] 在本地与测试环境分别跑一轮完整回归，确认 404、权限、数据串号与 AI 聊天/匹配功能均按预期工作