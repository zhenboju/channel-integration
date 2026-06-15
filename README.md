# channel-integration

渠道接入自动生成器 - Claude Code Skill

根据上游渠道接口文档，自动分析并生成符合项目统一模板的代码文件：
- **DTO 类**
- **Strategy 策略实现类**
- **Nacos 路由配置片段**

## 触发条件

当用户提到以下内容时自动触发：
- 接入新渠道
- 上游接口文档
- 新增策略
- channel integration
- 提供 API 文档内容或文件

## 工作流程

1. **解析** 接口文档 → 提取公司名、URL、签名方式、字段映射
2. **确认** 向用户展示分析结果，确认后继续
3. **生成** DTO + Strategy + Nacos 配置片段
4. **验证** 结构检查 + 代码检查 + 编译检查

## Skill 结构

```
channel-integration/
├── SKILL.md                              # 主文件 - 工作流程 + 决策指引
└── references/
    ├── strategy-template.md              # Strategy 完整代码模板 + 变体
    ├── dto-template.md                   # DTO 类模板 + 设计规则
    └── nacos-config-template.md          # Nacos YAML 配置模板 + 检查清单
```

## 安装

将  目录放置到项目的  下即可。
