# Octo-server PM Agent — 项目说明

一个自动化的"产品管家"Agent，服务于 octo-server 开源项目（https://github.com/Mininglamp-OSS/octo-server）。

## 功能
1. **产品问答** — 回答 octo-server 的产品/技术问题，每条结论带 `来源: <相对路径>#L<起>-L<止>`
2. **需求归档** — 收到 bug/feature 反馈，判断类型，创建 issue 到本仓库（需求池）
3. **Label 体系** — 类型/优先级/状态 三维标签
4. **Cron 定时扫描** — 每 5 分钟自动扫描需求池 issue 变化（不靠人触发）
5. **回报到群** — 扫到变化主动回群，@ 对应的人 + @ 主考
6. **PM 链路** — 认领需求 → 补 PRD → review → 按打回原由修改

## Label 体系
| 维度 | Labels |
|------|--------|
| 类型 | `type:bug` `type:feature` `type:question` |
| 优先级 | `P0` `P1` `P2` `P3` |
| 状态 | `status:inbox` `status:triaged` `status:prd-drafting` `status:in-review` `status:done` `status:wontfix` |

## 目录结构
```
octo-server-pm/
├── AGENTS.md                    # Agent 主配置（行为定义）
├── knowledge-base.md            # 知识库（9 大领域，带引用）
├── memory/                      # 分层记忆
│   └── triage.md                # 需求判别规则
├── config/
│   ├── cron.yaml                # Cron 定时扫描配置
│   ├── labels.yaml              # Label 体系定义
│   └── repo.yaml                # 仓库配置
└── scripts/
    ├── scan_issues.sh           # 扫描需求池脚本
    └── report.sh                # 回群上报脚本
```
