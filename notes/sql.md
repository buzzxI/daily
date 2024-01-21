这里面都是一些琐碎的东西

# 检索数据

## 检索不同的行

比如用户表，多个用户可以具有相同的的年龄，如果我们希望返回的是所有不同的年龄使用语句：

```mysql
select distinct age from user;
```

## 限制结果

这个其实写过，比如数据库中一共20条记录，现在一次返回只能5个

```mysql
select * from user limit 5
```

假如现在还是返回5个，如果我们希望返回的是范围是从5-9这5个

```mysql
select * from user limit 5 offset 5
```

> 注意这里的编号从0开始计数，比如返回前5个可以写成：
>
> ```mysql
> select * from user limit 5 offset 0
> ```

## 使用完全限定的表名

还是我们的user表，假如这个表在数据库`info`中

我们可以使用全限定名进行查询

```mysql
select info.user.age from iuser;
```

## 排序数据

使用关键字`order by`在查询的时候得到有顺序的结果集

简单来说：

```sql
select id from user order by id
```

一般来说，order by使用的子句使用的列是显示所选择的列，当然也不完全一定，比如：

```sql
select name from user order by id
```

当然可以通过不止一列对结果集进行排序，比如：

```sql
select name, id from user order by id, name
```

注意如果按照多列进行排序，肯定是有先后顺序的，比如上面的语句中，会先按照id进行排序，只有当id相同的时候会按照name排序

> 如果你order by的语句中的所有列都相同，那么结果集会按照数据库底层中的顺序呈现

默认情况下，我们使用order by语句对结果集进行排序得到的结果是升序的：$0\to 9$、$A\to Z$，当然也可以让结果集降序排列，即使用desc关键字，比如：

```sql
select name from user order by id desc
```

当然可以结合上面的多列排序和降序：

```sql
select name, id from user order by id desc, name
```

上面的语句将结果集按照id进行降序排列，如果id相同的话会按照名称升序排列

> 如果希望多列降序排序，那么每列都需要加上一个desc

涉及到文本的排序，和大小写有关系，比如在ASCII码中，小写字母排在大写字母的后面，但是在大多数数据库中是大写小写认为是相同的

> 好像是可以配置让二者不同，但是我不会

一个比较关键的是：**如果使用order by 关键字，确保它位于select语句的最后一个位置**

## 过滤数据

这个没什么好说的，就是使用where关键字

```sql
select * from user where id = 1;
```

什么大于等于小于...反正常用的都行

就说几个注意的地方：

* 不检查匹配：在mysql中`!=`和`<>`效果相同

  ```sql
  select * from user where id <> 1
  ```

  上面的这个会检索所有id不为1的用户

* 范围检索：

  ```sql
  select * from user where id between 1 and 10
  ```

  上面的会检索id从1到10的用户

  between检索一个范围内的数据，范围的上下限需要使用and分割，并且下限在左，上限在右侧

* 检查空字段

  ```sql
  select * from user where name is null
  ```

  检索当前列为空的行

上面的要注意，如果我们希望匹配不具有特定值的所有行，他将不会返回带有null值的行

比如有下面的表：

|  id  | name |
| :--: | :--: |
|  1   | buzz |
|  2   | NULL |

如果我们执行语句：

```sql
select * from user where name != 'buzz';
```

那么上面并不会返回带有NULL的那一行

* 使用IN关键字：这个好说

  ```sql
  select * from user where id in(1, 2);
  ```

  in关键字会匹配括号中的所有条件

* 使用NOT关键字，这个就是否定后面的条件

  ```sql
  select * from user where id not in (1, 2);
  ```

  他会匹配所以有id不是1和2的用户

* 使用通配符进行过滤：这个必须使用like关键字：

  * `%`用来匹配任意个数字符：要注意：`where name like '%'`并不会匹配name为null的项
  * `_`用来匹配单个字符：有且必须有一个字符

  使用通配符进行过滤的时会影响性能，所以能不用就不用；就算要用，尽量不要在搜索模式的开头使用

