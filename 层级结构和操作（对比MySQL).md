

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

## 5. settings 具体如何体现？

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



## 6. Elasticsearch 中完整层级

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

## 7. mapping、settings、documents 的具体查看方式

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

## 8. 核心总结

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

# ES和MySQL操作类比
先说明一个关键点：

> Elasticsearch 的 **index** 更像 MySQL 的 **table**，不是 MySQL 里的二级索引。
> MySQL 里的 `CREATE INDEX` 对应的是给表字段建查询索引；Elasticsearch 的 `index` 是数据集合本身。


---

## 1. MySQL 和 Elasticsearch 操作对照表

| 目的       | MySQL                                    | Elasticsearch                                      |
| -------- | ---------------------------------------- | -------------------------------------------------- |
| 创建数据集合   | `CREATE TABLE users (...)`               | `PUT /users`                                       |
| 创建字段结构   | `CREATE TABLE` 中定义 column                | `PUT /users` 时定义 `mappings.properties`             |
| 新增字段     | `ALTER TABLE users ADD city VARCHAR(50)` | `PUT /users/_mapping` 新增 `city`  (旧doc没有该字段）                |
| 修改字段类型   | `ALTER TABLE users MODIFY age BIGINT`    | 通常新建 index，然后 `_reindex`                           |
| 创建普通查询索引 | `CREATE INDEX idx_name ON users(name)`   | ES 一般不这样操作；字段是否可搜索由 mapping 决定                     |
| 修改索引级配置  | /                           | `PUT /users/_settings`                             |
| 修改副本数    | /                              | `PUT /users/_settings { "number_of_replicas": 2 }` |
| 修改主分片数   | /                              | 通常不能直接改，创建 index 时决定                               |
| 插入数据     | `INSERT INTO users ...`                  | `POST /users/_doc/1 { ... }`                       |
| 查询数据     | `SELECT * FROM users`                    | `GET /users/_search`                               |

---

## 2. Elasticsearch index 的创建和修改

### 2.1 创建 index

 MySQL：

```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  age INT,
  city VARCHAR(50)
);
```

Elasticsearch：

```json
PUT /users
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
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

这里：

```text
users 是 index
settings 是索引级配置
mappings 是字段结构定义
properties 里面是具体字段
```

Elastic 官方示例中，`PUT /books` 就是创建一个名为 `books` 的 index；文档数据会以 JSON 对象形式添加到可搜索的 index 中。([Elastic][2])

---

### 2.2 修改 index 的 settings

index 本身不像 MySQL 表那样经常直接 `ALTER TABLE`。Elasticsearch 中更常见的是修改 index 的 `settings`。

例如修改副本数：

```json
PUT /users/_settings
{
  "index": {
    "number_of_replicas": 2
  }
}
```

这个操作类似于调整表的运行配置，而不是修改字段结构。Elastic 官方 API 示例中，`PUT /my-index-000001/_settings` 可以修改动态 index setting，例如 `number_of_replicas`。([Elastic][3])

但不是所有 settings 都能随便改。比如：

```json
"number_of_shards": 2
```

主分片数通常只能在 index 创建时设置，创建后不能像普通配置一样直接改。Elastic 文档明确说明 `index.number_of_shards` 只能在 index 创建时设置。([Elastic][4])

所以常见情况是：

```text
number_of_replicas：常见，可修改
refresh_interval：常见，可修改
number_of_shards：创建时决定，后续一般不直接修改
analysis 分词器：创建时定义更常见；后续新增 analyzer 通常需要关闭 index 再修改再打开
```

---
## 3. mapping 和 field 的创建、修改

在 Elasticsearch 中，**field 字段的创建和修改，本质上就是 mapping 的创建和修改**。

因为：

```text
mapping 是字段定义的集合；
field 是 mapping.properties 里面的一个字段定义。
```

例如：

```json
{
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
      }
    }
  }
}
```

这里：

```text
mapping 是整个 mappings/properties 结构；
id、name、age 是具体 field。
```

所以：

```text
创建 field = 在 mapping.properties 里新增字段定义
修改 field = 修改 mapping 中这个字段的定义
```

---

### 3.1 创建 index 时定义 mapping 和 field

MySQL 中，字段通常在 `CREATE TABLE` 里定义：

```sql
CREATE TABLE users (
  id INT,
  name VARCHAR(50),
  age INT,
  city VARCHAR(50)
);
```

Elasticsearch 中，字段通常在创建 index 时通过 `mappings.properties` 定义：

```json
PUT /users
{
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

这里同时完成了：

```text
创建 mapping
创建 id、name、age、city 这些 field 定义
```

---

### 3.2 给已有 index 新增 field
Elasticsearch 在 mapping新增字段field：
  - 只是新增字段定义；
  - 不会自动修改已有 document；新写入或主动更新后的 document 才会真正拥有该字段。
  - 已有 document 中该字段是“不存在”，不是 null，也不是空字符串；
  - 想要旧的doc拥有该字段的需要主动更新旧数据补一个默认值。


MySQL 新增字段：

```sql
ALTER TABLE users ADD email VARCHAR(100);
```

Elasticsearch 新增 field：

```json
PUT /users/_mapping
{
  "properties": {
    "email": {
      "type": "keyword"
    }
  }
}
```

这个操作表示：

```text
修改 users 索引的 mapping；
在 mapping.properties 中新增 email 字段定义。
```

注意：

```text
新增 field 只是修改 mapping；
不会自动修改已有 document。
```

例如原来已有 document：

```json
{
  "id": "1",
  "name": "张三",
  "age": 25
}
```

新增 `email` 字段后，这条旧 document 不会自动变成：

```json
{
  "id": "1",
  "name": "张三",
  "age": 25,
  "email": null
}
```

它仍然是：

```json
{
  "id": "1",
  "name": "张三",
  "age": 25
}
```

也就是说：

```text
旧 document 中没有 email 字段；
不是 null；
不是空字符串；
而是字段不存在。
```

如果要给旧 document 补字段值，需要主动更新，例如：

```json
POST /users/_update_by_query
{
  "script": {
    "source": "ctx._source.email = 'unknown@example.com'",
    "lang": "painless"
  },
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "email"
        }
      }
    }
  }
}
```

这个操作会更新 `users` 索引中所有没有 `email` 字段的 document。

---

### 3.3 修改已有 field 的类型

MySQL 中修改字段类型比较常见：

```sql
ALTER TABLE users MODIFY age BIGINT;
```

Elasticsearch 中，已有 field 的类型通常不能直接修改。

例如原来是：

```json
"age": {
  "type": "integer"
}
```

如果想改成：

```json
"age": {
  "type": "keyword"
}
```

通常不能直接在原 index 上改。

常见做法是：

```text
1. 新建一个 index，例如 users_v2
2. 在 users_v2 中定义正确的 mapping
3. 使用 _reindex 把 users 的数据复制到 users_v2
4. 应用层切换到 users_v2，或者通过 alias 切换
```

示例：

```json
PUT /users_v2
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name": {
        "type": "text"
      },
      "age": {
        "type": "keyword"
      },
      "city": {
        "type": "keyword"
      }
    }
  }
}
```

然后执行：

```json
POST /_reindex
{
  "source": {
    "index": "users"
  },
  "dest": {
    "index": "users_v2"
  }
}
```

---

### 3.4 给已有 field 增加子字段

有些 mapping 修改是允许的，例如给已有字段增加 `multi-fields`。

比如原来：

```json
"name": {
  "type": "text"
}
```

现在希望 `name` 既可以全文搜索，又可以精确匹配、排序、聚合，可以增加一个 `keyword` 子字段：

```json
PUT /users/_mapping
{
  "properties": {
    "name": {
      "type": "text",
      "fields": {
        "keyword": {
          "type": "keyword"
        }
      }
    }
  }
}
```

之后可以这样使用：

```text
name：用于全文搜索
name.keyword：用于精确匹配、排序、聚合
```

不过需要注意：

```text
新增 multi-field 后，旧数据通常需要重新索引，新的子字段才会对旧数据完整生效。
```

---

### 3.5 通过 dynamic mapping 自动创建 field（方便但高风险-不建议使用）

如果没有提前定义 field，直接写入带新字段的 document：

```json
POST /users/_doc/1
{
  "id": "1",
  "name": "张三",
  "age": 25,
  "city": "北京"
}
```

如果 `city` 原来不在 mapping 中，并且 dynamic mapping 开启，Elasticsearch 可能会自动推断并创建 `city` 字段的 mapping。

例如自动生成：

```json
"city": {
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword"
    }
  }
}
```

这种方式叫：

```text
dynamic mapping 动态映射
```

但实际项目中通常不建议完全依赖 dynamic mapping，因为 ES 自动推断出来的类型不一定符合预期。

例如你可能希望：

```json
"city": {
  "type": "keyword"
}
```

但 ES 自动推断后可能变成：

```json
"city": {
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword"
    }
  }
}
```

所以更推荐显式定义 mapping。

---



##  ES 写法汇总

### 创建 index + mapping + settings

```json
PUT /users
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
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

### 新增字段

```json
PUT /users/_mapping
{
  "properties": {
    "email": {
      "type": "keyword"
    }
  }
}
```

### 修改动态 settings

```json
PUT /users/_settings
{
  "index": {
    "number_of_replicas": 2
  }
}
```

### 插入 document

```json
POST /users/_doc/1
{
  "id": "1",
  "name": "张三",
  "age": 25,
  "city": "北京"
}
```

### 查看 mapping

```json
GET /users/_mapping
```

### 查看 settings

```json
GET /users/_settings
```

### 查询数据

```json
GET /users/_search
```

---



