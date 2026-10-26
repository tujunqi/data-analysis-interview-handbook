# Python数据分析全解（Pandas/NumPy）：校招笔试与实习的硬通货

> 校招数据分析岗对Python的要求非常明确：**Pandas做数据清洗与聚合的熟练工，而不是算法工程师**。技术面考察形式通常有三类：①现场手写Pandas（如"用groupby算每个品类月销售额"）；②问概念（merge的四种连接、loc和iloc区别）；③结合简历项目追问代码细节。本篇按数据处理的真实流水线组织：**读取 → 数据结构 → 清洗 → 聚合 → 合并 → 时间序列 → NumPy → 端到端实战**，每个方法配代码+逐行注释+业务场景。
>
> 环境说明：Anaconda（自带pandas/numpy/jupyter）是标准起点；`pip install pandas` 亦可。示例数据均为虚构电商数据。版本：pandas ≥ 1.5（含2.x，注意 `append` 已废弃、用 `concat` 替代——报版本是**【加分点】**）。

---

## 第一节 数据读取：一切从 read_csv 的参数开始

### 1.1 read_csv 高频参数详解

```python
import pandas as pd

# 读取电商订单明细：CSV里常见的坑全在这几个参数里
df = pd.read_csv(
    'orders.csv',
    encoding='utf-8',        # 中文数据常见的坑：报UnicodeDecodeError就改 'gbk'（Windows导出常见）
    parse_dates=['order_date', 'pay_time'],   # 读取时直接把日期列解析成datetime，省一步to_datetime
    dtype={'user_id': str, 'sku_id': str},    # 关键！ID列读成int会丢前导0、还会溢出（18位手机号/订单号）
    usecols=['order_id', 'user_id', 'sku_id', 'order_date', 'amount', 'status'],
                             # 只读需要的列，大文件内存立减一半以上
    nrows=10000,             # 先读1万行试探结构，避免全量加载卡死——处理陌生大文件的第一步
    na_values=['NULL', '', '无'],             # 把这些值识别为缺失值（业务系统常导出"无"）
    thousands=','            # "1,234.5"这种带千分位的金额自动去逗号
)
```

**【避坑指南】** 校招手写题最爱埋两个雷：①ID列被读成数字（前导0丢失、精度丢失）→ `dtype=str`；②日期列是字符串 → `parse_dates` 或后续 `pd.to_datetime`。能在读数据这步就说出这两点，基本可以直接进入下一轮。

### 1.2 其他常用读取

```python
df_excel = pd.read_excel('月报.xlsx', sheet_name='明细', skiprows=2)  # skiprows跳过表头上方的标题行
df_sql = pd.read_sql('SELECT * FROM orders WHERE dt="2024-06-01"', conn)  # 从数据库直读（需SQLAlchemy连接）
df_chunk = pd.concat(pd.read_csv('big.csv', chunksize=100000), ignore_index=True)  # 分块读取超大文件
```

面试问答：**CSV文件10GB内存8GB怎么办？** 三选一：`chunksize` 分块迭代处理；只读 `usecols` 必要列+指定dtype降内存；转Parquet（列式存储，读取快5-10倍）。能答出Parquet直接加分。

### 1.3 读进来先看什么（数据体检三板斧）

```python
df.head()        # 看前5行长什么样，字段是否符合预期
df.shape         # (行数, 列数)，先确认数据量级
df.info()        # 每列类型与非空数——缺失情况、类型错误一眼看出，最常用的体检命令
df.describe()    # 数值列的均值/分位数/最值，扫一眼是否有负数金额、异常极大值
df['status'].value_counts(dropna=False)   # 类别列看分布，含NaN；订单状态分布常在此暴露口径问题
```

**业务场景**：拿到订单表先跑 `value_counts()` 看 `status`——发现"待支付"占40%，说明取数可能没过滤，GMV口径要先和业务确认，这就是"分析前先体检"的意义。

---

## 第二节 核心数据结构：Series、DataFrame与索引选取

### 2.1 两大数据结构

- **Series**：一维带标签数组（一列数据 + 索引）。
- **DataFrame**：二维表格（多个Series共用一个列索引 + 行索引）。可以理解为"带标签的Excel表"，但索引让对齐、切片、合并能力远超Excel。

