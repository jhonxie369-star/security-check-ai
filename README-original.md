# Security-Check Skills

AI 驱动的代码安全检查技能，基于 Joern 污点追踪和业务逻辑理解。

## 核心理念

- **方法论，不是规则**：提供检查思路，不是固定查询
- **Joern 是工具，不是规则**：AI 根据业务理解决定追踪什么污点
- **理解业务优先**：先理解业务，再发现漏洞
- **灵活应用**：不被示例局限，创造性地发现问题

## 架构

```
security-check/
├── SKILL.md                    # 主入口，检查流程
├── ARCHITECTURE.md             # 架构和设计理念
├── joern/                      # Joern 工具使用指南
├── knowledge/                  # 跨语言安全知识库
├── patterns/                   # 语言特定检查模式
│   ├── java/
│   ├── go/
│   ├── javascript/
│   ├── python/
│   └── cross-service/
└── lessons-learned/            # 经验记录
```

## 特性

- ✅ 支持多语言（Java、Go、JavaScript、Python）
- ✅ 基于 Joern CPG 的精确污点追踪
- ✅ 业务逻辑漏洞发现
- ✅ 五步检查流程（理解 → 猜测 → 验证 → 追踪 → 分析）
- ✅ 漏洞验证方法论
- ✅ 经验积累和持续改进

## 使用

详见 [SKILL.md](SKILL.md) 和 [ARCHITECTURE.md](ARCHITECTURE.md)

## License

MIT
