#  Mybatis 持久层框架

> 原文：https://lcnipys9u0xh.feishu.cn/wiki/VU1MwcbuKiIco5kkmQCc1W0lnFc

### 动态 SQL 与缓存机制

![image](../../../assets/RUitdnqdFoAt22xYxnRcUvign7f/OU1abC7UgownPSx3bEQc6jfWn0e.png)

### 1. MyBatis 的一级缓存和二级缓存有什么区别？

很多同学在使用 MyBatis 时能感受到“同一个查询好像不会重复查库”，但细说不清。我们一次性搞明白。

#### 🔎 先看两张缓存的作用域

> （此处为内嵌电子表格，需登录飞书查看）

#### 🧠 一级缓存深入

在同一个 SqlSession 里，执行 select 时，MyBatis 会先查 LocalCache（一个 PerpetualCache，底层是 HashMap）。如果命中，直接返回，不再查库。
重要特性：

- 在执行 update、delete、insert 提交后，或者执行 SqlSession.clearCache() 后，缓存会被清空。
- 如果多个 SqlSession 同时操作同一数据，一级缓存会导致读到旧数据（因为另一个会话修改了数据库，本会话缓存未更新）。所以在并发环境下，不能依赖一级缓存来保证数据一致性。

#### 🗄️ 二级缓存机制与配置

二级缓存是跨会话的。当某个 SqlSession 提交时，会将当前 select 的结果存入二级缓存区域；下一个新的 SqlSession 在相同 namespace 下执行相同查询时，就能从二级缓存直接取。

开启方式：

```
代码块
<!-- mybatis-config.xml -->
<settings>
    <setting name="cacheEnabled" value="true"/>
</settings>
<!-- 在 Mapper.xml 中添加 -->
<cache eviction="LRU" flushInterval="60000" size="512" readOnly="true"/>
```

或者在接口上标注：

```
代码块
@CacheNamespace(eviction = LruCache.class, flushInterval = 60000, size = 512, readOnly = true)
public interface UserMapper {
    @Select("SELECT * FROM users WHERE id = #{id}")
    User getUser(int id);
}
```

常见配置属性：

- eviction：回收策略 LRU/FIFO/SOFT/WEAK。
- flushInterval：刷新间隔（毫秒），到时间自动清空。
- size：缓存对象数量上限。
- readOnly：只读时性能更好，因为对象本身可被共享；读写时需要序列化拷贝。

二级缓存的查询流程：二级缓存 → 一级缓存 → 数据库。如果二级缓存命中，直接返回，不走一级缓存。

#### ⚠️ 避坑指南

1. 不要在分布式环境直接用内置缓存：默认二级缓存基于本地内存，多实例间数据不一致。务必换成 Redis、Hazelcast 等分布式缓存实现（如 mybatis-redis 适配）。