```python
s = pd.Series([86, 62, 120], index=['A店', 'B店', 'C店'])   # 一维：门店→客单价
df = pd.DataFrame({
    'user_id': ['u001', 'u002', 'u003'],
    'city':   ['杭州', '上海', '北京'],
    'amount': [86.0, 62.5, 120.0]
})
```

### 2.2 loc 与 iloc：面试必考的选取方式

```python
# loc：按"标签"选（行索引名、列名），含两端
df.loc[df['amount'] > 100, ['user_id', 'amount']]   # 布尔条件选行 + 列名列表选列
df.loc[0:2]                                          # 若索引是0,1,2的默认整数，loc[0:2]含第2行（标签闭区间）

# iloc：按"位置"选（第几行第几列），不含末端，纯Python风格
df.iloc[0:2, [0, 2]]    # 前2行 + 第0、2列（不含第2行，左闭右开）
```

**一句话记忆**：**loc用名字，iloc用位置**。**【面试高频】** "loc和iloc区别"问的是：①标签vs位置；②区间开闭性相反。附加分：提醒 `df[mask]` 布尔索引是最常用的日常写法，loc的价值在于"条件选行的同时精确选列"。

### 2.3 布尔索引与查询：日常80%的取数

```python
mask = (df['city'].isin(['杭州', '上海'])) & (df['amount'] >= 50)   # isin替代多or；&不能用and
hot = df[mask].copy()      # copy()避免SettingWithCopyWarning——筛选结果上再改列必须copy
hot = df.query("city in ['杭州','上海'] and amount >= 50")           # query写法更接近SQL，可读性好
```

**【避坑指南】** 两个条件必须各自带括号加 `&`（不能写 `and`）；从切片结果上直接赋值会触发 `SettingWithCopyWarning`，养成 `.copy()` 习惯。这两条是手写代码题的隐形扣分点。

---

## 第三节 数据清洗：占分析时间60%的活

> 校招项目面试最爱问"你的数据脏在哪、怎么处理的"。清洗四大金刚：**缺失、重复、异常、类型**，外加字符串处理。

### 3.1 缺失值：先查因，再选策略

```python
df.isna().sum()                       # 每列缺失计数——第一步永远是"看哪里缺、缺多少"
df.isna().mean().round(4)             # 缺失率；>30%的列先考虑是否整个字段不可用

# 三种处理策略（按业务含义选，不是拍脑袋fillna(0)）
df.dropna(subset=['user_id'])         # ① 关键字段缺失（连用户ID都没有）→ 删行，少量缺失(2%)时合理
df['amount'] = df['amount'].fillna(df['amount'].median())   # ② 金额缺失用中位数补（偏态分布不用均值！）
df['city'] = df['city'].fillna('未知')                       # ③ 类别缺失造"未知"类别，保留记录
df['pay_time'] = df['pay_time'].ffill()                     # ④ 时间序列近邻填充（前向填充），监控数据用
```

**业务场景**：订单表 `coupon_id` 缺失率80%——不是数据坏了，而是"没使用优惠券"的自然表现，正确处理是 `fillna('未用券')` 转成业务含义明确的类别。**【面试高频】** "缺失值怎么处理"满分答案结构：先看缺失机制（随机缺失还是系统性缺失）→ 按字段业务含义选删/补/标记 → 补的话偏态用中位数、类别造未知类 → 最后验证处理前后的分布没有大变形。

### 3.2 重复值

```python
df.duplicated(subset=['order_id']).sum()   # 订单号重复数——订单号理论唯一，有重复必是采集/导出问题
df = df.drop_duplicates(subset=['order_id'], keep='last')  # 保留最后一次状态（常见于状态流转表重复导出）
```

**业务场景**：从数仓导出的订单明细按状态流转可能一行订单多条记录（创建/支付/发货各一条），此时要按业务取"最终状态"：`sort_values('update_time').drop_duplicates('order_id', keep='last')`。区分"完全重复行"（直接删）与"业务键重复但内容不同"（按规则保留）——能讲出这个区别即加分。

### 3.3 异常值：三σ与IQR双引擎

