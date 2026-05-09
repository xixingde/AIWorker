---
description: 分析并细化需求——阶段一生成用户需求(用户视角)，阶段二转换为系统需求(指导开发)
metadata:
  version: "1.0.0"
  triggers:
    - 用户提到"需求分析"、"需求细化"、"生成需求文档"
    - 用户提供一句话需求或模糊需求说明时
    - 用户需要输出 user-story.md 或 requirement.md
  agents: []
  tools: []
  model: opus
skill: design
---

# /requirements

## 命令功能

将一句话需求或用户需求，经过两阶段处理输出完整的系统需求文档：