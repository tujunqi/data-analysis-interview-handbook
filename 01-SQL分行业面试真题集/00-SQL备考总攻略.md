# SQL备考总攻略（开库必读）

> 本篇是整个《SQL分行业面试真题集》的导读与方法论。8个行业分册、360道真题都在本目录下，但在刷题之前，请先用10分钟读完这一篇——它会告诉你：校招SQL到底考什么、怎么练、在考场上如何组织答题流程。**【面试高频】**【加分点】**【避坑指南】** 三类标签将在全库反复出现，含义如下：
>
> - **【面试高频】**：近两年校招面试（笔试+一面）反复出现的考点，必须熟练到"肌肉记忆"；
> - **【加分点】**：多数候选人答不出、答好了能拉开差距的点；
> - **【避坑指南】**：面试官明确用来筛人的坑，踩一次就可能挂。

---

## 一、为什么SQL是数据分析校招的第一道门槛

在国内校招数据分析岗（业务向）的招聘流程中，SQL的权重高得超出大多数应届生的想象，原因有三个：

1. **它是唯一100%会被考察的硬技能。** 统计学、Python、机器学习可以因团队而异，但只要这个岗位叫"数据分析师"，SQL必考。形式可能是：在线笔试（牛客/赛码平台、限时60-90分钟3-5题）、一面白板手写、二面业务场景题里夹带SQL，三种形式至少出现一种。
2. **它最能快速区分"会工具的人"和"有数据思维的人"。** 同样一道"计算7日留存率"，只会 `JOIN + GROUP BY` 的候选人和能主动确认"留存口径是按自然日还是滚动日、匿名设备算不算用户"的候选人，在面试官眼里是两个段位。
3. **它对应日常工作量的50%以上。** 入职后的取数、报表、看板搭建、临时分析需求，底层全是SQL。面试官筛SQL，本质上是在筛"入职后能不能直接干活"。

**【避坑指南】** 很多应届生把精力押在机器学习项目上，SQL停留在"会写SELECT"的水平。真实校招数据：业务向数据分析岗终面候选人的SQL通过率通常只有30%-40%，挂人最多的环节恰恰是最"基础"的SQL，而不是算法。

---

## 二、方言与工具怎么选

### 2.1 主方言：MySQL 8.0

国内校招笔面试的默认方言是 **MySQL 8.0**（少数公司用Hive/Spark SQL做笔试，如字节早期、美团部分部门，现在大多也已迁移到兼容ANSQL语法的引擎）。选择MySQL 8.0作为主方言的理由：

- 支持 **CTE（WITH子句）** 和 **窗口函数**（`ROW_NUMBER()`、`RANK()`、`LAG()`、`SUM() OVER` 等），这是进阶与高级题的核心武器；MySQL 5.7及更早版本不支持，网上老教程还在教"用变量模拟窗口函数"，校招备考不需要学那些。
- 语法接近标准SQL，迁移到Hive/Spark/Presto的成本低。

### 2.2 必须知道的 Hive/Spark SQL 差异（一面加分项）

| 差异点 | MySQL 8.0 | Hive/Spark SQL |
|---|---|---|
| 等值判断 | `=` 且 `NULL` 判断用 `IS NULL` | 同左，但字符串比较更宽松 |
| 日期函数 | `DATE_SUB`、`DATEDIFF`、`DATE_FORMAT` | `date_sub`、`datediff` 同名可用；格式化常用 `substr(dt,1,7)` 截取 `yyyy-MM-dd` 分区字段 |
| 字符串拼接 | `CONCAT(a,b)` | `CONCAT_WS`、`||`（Spark） |
| 类型转换 | `CAST(x AS CHAR)` | `CAST` 支持，另有 `to_date()`、`get_json_object` 等特有函数 |
| 全表扫描成本 | 一般单机，量级小 | 分区表必须带分区裁剪条件（`WHERE dt='2024-06-01'`），否则全表扫描直接被面试官标记为"没跑过生产" |
| 去重计数 | `COUNT(DISTINCT x)` 可用但大数据量会OOM | 大表官方建议 `GROUP BY x 再 COUNT`，或 `approx_count_distinct` |

**【加分点】** 在回答性能类问题时主动说一句："如果这是Hive表，我还会注意分区裁剪和 `COUNT(DISTINCT)` 的改写。" 一句话就能体现你了解生产环境，和纯刷题选手拉开差距。