* 使用正则表达式：这个需要使用关键字`regexp`

  * `.`用来匹配单个字符：相当于like中的`_`

  * 在mysql中正则表达式默认是不区分大小写的，如果希望区分大小写，需要添加关键字：`binary`：`where name regexp binary 'buzz'`

  * 正则表达式允许自身进行或运算：`where name regexp '1000|2000'`

  * 从多个字符中匹配一个：`where name regexp '[123]buzz'`；

    他等效匹配：`where name regexp '1buzz|2buzz|3buzz'`；就是说中括号内部每个字符算作一个匹配项

    此外`[^123]`将匹配除了1，2，3之外的所有项
    
    上面的可以写成：`[1-3]`表示范围匹配；显然：`[0-9]`表示匹配所有数字，而`[a-z]`表示匹配所有字母（如果不区分大小写的话）
    
  * 匹配特殊字符：因为特殊字符在正则表达式中具有特殊的含义，为了实现这些字符的匹配，需要进行转义：在特殊字符前加上：`\\`；比如：`where name regexp '\\.buzz'`
  
    正常情况的正则表达式中只需要一个反斜线表示转移，而在mysql中需要使用两个：MySQL
    自己解释一个，正则表达式库解释另一个  
  
  * 匹配字符类：正则表达式中具有字符类，其实就是关键字：
  
    ![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_regexp_%E5%AD%97%E7%AC%A6%E7%B1%BB.png)
  
    上面这个表格有点问题：其实：`[:alpha:]`仅仅相当于：`a-zA-z`，我们在使用posix字符类的时候需要形式上使用两个中括号：`[[:alpha:]]`
  
  * 匹配多个示例：我们上面说的都只能表示匹配一个，但正则表达式当然支持匹配多个字符：
  
    ![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_regexp_%E9%87%8D%E5%A4%8D%E5%85%83%E5%AD%97%E7%AC%A6.png)
  
    几个例子：
  
    * `'sticks?'`：它可以匹配：`stick`和`sticks`因为`?`表示前一个字符匹配0个或1个
    * `[[:digit:]]{4}`：它可以匹配四个连续的数字等效于：`[0-9][0-9][0-9][0-9]`
  
  * 定位符：我们上面的内容都是匹配一个字符串中任意位置的模式，为了匹配特定的文本，需要使用定位符
  
    ![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_regexp_%E5%AE%9A%E4%BD%8D%E7%AC%A6.png)
  
    现在我们有一个需求，匹配开头为数字或者小数点的字符串：`'^[0-9.]'`
  
    这里其实已经是第二次见到字符：`^`了，上一次是在集合中，他表示否定这个集合，如果在集合外表示字符串的开始
  
  * 其实在mysql中可以不使用表，仅使用字符串用来测试正则表达式是否匹配，语法为：`select '字符串' regexp '匹配文本'`，返回0的时候表示不匹配，返回1的时候返回匹配；比如：`select 'hello regexp '[0-9]'`，将返回0，显然`'hello'`不能和数字进行匹配    

# 创建其他字段

计算字段并不真实存在于数据库的表中，他是运行时在select语句内创建的

> 这种转换其实可以让DBMS做，也可以在数据返回后，我们手动格式化，但一般交给数据库去做更快

## 拼接字段

简单说我们自定义生成的列可以是由原来数据中的两列组合形成的，通过`Concat()`函数可以实现这个功能，比如：

```sql
select Concat(name, '(', sex, ')') from person;
```

上面的返回值的格式是：`name(sex)`

函数`Concat()`可以拼接多个串，每个串之间通过`,`分割（比如上面的就是拼接了4个串）

## 格式化字段

比如我们希望返回的结果中不包含多余的空格（字段前面的空格或者字段后面的空格）

通过：`RTrim(字段名)`可以实现去除字段右侧的空格；而通过`LTrim(字段名)`可以实现去除字段左侧的空格

如果希望同时去除左右侧的空格可以使用：`Trim(字段名)`，比如：

```sql
select Trim(name) from person
```

## 别名

其实相当于给导出的列赋一个别名，这个其实一个as关键字就好了，比如：

```sql
select Concat(Trim(name), '(', Trim(sex), ')') as personInfo from person;
```

## 执行计算

对于格式为数字的列，我们执行加减乘除的操作：比如：

```sql
select amount, item_price, amount*item_price as total from products;
```