```python
# IQR法（偏态数据友好，如金额）
q1, q3 = df['amount'].quantile([0.25, 0.75])
iqr = q3 - q1
upper, lower = q3 + 1.5 * iqr, q1 - 1.5 * iqr
outliers = df[(df['amount'] > upper) | (df['amount'] < lower)]

# 三σ法（近正态数据，如会话时长（对数变换后））
mean, std = df['duration'].mean(), df['duration'].std()
outliers2 = df[df['duration'].abs() - mean > 3 * std]   # 示意写法，实际用 (df['duration']-mean).abs() > 3*std
```

**处理原则（面试标准答案）**：先查因再动手——录入错误（金额为负、年龄999）→修正或删；真实极端值（大R用户）→**保留但分层分析**，绝不混在整体里算均值；测试订单→按业务规则（测试账号/内部单标记）剔除并记录口径。**"删不删"永远先问"这个值是真的吗、删了影响什么结论"**。

### 3.4 类型转换与字符串处理

```python
df['order_date'] = pd.to_datetime(df['order_date'], errors='coerce')  # 无法解析的变NaT而不是报错中断
df['amount'] = pd.to_numeric(df['amount'], errors='coerce')           # "1,234"或脏值转数字，转不动的标NaN
df['user_id'] = df['user_id'].astype(str).str.zfill(8)                # 统一ID位数（前导0对齐）
df['city'] = df['city'].str.strip().str.replace('市', '', regex=False)  # "杭州市"→"杭州"，维度统一才能聚合

# 时间衍生列（分析最常用）
df['ym'] = df['order_date'].dt.to_period('M').astype(str)   # "2024-06"，月度维度
df['hour'] = df['order_date'].dt.hour                        # 小时段，做时段分析
df['weekday'] = df['order_date'].dt.dayofweek                # 0=周一，配合isin([5,6])标记周末
```

**业务场景**：城市字段"杭州市/杭州/HZ"混写会导致聚合出三个"城市"，先标准化再分组是数据清洗里最常见的实际工作。`errors='coerce'` 的价值：让脏数据"安静地变成NaT/NaN"进入缺失值流程，而不是让整个脚本崩溃。

---

## 第四节 分组聚合：groupby 是 Pandas 的灵魂

### 4.1 groupby 的心智模型：拆-算-合

`df.groupby('city')['amount'].sum()` 内部做三件事：**按city把表拆成若干小表 → 每张小表算amount的和 → 合成一张结果表**。面试被问"groupby原理"就答这三个词（Split-Apply-Combine），并补一句"和SQL的GROUP BY语义一致"。

### 4.2 单层与多层聚合（必会）

```python
# 每个城市的订单数与销售额（一次聚合多个指标）
city_agg = df.groupby('city').agg(
    order_cnt=('order_id', 'nunique'),   # 用命名聚合：结果列名=业务含义，比 'order_id','nunique' 元组式可读
    gmv=('amount', 'sum'),
    avg_price=('amount', 'mean')
).reset_index()                          # groupby键默认变索引，reset_index还原成普通列，下游merge才方便

# 多维分组：城市×月份 销售额交叉表（等价SQL的 GROUP BY city, ym）
month_city = df.groupby(['ym', 'city'])['amount'].sum().unstack()   # unstack把city从行提到列，得到宽表
```

**【避坑指南】** 计数三兄弟别混：`count()` 计非空个数、`size()` 计行数（含NaN）、`nunique()` 去重计数。"下单用户数"必须用nunique——用size会把"一人多单"重复计。

### 4.3 agg / transform / apply 三兄弟（高频概念题）

```python
# agg：返回"每个组一行"的汇总表
df.groupby('city')['amount'].agg(['sum', 'mean', 'count'])

# transform：返回"与原表同长度"的列——把组级结果"广播"回每一行，最适合做"组内对比"
df['city_avg'] = df.groupby('city')['amount'].transform('mean')   # 每行带上所属城市的平均客单价
df['vs_city_avg'] = df['amount'] - df['city_avg']                  # 每单相对城市均值的高低

# 组内排名（窗口函数思维的Pandas写法）
df['rank_in_city'] = df.groupby('city')['amount'].rank(ascending=False, method='dense')

# apply：最灵活也最慢，复杂自定义逻辑才用（能向量化就别apply）
def top2(sub):                                   # 每个城市销售额前2的订单
    return sub.nlargest(2, 'amount')
top_orders = df.groupby('city', group_keys=False).apply(top2)
```