### 2.3 练习环境

- 本地装 MySQL 8.0 + DBeaver/Navicat，或直接用 [LeetCode数据库题](https://leetcode.cn/problemset/database/)、牛客SQL题库（免费，含"SQL大厂真题"专区）在线练习；
- 本仓库所有SQL均可在 MySQL 8.0 直接运行，示例数据在每题"表结构说明"模块中给出，建表后可自行插入练习。

---

## 三、校招SQL题型地图：8大题型与识别信号

刷题不是越多越好，而是要**按题型建立"条件反射"**。下面是校招SQL的8大题型，每类给出识别信号和核心武器。本库8个行业分册的题目全部落在这8类之内。

| 题型 | 识别信号 | 核心武器 | 难度分布 |
|---|---|---|---|
| ① 基础聚合 | "统计各…的数量/总和/均值/占比" | `GROUP BY` + 聚合函数 + `HAVING` + `CASE WHEN` | 入门 |
| ② 排名与TopN | "每个类目销量前3""消费前20%""排名第N" | 窗口函数 `ROW_NUMBER/RANK/DENSE_RANK`、`NTILE` | 进阶 |
| ③ JOIN组合 | "订单+用户""退款+支付""多表拼指标" | `LEFT JOIN` 语义辨析、预聚合防膨胀 | 入门-进阶 |
| ④ 留存/复购 | "7日留存""当月购买且上月也购买""复购率" | 自JOIN + `DATEDIFF`、LEFT JOIN 判存 | 进阶 |
| ⑤ 连续行为 | "连续3天登录""连续签到""最长连续天数" | `DATE_SUB(日期, ROW_NUMBER())` 分组法 | 进阶-高级 |
| ⑥ 漏斗/路径 | "曝光→点击→下单转化率""五步漏斗" | 多次聚合 + JOIN（或窗口）拼接各步人数 | 高级 |
| ⑦ 同比/环比/滑动 | "环比上月""同比去年""7日移动平均" | `LAG/LEAD`、窗口 `ROWS BETWEEN` | 进阶-高级 |
| ⑧ 复杂口径指标 | "LTV""ROI""贡献度拆解""异动归因" | 多层CTE建模 + 窗口 + 简化DID | 高级 |

**【面试高频】** ④⑤⑦三类的组合题（如"连续7天登录且发生支付的用户，计算其后续30天的ARPU环比变化"）是头部大厂笔试压轴的固定形态，务必把模板练到能默写。

---

## 四、30天刷题路径（配合本仓库使用）

- **第1周（打地基）**：本库4个行业（建议电商+金融+零售+本地生活）的入门级各15题。目标：单题10分钟内写出无bug的第一解法，重点消化每题的"性能分析与优化"模块。
- **第2周（上强度）**：8个行业进阶级。目标：每个题型至少手写过5遍，"面试延伸考点"里的追问能口头回答。
- **第3周（攻难点）**：8个行业高级篇。连续行为、漏斗、滑动窗口三类反复刷；每题先自己写，再对照两种参考解法，体会"同一问题不同技术路线"的取舍。
- **第4周（模拟考）**：随机跨行业抽题，限时60分钟3题；按第五节的考场流程完整走一遍。同时开始背第六节的八股快答。

**【避坑指南】** 不要只"看懂"不动笔。SQL是肌肉记忆型技能，看懂参考解法和独立写对之间隔着至少20遍的练习量。刷本库时，请务必逐题在真实环境跑通。

---

## 五、考场上如何组织答题（读题三步 + 检查清单）

### 5.1 读题三步法（每道题动笔前30秒完成）

1. **确认主体与粒度**：结果表一行代表什么？（一个用户？一个商品？一个"用户×月份"？）粒度搞错，后面全错。
2. **确认口径与边界**：
   - 时间边界："近30天"含今天吗？按自然日还是滚动24小时？
   - 状态过滤：算GMV剔除未支付/已退款订单吗？
   - 去重口径：`COUNT(DISTINCT user_id)` 还是次数？
   - **【加分点】** 在面试中主动把口径假设说出口（"我假设剔除退款订单，如果没有这个要求我会用全部已支付订单"），体现业务感，即使题目没说清楚也不算错。
3. **确认输出格式**：要排序吗？保留几位小数？占比用百分数还是小数？

### 5.2 交卷前检查清单（60秒过一遍）

- [ ] `JOIN` 后有没有因一对多关系导致**指标翻倍**（比如用户表和订单表直接JOIN后COUNT用户数）？
- [ ] `COUNT(字段)` vs `COUNT(*)`：字段为NULL时是否被漏算？
- [ ] 除法有没有防零：`ROUND(a/b, 4)` 改成 `ROUND(a / NULLIF(b,0), 4)` 更稳？
- [ ] 时间比较用 `>= '2024-06-01' AND < '2024-07-01'` 而不是 `BETWEEN`（避免把边界日算两次或漏算时分秒）？
- [ ] 窗口函数里的 `PARTITION BY` 和 `ORDER BY` 是否都写对了？漏掉 `ORDER BY` 的聚合窗口会变成"全范围累计"。
- [ ] 结果有没有按题目要求排序/限制条数？

**【避坑指南】** 校招笔试挂人的第一大原因是"JOIN膨胀导致指标翻倍"，第二大原因是"时间边界差一天"。这两条在写完代码后各花15秒检查，性价比极高。

---

## 六、SQL高频八股20问快答

以下是面试问答环节（不写代码）最常问的20个问题，答案要点控制在面试官30秒能听完的篇幅。

1. **WHERE 和 HAVING 的区别？** WHERE在分组前过滤行（不能用聚合函数），HAVING在分组后过滤组（可以用聚合函数）。执行顺序：FROM→WHERE→GROUP BY→HAVING→SELECT→ORDER BY→LIMIT。
2. **DROP、DELETE、TRUNCATE 区别？** DELETE逐行删、可回滚、可带WHERE；TRUNCATE清空整表、重置自增、快、DDL不可回滚；DROP连表结构一起删。
3. **窗口函数和GROUP BY的区别？** GROUP BY把多行压成一行（明细信息丢失），窗口函数每行都保留、在"额外列"上计算组内聚合/排名/偏移。
4. **RANK、DENSE_RANK、ROW_NUMBER 区别？** 遇到并列时：RANK并列同名次且跳号（1,1,3）；DENSE_RANK并列同名次不跳号（1,1,2）；ROW_NUMBER强制唯一（1,2,3）。取"前N名含并列"用DENSE_RANK+子查询，取"恰好N条"用ROW_NUMBER。
5. **LEFT JOIN、INNER JOIN、FULL JOIN 语义？** LEFT保留左表全部行；INNER只保留匹配行；MySQL不支持FULL JOIN，可用 `LEFT JOIN UNION RIGHT JOIN` 模拟。
6. **JOIN 的底层原理？** MySQL 8.0：小表驱动大表的Nested Loop + Block Nested Loop/Hash Join（8.0.18+）。驱动表选小表、被驱动表关联字段有索引是关键。
7. **为什么大表JOIN要预聚合？** 先按粒度GROUP BY缩容再JOIN，避免"先膨胀后过滤"，扫描量和shuffle大幅下降。
8. **COUNT(\*)、COUNT(1)、COUNT(列) 区别？** 前两者等价（统计行数，包含NULL行），COUNT(列)不计该列为NULL的行。
9. **NULL 参与比较和聚合会怎样？** `NULL = NULL` 结果是NULL不是TRUE；`NOT IN` 子查询里有NULL会导致全空；聚合函数忽略NULL但 `COUNT(*)` 除外；`SUM` 全NULL返回NULL可用 `COALESCE` 兜底。
10. **索引什么时候失效？** 对索引列用函数/运算、隐式类型转换（字符串列传数字）、前导模糊 `LIKE '%x'`、OR 连接非索引列、联合索引不满足最左前缀、优化器判断回表代价高于全表扫描。
11. **什么是回表？什么是覆盖索引？** 二级索引只存索引列+主键，查其他列需回主键索引再查一次=回表；查询所需列全在索引里=覆盖索引（EXTRA显示Using index），可免回表。
12. **UNION 和 UNION ALL？** UNION去重（内部多一次排序/哈希去重），UNION ALL不去重直接拼接。两段数据天然不重复时必须用UNION ALL，性能差数倍。
13. **EXISTS 和 IN 的选择？** 子查询结果集小用IN，外表小用EXISTS（本质是谁做驱动表）；MySQL 8.0优化器已能自动改写，但面试仍考理解。
14. **如何去重保留一条？** `ROW_NUMBER() OVER(PARTITION BY 业务键 ORDER BY 时间 DESC)=1` 筛 rn=1；或GROUP BY 业务键取 `MAX(id)` 再JOIN回表。
15. **"连续N天"问题的通用解法？** 按用户分组对日期排序得rn，计算 `DATE_SUB(日期, INTERVAL rn DAY)` 得到"分组键"，相同分组键即连续区间，再按分组键GROUP BY数天数≥N。这是本库反复出现的模板（见第七节）。
16. **EXPLAIN 主要看什么？** type（至少range，ALL=全表扫描）、key（实际用的索引）、rows（预估扫描行数）、Extra（Using filesort/Using temporary 是危险信号，Using index 是好信号）。
17. **慢SQL怎么排查？** EXPLAIN看执行计划→检查索引与JOIN顺序→检查是否隐式转换→大表是否预聚合/分区→必要时加覆盖索引或改写逻辑。
18. **什么是笛卡尔积？什么时候发生？** JOIN条件缺失或条件写在WHERE外导致两表所有行两两组合，行数=m×n，是大事故级错误，写JOIN永远显式写ON条件。
19. **窗口函数的执行顺序？** 在GROUP BY/HAVING之后、ORDER BY之前执行；因此窗口函数不能直接出现在WHERE里（此时GROUP BY还没执行），需要套一层子查询/CTE再过滤。
20. **一行数据在表中如何判断"最近一次"？** `ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY create_time DESC)=1`；不要用 `MAX(create_time)` 后再关联两次（多一次JOIN），除非面试官追问性能权衡。

**【加分点】** 第15、19、20问是"背八股"和"真懂窗口函数"的分水岭，能主动举出本库高级篇的例子来讲，效果远好于背定义。

---

## 七、通用SQL模板速查（默写级）

以下5个模板覆盖了校招高级题80%的组合形态，建议打印贴墙。示例表统一为：用户行为表 `user_behavior(user_id, dt, action, item_id)`，订单表 `orders(order_id, user_id, dt, amount, status)`。

### 模板1：N日留存率（次日/7日留存通用改法）

```sql
-- 7日留存：新用户在首日后第7天仍有行为的比例
WITH new_users AS (          -- 第一步：圈出每天的新用户
    SELECT user_id, MIN(dt) AS first_dt
    FROM user_behavior
    GROUP BY user_id
),
retained AS (                -- 第二步：行为表自关联，判断第7天是否回访
    SELECT n.first_dt,
           COUNT(DISTINCT n.user_id) AS new_cnt,               -- 每天新增人数
           COUNT(DISTINCT b.user_id)  AS retain_cnt            -- 第7天仍活跃人数
    FROM new_users n
    LEFT JOIN user_behavior b
           ON b.user_id = n.user_id
          AND b.dt = DATE_ADD(n.first_dt, INTERVAL 7 DAY)      -- 精确命中第7天（N日留存）
    GROUP BY n.first_dt
)
SELECT first_dt,
       new_cnt,
       ROUND(retain_cnt / new_cnt, 4) AS retain_rate_7d        -- 留存率，分母为新用户数
FROM retained
ORDER BY first_dt;
-- 变体：滚动留存把等值命中改为 b.dt BETWEEN first_dt+1 AND first_dt+7
```

### 模板2：连续N天行为（连续登录/连续下单通用）

```sql
-- 找出连续3天都有下单的用户
WITH base AS (
    SELECT DISTINCT user_id, DATE(create_time) AS dt      -- 一人一天只留一行（同日多单去重）
    FROM orders
    WHERE status IN ('已支付', '已完成')                   -- 口径：只算有效订单
),
ranked AS (
    SELECT user_id, dt,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY dt) AS rn   -- 组内日期序号
    FROM base
),
grp AS (
    SELECT user_id, dt, rn,
           DATE_SUB(dt, INTERVAL rn DAY) AS grp_key       -- 核心技巧：日期减序号，连续日期得到同一分组键
    FROM ranked
)
SELECT user_id
FROM grp
GROUP BY user_id, grp_key
HAVING COUNT(*) >= 3;                                     -- 同一分组键内天数≥3即存在连续3天
```

### 模板3：组内TopN（每个类目销售额前3的商品）

```sql
WITH item_sales AS (
    SELECT category_id, item_id, SUM(amount) AS gmv       -- 先按类目×商品聚出销售额
    FROM order_items JOIN orders USING(order_id)
    WHERE orders.status = '已完成'
    GROUP BY category_id, item_id
),
ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY gmv DESC) AS rn  -- 类目内排名
    FROM item_sales
)
SELECT category_id, item_id, gmv, rn
FROM ranked
WHERE rn <= 3;                                            -- "恰好前3条"用ROW_NUMBER；"并列都要"换DENSE_RANK
```

### 模板4：同环比（LAG取上期值）

```sql
WITH monthly AS (
    SELECT DATE_FORMAT(dt, '%Y-%m') AS ym, SUM(amount) AS gmv   -- 月度GMV
    FROM orders WHERE status = '已完成'
    GROUP BY DATE_FORMAT(dt, '%Y-%m')
)
SELECT ym, gmv,
       LAG(gmv)  OVER (ORDER BY ym) AS gmv_last_month,          -- 上月GMV
       LAG(gmv,12) OVER (ORDER BY ym) AS gmv_last_year,         -- 去年同月GMV（同比）
       ROUND((gmv - LAG(gmv) OVER (ORDER BY ym))
             / LAG(gmv) OVER (ORDER BY ym), 4) AS mom_rate,     -- 环比=(本月-上月)/上月
       ROUND(gmv / SUM(gmv) OVER (ORDER BY ym
             ROWS BETWEEN 11 PRECEDING AND CURRENT ROW), 4) AS ma12_share  -- 12个月滑动占比示意
FROM monthly
ORDER BY ym;
```

### 模板5：漏斗转化（多步行为人数逐步收窄）

```sql
-- 曝光→点击→加购→支付 四步漏斗（限定同一用户同一天内）
WITH step_users AS (          -- 每一步各自聚出"当天完成该行为的用户集合"
    SELECT dt, COUNT(DISTINCT user_id) AS uv_view   FROM user_behavior WHERE action='曝光' GROUP BY dt
    UNION ALL
    SELECT dt, COUNT(DISTINCT user_id) AS uv_click  FROM user_behavior WHERE action='点击' GROUP BY dt
    UNION ALL
    SELECT dt, COUNT(DISTINCT user_id) AS uv_cart   FROM user_behavior WHERE action='加购' GROUP BY dt
    UNION ALL
    SELECT dt, COUNT(DISTINCT user_id) AS uv_pay    FROM user_behavior WHERE action='支付' GROUP BY dt
)
SELECT dt,
       MAX(uv_view)  AS step1,
       MAX(uv_click) AS step2,
       MAX(uv_cart)  AS step3,
       MAX(uv_pay)   AS step4,
       ROUND(MAX(uv_pay) / MAX(uv_view), 4) AS overall_cvr   -- 端到端转化率
FROM step_users
GROUP BY dt
ORDER BY dt;
-- 严格口径（要求同一用户逐步完成）需按用户分别取各步最早时间再用窗口/JOIN判断先后
```

---

## 八、本目录文件索引

| 行业 | 入门级 | 进阶级 | 高级 |
|---|---|---|---|
| 电商 | `电商/01-入门级.md` | `电商/02-进阶级.md` | `电商/03-高级.md` |
| 金融 | `金融/01-入门级.md` | `金融/02-进阶级.md` | `金融/03-高级.md` |
| 教育 | `教育/01-入门级.md` | `教育/02-进阶级.md` | `教育/03-高级.md` |
| 零售 | `零售/01-入门级.md` | `零售/02-进阶级.md` | `零售/03-高级.md` |
| 游戏 | `游戏/01-入门级.md` | `游戏/02-进阶级.md` | `游戏/03-高级.md` |
| 本地生活 | `本地生活/01-入门级.md` | `本地生活/02-进阶级.md` | `本地生活/03-高级.md` |
| SaaS | `SaaS/01-入门级.md` | `SaaS/02-进阶级.md` | `SaaS/03-高级.md` |
| 物流 | `物流/01-入门级.md` | `物流/02-进阶级.md` | `物流/03-高级.md` |

> 使用建议：只投递某一两个行业的同学，请把目标行业的45题全部刷完，再任选2个邻近行业补充（如投电商可补零售、本地生活；投金融可补SaaS——两者都讲流失与留存；投游戏可补电商——都讲付费与活动）。