返回值将包含三列：`amount`、`item_price`、`total`，其中最后一列大小等于前两列大小乘积

## 设置默认值

* 创建表时：比如字段`hits`我们希望不插入的时候就填入0，使用命令：

  ```sql
  create table test(
   id int primary key auto_increment,
   hits int default 0,
   name varchar(255)
  )engine = innodb;
  ```

* 创建好表后修改列：

  ```sql
  alter table test alter hits set default 0;
  ```



# 数据处理函数

## 文本处理函数

其实前面已经有和文本处理相关的函数了，就是`Trim()`函数

还有几个：

* 大小写：

  * `Upper()`返回大写
  * `Lower()`返回小写

* 字符串长度：`Length()`

* 返回字符串的soundex值：`Soundex()`：soundex是一个将任何文本串转换为描述其语音表示的字母数字模式的算法 ；soundex考虑了类似的发音字符和音节，使得能对串进行发音比较而不是字母比较。 

  > 离谱

## 日期和时间处理函数

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_%E6%97%A5%E6%9C%9F%E5%92%8C%E6%97%B6%E9%97%B4%E7%B1%BB%E5%9E%8B%E6%95%B0%E6%8D%AE.png)

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_%E6%97%A5%E6%9C%9F%E5%92%8C%E6%97%B6%E9%97%B4%E5%A4%84%E7%90%86%E5%87%BD%E6%95%B0.png)

举个例子吧：假如我希望筛选所有日期在：`2022-01-10`的数据，然而我们时间数据在数据库中存储的时候使用的类型是datetime，如果我们直接一个：

```sql
select * from person where birthday = '2022-01-10'
```

这样是不行的，因为他会默认后面的时间段为`00:00:00`，我们并不能检索到当天的全部数据

正确的写法是：

```sql
select * from person where date(birthday) = '2022-01-10'
```

所以如果我们希望通过日期过滤的时候，尽量使用`date()`函数；而如果是对时间进行过滤的时候使用`time()`函数

现在来一个难的：假如我们希望筛选所有日期在：`2022-01`的数据项：

两种方式：

```sql
select * from person where date(birthday) between '2022-01-01' and '2022-01-31';
```

```sql
select * from person where year(birthday) = '2022' and month(birthday) = '01'
```

## 数据处理函数

这些其实用的比较少

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86%E5%87%BD%E6%95%B0.png)

# 汇总函数

## 聚集函数

这种函数，运行在行上，计算返回单个值

比如：确定表中的数据行数、确定所有行的和、找到某一列的最值、均值

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/mysql_%E8%81%9A%E9%9B%86%E5%87%BD%E6%95%B0.png)

* avg函数：这个函数只能接受一个参数，如果计算多个列需要使用多个函数；他会忽略值为null的行
* count函数：如果我们写成：`count(*)`它将会计算所有的行；而如果写成：`count(column)`它计算的行中将不包含该列中为null的行

## 聚集不同的行

这个需要使用distinct关键字

比如：

```sql
select avg(distinct price) from product;
```

我们在对count函数使用`distinct`关键字的时候需要指明列名：

```sql
select count(distinct name) from person;
```

> 很显然，因为如果你不写列名，他就不知道按哪个行来了

# 分组数据

这个不好解释，举个例子：如果我们现在需要查询特定id的产品数量：

```sql
select count(*) from products where id = 1;
```

但如果我们希望查询所有id的产品数量，并以id分类：

```sql
select count(*), id from product group by id;
```

比如数据库：

```shell
mysql> select * from product;
+-----+------+---------------+
| pid | name | manufactoryId |
+-----+------+---------------+
|   1 | buzz |             1 |
|   2 | Buzz |             1 |
|   3 | BuZZ |             1 |
|   4 | a    |             2 |
|   5 | b    |             2 |
|   6 | c    |             2 |
+-----+------+---------------+
mysql> select count(*) as product_count, manufactoryId from product group by manufactoryId;
+---------------+---------------+
| product_count | manufactoryId |
+---------------+---------------+
|             3 |             1 |
|             3 |             2 |
+---------------+---------------+
```

可以看到制造商id为1的一共有三个产品，同理制造商id为2的也有三个产品

mysql内部会根据group by将所有数据分类，然后再将其聚合

