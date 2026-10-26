# 分层开发流程

本仓库采用 [How the Claude Code team does code review](https://claude.dev/blog/solving-code-review-at-anthropic/) 所述的分层验证方法，具体执行沿用 [贡献约定](../CONTRIBUTING.md) 和 [审查重点](../REVIEW.md)，目标是减少真实缺陷与人的等待，同时保持回滚率和事故率稳定

## 从实现到 PR

1. 阅读项目入口与相关模块说明，明确问题、验收证据、系统边界、风险、负责人和恢复方案，保持变更小而可审查
2. 运行适用的真实验证，记录命令、结果及缺口；缺陷修复优先验证根因，界面变更提供截图与交互证据
3. 在开 PR 前，使用全新上下文的独立 Agent 审查完整差异与周边代码，同时明确执行安全审查；给出变更意图和真实 base/head，不能只让 Reviewer 接受作者解释
4. 核实发现，修复有效问题或写明不采纳理由，填写 PR 的验证、风险、恢复、AI 与安全审查证据

本地独立 AI 审查是当前可用路径；这些文档不会自动安装 GitHub AI Reviewer 或安全审查服务，未运行的服务不能记作通过

## 验证证据

按 [验证](../CONTRIBUTING.md#验证) 选择检查，保留现有 CI，不为纯文档改动重复构建无关应用

| 范围 | 从仓库根目录运行 |
| --- | --- |
| 文档 | 逐段核对内容、相对链接与命令，再运行 `git diff --check` |
| Rust 格式 | `cargo +1.97.0 fmt --all --manifest-path backend/Cargo.toml -- --check` |
| Rust 静态检查 | `RUST_MIN_STACK=16777216 cargo +1.97.0 clippy --manifest-path backend/Cargo.toml --all-targets --all-features --locked -- -D warnings` |
| Rust 测试 | `RUST_MIN_STACK=16777216 cargo +1.97.0 test --manifest-path backend/Cargo.toml --test main --locked` |
| 前端 | 安装仓库锁定依赖后运行 `pnpm --dir frontend format:check`、`pnpm --dir frontend build`，行为变化另附界面证据 |

数据库/Redis 集成检查使用专用测试服务和 `CPR_TEST_DATABASE_URL`、`CPR_TEST_REDIS_URL`，缺少环境而跳过不能算通过；不得使用生产凭据代替

## 风险与人的职责

风险由变更决定：普通文案可按低风险审查，产品行为按中风险，鉴权、凭据、隔离、迁移、插件权限、自动更新、CI、发布和架构约定按高风险；修改安全策略的文档也属敏感变更

每个变更有一位负责的人。初始阶段低风险也保留负责人确认；可信作者的快速通道须在得到授权且回滚/事故证据稳定后才放宽。中风险需要一位了解变更的人核对最终提交及证据，可以由负责人承担；高风险需要负责人及另一位具备相关能力的独立审查者。Agent 的批准不算人工批准；GitHub PR 作者不能给自己的 PR 提供 approving review，负责人在 GitHub 之外的确认不能代替平台要求的独立审批，也不能降低既有更严格规则

初始可信作者为 `xujinyang`；其他作者即使风险较低，也需要额外一位合格人工审查者

人工确认必须针对实际最终提交，新增提交或手动冲突解决后复核受影响内容。所有 AI 与安全发现修复或带理由关闭、讨论解决、适用检查通过后才能合并。通常一轮完整审查加修复验证即可，后续仅看新增差异和关联风险；三轮后重新评估方案，不无限循环

## 仓库设置与规则保护

[规则文件保护](../CONTRIBUTING.md#规则文件保护) 仍然有效：一般 PR 不修改 AGENTS.md、CONTRIBUTING.md、.agents/skills/ 或 .github/。需要调整入口、PR 模板、CODEOWNERS 或 CI 时，由维护者按现有渠道处理，不在 PR 中禁用保护

维护者接入时核对以下设置，文件存在不等于服务端已经强制执行：

- 保留现有质量、安全和构建工作流，确认 fork 的 Actions 已启用且检查实际运行后，再指定必需检查
- 在支持的分支规则中要求讨论解决、撤销过期审批、敏感路径人工审查，并安排非 PR 作者的合格审批人
- PR 证据保留 `Testing`、`Risk`、`How to revert`、负责人、审查提交、AI 工具/实际型号、安全发现与处理理由；正式模板调整由维护者处理，在此之前写入现有 PR 正文
- GitHub AI 与安全 Reviewer 尚需单独配置；缺少 Key、安装或权限时应标明缺口，不能用空任务显示通过，也不能让贡献分支代码在有密钥的特权任务中运行
- Merge Queue 仅在平台支持、并且必需工作流已支持 `merge_group` 后启用，否则合并前更新到当前 main 并重新验证

这些文档不更改服务端分支保护、自动合并或发版权限；本机的 Codex 全局规则要求读取 REVIEW.md，其他机器的入口需由维护者同步

## 放量、回滚与改进

新行为按适用范围使用功能开关，先内部验证，再逐步放量。PR 或发布记录写明负责人、监控信号、停止条件和可执行恢复步骤。配置或数据迁移必须核查旧版本兼容性和备份恢复，不能假定 git revert 能恢复数据

事故与回滚记录包括问题 PR、恢复方案和漏检层，把问题加入回归测试或审查评估集，并改进 REVIEW.md、项目说明或 lint。更换 Reviewer 的提示词、工具、模型后重跑评估，同时保留已知正常变更来衡量误报。跟踪审查轮数、人工审批/等待、误报、回滚和事故，避免仅把人的工作从审查转移到救火
