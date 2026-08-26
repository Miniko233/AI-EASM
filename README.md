# AI-EASM
基于 Claude Code 的多 Agent 协作系统，自动化完成企业外部攻击面的"发现 → 测绘 → 分析 → 报告"全流程。
## 0x01 这是什么

输入一个公司名、域名或 IP，系统调度 4 个 AI Agent 协作，最终输出一份包含 **IP/域名、Web 资产、端口服务、APP 应用** 的 Markdown + Excel 标准化报告。

```
用户输入（公司名 / 域名 / IP）
    │
    ▼
Agent 1: 目标情报 ──→ ICP 备案 + DNS 解析 → Target Profile
    │
    ▼
Agent 2: 资产发现 ──→ 子域名收集 + 端口扫描 + 指纹识别 → 原始资产
    │
    ▼
Agent 3: 资产智能 ──→ 去重合并 + 标准化 → 结构化资产清单
    │
    ▼
Agent 4: 报告生成 ──→ Markdown + Excel → 最终报告
```

<img src=".images/image-20260811165333623.png" alt="image-20260811165333623" style="zoom:80%;" align="left"/>

## 0x02 项目结构

<img src=".images/image-20260811165635904.png" alt="image-20260811165635904" style="zoom:80%;" align="left"/>

```
.
├── CLAUDE.md                     # 项目主配置（Workflow Manager 指令）
├── docs/
│   ├── workflow.md               # 四阶段流水线定义
│   └── agent-design.md           # 4 个 Agent 的职责与权限
├── .claude/
│   ├── agents/                   # Agent 定义文件
│   ├── skills/                   # Skill（专业技能包）
│   │   ├── origin-ip-judge/      #   原站 IP 判断（CDN/WAF 识别）
│   │   └── scopesentry-mcp/      #   ScopeSentry 扫描平台操作指南
│   └── settings.local.json       # 本地 MCP 服务器配置
├── tools/
│   ├── scripts/                  # Python 数据处理脚本
│   │   ├── requirements.txt      #   Python 依赖清单
│   │   ├── dns_resolve.py        #   DNS 并发批量解析（A/CNAME）
│   │   ├── build_profile.py      #   Target Profile 构建
│   │   ├── merge_subdomains.py   #   子域名合并去重
│   │   ├── build_verified.py     #   资产标准化与去重
│   │   └── build_report.py       #   报告生成（Markdown + Excel）
│   └── ICP_Query-main/           # ICP 备案查询本地服务MCP（Python + Rust）
└── workspace/
    ├── templates/
    │   ├── context.json          # 项目上下文模板
    │   └── target-profile.json   # 目标画像模板
    └── projects/
        └── <项目名>/              # 每个目标一个目录
            ├── context.json
            ├── agent1-target-intelligence/
            ├── agent2-asset-discovery/
            ├── agent3-asset-intelligence/
            └── agent4-report/
```
## 0x03 Agent说明

### Agent 1：Target Intelligence

根据公司名/域名/IP/备案号建立目标画像(ICP备案、DNS 解析、CDN/WAF 判断)

### Agent 2：Asset Discovery

执行两阶段资产扫描(子域名收集 >端口/指纹/资产测绘)，调度 ScopeSentry 平台

### Agent 3：Asset Intelligence

对原始数据去重、合并、标准化，输出结构化资产清单

### Agent 4：Report

生成Markdown+Excel报告(IP/域名、Web资产、端口服务、APP应用)

## 0x04 快速使用

#### 0-01 环境

1、**Claude Code** 

2、**Python 3.11+** + 依赖：

```ruby
pip install -r tools/scripts/requirements.txt
```

#### 0-02 模型选择

建议使用**DeepSeek**，搭建是以主会话（deepseek-v4-pro）+subagent（deepseek-v4-flash）构建的

如果选择使用其他模型，主会话无影响，需要修改一下subagent的模型映射

.claude/settings.local.json

```ruby
{
  "env": {
    "ANTHROPIC_DEFAULT_FABLE_MODEL": "deepseek-v4-flash" #修改成使用的模型
  }
}
```

#### 0-03 MCP添加

| MCP服务器   | 用途                     |
| ----------- | ------------------------ |
| icp-query   | 工信部 ICP 备案查询      |
| hunter-mcp  | 奇安信鹰图网络空间测绘   |
| scopesentry | ScopeSentry 主动扫描平台 |

根据以下项目地址部署添加：

icp-query：https://github.com/HG-ha/ICP_Query

hunter-mcp：https://github.com/PiggyHurry/hunter-mcp

scopesentry：https://mp.weixin.qq.com/s/jweBcSH61ncnxo4Iv06kiQ

#### 0-04 运行

在项目文件夹下启动 Claude Code 会话，直接输入：

```
帮我做下 <公司名/域名/IP> 的外部攻击面/信息收集
```

