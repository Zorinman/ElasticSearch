
我把 `settings` 放到了 **index 这一层**，并补充了它和 MySQL 的对比边界：它可以类比为“表级配置/存储配置的一部分”，但没有完全一一对应关系。

# MySQL 与 Elasticsearch 的结构类比

可以先用最简单的层级关系理解：

```text
MySQL:
数据库 database
  → 数据表 table
    → 行 row
      → 字段 column

Elasticsearch:
集群 cluster
  → 索引 index
    → 文档 document
      → 字段 field
```

更直观一点：

| MySQL        | Elasticsearch             | 说明                                              |
| ------------ | ------------------------- | ----------------------------------------------- |
| 数据库 database | 集群 cluster / namespace 概念 | ES 里没有完全等价的“数据库”概念                              |
| 表 table      | 索引 index                  | 一类数据的集合                                         |
| 行 row        | 文档 document               | 一条具体数据，通常是 JSON                                 |
| 列 column     | 字段 field                  | JSON 里的属性                                       |
| 表结构 schema   | 映射 mapping                | 定义字段类型、是否分词、如何索引等                               |
|  /   | settings                  | ES 索引级配置，例如分片、副本、刷新间隔、分词器配置等；与 MySQL 没有完全一一对应关系 |
| SQL 查询       | Query DSL                 | ES 的 JSON 查询语法                                  |

---

## 1. MySQL 的数据结构

MySQL 中，一张表可以理解为：

```text
表 table
  = 表结构 schema
  + 表数据 rows
```

例如有一张 `users` 表：

```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  age INT,
  city VARCHAR(50)
);
```

这段 SQL 定义的是 **表结构**。

它规定了：

```text
表名：users

字段：
  id    INT，主键
  name  VARCHAR(50)
  age   INT
  city  VARCHAR(50)
```

然后插入一行数据：

```sql
INSERT INTO users (id, name, age, city)
VALUES (1, '张三', 25, '北京');
```

这就是 **表数据**。

所以完整的 `users` 表可以理解为：

```text
users 表
  ├── 表结构：
  │     id INT PRIMARY KEY
  │     name VARCHAR(50)
  │     age INT
  │     city VARCHAR(50)
  │
  └── 表数据：
        1, 张三, 25, 北京
        2, 李四, 30, 上海
```

MySQL 里可以通过这些方式查看表结构和数据：

```sql
-- 查看表结构
DESC users;

-- 查看建表语句
SHOW CREATE TABLE users;

-- 查看表数据
SELECT * FROM users;
```

简单来说：

```text
MySQL 的表结构规定数据长什么样；
表数据必须按照这个结构存进去。
```

---

## 2. Elasticsearch 的 index、mapping、settings 和 documents

Elasticsearch 中，一个索引可以理解为：

```text
索引 index
  = mapping
  + settings
  + documents
```

也就是：

```text
users 索引
  ├── mapping：字段结构定义
  ├── settings：索引级配置
  └── documents：真实 JSON 数据
```

其中：

```text
mapping  决定“字段怎么理解”
settings 决定“索引怎么运行”
documents 是真正存进去的数据
```

---

## 3. mapping 是什么？

`mapping` 类似 MySQL 的表结构。

它定义文档中有哪些字段、字段是什么类型、是否分词、如何索引等。

例如创建一个 `users` 索引：

```json
PUT users
{
  "mappings": {
    "properties": {
      "id": {
        "type": "integer"
      },
      "name": {
        "type": "text"
      },
      "age": {
        "type": "integer"
      },
      "city": {
        "type": "keyword"
      }
    }
  }
}
```

这里的 `mappings` 就类似 MySQL 的表结构。

它定义了：

```text
索引名：users

字段：
  id    integer
  name  text
  age   integer
  city  keyword
```

然后写入一条文档：

```json
POST users/_doc/1
{
  "id": 1,
  "name": "张三",
  "age": 25,
  "city": "北京"
}
```

这条 JSON 数据就是 Elasticsearch 中的一条 **document 文档**。

