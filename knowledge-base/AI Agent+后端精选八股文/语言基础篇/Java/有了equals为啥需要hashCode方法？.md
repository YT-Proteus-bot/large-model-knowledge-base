---
title: 有了equals为啥需要hashCode方法？
topic: AI Agent+后端精选八股文
sub_topic: 语言基础篇/Java
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/NrtuwSsQSiNelgk2gSAc32iZnEb
source_path: AI Agent+后端精选八股文/语言基础篇/Java/有了equals为啥需要hashCode方法？.md
questions: 4
chars: 1807
build_date: '2026-11-20'
images_removed: 0
---

# 有了equals为啥需要hashCode方法？

> 主题：AI Agent+后端精选八股文 › 语言基础篇/Java　｜　原始文件：AI Agent+后端精选八股文/语言基础篇/Java/有了equals为啥需要hashCode方法？.md

有了 equals 为什么还要 hashCode？

这个问题问到了 Java 集合框架的根基。简单说：equals 负责判断两个对象是否“逻辑相等”，hashCode 负责快速定位对象在哈希表里的存储位置。如果只重写 equals 而忽略 hashCode，哈希表相关的集合（HashMap、HashSet、Hashtable）就会行为错乱，出现“明明相等的对象却找不到”的诡异 bug。

## 题1：两者的核心分工

> 〔原文此处为飞书内嵌组件，内容未随导出〕

关键点：在 HashMap 这类结构中，先用 hashCode 快速定位桶（O(1)），再用 equals 在桶内精确比对。如果 hashCode 没跟上 equals 的逻辑，第一步就定位错了，后面的 equals 根本不会被调用。

## 题2：契约：必须遵守的规则

Oracle 官方文档明确规定了 equals 和 hashCode 的契约：

1. 如果两个对象通过 equals(Object) 判定相等，它们的 hashCode 必须相等。
1. 两个 equals 不等的对象，hashCode 不一定不等（哈希冲突是允许的），但好的算法应尽量分散。
1. 在同一个程序运行期间，同一个对象的 hashCode 必须保持不变，除非对象中影响 equals 的属性被修改。

违反第 1 条的后果：

假设你定义了一个 User 类，以 id 为相等依据，并正确重写了 equals，但没有重写 hashCode，而是继承了 Object 的默认实现（通常返回对象的内存地址变换值）。那么两个 id 相同的 User 对象虽然 equals 为 true，但 hashCode 几乎肯定不同。

当把它们存入 HashMap 时，它们会散列到不同的桶里，导致 map.get(new User(1)) 返回 null，因为找不到对应的桶——哪怕这个 key 与 map 中已有的 key 逻辑相等。

## 题3：验证：一个经典翻车现场

```
public class User {
    private Long id;
    private String name;
    // 只重写了 equals，没有重写 hashCode
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return Objects.equals(id, user.id);
    }
}
// 使用场景
Map<User, String> map = new HashMap<>();
User u1 = new User(1L, "张三");
map.put(u1, "张三的数据");
User u2 = new User(1L, "张三");
System.out.println(map.get(u2)); // 输出 null！
```

分析：u1.equals(u2) 为 true，但两者的 hashCode 不同，导致 put 和 get 时计算出的桶索引不同，get 在错误的桶里搜寻，找不到匹配的 entry。

## 题4：哈希表内部流程（为什么必须成对重写）

一次 HashMap.put(key, value) 的操作大致分四步：

1. 计算哈希：调用 key.hashCode()，并通过扰动函数混合高位信息，得到最终哈希值。
1. 定位桶：根据哈希值取模，找到应插入的桶索引。
1. 冲突处理：若该桶已有元素，遍历链表/红黑树，对每个节点调用 key.equals(k) 判断是否相等。
1. 替换或新增：若存在 equals 相等的节点，覆盖旧值；否则插入新节点。

关键点：第 2 步只用 hashCode，第 3 步才用 equals。如果两个逻辑相等的对象 hashCode 不同，它们会散入不同的桶，第 3 步永远不会执行，导致 containsKey、get、remove 全部失效。
