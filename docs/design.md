# 设计理念  
* 简单  
* 统一  
* 容易扩展  
* 正交  
* 实用主义(适当冗余)

# 目标
* 容易稳定快速的获取高质量的金融数据  
* 容易灵活可靠的研究,回测，交易  

# 手段  
* 中间层,先定义数据格式,爬虫去适应数据格式,connector去结构化存储之
* 统一存储,股票,商品,期货等的交易数据格式统一
* 统一计算,技术指标的通用性,不依赖特定标的
* 策略框架与交易gateway解耦

# 数据分类
1. security(交易标的)  
stock  
future  
bond  
fund  
分为
  * 交易数据  
    * 基础数据:tick  
    * 生成数据:k线，均线，macd等技术指标
  * 基本面数据  
    * 基础数据:  
      stock->资产负债表,利润表，现金流量表  
      future->商品库存，生产成本  
      bond->借贷方的现金流
    * 生成数据:  
      stock->PE,PB,流动比率等  
      future->
  * 消息面数据  
    * 业绩预告  
    * 概念  　

  可以对基础数据和生成数据进行任意计算，从而生成自己需要的数据
2. 宏观数据  
gdp cpi pmi 利率等  
房价 房屋成交量  

# 数据结构
1. security  
  * code  
    习惯称呼，比如000338指的是潍柴动力，但在整个市场里面这个标识不一定唯一，比如某些基金代码可能跟某些股票代码一样
  * exchange  
    所属交易所  
  * type  
    证券的类型,可选值:stock,future
  * id  
      对于交易标的，有一个全局唯一的id  
      id = type_exchange_code
2. k线  
  * 前复权(qfq)，后复权(hfq)，不复权(bfq)  
    三种类别都需要存储，前复权方便从现在的角度去选股，而对于持仓市值的变化需要用后复权去计算。  
  * 默认存储日线级别,其他级别可以根据tick进行计算  
  * csv结构
```
  timestamp,code,low,open,close,high,vol,turnover,factor
  19991230,600000,24.90,24.65,24.75,24.99,2333200,57888237,1
```
3. tick
  * csv结构  
```
  timestamp,price,vol,turnover,direction(买:1,卖:-1,中性:0)
  14:59:55,10.02,407,408014,0
```

# 技术选型
* 数据库(存储)
本质上，所有数据都是时间强相关的，所以时间索引是必须的;从回测和实时交易的角度，数据应该是时间有序的;  
结论:  
一般时间相关的数据存es,这种数据作为选股或者交易辅助，能够通过指定时间，快速查询，包括原始数据和一些经过复杂计算得到的数据。  
tick和k线数据存kafka,可快速进行消费(回测，交易)  
交易框架可以以kafka为基础，对相应级别的k线数据或者tick数据进行消费,然后使用时间相关函数查询指标， 进行交易  
也可以使用时间相关函数进行计算， 产生交易信号， 再去消费行情数据  

* 计算框架  
在kafka存储行情数据的前提下，可以对其进行相应计算，从而产生相应指标，该类指标称为技术指标，可以将其存储到es;  
es存储的相应数据也可以进行再加工，并进行存储;  

* 监控
grafana配合es可以非常灵活的设置各种监控

# 计算  
* 技术指标  
输入:一段时间的量价  
输出:计算结果  
由于tick,日K的存储采取"文件","数据库"等多种方式,相应的，计算也可以有多种方式.  
  * 文件  
  优点:简单，可用pandas直接计算  
  缺点:多品种关联计算太麻烦;基本上只适合"个人""单机"    
  结论:可提供简单的计算，或者作为中间计算后将结果存到数据库  

  * 数据库  
  优点:并发，容错，事务等保障  
  缺点:复杂  
  结论:用于复杂计算  

  计算结果的再存储，相当于"流"的视图.  
  交易时段的实时计算基于tick,并影响需要计算的指标.  

* 策略运行  
策略可以设定起始和结束时间，如果没有结束时间，会一直运行,也可以设置休息的periods  
对于策略来说，可以获取(timestamp<=current_time;level>=current_level)的任意"指标"  
两个考虑:保证不使用未来函数;大级别是由小级别构成的，在小级别里面可以获取"变化的"大级别指标  

# 架构图
1.为了最大的灵活性，以文件(json,csv等)的形式存储历史数据，并提供脚本对其进行结构化(to kafka,es);在历史数据校验ok后，实时数据的生成同时存文件和数据库  
2.数据管理,检查数据的完整性，准确性  

# 问题　　
* 数据源?  
爬虫(新浪财经,东方财富等),UGC,人工录入, 购买
* 数据完整性算法  
* 需要存文件作为中间状态吗？  
对于实时性要求高的，不适合存文件；
对于结构固定的历史数据，可以存文件。　　
爬数据存文件可以不依赖kafka，但提供脚本来结构化数据:
file->kafka->flink->elastic search

#  能做什么？
在全市场，多维度的视角下
* 快速生成自己需要的数据，并提供统一的访问接口和图表展示  
* 自定义监控和告警  
* 灵活的交易策略和回测  
* 云平台下大数据，AI能力
* 垂直搜索,能回答此类问题：中国2017年融资额，A股分红最多的公司
