# 大众点评调研工作流 (OpenCLI)
## 前提

本指南指导 Agent 如何利用搜索工具获取大众点评（Dianping）数据，并初筛出目的地附近的 Top 10 候选餐厅。

大众点评用于餐厅硬信号：口味、排队、踩雷、价格、区域和是否值得。
小红书只补氛围、近期体验、拍照和软性提醒。

OpenCLI 已提供大众点评 browser adapter，目标站点是 `www.dianping.com`。

## 环境要求

- Chrome 已登录 `dianping.com`
- 已安装 OpenCLI Browser Bridge 扩展
- 优先使用 PC 站；移动站对非移动 UA 限制较多

## 1. 搜索策略 (Search Strategies)
优先使用CLI搜索策略，进行搜索，在大众点评网页端直接搜索存在验证码或登录限制的情况下，建议使用搜索引擎进行定向抓取。

### CLI搜索策略
围绕当天主区域搜索，不搜泛词。

```bash
opencli dianping search "银座 午餐" --city 东京 --limit 10 -f json
opencli dianping search "有乐町 晚餐" --city 东京 --limit 10 -f json
opencli dianping search "新宿 居酒屋" --city 东京 --limit 10 -f json
```

命令格式：

```bash
opencli dianping search "<keyword>" --city <name-or-id> --limit <n> -f json
```

`--city` 可用中文、拼音或大众点评 cityId；省略时使用当前 cookie 里的城市。


### 常用搜索语法 (Google Search / DuckDuckGo Queries)
使用 `search_web` 工具，利用以下组合进行定向检索：

1. **按目的地与菜系口味搜索**：
   - 语法：`site:dianping.com <目的地> <口味/菜系> 推荐`
   - 示例：`site:dianping.com 上海静安寺 本帮菜 推荐`
2. **按就餐目的与环境标签搜索**：
   - 语法：`site:dianping.com <目的地> <就餐目的/环境要求> 餐厅`
   - 示例：`site:dianping.com 北京三里屯 约会 景观餐厅`
3. **查特定店铺的评分与客单价**：
   - 语法：`site:dianping.com/shop <店铺名称> <目的地>`
   - 示例：`site:dianping.com/shop 荣叔黄鱼面 新天地`

## 2. 数据提取要点 (Data Extraction)

从搜索结果的网页快照或描述片段中，重点提取并记录以下关键字段：
- **店铺全称**（防止品牌混淆）
- **大众点评星级评分**（如 4.5/5.0 或五星/准五星）
- **人均客单价**（RMB/人）
- **主营菜系/特色**（如 “潮州菜”、“高空法餐”）
- **地理位置**（具体商圈或路段，例如 “太古里”、“巨鹿路”）
- **热门推荐菜/招牌菜**

### 查看店铺详情

搜索结果里的 `shop_id` 可以继续查详情。

```bash
opencli dianping shop <shop_id> -f json
opencli dianping detail <shop_id> -f json
```

也可以传完整店铺 URL：

```bash
opencli dianping shop "https://www.dianping.com/shop/<shop_id>"
```

## 3. 初筛与过滤规则 (Filtering Rules)

根据提取到的数据，执行以下规则过滤，整理出 Top 10 候选列表：
1. **位置匹配性**：必须在目的地商圈或其步行/打车 10 分钟范围之内。
2. **预算契合度**：人均客单价需在用户要求的预算区间内。若用户只说“便宜”，则过滤出当地该菜系人均较低的店铺；若无预算限制则不作硬性过滤。
3. **评分底线**：原则上点评评分需在 **4.0 分及以上**（新开业暂无评分但热度极高的可作为特例保留，但需特殊说明）。
4. **去重与精简**：去除分店重复的情况，只保留距离目的地最近或评分最高的那家分店。
5. **数量控制**：精简为最符合需求的 **10 家候选餐厅**，并按照点评评分与相关度从高到低排列。

### 判断标准

优先看：

- `rating`：基础稳定性
- `reviews`：评价量，太少说明信号弱
- `price`：是否符合预算
- `cuisine`：是否适合当前这顿饭
- `district`：是否落在当天区域
- 评价关键词：排队、踩雷、服务、游客店、性价比、是否值得专门去

不要为了高分店扭曲路线。餐厅默认是当天区域里的补给点，只有预约餐、强目的餐、用户明确指定的店，才允许成为路线锚点。

### 写回格式

每顿饭只保留 2-3 个候选。

```md
午餐区域：银座 / 有乐町
主推：店名 A
- 大众点评：评分稳定，评价量够，适合午餐，不需要专门绕路
- 小红书：近期反馈氛围好，拍照友好

备选：店名 B
- 大众点评：离地铁近，排队风险低
- 小红书：更像工作日简餐
```

### 常见坑

- 只按评分选店，不看它是否在当天区域
- 为了一家店反向规划半天路线
- 把小红书种草当成餐厅硬口碑
- 忽略排队、预约和营业时间
- 搜索词太泛，得到一堆游客店

## 官方参考

- https://github.com/jackwener/OpenCLI/blob/main/docs/adapters/browser/dianping.md
