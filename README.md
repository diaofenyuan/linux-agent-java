# linux-aiops-agent（Java 版）

面向 Linux 操作系统的安全智能运维 Agent 的 Java 重写版，移植自 [diaofenyuan/Linux-agent](https://github.com/diaofenyuan/Linux-agent)。

- 后端：Spring Boot 3.x + Java 17，内嵌 Tomcat，默认监听 `127.0.0.1:18080`
- 存储：与原版一致的 JSON/JSONL 本地文件（无外部数据库）
- 前端：自建 Vue 3 + TypeScript + Vite + Element Plus 控制台（`frontend/`），构建产物输出至 `frontend/build`，后端静态托管

## 环境要求

- JDK 17+
- Maven 3.8+

## 快速启动

**Windows 一键运行**：双击项目根目录的 `start.bat` 即可——自动检测 Java/前端产物/后端 jar（缺失时用 Maven 构建）、后台启动服务、等待健康检查并自动打开浏览器访问控制台。数据存放在 `.runtime/`，日志在 `.runtime/console.log`。再次双击会直接打开已运行的控制台；`stop.bat` 一键停止。

```bash
# 手动方式
mvn spring-boot:run

# 或打包运行
mvn -DskipTests package
java -jar target/linux-aiops-agent-1.0.0.jar
```

健康检查：

```bash
curl http://127.0.0.1:18080/api/health
```

## 目录结构

```
src/main/java/com/linux/agent/
├── ApiApplication.java      # 入口（对应 backend/ops_agent_api.py）
├── config/                  # AppProperties（OPS_AGENT_* 环境变量）、WebConfig
├── web/                     # 统一响应、错误码、异常、请求ID、分页
├── security/                # 会话、PBKDF2 密码、CSRF、RBAC（identity.py/rbac.py）
├── repository/              # JSON 文件访问（backend/repositories/*）
├── service/                 # 业务服务（backend/services/*）
├── controller/              # 路由适配（backend/api/*_routes.py）
└── audit/                   # 审计 JSONL Hash 链（audit/store.py）
```

## 与原项目的模块映射

| 原项目（Python） | 本项目（Java） |
| --- | --- |
| `backend/ops_agent_api.py` | `ApiApplication` + Spring MVC |
| `backend/core/response.py errors.py` | `web/ApiResponse` `web/ErrorCode` |
| `backend/core/context.py pagination.py` | `web/ReqCtx` `web/PageResult` |
| `backend/identity.py rbac.py` | `security/*` |
| `backend/repositories/*` | `repository/*` |
| `backend/services/*` | `service/*` |
| `backend/api/*_routes.py` | `controller/*` |
| `audit/store.py` | `audit/AuditService` |
| `realtime_monitor.py alert_rules.py os_collectors.py` | 待建：`monitor/` |
| `executor/ guardrails/ mcp_tools/ agent/` | 待建：`executor/ guardrails/ mcp/ chatops/` |

## 配置

沿用原版环境变量名（前缀 `OPS_AGENT_`），未设置时默认写入 `./.runtime/`：

`OPS_AGENT_DATA_DIR`、`OPS_AGENT_AUDIT_LOG`、`OPS_AGENT_AUTH_STORE`、`OPS_AGENT_ASSET_STORE`、`OPS_AGENT_SERVICE_CENTER_STORE`、`OPS_AGENT_INCIDENT_STORE`、`OPS_AGENT_API_CONFIG`、`OPS_AGENT_EXECUTION_STORE`、`OPS_AGENT_GUARDRAILS_STORE`、`OPS_AGENT_GUARDRAILS_APPROVAL_STORE`、`OPS_AGENT_TERMINAL_LOG_DIR`、`OPS_AGENT_ORCHESTRATION_DIR`、`OPS_AGENT_MCP_STORE`、`OPS_AGENT_MCP_INVOCATION_LOG`、`OPS_AGENT_CHATOPS_STORE`、`OPS_AGENT_ALERT_RULES_CONFIG`、`OPS_AGENT_ALERT_STATE_STORE`、`OPS_AGENT_METRICS_HISTORY_STORE`、`OPS_AGENT_LATEST_SNAPSHOT_STORE`、`OPS_AGENT_TEST_RUN_DIR`、`OPS_AGENT_PORT`、`OPS_AGENT_BIND`、`OPS_AGENT_FRONTEND_DIR`

## 已实现

- 统一响应 `{ok, code, message, data, meta, request_id}` 与稳定错误码（ok/validation_error/not_found/unauthenticated/permission_denied/csrf_required/internal_error 等）
- 请求 ID 过滤器（`X-Request-Id`）、审计日志 Hash 链（SHA-256，支持 `/api/v1/audit/verify` 校验）
- 认证链路：`GET /api/auth/bootstrap` → `POST /api/auth/init-admin` → `POST /api/auth/login`
- HttpOnly Cookie `ops_session` 会话、写请求 `X-CSRF-Token` 校验、RBAC 权限点拦截
- PBKDF2WithHmacSHA256 密码哈希、登录失败锁定（5 次锁 15 分钟）

## 路线图

1. ✅ 工程骨架 + 统一响应/错误码 + 健康检查
2. ✅ 认证/会话/CSRF/RBAC + 审计 Hash 链
3. ✅ 成员管理 + 角色目录/自定义角色（`/api/users*` `/api/roles*`）
4. ✅ 资产与拓扑中心（CRUD、分组、标签、应用、端口、依赖边、快照、同步 manual/local/config）
5. ✅ OS 只读感知采集 + 监控快照/历史 + 告警规则引擎（模拟、确认/升级/恢复、抑制、通知）+ 风险扫描
6. ✅ 服务中心与配置版本（保存/详情/diff/drift/rollback/lifecycle/变更记录）
7. ✅ 受控终端（allowlist 只读命令、模板/渲染、历史/回放）+ 文件管理（浏览/读写/上传下载、备份/diff/恢复、路径边界防护）
8. ✅ 安全护栏（内置种子规则 + 自定义规则 CRUD/启停/测试、豁免管理、意图+命令风险评估、审批请求记录）
9. ✅ ChatOps（外部 Agent API 调用 + 命令提取 + 护栏复核 + 高风险确认流）/ MCP 工具（内置只读工具集、JSON-RPC facade、插件注册启停卸载）/ Agent API 配置（地址白名单 SSRF 防护、密钥掩码、连通性测试）
10. ✅ 平台健康与管理员接口（runtime/context、admin/summary、admin/health、`/metrics` Prometheus + loopback 访问控制）
11. ✅ 审计查询（多条件检索）、导出 JSONL、归档（保留策略）、取证视图、合规报告
12. ✅ 自动化验收（内置 8 项自检用例、运行记录、Markdown 报告）+ 部署交付（systemd/RPM/安装卸载/冒烟脚本）

**已按需求裁剪**：事件工单、结构化审批执行器。

## 运行测试

```bash
mvn test
```

## 接入前端

前端已内置（Vue 3 + TS + Element Plus），开发与构建：

```bash
cd frontend
npm install

# 开发模式（代理 /api 到 127.0.0.1:18080）
npm run dev

# 类型检查 + 构建产物到 frontend/build（后端默认托管目录）
npm run typecheck
npm run build
```

构建后直接启动后端即可访问控制台：

```bash
mvn -DskipTests package
java -jar target/linux-aiops-agent-1.0.0.jar
# 浏览器打开 http://127.0.0.1:18080/
```

首次访问会引导初始化管理员账号；登录后可使用全部模块：总览、资产中心、监控告警、服务中心、受控终端、文件管理、安全护栏、ChatOps、MCP 工具、Agent API 配置、审计中心、用户与角色、验收测试。