**【面试高频】** "agg和transform区别"标准答案：**返回形状不同**——agg压缩成组级汇总，transform广播回原表行级；要给每行打"所属组的统计值/组内占比"用transform，要出汇总报表用agg。这题考的是是否真理解分组体系。

### 4.4 pivot_table：Pandas 版透视表

```python
pt = pd.pivot_table(
    df, index='city', columns='ym', values='amount',   # 行=城市，列=月份，值=金额
    aggfunc='sum', fill_value=0,                        # 无数据填0（透视表默认NaN）
    margins=True, margins_name='合计'                    # 带行列合计，报表直接可用
)
```

与 `groupby+unstack` 等价，但 `margins`（自动合计）和 `fill_value` 更省事；类别多时列会爆炸，先筛TopN维度。

---

## 第五节 合并与重塑：merge 四种连接 + concat + melt

### 5.1 merge：Pandas 版 JOIN（必考）

```python
# 订单表（明细）关联用户维表（画像）：LEFT JOIN保订单全量
df2 = df.merge(
    users[['user_id', 'city', 'member_level']],   # 维表只取需要的列，防列名冲突+省内存
    on='user_id', how='left'
)

# how 四种连接语义（必须能画图讲清）
# inner：两边都有的键      → 只保留匹配订单
# left ：左全右匹配        → 订单全保留，没匹配的用户画像为NaN（新客不在维表？先查维表更新延迟！）
# outer：两边全保留        → 对账场景用（找互相缺的记录）
# right：右全左匹配        → 少用，等价于交换左右后的left
```

**【避坑指南】merge三大事故**：
1. **行数爆炸**：右边键不唯一（一个user_id在维表出现2次）→ 左表每单被复制2份，GMV翻倍。**防御性检查：`users['user_id'].is_unique` 或merge后比对行数**——这题几乎每场技术面都会以某种形式出现；
2. **列名冲突**：两张表都有 `city` → 后缀 `_x/_y`，join前先重命名；
3. **键类型不一致**：一边str一边int → 匹配全部失败静默返回NaN，merge前统一dtype。

```python
# validate参数是官方防呆：合并前声明键的唯一性，不满足直接报错
df.merge(users, on='user_id', how='left', validate='many_to_one')   # 左多右一，维表重复立刻报错
```

**【加分点】** 知道 `validate` 参数的候选人极少，主动写出可展示"生产级严谨"。

### 5.2 concat：纵向堆叠

```python
jun = pd.read_csv('2024-06.csv'); jul = pd.read_csv('2024-07.csv')
full = pd.concat([jun, jul], ignore_index=True)   # 月度文件纵向拼接；ignore_index重建行索引
```

**与merge的区别一句话**：merge按"键"横向关联，concat按"轴"直接堆叠。拼接后核对总行数=两表行数和（concat不匹配键，错位也不会报错，行数核对是唯一防线）。

### 5.3 melt 与 pivot：长宽表互转

```python
# 宽转长：把"1月 2月 3月"多列 → "月份"一列（画图/入库都要长表）
long_df = wide_df.melt(id_vars=['city'], value_vars=['1月', '2月', '3月'],
                       var_name='月份', value_name='销售额')
# 长转宽：就是pivot/pivot_table/unstack，见4.4
```

**业务场景**：业务方给的Excel是"门店为行、1-6月为列"的宽表，melt转长表后才能groupby/merge/画折线图。"长表是分析友好的形态，宽表是展示友好的形态"——这句总结值得背。**【面试高频】**

---

## 第六节 时间序列处理：同环比、滑动窗口、留存

> 留存、复购、同环比是校招Pandas笔试的三大高峰，全部属于时间序列操作。核心武器：`shift`（取上期）、`rolling`（滑动窗口）、`resample`（重采样）、`expanding`（累计）。

### 6.1 shift：同环比的基石

```python
daily = df.groupby('ym')['amount'].sum().reset_index()   # 月度GMV序列
daily['gmv_prev'] = daily['amount'].shift(1)              # 上一行（上个月）的GMV——shift按位置平移
daily['mom'] = daily['amount'] / daily['gmv_prev'] - 1    # 环比 = 本月/上月 - 1
daily['yoy'] = daily['amount'] / daily['amount'].shift(12) - 1  # 同比 = 平移12位（12个月前）
```

