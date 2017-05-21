# 性能优化之SQL

### 背景

​	根据查询条件筛选符合需求的日志，以当前登录的用户为切入点，若为首席交易员则需要查询该首席交易员关联的交易分组下的所有交易做的交易，若为普通交易员则只查询该交易员做的交易。数据库中交易日志的每一条数据都是从taker方来看的，所以需要对查询的数据根据当前登录的交易员的身份进行转换，例如数据库中一条记录如下：

| TKR_CD  | CP_CD   | DIR  | AMT  | ROLE  | ID   |
| ------- | ------- | ---- | ---- | ----- | ---- |
| yc@abci | yc@icbc | B    | 4523 | maker | 1    |
| yc@hbcb | yc@bcoh | S    | 5678 | taker | 2    |

如果当前登录系统进行查询的交易员是yc@abci,在不做处理的情况下他看到的交易方向是"B"，而实际上该交易员在交易单号为1的交易中是maker，所以对其而言真实交易方向是"S".

### 初级版

​	由于在进行SQL查询之前无法获取当前登录的交易员在每笔交易中的身份(**taker** or **maker**)，需要根据身份进行转换的查询条件（如 方向）没有进行处理，所以需要对查询出的结果进行如下处理:

- 用单独的SQL进行身份判断，循环遍历
- 用单独的SQL进行机构码转换，循环遍历
- 用单独的SQL进行拆入拆出总额的计算
- 用单独的SQL进行交易总数的判断


### 增强版

- 需要在循环中重新查库确定的数据在第一次SQL查询中通过**left join**查出字段结果
- 需要进行拆入拆出总和判断的数据在第一次SQL查询中通过**case when**和**sum()**算出结果作为新的字段展示

### 最终版

- SQL中所有进行了**left join**的字段增加索引
- SQL中的视图替换成普通表的**union**，避免索引无效
- **trunc()**函数修改，避免索引无效，因为在索引列上使用函数时不会用到索引



```mysql
with myFlrs as
	(select FLR_CD,CHF_TRDR_INDCTR
    	FROM a
    where usr_cd = ?)
select t.*,
	(case when t.dir = 'S' then t.LNDNG_AMT end) SAMT,
	(case when t.dir = 'B' then t.LNDNG_AMT end) BAMT,
	sum(case when((t.asofrole = 'taker' and t.dir = 'S') or (t.asofrole = 'maker' and t.dir = 'B')) then t.LNDNG_AMT else 0 end)) over() as S_SUM,
	sum(case when((t.asofrole = 'taker' and t.dir = 'B') or (t.asofrole = 'maker' and t.dir = 'S')) then t.LNDNG_AMT else 0 end)) over() as B_SUM
from(
	select t1.*,
		   t2.flr_cd as tkrFlrCd,
		   t3.flr_cd as cpFlrCd,
		   case when f1.instn_cd = ? then 'taker' else 'maker' end asofrole
		from((select * from V) t1 left join myFlrs t2 on
            t1.tkr_flr_cd = t2.flr_cd left join flr_bsc_info f1 on 
            t1.tkr_flr_cd = f1.flr_cd)) t
where 1=1
and ...
```

### 终结版

- 进行Left join 的表增加过滤条件(更熟悉需求带来的性能提升)
- 进行Left join之前进行条件筛选(**where**)，而不是对join之后的表进行条件筛选