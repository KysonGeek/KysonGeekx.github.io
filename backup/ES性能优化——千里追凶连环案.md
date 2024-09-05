> ES支持多种查询模式，其中应用较为广泛的是模糊查询，它也导致多种性能问题。读完此文后，你将会明白模糊查询在ES引擎中的底层实现逻辑，学会系统性排查ES性能问题，并能更好的设计ES-mapping、优化业务系统使用的ES-dsl语句。

> 前情提要：为了解决召回服务查询dws慢的问题，我们引入了es，将hive表的数据按天分片导入es中，在这期间，凶手也悄悄潜入。通过本文，让我们来揭晓凶手的真实面目
>
> ES检索相关的名词解释：
>
> keyword——字符串类型，不支持分词（一般用于等值搜索）
>
> text——字符串类型，默认按照空格分词（一般用于模糊搜索）
>
> wildcard（字段类型）——通配符类型，分词+doc_value 支持正则匹配（用于正则匹配）
>
> wildcard（搜索关键词）—— 正则匹配，可以搭配text和wildcard（字段类型）使用
>
> doc_value——写入es的原始文档

# 1. 案发现场

策略同学报案：数据索引服务偶尔超时报错
![image](https://github.com/user-attachments/assets/480cc9d2-664d-45d2-a620-35e6b3cab15f)

<div align="center">
图一：案发现场十点
</div>

![image](https://github.com/user-attachments/assets/f750fe74-4150-4f9a-93cf-869ec4030168)



# 2. 案件分析

针对接口超时我们进行了内部分析，主要得出以下几个结论

- hive同步es时并发量设置的比较大，导致cpu飙升到80+

![image](https://github.com/user-attachments/assets/413e866e-20f1-4a81-8216-223c024470af)

- es服务申请时，一共四个pod，每个pod规格：16核36G。当时硬盘申请了2T，按照es内存/索引(500G)大小 = 2/3的规范来看，申请的es-pod规格低了
- 梳理策略组使用的字段时判断cert_name不会用于模糊搜索，所以当时es索引使用了keyword（不分词，主要用于等值搜索），通过日志发现，此字段也用了模糊搜索，导致es查询性能下降

![image](https://github.com/user-attachments/assets/51d9a374-48c5-4993-8b46-252e773a64b4)

- 视频召回时，调用es索引服务拼写的条件不符合常理。例如：

（cert_name like '%用车%' or cert_name like '%汽车%' or cert_name like '%车%'）

![image](https://github.com/user-attachments/assets/d69d26bb-b3b3-4293-8aeb-a177e4891e9c)

# 3. 锁定&捉拿凶手

针对上面的问题我们一一给出了逮捕方案

- 降低hive同步es的并发量，从3k改成了1.5k 

![image](https://github.com/user-attachments/assets/b9effe61-73a5-4d6f-b0c8-1f523d23e710)

- 提升pod规格，16U32G ->32U64G
- 修改索引，将cert_name的字段类型由keyword改成wildcard
- 告诉策略同学，修改不符合常理的搜索条件。将超时时间由2s设置成10s

# 4. 隐藏在背后的真凶

> 初步解决方案实行后，hive同步es时，不再会有大规模的超时了，拨开云雾后，凶手也随之漏出了真面目

| 字段类型 | 是否支持分词 | 模糊搜索关键词  | 优点                                           | 缺点                                                         | 备注                                                         |
| -------- | ------------ | --------------- | ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| keyword  | 否           | wildcard        | 等值搜索快，索引存储小                         | 支持等值，模糊搜索性能差、分词搜索                           | 适用于行业，标签等                                           |
| text     | 是           | match，wildcard | 支持自定义分词器，支持模糊搜索，支持纠错搜索等 | 索引存储大，模糊搜索可能不精准(默认按照空格分词，可以指定分词器，使用match搜索时只会对分词结果进行匹配；使用wildcard检索会导致cpu飙升不建议使用) | 适用于对于模糊搜索结果要求不严格的情况（允许部分结果，不符合模糊匹配） |
| wildcard | 是           | wildcard        | 新特性，优化了模糊搜索，支持正则               | 索引存储大，单字，双字模糊搜索性能差，特耗cpu                | 当前采用此方案                                               |

*模糊搜索字段类型选型对比*

上线一段时间后，发现每天都会有零星的接口超时，这次深入分析了一下耗时原因，定位在了单字模糊搜索。虽然 es7.9 之后新增了wildcard类型的字段进行优化正则匹配，但是各种野史还是不推荐使用wildcard进行搜索，主要一个原因是太耗cpu。在敢为极致字节范附体的情况下，又进一步探究wildcard的实现逻辑

> 刚定位到单字模糊搜索时，和策略同学沟通过“能否去掉单字模糊搜索”。策略同学给出数据，去掉单字模糊搜索后会对采纳率有影响[单字匹配搜索问题](https://bytedance.feishu.cn/docs/doccnTdrVliyKBZ79D7IWfKu9uf) 

在了解wildcard之前，我们先了解一下es用到的分词器n-gram

![499ffc70-e776-41e8-8c29-319c224ff4f4](https://github.com/user-attachments/assets/e31e6699-1fc3-4f88-81bb-5deaa3fa7e28)

| n-gram | 示例数据              | 分词结果                        | 备注 |
| ------ | --------------------- | ------------------------------- | ---- |
| 1-gram | 你们，还好吗          | ["你","们","还","好","吗"]      |      |
| 2-gram | ["你们""还好","好吗"] |                                 |      |
| 3-gram | ["还好吗"]            | match搜索“你好”，能不能搜出来？ |      |

```
真実はいつもひとつ
```

在使用profile命令解析wildcard的查询逻辑时现了背后的真凶

| 搜索条件              | profile解析的结果                                            | 搜索条件拆解后的执行顺序                                     |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 单字模糊搜索 "们"     | ![image](https://github.com/user-attachments/assets/ff3a279d-d984-43bb-99f9-24175a1b01ed) | match_all拿出原始数据进行正则匹配                            |
| 俩字模糊搜索 "你们"   |![image](https://github.com/user-attachments/assets/8e577a62-dd5b-49f1-825d-fe9e860732ed) | 前缀匹配"你们*"拿出原始数据进行正则匹配                      |
| 仨字模糊搜索 "你们好" | ![image](https://github.com/user-attachments/assets/9f16aa64-0609-407c-8bb1-7b90a8f7b7ae) | term匹配"你们好"拿出原始数据进行正则匹配                     |
| 多字模糊搜索          | ![image](https://github.com/user-attachments/assets/8210fa18-f250-44aa-a708-ad250c47c77c) | 将搜索语句拆分成多组三个字的数据进行term匹配拿出原始数据进行正则匹配 |

通过上面表格，我们有充分的理由推断出：wildcard类型字段在存储时会将数据按照3-gram分词+doc_value（原始的文档数据）

理由如下：

- 搜索词性能比对，term（等值搜索）>前缀匹配
- 模糊搜索三个字时用的是term
- 模糊搜索超过三个字时，将搜索词拆成长度3的多个搜索词，并进行term
- 第一次匹配后，再进行精准匹配，why？

🌰 es字段中存储了一段文字：“社会主义核心价值观之图书会主流”

> 分词后的结果：社会主、会主义……图书会、会主流

当我模糊搜索“社会主流”时，es内部进行了如下步骤

1. 当进行到按照3-gram分词时，“社会主流”被分为“社会主”和“会主流”
2. 进行term匹配的时候，会将“社会主义核心价值观之图书会主流”匹配出来
3. 我们发现“社会主义核心价值观之图书会主流”中并没有包含“社会主流”这四个字
4. 所以最后一步需要对doc_value进行正则匹配

此处提出两个问题供大家思考

_1. 当模糊搜索关键词长度是3的情况下，是否还需要进行doc_value正则匹配？
2. 当模糊搜索关键词是“主流”时，es会先进前缀匹配“主流*”，但是“社会主义核心价值观之图书会主流”在分词时，由于主流是最后两个字，所以分词结果的最后一个三字结果是“会主流”，请问：当我搜索“主流”时能否搜出“社会主义核心价值观之图书会主流”？_

<img width="752" alt="image" src="https://github.com/user-attachments/assets/0db1ac64-31f8-4f77-8a64-de6726e43f07">

# 5. 逮捕计划

真凶的面目已经完全裸露出来了

- 单字模糊搜索，matchAll对所有的doc_value进行正则匹配
- 俩字模糊搜索，正则匹配前缀，对筛选出来的doc_value进行正则匹配
- 仨个字以上模糊搜索，term匹配，对筛选出来的doc_value进行正则匹配，性能还行。不算凶手

当我们一步一步分析到此时，逮捕方案也慢慢水到渠成，既然wildcard只用了3-gram，解决了大于两个字的模糊搜索，那我们为什么不用1~2gram来解决三个字以下的模糊搜索呢（能想到此方案有一个必备前提，es中可以对一个字段采用不同分词器分别解析存储）

随后我们设计了新的es-mapping。以“primary_name”举例，既要用wildcard分词，也要用ngram分词

```JSON
//mapping
{
    "primary_name": {
        "type": "wildcard",
        "fields": {
            "ngram": {
                "type": "text",
                "analyzer": "ngram_analyzer" 
            }
        }
    }
}
//setting
{
    "ngram_tokenizer": {
        "token_chars": [ //将以下字符也包含在分词中
            "letter",
            "digit",
            "whitespace",
            "punctuation",
            "symbol"
        ],
        "min_gram": "1",
        "type": "ngram",
        "max_gram": "2"
    }
}
```

这样设计后

- 对于三个字及以上的模糊搜索我们采用wildcard（term+doc_value）的方式

```JSON
//query dsl
{
  "query": {
    "wildcard": {
      "primary_name": {
        "wildcard": "*社会主义好*"
      }
    }
  }
}
```

- 对于一个或者两个字的模糊搜索我们采用term的方式

```JSON
//query dsl
{
  "query": {
    "term": {
      "primary_name.ngram": {
        "value": "社会"
      }
    }
  }
}
```

# 6. 真凶授首

在测试集群hl实施方案后，进行了一下es查询耗时对比

| 查询方式                 | 耗时  | 证明                                                         | 备注           |
| ------------------------ | ----- | ------------------------------------------------------------ | -------------- |
| 旧的wildcard双字模糊搜索 | 6s    |![image](https://github.com/user-attachments/assets/5b3f6378-959c-481a-999c-986303e0dbb1) | 耗时相差800+倍 |
| 新的term双字模糊搜索     | 6.8ms |![image](https://github.com/user-attachments/assets/16053b19-08d0-4c84-a6ba-cd8fe7a1a6ed) |                |
|                          |       |                                                              |                |
| 旧的wildcard单字模糊搜索 | 700ms | ![image](https://github.com/user-attachments/assets/7e7743bc-bc6b-4f54-a387-647d2e47c867) | 相差100+倍     |
| 新的term单字模糊搜索     | 5.5ms | ![image](https://github.com/user-attachments/assets/60c4c905-2ba2-46aa-97cd-b794806b667d) |                |

上表中对比的仅仅是一个字段的模糊搜索。实际使用中，一次查询中可能会包含多个模糊查询

**上线后的真实效果**

![image](https://github.com/user-attachments/assets/4d178527-4e96-4d29-940b-69a06e9638a6)
<div align="center">
09-15（上线前），99线
</div>


![image](https://github.com/user-attachments/assets/a157a67b-eb4c-49e9-9ad0-e299aeecffac)
<div align="center">
09-16（上线后）,99线
</div>


| 监控时间           | MAX       | p99       |
| ------------------ | --------- | --------- |
| 09-15，09:00~19:00 | 3.53 s    | 631.12 ms |
| 09-16，09:00~19:00 | 592.07 ms | 180.71 ms |

从上面的监控数据可以看出查询接口

1. p99耗时降低约 70% = 1 - ( 180ms / 600ms )
2. 不再有突刺，MAX不会超过1s，降低了约  80% = 1 - ( 600ms / 3.5s )

![image](https://github.com/user-attachments/assets/6b890dee-54c7-4188-93c1-1fa61ed7471b)
<div align="center">
上线前后两天失败率
</div>

