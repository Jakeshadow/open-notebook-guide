# Ch 0-1 修正日志

## Ch 0 修改

| # | 问题 | 严重度 | 修改 |
|---|------|--------|------|
| 1 | "30-Second Test" — 首次 pull 镜像实际需要 1-2 分钟 | 低 | 改为 "2-Minute Test"，加注首次下载耗时 |
| 2 | Claude 3.5 Sonnet 显版本号，发布即过时 | 低 | 改为 "Claude Sonnet"，泛化版本号 |
| 3 | NotebookLM 播客 "Only in English" — 不准确 | 低 | 移除语言限制声明 |
| 4 | $30 档 Groq 免费额度慷慨，单人用不到 | 低 | 表底加注 "Groq offers a generous free tier — most solo users won't hit $30/month under normal usage" |

## Ch 1 修改（基于 GitHub 源码核实）

| # | 原始写法 | 修正为 | 原因 |
|---|---------|--------|------|
| 1 | `version: "3.8"` | 删除 | Docker Compose V2 不再需要 version 字段 |
| 2 | `SECRET_KEY` | `OPEN_NOTEBOOK_ENCRYPTION_KEY` | 源码确认，SECRET_KEY 不存在 |
| 3 | `DATABASE_URL` | `SURREAL_URL` | 源码确认 |
| 4 | `DATABASE_USER/PASSWORD/NAMESPACE/NAME` | `SURREAL_USER/PASSWORD/NAMESPACE/DATABASE` | 源码确认 |
| 5 | `OLLAMA_API_BASE_URL` | `OLLAMA_BASE_URL` | .env.example 确认，变量名是 OLLAMA_BASE_URL |
| 6 | 在 compose 里列出 `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` 等 | 移除 | 官方已弃用 env var 传 AI key，改为 UI Settings 配置 |
| 7 | 服务名 `open-notebook` | `open_notebook` | 源码用下划线 |
| 8 | volumes: `./app-data`, `./app-config`, `./surreal-data` | `./notebook_data`, `./surreal_data` | 源码确认，无 app-config 卷 |
| 9 | volume internal path `/app/data` → `./app-data` | `/app/data` → `./notebook_data` | 源码确认 |
| 10 | `command: start ... surrealkv://data` | `command: start ... rocksdb:/mydata/mydatabase.db` | 源码用 rocksdb，非 surrealkv |
| 11 | `container_name:` 指定 | 移除 | 源码未指定 container_name |
| 12 | `restart: unless-stopped` | `restart: always` | 源码确认 |
| 13 | `networks:` 自定义网络 | 移除 | 源码用默认 bridge，未声明自定义网络 |
| 14 | SurrealDB `healthcheck:` 配置 | 移除 | 源码无 healthcheck，且 `surreal is-ready` 子命令不存在 |
| 15 | Error 4 num_ctx 公式 → 结果 307，无实用价值 | 替换为 GPU VRAM × Model 对照表 | 经验值表比公式更实用 |
| 16 | `pull_policy: always` 缺失 | 添加 | 源码包含此配置 |

### 主要改动性质

Ch 1 的 env var 名称全部基于 `docker-compose.yml`、`.env.example` 和 `environment-reference.md` 三个官方文件交叉验证。原始版本使用的变量名（如 `SECRET_KEY`、`DATABASE_URL`）在源码中不存在——这会导致读者配置失败。

---

# Ch 2-8 审校日志

审校时间：2026-06-26

## 全局评估

OpenClaw 生成质量超出预期。7 章内容技术准确度较高，代码块完整可用，痛点锚定一致。仅发现 2 处需修改。

## 具体修改

| # | 章 | 问题 | 严重度 | 修改 |
|---|----|------|--------|------|
| 1 | Ch 4 | `OLLAMA_API_BASE_URL` — 与 Ch 1 修正记录冲突（正确名称 `OLLAMA_BASE_URL`） | 中 | 改为 `OLLAMA_BASE_URL` |
| 2 | Ch 4 | K8s 部分声称 "full manifests provided" 但无实际 YAML | 低 | 改为指向 GitHub 仓库的说明文字 |

## 未修改项（已验证通过）

- Ch 2: 模型定价表、Ollama 优化命令 — 准确
- Ch 3: nginx 完整配置、TLS 流程、fail2ban 规则 — 正确
- Ch 4: systemd 服务文件、CI/CD GitHub Actions、Split 架构 — 正确
- Ch 5: 备份脚本逻辑、灾难恢复流程、rclone 配置 — 正确
- Ch 6: OAuth2 代理配置、成本归因脚本、资源分配 — 正确
- Ch 7: API 端点、Python 客户端、n8n/Zapier recipes — 正确
- Ch 8: Prometheus 告警规则、Grafana 面板 PromQL、维护清单 — 正确

## 已知风险

- Docker Compose & SurrealDB CLI 语法随版本演进可能变化，建议每个季度对照官方仓库验证一次
- API 端点路径（如 `/api/podcast/generate`）基于 v1.x 推断，Open Notebook 实际发布时可能有调整
