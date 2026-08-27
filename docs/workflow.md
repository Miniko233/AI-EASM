# AI-EASM Workflow

## 四阶段流水线

```
用户输入
  │
  ▼
┌─────────────────────────────────────────────────────┐
│ Agent 1: Target-Intelligence-Agent                  │
│                                                     │
│ 公司名 → ICP 查询 (web/app/mapp/kapp) → DNS 解析    │
│ 域名   → ICP 反查公司名 → 全量扩展 → DNS 解析       │
│ IP     → 直接记录                                    │
│                                                     │
│ 输出: target-profile.json                           │
│   company / domains[] / apps[] / mini_programs[]    │
│   quick_apps[] / ips[] / metadata                   │
└─────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────┐
│ Agent 2: Asset-Discovery-Agent                      │
│                                                     │
│ Phase 0: Hunter 被动搜索根域名（半年数据）           │
│           → 提取已知子域名                           │
│                                                     │
│ Phase 1: EASM-Subdomain                             │
│           (subfinder + ksubdomain)                  │
│           → 查询新子域名 → 合并去重                  │
│                                                     │
│ 全部目标 → Phase 2: EASM-PortScan                    │
│             (hunter, SkipCdn, SkipSameIP,           │
│              RustScan, fingerprintx, httpx,         │
│              WebFingerprint)                         │
│           → 查询端口/Web/IP聚合资产                  │
│                                                     │
│ 只有 IP → 跳过 Phase 0/1, 直接 Phase 2                │
│                                                     │
│ 输出: phase0_hunter_subdomains.json                  │
│       phase1_subdomains.json / phase2_assets.json   │
│       phase2_ip_assets.json                          │
└─────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────┐
│ Agent 3: Asset-Intelligence-Agent                   │
│                                                     │
│ ip_domains:   IPAsset → IP↔域名映射                  │
│ web_assets:   Assets → IP:Port+BodyHash 去重        │
│               (同内容合并, 优先域名URL)              │
│ port_services: IPAsset → IP+Port 去重聚合           │
│                                                     │
│ 输出: verified-assets.json                           │
│   ip_domains[] / web_assets[] / port_services[]     │
│   metadata                                           │
└─────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────┐
│ Agent 4: Report-Agent                               │
│                                                     │
│ 四部分 + APP应用                                     │
│                                                     │
│ 输出: report.md + report.xlsx                        │
│   Sheet: 概览 / IP与域名 / Web资产 / 端口服务 / APP  │
└─────────────────────────────────────────────────────┘
```

## 数据流

```
workspace/projects/<项目>/
│
├── context.json                          ← Project Context（唯一可信状态）
│
├── agent1-target-intelligence/
│   ├── raw/                              # ICP+DNS 原始数据
│   │   ├── icp_web.json
│   │   ├── icp_app.json
│   │   ├── icp_mapp.json
│   │   ├── icp_kapp.json
│   │   ├── domains.txt
│   │   └── dns_result.json
│   ├── target-profile.json               # → Agent 2/3/4 只读
│   └── reflection.json                   # Agent 1 复盘
│
├── agent2-asset-discovery/
│   ├── raw/                              # 扫描原始数据
│   │   ├── phase0_hunter_subdomains.json  # → Agent 3 只读
│   │   ├── phase1_subdomains.json        # → Agent 3 只读
│   │   ├── phase2_assets.json            # → Agent 3 只读
│   │   └── phase2_ip_assets.json         # → Agent 3 只读
│   └── reflection.json                   # Agent 2 复盘
│
├── agent3-asset-intelligence/
│   ├── verified-assets.json              # → Agent 4 只读
│   └── reflection.json                   # Agent 3 复盘
│
└── agent4-report/
    ├── report.md
    ├── report.xlsx
    └── reflection.json                   # Agent 4 复盘
```

每个 Agent 只写自己的目录，只读上游目录。

## 异常处理

### 原则

单个步骤失败不终止整个流水线。记录错误，尽可能继续。

### 各阶段策略

| Agent | 异常场景 | 处理 |
|-------|---------|------|
| Agent 1 | ICP 查询无结果 | DNS 仍执行（如有域名输入） |
| Agent 1 | DNS 解析失败 | 记录 error，继续 build_profile |
| Agent 2 | Phase 1 任务失败 | 用 Target Profile 已有子域名继续 Phase 2 |
| Agent 2 | Phase 2 任务失败 | 保存已查询到的部分结果 |
| Agent 3 | 输入文件缺失 | 跳过对应部分，在 notes 中记录 |
| Agent 4 | Excel 生成失败 (openpyxl) | 至少保证 Markdown 输出 |

## 工具与权限矩阵

| Agent | MCP | Shell | 只读 | 写入 |
|-------|-----|-------|------|------|
| Agent 1 | icp-query | dns_resolve.py, build_profile.py | docs/ | agent1/ |
| Agent 2 | scopesentry, hunter-mcp | 无 | agent1/ | agent2/ |
| Agent 3 | 无 | build_verified.py | agent1/, agent2/ | agent3/ |
| Agent 4 | 无 | build_report.py | agent1/, agent3/ | agent4/ |

## 设计原则

- **Python 负责数据，LLM 负责推理** — 去重、合并、标准化由脚本完成
- **原始数据不修改** — 所有处理生成新文件
- **所有结论可追溯** — source/task_name 字段记录来源
- **每个 Agent 一个阶段** — 不跨阶段混合职责