## 应用

现在有一个需求，查询某个用户的文章的评论个数，在数据库中有文章表和评论表两个表，评论表通过外键 fk_aid 和文章表关联

这里涉及多表查询操作，同时也涉及到分组操作

```sql
select 
count(tc.cid) as comment_count
from t_article as ta left join t_comment as tc on ta.aid = tc.aid
where ta.uuid = ''
group by ta.aid
```

注意到这里就涉及到了数据分组，即统计某种分组下的个数

# group by 和 order by 同时使用

先说结论：

*   order by 需要在 group by 后面
*   order by 的 prop 需要是聚合函数，或者在 group by 中出现

目前的应用的话，有一个分页查询文章列表的 sql，

```sql
<select id="getArticleListByUuid" parameterType="string" resultMap="articleMap">
    select
        ta.aid as aid,
        ta.title as title,
        ta.cid as cid,
        t.name as category_name,
        ta.preview_url as preview_url,
        ta.`desc` as `desc`,
        ta.create_time as create_time,
        ta.update_time as update_time,
        ta.post_state as post_state,
        ta.hits as hits,
    	count(tc.cid) as comment_count
    from
        t_article as ta
        left join t_comment tc on ta.aid = tc.aid and tc.valid = 1
        left join t_category t on ta.cid = t.cid
    <where>
        ta.uuid = #{uuid} and
        <foreach collection="cidList" item="item"
        open="ta.cid in (" separator="," close=")" nullable="true">
            #{item}
        </foreach>
        and
        <foreach collection="stateList" item="item"
        open="ta.post_state in(" separator="," close=")" nullable="true">
            #{item}
        </foreach>
    </where>
    group by ta.aid
    order by ${sortBy} ${order}
</select>
```

注意到这里一共查询了三个表，一个文章表(主表)，一个评论表(副表)，一个分类表(副表)，通过 left join 进行外连接，通过文章表的主键进行分组(主要是针对评论表的数量)

排序的时候使用参数的形式传入，这里的参数可能是创建日期、评论数、访问量，显然这些变量既不是聚合函数，也不是 group 分组中的内容，所以如果直接这样查询的话，是无法得到理想的排序顺序的

所以这里在 group 中额外添加属性 sortBy，

# 外键

```sql
lcreate table tableName(
	...
    constraint 外键名 foreign key (键列) references 主表名称(主表列名称)
);
```

现在考虑一个很简单的例子，加入现在有两个表，一个学生表，一个教师表

学生表中的每个学生有自己的信息，同时每个学生绑定一个老师

老师表中的老师具有自己信息

学生和老师是多对一的关系

## 创建外键

创建表：

```sql
create table teacher(
id int not null primary key auto_increment,
name varchar(30));

create table student(
id int not null primary key auto_increment,
name varchar(30),
tid int,
constraint fktid foreign key(tid) references teacher(id));
```

现在我们如果希望向student表中添加数据，必须保证在teacher表中有某一行和其关联

所以我们添加学生数据的时候，在填入tid的时候需要保证在teacher表中有某一行的id和其对应；

同时对于teacher表中，如果某一行已经和student表中建立了联系，我们也不能删除这一行

外键是一种约束，我们上面给外键起了一个名字：fktid

起始可以在创建表是之后再添加这个外键的，执行语句：

```sql
alter table student add constraint fktid foreign key (tid) references teacher(id);
```

## 删除外键

如果希望删除这个外键需要执行语句：

```sql
alter table student drop foreign key if exists fktid;
```

## 级联操作

我们建立好外键的映射关系后，现在问题在于我们无法正常删除数据了

再teacher表中，如果我们希望删除，需要先把student表中对应的tid取值的先删除掉，然后再删掉teacher表中的项

既然现在是多对一的关系，一个正常的思路是，主表中（即一）某列进行了改动，我们希望子表中（即多）对应的列也发生改动；在我们这个具体的例子中，我们希望，如果我们更改了某个老师的id，我们希望在student表中对应外键也得到更改，如果我们在teaher表中删除了某个老师，我们希望在student表中也删除对应的行

上面这种操作其实就是级联操作

其实级联操作需要在创建表的时候进行配置：

