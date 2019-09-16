#### 查询数据

- ##### 基本查询

  ~~~sql
  select * from table_name;
  ~~~

- ##### 条件查询

  ~~~sql
  -- 条件表达式中可以有 NOT, AND, OR, 优先级依次递减
  select * from table_name where <条件表达式>
  ~~~

- ##### 投影查询（查询指定列，列的别名使用）

  ~~~sql
   select 列名1，列名2， 列名3 ... from 表名; 
  ~~~

- ##### 排序（默认使用升序 ASC， ORDER BY 必须放在 WHERE 子句后面）

  ~~~sql
  select * from table_name order by 列名1 desc(asc), 列名2 desc(asc)... ;
  ~~~

- ##### 分页查询

  ~~~sql
  select * from students order by id limit 15 offset 0;
  ~~~

- ##### 聚合查询

  ~~~sql
  --例子：统计 students 表中男性的数量
  	SELECT 
  		COUNT(*) boys 
  	FROM students 
  	WHERE gender = 'M';
  -- 例子：统计男生平均成绩
  	select 
  		AVG(score) average 
  	FROM students 
  	WHERE gender = 'M';
  -- 例子：按照 class_id 分组
  	select 
  		COUNT(*) num 
  	from students 
  	group by class_id;
  -- 例子：查询每个班级男生和女生的平均分及班级名称
  	select * 
  	from (
          select 
          	class_id, 
          	gender, 
          	avg(score)
          from students
          group by class_id, gender
      	) 
      AS temptable 
      left outer join classes
      on temptable.class_id = classes.id;
  
  ~~~

- ##### 多表查询

  ~~~sql
  SELECT
      s.id sid,
      s.name,
      s.gender,
      s.score,
      c.id cid,
      c.name cname
  FROM students s, classes c
  where s.gender  = 'M' and c.id = 1;
  ~~~

- ##### 连接查询

  ~~~sql
  -- 1. inner join(选出两张表都存在的记录)
  	select * 
  	from students 
  	inner join classes 
  	on classes.id = students.class_id;
  
  -- 2. left outer join(选出左表存在的记录)
  	select * 
  	from students 
  	left outer join classes 
  	on students.class_id = classes.id;
  
  -- 3. right outer join(选出右表存在的记录，右连接相当于左连接的主表和连接表交换位置)
  	select * 
  	from students 
  	right outer join classes 
  	on students.class_id = classes.id;
  
  -- 4. full outer join(选出左右表都存在的记录)
  	select * 
  	from students 
  	full outer join classes 
  	on students.class_id = classes.id;
  ~~~

#### 修改数据

- **插入数据**

  ~~~sql
  -- 例子：向 students 表中添加一条记录
  	insert into students(class_id, name, gender, score) 
  		VALUES (2, '大牛', 'M', 80);
  -- 例子：向 students 表中添加多条记录
  	insert into students(class_id, name, gender, score) 
  		VALUES (1, '大宝', 'M', 87), (2, '二宝', 'M', 81);
  ~~~

- **更改数据**

  ~~~sql
  -- 例子：修改 students 表中的 name 和 score 两个字段名
  	update students set name = '大牛', score = 90 where id = 1;
  ~~~

- **修改数据**

  ~~~sql
  -- 例子：删除表中的 id = 1 的记录
  	delete from students where id = 1;
  ~~~

#### 使用  SQL 语句

- **插入或替换**

  ~~~sql
  -- 若当前插入记录已经存在于数据库表中，则先删除原记录，再插入新纪录
  REPLACE INTO student(id, class_id, name, gender, score) 
  	values (1, 1, '小明', 'F', 98);
  ~~~

- **插入或更新**

  ~~~sql
  -- 若当前插入记录已经存在于数据库表中，则更新该记录
  INSERT INTO students(id, class_id, name, gender, score)
  	values (1, 1, '小明', 'F', 99) 
  	ON DUPLICATE name = '小明', gender = 'F', score = 99;
  ~~~

- **插入或忽略**

  ~~~sql
  -- 若当前插入记录已经存在于数据库表中，则忽略当前插入记录
  INSERT IGNORE INTO students (id, class_id, name, gender, score) 
  	VALUES (1, 1, '小明', 'F', 99);
  ~~~

- **快照**

  ~~~SQL
  -- 对一个表进行快照，即复制一份当前标的数据到一个新表
  CREATE TABLE students_of_class1 
  	select * from students 
  		where class_id = 1;
  ~~~

- **写入查询结果集**

  ~~~~sql
  /* 
   * 将查询结果写入到一个表中
   * 1. 新建一个统计表
   * 2. 将数据写入到表中
   */
  CREATE TABLE statistics (
      id BIGINT NOT NULL AUTO_INCREMENT,
      class_id BIGINT NOT NULL,
      average DOUBLE NOT NULL,
      PRIMARY KEY (id)
  );
  INSERT INTO statistics (class_id, average) 
  	SELECT class_id, AVG(score) 
  	FROM students 
  	GROUP BY class_id;
  ~~~~