---
title: "如何使用pymysql来操作Mysql数据库"
description: "在本篇文章中我将讲解如何使用pymysql库来操作mysql数据库以及注意事项"
date: 2026-05-16
lastmod: 2026-05-16
weight: 4
categories:
    - Tutorial
    - Tech Stack
tags:
    - 教程
    - 技术栈
    - pysql
---

# 如何使用pymysql来操作mysql数据库

---

- 作者：山财小蒋
- 联系方式：2018036661@qq.com
- 创作不易，转载请注明出处，欢迎批评探讨

---

- Mysql是一个快速开源且易使用的关系型数据库，学习并掌握它是很有必要的，但是只学sql语句而不会使用其他编程语言来建立与mysql的操作关系的话，在实际项目开发难免会遇到很多问题，这也是我的经验之谈；

- pymysql是python中常用于操作mysql的一个库，除此之外还有mysql-connertor和mysqldb等库。

- 本篇文章中我将讲解如何使用pymysql库来操作mysql数据库；

- 接下来我来进行一段完整的代码演示，首先在mysql server中建立实例用的数据库和数据表：

```sql
create database employee
use employee
create table emp (eid int, salary int);
```

- 接下来我们尝试在python中用pymysql来操作它：

```python
import pymysql

connection = pymysql.connect(
    host="localhost",
    user="root",
    password="Jl.2018036661",
    database="employee",
    charset="utf8mb4",
    cursorclass=pymysql.cursors.DictCursor
)

try:
    with connection.cursor() as cursor:
        sql = "INSERT INTO emp (eid, salary) VALUES (%s, %s)"
        data = (1, 10000)
        cursor.execute(sql, data)
        connection.commit()
        print("数据插入成功")

    with connection.cursor() as cursor:
        sql = "SELECT * FROM emp"
        cursor.execute(sql)
        result = cursor.fetchall()
        print("查询结果:", result)

except Exception as e:
    print("数据库操作失败:", e)
finally:
    connection.close()
```

- 得到如下结果：

```
数据插入成功
查询结果: [{'eid': 1, 'salary': 10000}]
```

---

## 游标

- pymysql库中一个很重要的概念是“游标”，有了它我们才能精准的操作mysql数据库，一般我们使用pymysql数据库都分为以下四个流程：

- 1.连接数据库
- 2.创建游标
- 3.执行sql语句
- 4.关闭连接

- 在官方定义中，游标是数据库里用来遍历、执行 SQL、读取结果集的数据操作接口，游标由两部分组成，分别是conn和cursor，而在我看来游标其实就是一个“定位器”，数据库连接conn负责打通通道，游标cursor是负责在通道里干活的工具 / 指针，没有游标连上数据库也无法执行任何 SQL，可以说
如果没有它们俩你的程序就不知道要在什么时候什么地方执行哪条sql语句，操作mysql数据库也就成为了无稽之谈；


## 游标的核心作用
1. 执行 `增删改查` SQL 语句
2. 接收数据库返回的**查询结果**
3. 逐条遍历数据表数据
4. 定位数据行、批量操作数据

## 两种常用游标类型
### 1. 默认游标：普通游标（元组格式）
取出数据是 **元组 tuple**，按顺序取值
```python
cursor = conn.cursor()
```

### 2. 字典游标
取出数据是 **字典 dict**，**通过字段名取值**，超级方便
```python
cursor = conn.cursor(pymysql.cursors.DictCursor)
```

---

## 游标 4 大核心方法
1. `cursor.execute(sql)` → 执行SQL语句
2. `cursor.fetchone()` → **取第一条数据**
3. `cursor.fetchall()` → **取出全部数据**
4. `cursor.fetchmany(n)` → 取出指定条数数据

## 指针原理
游标内部有**数据指针**：
- 查完数据后，指针**默认在第一条前面**
- `fetchone()` 取一条 → 指针**向后移动一位**
- 取完所有数据后，指针走到末尾，**再取就是空**

---

## 完整分步实战代码演示
### 1. 准备数据库表
```sql
CREATE TABLE coin_price(
    id INT AUTO_INCREMENT PRIMARY KEY,
    symbol VARCHAR(30),
    price DECIMAL(20,2),
    time DATETIME
);
```

### 2. 基础连接 + 创建游标
```python
import pymysql

# 1. 建立数据库连接
conn = pymysql.connect(
    host="localhost",
    user="root",
    password="你的密码",
    database="你的库名",
    charset="utf8mb4"
)

# 2. 创建【普通游标】
cursor = conn.cursor()
```

### 3. 用游标执行插入
```python
sql = "INSERT INTO coin_price(symbol,price,time) VALUES(%s,%s,%s)"
data = ("BTC/USDT",67890.5,"2026-05-16 12:00:00")

# 游标执行SQL
cursor.execute(sql,data)
# 增删改必须提交
conn.commit()
print("插入成功，自增ID：",cursor.lastrowid)
```

### 4. 游标查询 + 指针移动演示
```python
# 查询所有数据
sql = "SELECT * FROM coin_price"
cursor.execute(sql)

# ① 取第一条
row1 = cursor.fetchone()
print("第一条数据：",row1)

# ② 再取第二条（指针已经往后走了）
row2 = cursor.fetchone()
print("第二条数据：",row2)

# ③ 取剩下所有
all_rest = cursor.fetchall()
print("剩余全部：",all_rest)
```

### 5. 字典游标
```python
# 重新创建字典游标
dict_cursor = conn.cursor(pymysql.cursors.DictCursor)
dict_cursor.execute("SELECT * FROM coin_price")

# 拿到字典数据，直接用字段名取值
data = dict_cursor.fetchone()
print("交易对：",data["symbol"])
print("价格：",data["price"])
print("时间：",data["time"])
```

### 6. 批量读取指定条数
```python
dict_cursor.execute("SELECT * FROM coin_price")
# 一次性取3条
many_data = dict_cursor.fetchmany(3)
for item in many_data:
    print(item["symbol"],item["price"])
```

### 7. 重置游标指针
取完数据指针到末尾，想从头再读：

```python
# 回到开头
dict_cursor.scroll(0,mode="absolute")
```

### 8. 关闭释放资源
```python
# 先关游标，再关连接
cursor.close()
dict_cursor.close()
conn.close()
```

---

### 通俗易懂总结
1. **连接 = 修路**，游标 = **路上的运输车**
2. 所有SQL命令**必须交给游标执行**
3. 查询出来的数据，全存在**游标结果集**里
4. `fetch` 系列方法就是**从游标里拿数据**
5. 字典游标写量化项目**最舒服**，不用记字段顺序
6. 增删改一定要 `conn.commit()`，查询不需要

---

### 最简使用流程背诵
```
1. connect() 建连接
2. cursor() 拿游标
3. execute() 执行SQL
4. fetchxxx() 拿结果
5. commit() 提交改动
6. close() 关闭释放
```