# Gold & News — 黄金与新闻一体化

黄金分析与新闻汇总的组合助手，由两个 skill 构成：

- **gold-price**: 金价获取（双源交叉 gold-api + goldprice.today）、美国宏观数据追踪（CPI/PCE/NFP/Fed/褐皮书）、持仓管理、Fed 传声筒信号链。
- **news-summary**: 新闻多维度汇总（六维度强制清单），且"新闻汇总"总会伴随"黄金日报"二合一输出。

## 使用触发
- 用户说「新闻汇总 / 黄金日报 / 金价多少 / XAU / 追踪Fed / 褐皮书」→ 调用对应 skill
- 「新闻汇总」必伴随黄金日报（见 news-summary C0 硬联动）

完整的输出骨架见 `skills/gold-price/output-templates.md`，宏观数据追踪在 `skills/gold-price/黄金数据追踪表.md`。
