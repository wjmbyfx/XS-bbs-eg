# Gin + Gorm + go-redis 练手项目详细设计文档

> 文档目标：给你一份“可照着做”的详细方案，从需求拆分、架构分层、数据建模、接口设计到开发节奏，帮助你高质量完成一个中小型社区后端练手项目。

## 1. 项目定位

### 1.1 项目名称
XS-BBS（轻量论坛后端，面向接口开发）。

### 1.2 练手目标
- 掌握 Gin 的路由组织、中间件、参数绑定与统一响应。
- 掌握 Gorm 的建模、关联查询、事务、分页、软删除与索引设计。
- 掌握 go-redis 的缓存读写、排行榜、计数器与缓存一致性基础策略。
- 建立“Controller -> Service -> Repository”分层思维。
- 形成可持续扩展的工程结构（后续可加消息队列、搜索、审核系统）。

### 1.3 业务范围（MVP）
- 用户模块：注册、登录、获取用户信息。
- 社区模块：社区列表、社区详情。
- 帖子模块：发帖、帖子列表、帖子详情。
- 互动模块：帖子点赞 / 取消点赞（Redis + DB 最终一致）。

---

## 2. 需求拆分

## 2.1 功能性需求
1. 用户可注册并登录，登录后拿到 JWT。
2. 登录用户可发布帖子，帖子挂靠某个社区。
3. 游客可查看社区列表、帖子列表、帖子详情。
4. 登录用户可对帖子点赞或取消点赞。
5. 列表接口支持分页。

## 2.2 非功能性需求
- 可维护性：分层清晰，接口统一错误码。
- 可观测性：请求日志 + panic recover + 慢查询日志。
- 可扩展性：模块化目录设计，便于新增评论、收藏、关注。
- 一致性策略：允许点赞计数短时间延迟落库（异步化可演进）。

---

## 3. 整体架构设计

采用经典三层：

1. **Controller（接口层）**
   - 只做：参数校验、调用 service、组装响应。
   - 不做：复杂业务与 SQL。

2. **Service（业务层）**
   - 只做：业务编排、事务边界、缓存策略决策。
   - 不做：直接处理 HTTP 细节。

3. **Repository（数据层）**
   - 只做：DB/Redis 读写细节。
   - 对 service 暴露业务语义方法（如 `CreatePost`、`GetPostList`）。

依赖方向：`controller -> service -> repository`（单向依赖）。

---

## 4. 目录设计建议

```text
cmd/
  main.go                     # 程序启动，组件初始化，路由挂载
internal/
  app/
    user/
      controller/
      service/
      repository/
      model/
    community/
      controller/
      service/
      repository/
      model/
    post/
      controller/
      service/
      repository/
      model/
  pkg/
    ginx/                     # 统一响应、参数解析、上下文用户获取
    middleware/               # jwt、日志、recover、cors
    constant/                 # 错误码、常量定义
pkg/
  conf/                       # 配置读取
  database/                   # gorm 初始化
  cache/                      # redis 初始化
  logger/                     # zap 初始化
docs/
  swagger.yaml
script/
  my_app.sql
```

说明：
- `internal/app` 放业务模块，按领域拆分，便于后续拆微服务。
- `internal/pkg` 放业务强相关公共能力。
- `pkg` 放可复用基础组件。

---

## 5. 数据库设计（MySQL）

## 5.1 表设计概览

1. `user`：用户信息。
2. `community`：社区信息。
3. `post`：帖子主体。
4. `post_like`（可选）：点赞关系（若全放 Redis，可后续异步落地）。

## 5.2 关键字段建议

### user
- `id` bigint PK（雪花ID或自增）
- `username` varchar(64) UNIQUE
- `password` varchar(256)（哈希）
- `email` varchar(128)（可选）
- `status` tinyint（0正常/1禁用）
- `created_at` / `updated_at`

索引：
- `uniq_username(username)`

### community
- `id` bigint PK
- `name` varchar(128)
- `introduction` varchar(1024)
- `created_at` / `updated_at`

索引：
- `idx_name(name)`

### post
- `id` bigint PK
- `title` varchar(256)
- `content` text
- `author_id` bigint
- `community_id` bigint
- `status` tinyint（0正常/1删除/2审核中）
- `created_at` / `updated_at`

索引：
- `idx_author(author_id)`
- `idx_community_created(community_id, created_at desc)`

### post_like（可选持久化表）
- `id` bigint PK
- `post_id` bigint
- `user_id` bigint
- `created_at`

索引：
- `uniq_post_user(post_id, user_id)`
- `idx_user(user_id)`

---

## 6. Redis 设计

## 6.1 Key 规范
- `bbs:post:like:users:{postID}` -> Set，存点赞用户ID。
- `bbs:user:like:posts:{userID}` -> Set，存用户点赞过的帖子ID。
- `bbs:post:score` -> ZSet，存帖子热度分值。
- `bbs:post:detail:{postID}` -> String(JSON)，帖子详情缓存（可选）。

## 6.2 缓存策略
1. **详情缓存**：先查缓存，miss 查 DB 并回填，设置 TTL（如 5~15 分钟）。
2. **点赞计数**：
   - 点赞：`SADD post_like_users`，`ZINCRBY post_score +1`
   - 取消点赞：`SREM ...`，`ZINCRBY post_score -1`
3. **排行榜**：基于 `ZREVRANGE bbs:post:score`。

