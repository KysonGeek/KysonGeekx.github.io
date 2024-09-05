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

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ODY5NzBkNjQyMzY1Y2NjZjAyMmNkMWVkMDE4YWI4MDdfNFJKUkVvYXVYWEdSdHZqRGNwbjI4aG9rd0NTUTFZeGJfVG9rZW46Ym94Y25Pb0ViUFQ4MGdLTnRwbktWcFNHS09mXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZWZlNjhkY2VkOTM3NDk1NTAwOWQ5ODAyYjNhZTY1NzFfZGZVeU1qN1BqRVIySDlIdHdPZGFxOFBHaW15anJ5ZkxfVG9rZW46Ym94Y25NUkNlTlJxRXZXb2Niczh1V0RHempjXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

图一：案发现场十点

# 2. 案件分析

针对接口超时我们进行了内部分析，主要得出以下几个结论

- hive同步es时并发量设置的比较大，导致cpu飙升到80+

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZDY4NjQ0MDI5ODAxMTZiODM1ZTFlZTBjMTIyYWNkMDlfN2NHVmNBdEJoQmxjMTBsWUtNVWxWNUlnZmg3cjJ2bEVfVG9rZW46Ym94Y25OUjNkWVl4MjFYMTNlNkpZaXN6T1VmXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

- es服务申请时，一共四个pod，每个pod规格：16核36G。当时硬盘申请了2T，按照es内存/索引(500G)大小 = 2/3的规范来看，申请的es-pod规格低了
- 梳理策略组使用的字段时判断cert_name不会用于模糊搜索，所以当时es索引使用了keyword（不分词，主要用于等值搜索），通过日志发现，此字段也用了模糊搜索，导致es查询性能下降

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=NzVmMGJhNTFmNzVlNjZhZDA2ZWZiZTlmZjhhODU2YTNfa29uNFB6d3hCQ3czWk5tMEdIbXY3a2U2TXJNNzU2cU1fVG9rZW46Ym94Y250M3REOUxGTjZCYWlINzZhU0dWeVpjXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

- 视频召回时，调用es索引服务拼写的条件不符合常理。例如：

（cert_name like '%用车%' or cert_name like '%汽车%' or cert_name like '%车%'）

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=MGQ0ZDQ0ZmJiYjU4OGQ4YTc4OTEzOGU4YmE2ZTNjMTVfOHM4Z0VBcDRlcENqcUJ4RUt1V3BQbjdtb2pGNjQ3Z2lfVG9rZW46Ym94Y244UkQ2TDhtQzJmVndzdDQ5OEZSQU1oXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

# 3. 锁定&捉拿凶手

针对上面的问题我们一一给出了逮捕方案

- 降低hive同步es的并发量，从3k改成了1.5k 

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=NWJmOWVjYjFkMTNjNDExNjY0MGI2NWZmNmM1MGMwZmZfWFhOd0lWVnl6ZEdYU3hZcDI1aDhkVU1qVmlTQzdCY25fVG9rZW46Ym94Y25hbUVINzh0ZEFNSnhkemllRFFLZ1FXXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

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

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZjlkMTVkZDg0ZWIxYWI4MTQyNmMyMDIxNDdhYTk1ODBfaFFEOE9EZnd0Q0xFejZxMDlrdXdjM3pJUGRaQjZGbzFfVG9rZW46Ym94Y245ZFBuQ0N5NWc2MWhZS1hMbkx2THdjXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

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
| 单字模糊搜索 "们"     | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=NjIyMWNkNDY2ZDhhNjNkMWVjZjRlMjc0YjYyZmQ0YjZfQ2JHYkdtS0cwVU5iNngzaGtoZlNFU0NIN2hsOW5majVfVG9rZW46Ym94Y24wbGdjNmYxUm9SaXQ5Q1RwQlRycVRhXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) | match_all拿出原始数据进行正则匹配                            |
| 俩字模糊搜索 "你们"   | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZGI5NTU4YTdlNTVkZTA0YTU3NTVmYWYzNGM5MDJmMWNfMlhvMzBRZGZYRkV5SkpGQ3gzdlhDbmdvTmJySnhFd1FfVG9rZW46Ym94Y25TdGZYaW8xY2x6QzRFdFNxbUZTbGxZXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) | 前缀匹配"你们*"拿出原始数据进行正则匹配                      |
| 仨字模糊搜索 "你们好" | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=YzYzNzEyNDk3NGE3ODlmYzA4NThjYjNlOGE1NDkzMzZfNjlydGtDQUlIbW44UjFzMXVxcTJNdU8xeHZDYmZIbWtfVG9rZW46Ym94Y241RTRHQjV2aWpUUDVFNzlEQnpEeHE3XzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) | term匹配"你们好"拿出原始数据进行正则匹配                     |
| 多字模糊搜索          | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=OTY1NWE0MTYxYzU4OWQxOGY4OTdmMmY4M2FiZDA5NDlfZ2dBZHI2Y1g1MjY5emZ2OG5uSWVjTzdrYmM2cHg2MXdfVG9rZW46Ym94Y25JUk1YWHA2ajJuM3FZdVlibWM3ckllXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) | 将搜索语句拆分成多组三个字的数据进行term匹配拿出原始数据进行正则匹配 |

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

