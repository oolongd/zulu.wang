---
category: MySQL
tags:
  - MySQL
date: 2020-02-14
title: 利用内存表及存储过程批量生成 MySQL 测试数据
vssue-title: Generate Test Data To MySQL
---

当我们在做一些性能或者特性测试对时候，往往需要生产大批量的数据进行测试。我们可以通过编写例如 bash、python 脚本等方式来实现，但是这种效率并不是很高。本文将利用 MySQL 的内存表及存储过程来高效的创建测试数据。

## 一、创建数据表
```SQL
CREATE TABLE `contacts` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `f_name` varchar(20) NOT NULL,
  `l_name` varchar(20) NOT NULL,
  `phone_number` varchar(20) NOT NULL,
  `create_time` datetime NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
```

## 二、 创建内存表
```SQL
CREATE TABLE `contacts_memory` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `f_name` varchar(20) NOT NULL,
  `l_name` varchar(20) NOT NULL,
  `phone_number` varchar(20) NOT NULL,
  `create_time` datetime NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MEMORY DEFAULT CHARSET = utf8mb4;
```
之所以使用 memory 存储引擎是因为其基于内存，速度快。利用存储引擎可以快速的往 MySQL 插入数据，增加生成数据的速度。

## 三、 存储过程

### 1. 创建随机字符串
```SQL
CREATE DEFINER=`root`@`%` FUNCTION `rand_string`(n INT) RETURNS varchar(255) CHARSET utf8mb4
    DETERMINISTIC
BEGIN
    DECLARE chars_str varchar(100) DEFAULT 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    DECLARE return_str varchar(255) DEFAULT '' ;
    DECLARE i INT DEFAULT 0;
    WHILE i < n DO
        SET return_str = concat(return_str, substring(chars_str, FLOOR(1 + RAND() * 62), 1));
        SET i = i + 1;
    END WHILE;
    RETURN return_str;
END
```
### 2.创建随机时间

```SQL
CREATE DEFINER=`root`@`%` FUNCTION `rand_datatime`(start_datetime DATETIME,end_datetime DATETIME) RETURNS datetime
    DETERMINISTIC
BEGIN
    DECLARE sub INT DEFAULT 0;
    DECLARE ret DATETIME;
    SET sub = ABS(UNIX_TIMESTAMP(end_datetime) - UNIX_TIMESTAMP(start_datetime));
    SET ret = DATE_ADD(start_datetime, INTERVAL FLOOR(1 + RAND() * (sub + 1)) SECOND);
    RETURN ret;
END
```

> Returns a random floating-point value v in the range 0 <= v < 1.0. To obtain a random integer R in the range i <= R < j, use the expression FLOOR(i + RAND() * (j − i))

> 因为 MYSQL RAND() 函数生成的随机数的返回是 0 <= v < 1.0 因此产生 i <= R <= j 需要使用 FLOOR(i + RAND() * (j − i + 1))

### 3. 生成测试数据存储过程
```SQL
CREATE DEFINER=`root`@`%` PROCEDURE `add_contacts`(IN n int)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE (i <= n) DO
        INSERT INTO contacts_memory (f_name, l_name, phone_number, create_time) VALUES (rand_string(4), rand_string(8), FLOOR(RAND() * 10000000000000), rand_datatime("2019-12-01 00:00:00", '2020-01-31 23:59:59'));
        SET i = i + 1;
    END WHILE;
END
```
## 四、 调用存储过程生成测试数据
```SQL
CALL add_contacts(1000000)
```
参数决定了需要生成多少条记录，当生成的记录过多时，可能会提示当前内存已满。

> 1114 - The table 'contacts_memory' is full, Time: 361.549000s

可以通过修改 `max_heap_table_size` 变量来控制内存表允许使用的变量大小。注意修改后只对之后创建的表，或者执行 ALTER TABLE 操作的表有效。

> This variable sets the maximum size to which user-created MEMORY tables are permitted to grow. The value of the variable is used to calculate MEMORY table MAX_ROWS values. Setting this variable has no effect on any existing MEMORY table, unless the table is re-created with a statement such as CREATE TABLE or altered with ALTER TABLE or TRUNCATE TABLE. A server restart also sets the maximum size of existing MEMORY tables to the global max_heap_table_size value.

```SQL
-- 查看当前内存表大小限制
SHOW variables like 'max_heap_table_size'
-- 重新设置当前内存表大小为 1GB
SET max_heap_table_size = 1024 * 1024 * 1024 * 1;
```

## 五、 将数据从内存表插入普通表
```SQL
INSERT INTO contacts SELECT * FROM contacts_memory
```
