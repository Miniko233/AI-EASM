# Agent Design

本文档定义 AI-EASM 项目中所有 Agent 的职责、工作流和权限。

------

## Agent 1: Target-Intelligence-Agent

**定义文件**: `.claude/agents/Target-Intelligence-Agent.md`

**职责**: 根据用户输入建立目标画像（Target Profile）。

### 输入路由

| 输入类型 | 处理方式 |
|----------|----------|
| 公司名称 | ICP Query 4 种类型全量扩展 → DNS → IP 判断 |
| 域名 | 去 www → ICP 获取公司名 → 全量扩展 → DNS → IP 判断 |
| IP / IP 段 / APP / 备案号 | 直接记录，不做扩展 |

### 可用工具

| 工具 | 用途 |
|------|------|
| `mcp__icp-query__icp_query` | ICP 备案查询（唯一允许的 MCP） |
| `python tools/scripts/dns_resolve.py` | DNS 解析（唯一允许的 Shell 命令） |

### 可用 Skill

| Skill | 用途 |
|-------|------|
| `origin-ip-judge` | 判断 DNS 解析 IP 是否为原站（非 CDN/WAF） |

### 输出

```
workspace/projects/<目录名>/target-profile.json
```

目录名优先级：公司名 > 域名 > IP > 原始输入

### 权限边界

- 仅允许 1 个 MCP 工具 + 1 条 Shell 命令
- 读/写仅限 `workspace/projects/<目标>/`
- 禁止安装任何工具或依赖

------

## Agent 2: Asset-Discovery-Agent

**定义文件**: `.claude/agents/Asset-Discovery-Agent.md`

**职责**: 根据 Target Profile 执行两阶段资产扫描发现。

### 工作流

```
读取 target-profile.json
    │
    ├── 有根域名 → Phase 0: Hunter 被动子域名（半年数据）
    │             → Phase 1: 子域名收集 (SubdomainScan + SubdomainSecurity)
    │             → 查询新子域名 → 与 Target Profile + Hunter 子域名合并去重
    │             → Phase 2: 端口/指纹/资产测绘 (EASM-PortScan)
    │
    └── 只有 IP → 直接 Phase 2
```

### 可用工具

| 工具 | 用途 |
|------|------|
| `mcp__hunter-mcp__hunter_search` | Phase 0: 鹰图被动搜索根域名的已知子域名 |
| `mcp__scopesentry__*` | ScopeSentry MCP（Phase 1/2 主动扫描） |

### 输出

```
workspace/projects/<项目>/agent2-asset-discovery/raw/
├── phase0_hunter_subdomains.json
├── phase1_subdomains.json
├── phase2_assets.json
└── phase2_ip_assets.json
```

------

## Agent 3: Asset-Intelligence-Agent

**定义文件**: `.claude/agents/Asset-Intelligence-Agent.md`

**职责**: 对 Agent 1/2 原始数据去重、合并、标准化，输出结构化资产清单。

### 工作流

```
读取 target-profile.json + phase2_*.json
    │
    ├── ip_domains[]     IP → 域名映射（IPAsset + Target Profile）
    ├── web_assets[]     URL去重（IP:Port + BodyHash，优先域名URL）
    └── port_services[]  端口服务聚合（IP + Port 去重）
    │
    ▼
  build_verified.py → LLM 审核 → verified-assets.json
```

### 输出

```
workspace/projects/<项目>/agent3-asset-intelligence/verified-assets.json
```

包含：ip_domains / web_assets / port_services / metadata

### 可用工具

仅允许 Python 脚本 `build_verified.py`，不调用任何 MCP。

------

## Agent 4: Report-Agent

**定义文件**: `.claude/agents/Report-Agent.md`

**职责**: 读取标准化资产数据，生成 Markdown + Excel 标准化报告。

### 输出

```
workspace/projects/<项目>/agent4-report/
├── report.md
└── report.xlsx
```

报告四部分：IP与域名 / Web资产 / 端口服务 / APP应用

Excel 以 Sheet 页区分各部分。

### 可用工具

仅允许 `build_report.py`，不调用任何 MCP。