```sql
alter table student add constraint fktid foreign key (tid) references teacher(id)
on update cascade on delete cascade
```

上面配置了外键同步跟新，同步删除

现在我们就可以删除teacher表中的项了，同时对应于student表中也会被删除；同时如果改变了teacher表中的某一行的主键，在student表中对应的外键也会得到更新

严格意义上的行为不只有同步删除：

```sql
on delete action
```

我们可以指定的`action`：

* `cascade`：同步删除
* `set null`：删除父表中的项时，子表中的项会被设置为null（这个条件成立的前提是子表对应列中允许存在null值）
* `no action`、`restrict`：不允许删除父表中的项

> `on update`也差不多

# 多表

多表的关系：

* 一对一：这个用的太少了：两张表中项是一对一的关系，实现的时候可以在任意一张表中添加唯一的外键指向另一张表的主键
* 一对多：这个其实用的多，我们上面的例子就是一对多的；为了建立这种关系，需要在一对多的中，多的那个表中添加一个外键，让这个外键指向一的那个表（一般指向的是主键），比如上面的学生和老师的例子
* 多对多：比如说学生和课程的关系

好吧现在其实就是多对多的关系不知道该怎么表达，下面具体以学生和课程举例：

学生表：

| sid  | name |
| :--: | :--: |
|  1   | buzz |
|  2   |  a   |

课程表：

| cid  | name |
| :--: | :--: |
|  1   | 数学 |
|  2   | 英语 |

现在我们知道对于学生表而言，sid是主键，对于课程表而言，cid是主键

为了实现映射关系，需要借助第三张表：假设起名叫stu_cur

这个第三方的表中至少有两列，分别为上面两张表的主键：

| sid  | cid  |
| :--: | :--: |
|  1   |  1   |
|  1   |  2   |
|  2   |  1   |

要明确的是，这两列都不是主键他们可重复，同时配置sid作为学生表的外键，cid作为课程表的外键，如上面的配置，即可实现多表的映射，

其实上面的还存在一定问题，我们知道从`sid`到`cid`的映射关系对是唯一的，即表中应该仅有一行`sid = 1`同时`cid = 1`，这个时候就需要联合主键了

## 笛卡尔积

现在有两个集合A、B，集合的笛卡尔积就是取两个集合中的组合

> 结果集的大小为A集合的大小和B集合大小的乘积

普通的多表查询，形如：

```sql
select * from tableA, tableB
```

这种形式的，查出来就是笛卡尔积，如果我们多表中是有联系的，比如说多对一的关系（即存在外键），那么结果集不仅数量存在冗余，且结果也是不正确的

如果要正常完成多表查询，需要消除笛卡尔积 

## 多表查询分类

现在有两张表：

部门表：`dept`、员工表：`emp`

对于部门表：

| id   | name |
| ---- | ---- |

分别对应部门的名称和部门的主键id

对于员工表：

| id   | name | sex  | salary | join_date | dept_id |
| ---- | ---- | ---- | ------ | --------- | ------- |

注意最后一项，`dept_id`，显然这是一个外键，对应着部门表中的某个id

### 内连接查询

* 隐式内连接

  针对上面的两个表：

  ```sql
  select * from emp,dept where emp.'dept_id' = dept.'id'
  ```

  然而在实际查询中，我们可能不是查询所有的数据，而仅仅是表中的某几列此时我们的select语句就不能是`*`

  比如上面的两张表，现在需要多表查询员工的名称，性别和部门的名称：一个比较直观的认知是将`*`改为具体的属性值，然而实际操作的时候发现，员工的名称和部门的名称都是`name`，所以这个写属性的时候需要将名字加上限定：

  ```sql
  select emp.'name', emp.'sex', dept.'name' from emp, detp where emp.'dept_id' = dept.'id'
  ```

  然而这种太长了，表名越长，sql语句越长，所以现在其实可以在查询的时候给表起一个别名

  ```sql
  select t1.'name', t1.'sex', t2.'name' from emp as t1, dept as t2 where t1.'dept_id' = t2.'id'
  ```

  上面的语句通过as关键字给表起了别名，实际中这个as关键字可以省略，而使用空格分割，比如：

  ```sql
  select t1.'name', t1.'sex', t2.'name' from emp t1, dept t2 where t1.'dept_id' = t2.'id'
  ```

  格式化一下：

  ```sql
  select 
  	t1.'name',
  	t1.'sex',
  	t2.'name'
  from 
  	emp as t1,
  	dept as t2
  where
  	t1.'dept_id' = t2.'id';
  ```