**【避坑指南】** shift前必须先排序（`sort_values('ym')`）并保证序列完整（缺月会错位）——"环比算错90%是没排序或缺期"；首行shift结果为NaN是正常现象，除法防零用 `np.where(denom!=0, ...)` 或直接容忍NaN。

### 6.2 rolling：滑动窗口（7日均线、异常平滑）

```python
daily = daily.sort_values('ym')
daily['gmv_ma7'] = daily['amount'].rolling(7).mean()                      # 7期简单均线，看趋势去噪
daily['gmv_ma7c'] = daily['amount'].rolling(7, min_periods=1).mean()      # min_periods让前6期也有值（不足按已有期算）
daily['gmv_max7'] = daily['amount'].rolling(7).max()                      # 7日峰值——容量/预警场景
```

**业务场景**：GMV日波动大（周末效应、大促），运营看"7日移动平均"判断真实趋势，避免被单日脉冲带偏。rolling+除法还能算"滚动7日复购率"等滚动指标。

### 6.3 resample：重采样（日→周/月聚合）

```python
ts = df.set_index('order_date')['amount']          # resample要求时间作索引
weekly = ts.resample('W').sum()                    # 按周求和（W默认周日为界，'W-MON'周一为界）
monthly = ts.resample('MS').sum()                  # 按月求和（MS=月初起，M=月末止——报表口径要说明）
daily_complete = ts.resample('D').sum().fillna(0)  # 重采样会补出无数据的日期（补0），是修"缺日"的正规方法
```

**【加分点】** 能说出 `'M'` 与 `'MS'` 的区别（月末标签 vs 月初标签）并指出"画月度折线图用MS，否则最后一个点会落在月末当天"，说明你真踩过这个坑。

### 6.4 N日留存：Pandas 完整实现（必背）

```python
# 目标：计算每日新增用户的次日/7日留存率
# 登录表：login(user_id, dt)；dt为日期字符串，先转datetime
login = pd.read_csv('login.csv', parse_dates=['dt'], dtype={'user_id': str})

first_login = login.groupby('user_id')['dt'].min().rename('first_dt').reset_index()  # 每个用户的首次登录日
merged = first_login.merge(login, on='user_id')                                       # 用户所有登录记录拼上首登日
merged['days_since'] = (merged['dt'] - merged['first_dt']).dt.days                    # 距首登天数

ret = merged[merged['days_since'].isin([1, 7])].groupby(['first_dt', 'days_since'])['user_id'].nunique().unstack()
ret = ret.rename(columns={1: 'next_day', 7: 'day7'})
cohort = first_login.groupby('first_dt')['user_id'].nunique().rename('new_users')
report = cohort.to_frame().join(ret).assign(
    次留率=lambda x: (x['next_day'] / x['new_users']).round(4),
    七日留率=lambda x: (x['day7'] / x['new_users']).round(4)
)
```

**讲解要点（面试口述稿）**："我按'先定义、再拼表、后算率'三步走：先算每个用户的首次登录日形成同期群，再计算每次登录距首登的天数，最后在第1天、第7天分别数回访用户数除以当日新增。分子分母都用nunique去重，这是留存口径最容易错的地方。"——这段话能把SQL模板1的思路用Pandas复述一遍，SQL/Pandas双修的形象立住了。

---

## 第七节 NumPy 速览：向量化思维

```python
import numpy as np
arr = np.array([1, 2, 3, 4, 5])

arr * 2                     # 向量化运算：无需for循环，全数组同时乘2（底层C实现，快百倍）
(arr > 3).sum()             # 布尔数组求和=满足条件的元素个数（True当1用）
np.where(arr > 3, '高', '低')  # 向量版if-else，Pandas造标签列常用：df['档'] = np.where(df['amount']>100, '高', '低')
np.clip(arr, 2, 4)          # 截断到[2,4]：清洗时把离谱值压回边界（winsorize缩尾的雏形）
np.random.default_rng(42)   # 新版随机数生成器（固定种子保证可复现——实验/造数必带）
```

**面试问答**：**为什么Pandas快？** 底层是NumPy的C数组连续存储+向量化运算，避免Python逐元素循环；**为什么还要少用apply？** apply对每行调用Python函数，性能接近循环，能转成向量化表达式（`np.where`、算术运算）就转。**【加分点】** "先向量化、apply是兜底"是Pandas性能观的一句话总结。

