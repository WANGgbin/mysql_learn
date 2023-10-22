描述 mysql 中的 foreign key.

# 作用

外键约束的目的时保证数据完整性/一致性。

比如，有两张表：用户表、订单表。订单表一定属于某个人，基于此约束关系，我们可以在订单表中创建一个外键，指向订单表的唯一索引。如果一个用户存在订单的话，
我们是不无法直接删除该用户的，因为删除用户后，用户对应的订单记录中存储的用户信息就是无效的。这就是保证数据完整性/一致性的体现。


# 定义

我们通过以下的语法来创建 foreign key.

- 创建数据表时候指定

```sql
create table person (
    id bigint unsigned not null primary key auto_increment
) engine = innodb;

create table orders (
    id bigint unsigned not null primary key auto_increment,
    price double,
    person_id bigint unsigned,
    foreign key(person_id) references person(id) restrict
) engine = innodb;
```

- 更改数据表

```sql
alter table orders
add foreign  key(person_id) references person(id) restrict;
```

# 属性

foreign key 的属性决定了删除/更新父表时数据库的行为。

属性，包括以下几种：
- restrict(默认属性)
  
    不允许更新/删除 父表中记录

- cascade

    更新：子表记录中对应的 foreign key 同步更新
    删除：子表记录删除
  
- set null
    
    > 特别需要注意：只有 foreign key 可以为 NULL， 该属性才会生效
  
    更新/删除：子表记录中对应的 foreign key 置为 NULL,

设置属性的时候，可以分别定义删除、更新时的行为。比如：
```sql
alter table orders
add foreign  key(person_id) references person(id) on delete restrict on update cascade;
```