* 显式内连接 

  ```sql
  select 字段列表 from 表名1 [inner] join 表名2 on 条件
  ```

  查询的是两个表的交集

使用内连接进行查询，需要明确：

* 从哪些表中查询：要明确表之间的关系
* 取出非法数据的条件：比如上面的例子中条件就是员工表的`dept_id`和部门表的`id`相同
* 查询的字段：因为实际中并不是查询所有的列

**建议使用显式内连接**

###  外连接查询

* 左外连接：

  ```sql
  select 字段列表 from 表1 left [outer] join 表2 on 条件；
  ```

   查询的是左表和交集的部分

* 右外连接

  ```sql
  select 字段列表 from 表1 right [outer] join 表2 on 条件；
  ```

  查询的是右表和交集部分

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/sql_join.png)

### 子查询

查询中嵌套查询

比如上面的表中，查询工资最高的员工的信息：

```sql
select * from emp where salary = (select max(salary) from emp);
```

其实要分一下类：

* 子查询结果集为单行单列：我们上面的举例就是结果集为单行单列的

  子查询结果集作为条件，使用运算符：`<, =, >, <=, >=,...`用作判断

* 子查询结果集为多行单列：比如查询某些部门下所有员工的信息：

  ```sql
  select 
  	* 
  from 
  	emp 
  where 
  	dept_id 
  in (select 
      	id 
      from 
      	dept 
      where 
      	name = '部门1' 
      or 
      	name = '部门2');
  ```

  这种子查询的结果集也作为条件，而使用的运算符是in

* 子查询结果集为多行多列：

  比如希望得到员工入职在某日期后的员工和对应部门的信息：

  ```sql
  select * from dept t1, (select * from emp where join_date > '某个日期') t2 where t1.'id' = t2.'dept_id'
  ```

  这种子查询生成的是多行多列的，本身可以认为是一张虚拟表，参与进一步的查询

  当然上面的写法可以写成内连接的形式：

  ```sql
  select * from dept t1, emp t2 where t1.'id' = t2.'dept_id' and t2.'join_date' > '某日期'
  ```


# 范式

设计数据库时，需要遵循的一些规范。要遵循后边的范式要求，必须先遵循前边的所有范式要求

现在就考虑前三个范式：

* 第一范式（1NF）：每一列都是不可分割的原子数据项：只要能建表就肯定满足这个条件

* 第二范式（2NF）：在1NF的基础上，非码属性必须完全依赖于码（在1NF基础上消除非主属性对主码的部分函数依赖）

  * 函数依赖：A-->B,如果通过A属性(属性组)的值，可以确定唯一B属性的值。则称B依赖于A
    例如：学号-->姓名 （属性依赖）；（学号，课程名称） --> 分数  （属性组依赖）

  * 完全函数依赖：A-->B， 如果A是一个属性组，且B属性值需要依赖于A属性组中所有的属性值。
    例如：（学号，课程名称） --> 分数 

  * 部分函数依赖：A-->B， 如果A是一个属性组，则B属性值得确定只需要依赖于A属性组中某一些值即可。
    例如：（学号，课程名称） -- > 姓名

  * 传递函数依赖：A-->B, B -->C . 如果通过A属性(属性组)，可以确定唯一B属性的值，在通过B属性（属性组）的值可以确定唯一C属性的值，则称 C 传递函数依赖于A
    例如：学号-->系名，系名-->系主任

  * 码：如果在一张表中，一个属性或属性组，被其他所有属性所完全依赖，则称这个属性(属性组)为该表的码

    > 这不就是主键吗

    例如：该表中码为：（学号，课程名称）

       * 主属性：码属性组中的所有属性
       * 非主属性：除过码属性组的属性

  为了消除第二范式的部分依赖，一般需要拆分表 

* 第三范式（3NF）：在2NF基础上，任何非主属性不依赖于其它非主属性（在2NF基础上消除传递依赖）





