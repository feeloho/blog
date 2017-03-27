# MySQL 多列索引

多列索引也叫 联合索引, 也叫 复合索引. 在多列索引中有个 左前缀 的原则.

#### 数据表结构

```mysql
create table people(
	id int unsigned not null auto_increment,
	firstname char(20) not null,
	lastname char(20) not null,
	age tinyint unsigned not null,

	primary key (id)
);
```



当想通过 `firstname` `lastname` 以及 `age` 这3个字段查找记录的时候,

```mysql
select * from people where firstname="ya" and lastname="zi" and age=16
```

这时候如果使用 多列索引 可提高查询效率.



#### 创建多列索引

在建表语句里加上这行就行了: 

```mysql
key firstname_lastname_age (firstname, lastname, age)
```

若是已经创建表了, 想再添加索引, 使用下面的方法: 

```mysql
create index firstname_lastname_age on people(firstname, lastname, age)
```

或

```mysql
alert table people add index firstname_lastname_age (firstname, lastname, age)
```

其中 `firstname_lastname_age` 是索引的名称.



#### 左前缀原则

我创建了 多列索引 `(firstname, lastname, age)` 可以理解成是创建了 `(firstname)` `(firstname, lastname)` `(firstname, lastname, age)` 这3个.

那么, 左前缀原则 就是在字段上要从左往右开始依次使用. 不能跳着用.

```mysql
# 使用 firstname
firstname="ya"
# 使用 firstname lastname
firstname="ya" and lastname="zi"
# 使用 firstname lastname age
firstname="ya" and lastname="zi" and age=16

# 只能用上 firstname
firstname="ya" and age=16

# 使用 firstname lastname
firstname="ya" and lastname like "z%"
# 只能用上 firstname(因为lastname模糊查找左边内容未知, 不能和firstname连上)
firstname="ya" and lastname like "%i"
```