---

## 第八节 端到端实战：一条完整的分析管道

把前面所有知识串成"拿到原始CSV→产出经营结论"的标准流程。场景：分析6月各渠道的销售表现与客单价结构。

```python
import pandas as pd
import numpy as np

# ===== 第1步：读取与体检 =====
df = pd.read_csv('orders_202406.csv', parse_dates=['order_date'],
                 dtype={'user_id': str, 'order_id': str})          # ID读str防精度丢失，日期边读边解析
print(df.info(), df['status'].value_counts(dropna=False))          # 体检：类型/缺失/状态分布

# ===== 第2步：口径过滤 =====
paid = df[df['status'].isin(['已支付', '已完成'])].copy()            # GMV只算有效订单；copy()防告警

# ===== 第3步：清洗 =====
paid = paid.drop_duplicates(subset=['order_id'], keep='last')      # 订单号唯一化
q1, q3 = paid['amount'].quantile([.25, .75])
iqr = q3 - q1
paid = paid[paid['amount'].between(q1 - 3 * iqr, q3 + 3 * iqr)]    # 极端异常单放宽3倍IQR再截（保留大促真实大单）
paid['city'] = paid['city'].str.strip().str.replace('市', '', regex=False)  # 城市标准化

# ===== 第4步：维度衍生与关联 =====
paid['ym'] = paid['order_date'].dt.to_period('M').astype(str)      # 月维度
users = pd.read_csv('users.csv', dtype={'user_id': str})
paid = paid.merge(users[['user_id', 'channel', 'member_level']],
                  on='user_id', how='left', validate='many_to_one') # 关联画像并防维表重复

# ===== 第5步：分组聚合出结论 =====
summary = paid.groupby('channel').agg(
    orders=('order_id', 'nunique'),
    users=('user_id', 'nunique'),
    gmv=('amount', 'sum'),
).assign(
    avg_price=lambda x: x['gmv'] / x['orders'],                    # 客单价=GMV/订单数（口径一致）
    gmv_share=lambda x: (x['gmv'] / x['gmv'].sum()).round(4)       # 渠道GMV占比
).sort_values('gmv', ascending=False)
print(summary)

# ===== 第6步：新老客结构（验证"客单价下降是不是新客稀释"）=====
paid['first_order'] = paid.groupby('user_id')['order_date'].transform('min')  # 每单拼上该用户首单时间
paid['user_type'] = np.where(paid['order_date'] == paid['first_order'], '新客', '老客')
structure = paid.pivot_table(index='channel', columns='user_type',
                             values='amount', aggfunc='mean').round(1)        # 渠道×新老客客单价
print(structure)
```

**这段代码覆盖的面试考点**：读参数防雷（dtype/parse_dates）、口径过滤、去重、IQR异常、字符串标准化、merge+validate、命名聚合、占比与客单价的派生指标、transform判断新老客、透视表。面试时能边写边讲每步"为什么"，就是"真做过项目"的信号。

---

## 第九节 收尾：Pandas面试高频问答TOP10

1. **loc与iloc区别**——标签vs位置，区间开闭相反（第二节）。
2. **merge四种how的语义 + 行数爆炸防御**——第五节（validate/查唯一）。
3. **agg/transform/apply区别**——返回形状不同，transform广播回行级（第四节）。
4. **缺失值处理流程**——先看机制再选策略，偏态用中位数（第三节）。
5. **count/size/nunique区别**——非空计数/行数/去重计数（第四节）。
6. **同环比/滑动窗口怎么写**——shift+除法、rolling（第六节）。
7. **N日留存思路**——首登同期群+距首登天数+nunique分子分母（第六节）。
8. **SettingWithCopyWarning是什么**——切片视图副本歧义，`copy()`后操作。
9. **大文件读不进内存怎么办**——chunksize/usecols/dtype/Parquet（第一节）。
10. **Pandas和SQL的关系**——语义高度对应（merge=JOIN、groupby=GROUP BY、pivot_table=聚合+PIVOT）；海量数据SQL先行、Pandas做本地探索与建模前处理。答出这个定位，面试官就知道你理解工具边界。

**【全篇完】**
