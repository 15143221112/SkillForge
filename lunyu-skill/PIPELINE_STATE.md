# PIPELINE_STATE.md —《论语》书蒸馏流水线状态

- 对象：《论语》二十篇（通行本）
- 档位：single-first（面向读者可直接调用的一套"孔子决策/修身方法论"能力卡）
- 产物根：/home/user/.doubao/agent_mode/workspace/.user_skills/lunyu-skill/

| 阶段 | 状态 | 产物 |
|---|---|---|
| 阶段0 整书理解 | 完成 | BOOK_OVERVIEW.md |
| 阶段1 五路提取 | 完成（串行降级） | candidates/{framework,principle,case,counterexample,term}.md |
| 阶段1.5 三重验证 | 完成 | verified.md, coverage-audit.md |
| 阶段1.6 晋级门 | 完成 | destinations.json |
| 阶段2 RIA++ 能力卡 | 完成 | cards/*.md (10 张), verified.yaml |
| 阶段3 Zettelkasten | 完成 | GLOSSARY.md + 各卡 also_read |
| 阶段4 压力测试 | 完成 | pressure-test.md |
| 阶段5 编译 | 完成 | DIGEST.md, SKILL.md |

来源：一手 ctext.org《论语注疏》武英殿本（19 篇）+ 古诗文网颜渊篇（对勘）。
