# gold-news — 黄金分析与新闻汇总一体化 Plugin

> 从 `kira4094/my-skills` 拆出的合并 plugin：**gold-price × news-summary** 合一，保留两者紧密联动。

## 结构
```
gold-news/
├── .claude-plugin/plugin.json   ← manifest (skills: ./skills)
├── skills/
│   ├── gold-price/              ← 黄金分析与金价获取（5文件）
│   │   ├── SKILL.md
│   │   ├── XAU-Gold-price-acquisition-rules.md
│   │   ├── output-templates.md  （gold-price × news-summary 共用模板）
│   │   ├── user_macro_gold_interest.md
│   │   └── 黄金数据追踪表.md      （含褐皮书6.6追踪节）
│   └── news-summary/            ← 新闻汇总与分类（3文件）
│       ├── SKILL.md
│       ├── news-summary-auto-time.md
│       └── us-media-monitoring.md
└── version.json
```

## 联动关系（合并后 `../` 引用天然保持）
- news-summary C0 对「新闻汇总 + 黄金日报」二合一输出（详见 news-summary/ 引用 gold-price/output-templates.md）
- gold-price SKILL.md 引用 `../news-summary/news-summary-auto-time.md` 和 `us-media-monitoring.md`
- 两 skill 同层放 `skills/` 下，所有 `../xxx` 跨引用不变

## 版本号
- 仓库级 `version.json` 由 `update-version.cjs` 计算（break→∞ / feat→minor / fix→patch）
- `.claude-plugin/plugin.json` 的 version 由 update-version 同步
- 两个 skill 的 frontmatter version 独立维护（gold-price 1.x / news-summary 1.x）

## 说明
- 月度记录、记忆等 不随 plugin 存放（见 gold-price/SKILL.md 记忆存储位置）
- 待办：git 仓库初始化与发布地址待定（用户决定）
