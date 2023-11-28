---
layout: post
title: "stock/future data access (gambling)"
author: "melon"
date: 2023-06-27 21:09
categories: "2023"
tags:
  - gambling
---

### # realtime resources
concise data (v_s)
```text
url: http://qt.gtimg.cn/q=s_sh600519

data: v_s_sh600519="1~贵州茅台~600519~1728.38~17.33~1.01~18574~319929~~21711.87~GP-A";

format:
market  1: sh, 51: sz, 100: hk, 200: us
stock name
stock code
current price
gain/loss
gain/loss(percent)
volume
amount
-
market capitalization
```

detailed data (v_)
```text
url: http://qt.gtimg.cn/q=sh600519

data: GET url

format:
market  1: sh, 51: sz, 100: hk, 200: us
stock name
stock code
current price
prev close
current open
volume
waipan
neipan
buy1
buy1 volume
buy2
buy2 volume
buy3
buy3 volume
buy4
buy4 volume
buy5
buy5 volume
sell1
sell1 volume
sell2
sell2 volume
sell3
sell3 volume
sell4
sell4 volume
sell5
sell5 volume
request time
gain/loss
gain/loss (percent)
highest
lowest
current price/volume(100)/amount(yuan)
volume(100)
amount(yuan)
turnover rate
ttm shiyinglv
-
highest
lowest
zhenfu
liutongshizhi
zongshizhi
shijinglv
zhangtingjia
dietingjia
liangbi
-
junjia
dongtaishiyinglv
jingtaishiyinglv
-
-
-
chengjiaoe
...
```

combine request data, the returned data is separated by semi-colon.
```text
url: http://qt.gtimg.cn/q=s_sh600519,s_hk08270
url: http://qt.gtimg.cn/q=sh600519,hk08270
```

failed request:
```text
url: http://qt.gtimg.cn/q=s_sg514

data: v_pv_none_match="1";
```

pankoufenxi
```text
url: http://qt.gtimg.cn/q=s_pksh600519

data: v_s_pksh600519="0.411~0.166~0.299~0.124";

format:
buypandadan
buypanxiaodan
sellpandadan
sellpanxiaodan
```

realtime zijin liuxiang
```text
url: http://qt.gtimg.cn/q=ff_sh600519

data: GET url

format:
stock code
zhuli liuru
zhuli liuchu
zhuli jingliuru
zhuli jingliuru / zijin liuruliuchu zonghe
sanhu liuru
sanhu liuchu
sanhu jingliuru
sanhu jingliuru / zijin liurulliuchu zonghe
zijinliuruliuchu zonghe = (zhuli) liuru + liuchu + (sanhu) liuru + liuchu
-
-
stock name
date
...
```

<hr>

### # day resources
```text
# basic website
https://web.ifzq.gtimg.cn/appstock/app/fqkline/get

# param=stock code,day,start_date,end_date,trading_days,qfq/hfq
# trading_days: number of days for deriving data
# qfq: qianfuquan
# hfq: houfuquan

# examples
us AAPL (.0Q)
https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param=usAAPL.OQ,day,2020-3-1,2021-3-1,500,qfq
sh 600519 (no .OQ)
https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param=sh600519,day,2020-3-1,2021-3-1,500,qfq
sh 600519 (no .OQ)
https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param=sh600519,day,,,2,qfq
hk 01810 (no .0Q)
https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param=hk01810,day,2020-3-1,2021-3-1,500,qfq

# data desc: date, open, close, high, low, total lots(1 lot = 100 shares)
["2021-03-10", "1977.000", "1970.010", "1999.870", "1967.000", "51172.000"]

# another url to request kline data
https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?_var=kline_dayhfq&param=sh600519,day,,,320,hfq
```

<hr>

### # min data resources (1min, 5min, 10min...)
```text
# url:
https://web.ifzq.gtimg.cn/appstock/app/minute/query?code=sh600519
# 1min url (outdated)
https://web.ifzq.gtimg.cn/appstock/app/kline/mkline?param=sh600519,m1,,32
# 5min url (outdated)
https://web.ifzq.gtimg.cn/appstock/app/kline/mkline?param=sh600519,m5,,32
# 15min url (outdated)
https://web.ifzq.gtimg.cn/appstock/app/kline/mkline?param=sh600519,m15,,32
# 30min url (outdated)
https://web.ifzq.gtimg.cn/appstock/app/kline/mkline?param=sh600519,m30,,32
# 60min url (outdated)
https://web.ifzq.gtimg.cn/appstock/app/kline/mkline?param=sh600519,m60,,32
# 5day (min) url
https://web.ifzq.gtimg.cn/appstock/app/day/query?code=sh600519

# us stock, the .0Q is optional
https://web.ifzq.gtimg.cn/appstock/app/dayus/query?code=us.DJI
https://web.ifzq.gtimg.cn/appstock/app/dayus/query?code=usGOOD.OQ
```

<hr>

### # kline gif resources
```text
# day kline
http://image.sinajs.cn/newchart/daily/n/sh601006.gif
# realtime kline
http://image.sinajs.cn/newchart/min/n/sh601006.gif
# weekly kline
http://image.sinajs.cn/newchart/weekly/n/sh000001.gif
# monthly kline
http://image.sinajs.cn/newchart/monthly/n/sh000001.gif
```

<hr>

### # ctp resources
simnow: https://www.simnow.com.cn  
sfit: http://www.sfit.com.cn/5_2_DocumentDown_3.htm  

<hr>

### # 雪球
https://stock.xueqiu.com/v5/stock/realtime/quotec.json?symbol=SH601003,SZ001389

<hr>

### # pytdx
https://pytdx-docs.readthedocs.io/zh_CN/latest/pytdx_reader/

<hr>

### # 腾讯行情
上证指数：http://qt.gtimg.cn/q=s_sh000001  
道琼斯指数：http://qt.gtimg.cn/q=s_usDJI  
腾讯济安：http://qt.gtimg.cn/q=s_sh000847  
恒生指数：http://qt.gtimg.cn/q=s_r_hkHSI  
获取实时资金流向：http://qt.gtimg.cn/q=ff_sh600519  
获取盘口分析：http://qt.gtimg.cn/q=s_pksh600519  
获取简要信息：http://qt.gtimg.cn/q=s_sh600519  

另外，一些来源网址，一起记录备忘
行情接口A股篇 ---知乎的汇总贴
https://zhuanlan.zhihu.com/p/40802537


新浪等市场免费实时股票行情数据接口
http://www.joeychou.me/blog/34.html#price


腾迅股票数据接口 http/javascript
http://www.joeychou.me/blog/35.html


上证
http://qt.gtimg.cn/q=sh600000


深证
http://qt.gtimg.cn/q=sz000001


香港
http://qt.gtimg.cn/q=hk00001


基金
http://qt.gtimg.cn/q=jj000001


新浪期货数据接口(转)
https://blog.csdn.net/dodo668/article/details/82382675


新浪期货接口
https://blog.csdn.net/qq_37193537/article/details/89359425


新浪期货行情接口
https://www.cnblogs.com/fljie/p/7651089.html


股票接口
https://www.cnblogs.com/mxhmxh/p/10278644.html


股票数据API整理--- 包括：数据超市、雅虎、新浪、Google、和讯、搜狐、ChinaStockWebService、东方财富客户端、证券之星、网易财经
http://blog.sina.com.cn/s/blog_afae4ee50102wu8a.html

