 Redis规范
===========================
MySQL数据库与oracle、sqlserver等数据库相比，有其内核上的优势与劣势。
我们在使用MySQL数据库的时候需要遵循一定规范，扬长避短。本规范旨在帮助或指导RD、QA、OP等技术人员做出适合线上业务的数据库设计。
在数据库变更和处理流程、数据库表设计、SQL编写等方面予以规范，从而为公司业务系统稳定、健康地运行提供保障。
****
## 目录
|作者|lucbine|
|---|---

* [基础规范](#基础规范)
* [命名规范](#命名规范)
* [使用规范](#使用规范)


说明
-----------
以下所有规范会按照【高危】、【强制】、【建议】三个级别进行标注，遵守优先级从高到低。
对于不满足【高危】和【强制】两个级别的设计，DBA会强制打回要求修改。

基础规范
----------- 
   * 【强制】禁止在应用程序服务器上通过命令直连生产环境Redis服务(允许Redis作为本地缓存，允许访问本地127.0.0.1)。
   * 【强制】任何服务都不能保证自身100%可用，应用服务程序应当具备自动重连机制并不断提升自身的健壮性。
   * 【强制】Redis服务默认禁用高危命令包括：keys、flushdb、flushall。
   * 【建议】Redis服务免密访问存在数据安全风险，谨慎开启。
   * 【建议】应用服务侧对于Redis超时的设置要进行理性评估，建议大于300ms且重试机制小于3次，避免过度放大请求导致雪崩。

命名规范
-----------
   * 【强制】库名、表名、字段名，必须见名知意,使用消息字母，使用下划线分隔,禁止拼音英文混用,最长最好不要超过32个字符
   * 【强制】库名、表名格式：业务系统名称_子系统名，同一模块使用的表名尽量使用统一前缀。命名最好与公司的产品、业务线等相关联
   * 【强制】一般分库名称命名格式是“库通配名_编号”，编号从“0”开始递增，例如 "wenda_001"、以时间进行分库的名称格式是“库通配名_时间” 例如 "wenda_20100711" 
   * 【强制】库名、表名、字段名，禁止使用mysql的相关关键字
   * 【建议】临时库、临时表 必须使用temp作为前缀，并用日期做后缀 ，例如：temp_log_20200711，定期清理。
   * 【建议】备份库、备份表 必须使用bak作为前缀，并使用日期做后缀，例如：bak_log_20200711 ，定期清理。
   * 【建议】主键的名称以“pk_”开头，唯一键以“uk_”或“uq_”开头，普通索引以“idx_”开头，一律使用小写格式，以表名/字段的名称或缩写作为后缀。
   
使用规范
-----------

   * 【必须】禁止使用Redis多db(只用select 0)，多db场景应该使用多实例去替代
   * 【推荐】选择合适的数据类型 ，如果不确定使用其他数据类型（hash、set之类）性能会比string类型好，那就使用string数据类型
   * 【必须】使用二级缓存必须做crash测试
       目前业务为了减轻远端redis压力，会在应用服务本地部署一个redis，一定要考虑、测试这个redis crash后远端redis承受的压力
   * 【建议】必须设置key的过期时间、时间打散
      Redis作为内存数据库，缓存的定位，需要设置key的过期时间，保证reids存储是热点数据，避免reids资源、性能浪费
   * 【建议】对于数据有持久化需求的场景，建议使用MySQL进行数据存储，不要放在Redis存储
   * 【强制】key名不要包含特殊字符，如空格、换行、单双引号以及其他转义字符
   * 【强制】技术设计上避免热点key
   * 【建议】使用批量操作提高效率，但要注意控制一次批量操作的元素个数(例如500以内，实际也和元素字节数有关)。如果用pipeline，也注意批次下key数量限制在500以内
   * 【建议】 O(N)命令关注N的数量。例如hgetall、lrange、smembers、zrange、sinter等并非不能使用，但是需要明确N的值。有遍历的需求可以使用hscan、sscan、zscan代替
   * 【建议】避免多个应用使用一个Redis实例。正例：不相干的业务拆分，公共数据做服务化
   
#### 设计规范
   * 
   *  
#### 索引设计规范
   * 【强制】单个索引中每个索引记录的长度不能超过64KB （InnoDB单列索引长度不能超过767bytes，联合索引还有一个限制是长度不能超过3072bytes）
   * 【建议】单个表上的索引个数不能超过7个
   * 【建议】在建立索引时，多考虑建立联合索引，并把区分度最高的字段放在最前面。如列userid的区分度可由select count(distinct userid)计算出来。
   * 【建议】建表或加索引时，保证表里互相不存在冗余索引。
      对于MySQL来说，如果表里已经存在key(a,b)，则key(a)为冗余索引，需要删除。
   * 【强制】在多表join的SQL里，保证被驱动表的连接列上有索引，这样join执行效率最。小表驱大表，被驱动表加索引。
      (join查询在有索引条件下,驱动表有索引不会使用到索引、被驱动表建立索引会使用到索引)
   *
   
#### 分库分表
   * 【强制】分区表的分区字段（partition-key）必须有索引，或者是组合索引的首列。
   * 【强制】单个分区表中的分区（包括子分区）个数不能超过1024。
   * 【强制】上线前RD或者DBA必须指定分区表的创建、清理策略。
   * 【强制】访问分区表的SQL必须包含分区键。
   * 【建议】单个分区文件不超过2G，总大小不超过50G。建议总分区数不超过20个。
   * 【强制】对于分区表执行alter table操作，必须在业务低峰期执行。
   * 【强制】采用分库策略的，库的数量不能超过1024
   * 【强制】采用分表策略的，表的数量不能超过4096
   * 【建议】单个分表不超过500W行，ibd文件大小不超过2G，这样才能让数据分布式变得性能更佳。
   * 【建议】水平分表尽量用取模方式，日志、报表类数据建议采用日期进行分表。
   