## 6.3 一致性策略
- 方案A（入门推荐）：点赞关系实时写 Redis + 同步写 DB。
- 方案B（进阶）：写 Redis + 记录变更日志，异步批量落库（更高性能）。

---

## 7. 接口设计（REST）

统一约定：
- 响应体：`{ "code": 0, "msg": "ok", "data": {...} }`
- 鉴权：`Authorization: Bearer <token>`

## 7.1 用户模块
1. `POST /api/v1/user/register`
   - 入参：`username`, `password`, `re_password`
   - 出参：注册成功信息

2. `POST /api/v1/user/login`
   - 入参：`username`, `password`
   - 出参：`token`, `user_id`, `username`

3. `GET /api/v1/user/current`
   - 鉴权：是
   - 出参：当前登录用户信息

## 7.2 社区模块
1. `GET /api/v1/community/list`
2. `GET /api/v1/community/detail?id=xxx`

## 7.3 帖子模块
1. `POST /api/v1/post/create`
   - 鉴权：是
   - 入参：`title`, `content`, `community_id`

2. `GET /api/v1/post/list?page=1&size=10&community_id=1`
3. `GET /api/v1/post/detail?id=xxx`

## 7.4 点赞模块
1. `POST /api/v1/post/vote`
   - 鉴权：是
   - 入参：`post_id`, `action`（1点赞，-1取消）

---

## 8. 关键流程设计

## 8.1 注册流程
1. controller 校验参数。
2. service 检查用户名是否重复。
3. service 调用 hash 工具加密密码。
4. repository 写入 `user` 表。

## 8.2 登录流程
1. 查询用户。
2. 校验密码哈希。
3. 生成 JWT（含 userID、过期时间）。
4. 返回 token。

## 8.3 发帖流程
1. JWT 中间件解析当前 userID。
2. service 校验社区是否存在。
3. 生成 postID（雪花算法可选）。
4. repository 落库 post。
5. 初始化 Redis 热度分。

## 8.4 点赞流程
1. 校验帖子存在。
2. 根据 action 执行 `SADD/SREM`。
3. 同步更新热度 ZSet。
4. 需要强一致时同步写 `post_like` 表。

---

## 9. 中间件与通用能力

## 9.1 中间件链建议
1. `recovery`：兜底 panic，保证服务不崩。
2. `logger`：记录请求耗时、状态码、错误。
3. `cors`：跨域支持。
4. `jwt`：仅挂在需要鉴权的路由组。

## 9.2 统一错误码
- `0` 成功
- `1001` 参数错误
- `1002` 用户不存在
- `1003` 密码错误
- `1004` token 无效
- `2001` 帖子不存在
- `3001` 系统内部错误

建议在 `internal/pkg/constant/e` 中集中维护，避免魔法数字。

---

## 10. 配置与环境

`config.yaml` 建议包括：
- app：name, mode, port
- mysql：dsn, max_open_conns, max_idle_conns
- redis：addr, password, db
- jwt：secret, expire
- log：level, encoding

多环境建议：
- `config.dev.yaml`
- `config.test.yaml`
- `config.prod.yaml`

启动时通过环境变量切换。

---

## 11. 安全设计基础

- 密码必须哈希存储（bcrypt/argon2）。
- JWT secret 不写死在代码里。
- 发帖、点赞等写操作必须鉴权。
- 参数校验要有长度限制（标题、内容、用户名）。
- 对高频接口可加简单限流（后续可接入 Redis 限流）。

---

## 12. 开发里程碑（建议按周）

### 第1阶段：骨架搭建
- 初始化 Gin、Gorm、Redis、Zap、Viper。
- 跑通健康检查接口 `/ping`。
- 定义统一响应结构与错误码。

### 第2阶段：用户模块
- 注册/登录/JWT 鉴权。
- 完成用户接口联调与基本单测。

### 第3阶段：社区与帖子模块
- 社区列表、详情。
- 发帖、列表、详情。
- 完成分页与排序。

### 第4阶段：点赞 + 缓存
- 完成点赞接口。
- 引入 ZSet 热度排序。
- 设计缓存回源逻辑。

### 第5阶段：质量提升
- 增加接口测试。
- 补齐 Swagger。
- 加入 Makefile/脚本简化开发流程。

---

## 13. 测试策略

## 13.1 单元测试
- service 层优先，repository 可 mock。
- 覆盖核心业务分支：注册重复、登录失败、点赞重复等。

## 13.2 集成测试
- 使用测试库 + 测试 Redis。
- 覆盖关键链路：注册 -> 登录 -> 发帖 -> 点赞 -> 查询。

## 13.3 压测（可选）
- 使用 `wrk`/`hey` 对帖子列表与点赞接口压测。
- 观察 p95 延迟、错误率、QPS。

---

## 14. 可扩展路线（进阶）

1. 评论系统：楼中楼、评论点赞。
2. Feed 流：关注 + 时间线。
3. 审核系统：敏感词过滤、异步审核。
4. 消息通知：点赞/评论通知。
5. 搜索：接入 Elasticsearch。
6. 可观测性：Prometheus + Grafana + tracing。

---

## 15. 模仿开发建议

1. 先做“最小可用链路”：注册、登录、发帖、列表。
2. 再做 Redis 增强：点赞与热榜。
3. 最后补质量：测试、文档、异常处理、日志规范。
4. 每个模块都按固定模板创建：
   - `model`（实体与 DTO）
   - `repository`（数据访问）
   - `service`（业务）
   - `controller`（HTTP）

这样你会快速形成一套可复用的后端开发肌肉记忆。