所以可以理解为：

```text
users 索引
  mapping：
    id    integer
    name  text
    age   integer
    city  keyword

  documents：
    {
      "id": 1,
      "name": "张三",
      "age": 25,
      "city": "北京"
    }

    {
      "id": 2,
      "name": "李四",
      "age": 30,
      "city": "上海"
    }
```

---

## 4. settings 是什么？

`settings` 是 Elasticsearch 的 **索引级配置**。

它不是描述某个字段的，而是描述整个索引如何运行、如何存储、如何搜索。

例如：

```text
settings 可以控制：
  分几个主分片
  有几个副本
  多久 refresh 一次
  使用什么分词器
  最大分页窗口是多少
```

比如创建索引时同时指定 `settings` 和 `mappings`：

```json
PUT users
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1,
    "refresh_interval": "1s"
  },
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name": {
        "type": "text"
      },
      "age": {
        "type": "integer"
      },
      "city": {
        "type": "keyword"
      }
    }
  }
}
```

这个 `users` 索引可以理解为：

```text
users 索引
  settings：
    number_of_shards: 2
    number_of_replicas: 1
    refresh_interval: 1s

  mapping：
    id    keyword
    name  text
    age   integer
    city  keyword

  documents：
    {
      "id": "1",
      "name": "张三",
      "age": 25,
      "city": "北京"
    }
```
其中settings含义依次为
```text
主分片数：2
副本数：1
刷新间隔：1 秒
```

## 6. settings 具体如何体现？

settings 主要体现在创建索引和查看索引配置时。

查看某个索引的 settings：

```json
GET users/_settings
```

可能返回：

```json
{
  "users": {
    "settings": {
      "index": {
        "number_of_shards": "2",
        "number_of_replicas": "1",
        "refresh_interval": "1s",
        "provided_name": "users",
        "creation_date": "1780000000000",
        "uuid": "abc123"
      }
    }
  }
}
```

这说明 `users` 这个索引的配置包括：

```text
主分片数：2
副本数：1
刷新间隔：1 秒
索引名称：users
创建时间：creation_date
索引唯一 ID：uuid
```

---



## 8. Elasticsearch 中完整层级

从逻辑角度看：

```text
Elasticsearch 集群 cluster
  → 索引 index
      → settings：索引级配置
      → mapping：字段结构定义
      → documents：真实 JSON 文档
          → fields：字段
```

从分布式存储角度看：

```text
Elasticsearch 集群 cluster
  → 节点 node
    → 索引 index
      → 分片 shard
        → 文档 document
          → 字段 field
```

其中：

```text
cluster：
  整个 Elasticsearch 集群。

node：
  集群中的一台 Elasticsearch 实例。

index：
  一类数据的集合，类似 MySQL 的表，但不完全等同于表。

settings：
  index 的索引级配置，例如分片、副本、刷新间隔、分词器等。

mapping：
  index 中字段的结构定义，例如字段类型、是否分词、使用什么 analyzer。

shard：
  index 底层的分片，用于分布式存储和查询。

document：
  一条 JSON 数据，类似 MySQL 的一行记录。

field：
  document 中的字段，类似 MySQL 的列。
```

---

## 9. mapping、settings、documents 的具体查看方式

MySQL 里：

```text
查看表结构：
  DESC users;
  SHOW CREATE TABLE users;

查看表数据：
  SELECT * FROM users;
```

Elasticsearch 里：

```text
查看 mapping：
  GET users/_mapping

查看 settings：
  GET users/_settings

查看 documents：
  GET users/_search
  GET users/_doc/1
```

例如：

```json
GET users/_mapping
```

查看字段结构。

```json
GET users/_settings
```

查看索引配置。

```json
GET users/_search
```

查看索引中的文档数据。

---

## 10. 最核心总结

最简单的类比是：

```text
MySQL:
users 表
  = 表结构 schema
  + 表数据 rows

Elasticsearch:
users 索引
  = mapping
  + settings
  + documents
```