1. 当模糊搜索关键词长度是3的情况下，是否还需要进行doc_value正则匹配？
2. 当模糊搜索关键词是“主流”时，es会先进前缀匹配“主流*”，但是“社会主义核心价值观之图书会主流”在分词时，由于主流是最后两个字，所以分词结果的最后一个三字结果是“会主流”，请问：当我搜索“主流”时能否搜出“社会主义核心价值观之图书会主流”？

暂时无法在飞书文档外展示此内容

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
| 旧的wildcard双字模糊搜索 | 6s    | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=NWJhOWM3YzQ1NGVjODg1MTBlZmYxNmJiOGIyZjM5ZjZfZlAzbVRGWEZ2OHNZNUFmU1dRb3poNmlUUVZKdWpGN3pfVG9rZW46Ym94Y254RTNybXJFazlIVnhrSzFEYmFpNnhnXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) | 耗时相差800+倍 |
| 新的term双字模糊搜索     | 6.8ms | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZWE4NTc0NTE3ZGM5MjViZjNjNjU1NmE2ZmM3M2IyNGFfT05mRlF0eFdhcVJkSXprZDZaaTdSendSWWwxdW1LSUJfVG9rZW46Ym94Y25YaHFuaGYwT0dWYUhzVW9yQTA3bERnXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) |                |
|                          |       |                                                              |                |
| 旧的wildcard单字模糊搜索 | 700ms | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=Mzc2ZDVlNTg2N2U5ZDNhNDgyMTM0NWM2ZjNkMTQ1MTFfNlo5RVNEOWFPOUZPemwxRmlKT0VFOU1XaEVRQkpzMUNfVG9rZW46Ym94Y25YdXRJanh5N2ZLb2p3VHZXNk9vVHliXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) | 相差100+倍     |
| 新的term单字模糊搜索     | 5.5ms | ![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=Nzk1MWEzNWFmNTM5NTAyNjVjNTc5ZDExNzI4ZTczMGFfMXdVZW1HSnFUcWNJc1lYTGY5YnN1ZHVzTkNzRWVqb1BfVG9rZW46Ym94Y25obFRReXdPWnhMZXltZGhWejFLUEhWXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA) |                |

上表中对比的仅仅是一个字段的模糊搜索。实际使用中，一次查询中可能会包含多个模糊查询

**上线后的真实效果**

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=OTdmM2Y4ZGEwMGIxY2ZjYmRiNjE4YTQ2N2M5OTBhZTFfR1V6NUhkdUd6RmNUb3BBanBRTTh1dXRlZjJKcFRWSWVfVG9rZW46Ym94Y25aT0JhWE9yRWVnT1o5cW5jNWs4T0VBXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

09-15（上线前），99线

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=M2MzMmYzNjllZjMyMjIzNDM4NzVhNzYwY2U2NWFhMTNfcUtqeHZqUmpwOGdtV0tqaFFuaHl0MnhQam0wd3JPOUdfVG9rZW46Ym94Y25NbG40OThNVjUwTTk5ZHJSWU9JMUJmXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

09-16（上线后）,99线

| 监控时间           | MAX       | p99       |
| ------------------ | --------- | --------- |
| 09-15，09:00~19:00 | 3.53 s    | 631.12 ms |
| 09-16，09:00~19:00 | 592.07 ms | 180.71 ms |

从上面的监控数据可以看出查询接口

1. p99耗时降低约 70% = 1 - ( 180ms / 600ms )
2. 不再有突刺，MAX不会超过1s，降低了约  80% = 1 - ( 600ms / 3.5s )

![img](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=M2ExMjJkZmUwYzQzM2Y4NmQxOWM2ZjBlYTU0YjdiNWVfeFBNc1F0Z1JDQ1VmS09kQk1mZ08yckJUNEdFRkdwYnJfVG9rZW46Ym94Y25qWDFZTjRaNUJ6SmZKRjN4MDh4OWtjXzE3MjU1MjI2NDQ6MTcyNTUyNjI0NF9WNA)

上线前后两天失败率
