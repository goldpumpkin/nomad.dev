---
title: 数据库：CASE WHEN THEN ELSE END

date: 2017-05-20 22:08:00

comments: true

description: 数据库 case when then else end

tags:
---

### CASE WHEN 条件 THEN 改变的值 END

### 1.简单case函数，使用表达式确定返回值：

​	语法：

```sql
CASE title 
	WHEN expression1 THEN result1
	WHEN expression2 THEN result2
	...
	WHEN expressionN THEN resultN
ELSE default_res
```

### 2.搜索case表达式，使用条件确定返回值(一般使用)

​	语法：

```sql
CASE
	WHEN condition1 THEN result1
	WHEN condition2 THEN result2
	...
	WHEN conditionN THEN resultN
ELSE default_result
END
```



​	eg:

```sql
SELECT sex
	CASE 
	WHEN sex=0 then '男'
	WHEN sex=1 then '女'
	ELSE '未知'
	END
FROM student;
```

### 3.一些例子

​	**1>已知数据按照另外一种方式进行分组，分析。**

​		根据这个国家人口数据，统计亚洲和北美洲的人口数量

​		

| country | population |
| ------- | ---------- |
| 中国      | 600        |
| 美国      | 100        |
| 加拿大     | 100        |
| 英国      | 200        |
| 法国      | 300        |
| 日本      | 250        |



```sql
SELECT SUM(population),
	CASE country
		WHEN '中国' THEN '亚洲'
		WHEN '日本' THEN '亚洲'
		WHEN '美国' THEN '北美洲'
		WHEN '加拿大' THEN '北美洲'
		ELSE '其他'
	END
FROM table_A
	GROUP BY 
		CASE country
			WHEN '中国' THEN '亚洲'
			WHEN '日本' THEN '亚洲'
			WHEN '美国' THEN '北美洲'
			WHEN '加拿大' THEN '北美洲'
			ELSE '其他'
		END;	
```

​	**2>求各个分数段的人数**

```
SELECT 
	SUM(CASE WHEN score = 81 THEN 1 end) 81,
    SUM(CASE WHEN score = 82 THEN 1 end) 82,
    SUM(CASE WHEN score = 83 THEN 1 end) 83
 FROM SCORE;
```

​	注意：如果使用count(),也行。count()统计数据*总数*，sum()*求和*某字段下面的所有数据

​		

​		

​		

