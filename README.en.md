# ⚒️ SkillForge

> Mind Skill Forge — distill "expert brains" into reusable Agent Skills
> Package domain experts, classic philosophy, and engineering methodology into ready-to-use skill packs for AI Agents / LLM knowledge bases.

[中文](README.md) · [Changelog](CHANGELOG.md)

---

## ✨ Highlights

- **Expert-mind distillation**: each skill is not just a knowledge base but a "thinking OS" — core mental models, judgment rules, expression DNA, honesty boundaries.
- **Plug and play**: each skill is an independent directory + `SKILL.md`, compatible with mainstream Agent skill specs — copy into your skills directory to activate.
- **Multi-domain**: from **cold-region rice experts** (Zu Xiaoli / Dr. Yan) to **classic philosophy** (Confucius / Laozi / Analects) to **engineering methodology** (Xianghe).
- **Ready to test**: each skill ships with trigger words, judgment rules, application boundaries, and rollback thinking.

---

## 📦 Skill Packs

| Skill | Domain | Core Capability |
|---|---|---|
| **zuxiaoli-rice** | Agriculture | Zu Xiaoli · cold-region rice "four attacks / four defenses" |
| **yanboshi-rice** | Agriculture | Dr. Yan · cold-region rice expert Q&A |
| **kongzi-perspective** | Philosophy | Confucius · benevolence-ritual thinking framework |
| **laozi-perspective** | Philosophy | Laozi · Dao-nature / reversal decision wisdom |
| **lunyu-skill** | Philosophy | Analects · 10 capability cards |
| **xianghe-perspective** | Engineering | Xianghe · server ops / AI deploy / KB building |

---

## 🚀 Quick Start

```bash
git clone https://github.com/15143221112/SkillForge.git
cp -r SkillForge/zuxiaoli-rice <your-skills-dir>/
```

Then trigger via natural language, e.g. "用孔子的视角看这个决策" activates `kongzi-perspective`.

---

## 📁 Structure

```
SkillForge/
├── zuxiaoli-rice/       # SKILL.md + references/
├── yanboshi-rice/
├── kongzi-perspective/
├── laozi-perspective/
├── lunyu-skill/
├── xianghe-perspective/
├── README.md
└── LICENSE
```

---

## 📄 License

[MIT](LICENSE) © 2026 SkillForge contributors
