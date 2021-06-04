# MySQL必知必会

## 一、初识MySQL

### 1.1 数据库概念

前端：页面展示数据

后台：连接数据库，向前端提供处理接口、连接前端（控制视图跳转，和给前端传递数据）

数据库：存数据



什么是数据库？

数据库是所有软件体系中最核心的存在。

作用：存储数据，管理数据的软件。



数据库分类？

**关系型数据库：**

- postgresql、MySQL等
- 通过表与表之间、行与列之间的关系进行数据的存储。
- 关系型数据库将数据保存在不同的表中，而不是将所有的数据放在一个大仓库中，这阿姨给你就可以增加数据并提高灵活性

**非关系型数据库：**

- redis、mongdb等not only sql
- 对象存储，通过对象自身的属性来决定（特别是相对于SQL而言比较灵活动态）。



数据库管理系统DBMS？

- 数据库的管理软件，可以科学有效的管理我们的数据，维护和获取数据
- 我们使用的本质是一个数据库管理系统DBMS，因为它可以管理和操作我们的数据。



<img src="image/Snipaste_2021-06-01_22-08-58.png" alt="Snipaste_2021-06-01_22-08-58" style="zoom:65%;" />



### 1.2 数据库基本操作

连接MySQL数据库管理系统（对于SQL的使用我个人推荐使用vscode中的MySQL插件进行使用，而且它可以支持多种数据库，包括PostgreSQL）：

```bash
$> mysql -u <username> -p[password]
```

修改MySQL数据库密码：

```mysql
-- 修改密码，该密码会通过md5算法进行加密然后存储到mysql的user表中
update mysql.user
set authentication_string=password('yourrpasswd')
where user='root'
    and host='localhost';
-- 刷新权限
flush privileges;
```

MySQL数据库本身相关的基本操作：

```mysql
-- 查看所有的数据库
show databases;

-- 选择/切换数据库
use <database_name>;

-- 查看数据库中所有的表
show tables;
```



## 二、操作数据库

结构化查询语言中主要有4种类型的语言命令组成，包括：

- **数据定义语言DDL**：它用来负责创建或删除存储数据用的数据库以及数据库以及数据库中的表等对象，包括常见的create、drop、alter指令。
- **数据操纵语言DML**：它用来负责查询或变更表中的记录，包括常见的select、insert、update、delete指令。
- **数据控制语言DCL**：它用来负责确认或取消对数据库中的数据进行的变更，包括常见的commit、rollback、grant、revoke指令。

不过我们主要接触的SQL语句还是数据操纵语言DML，即CRUD增删改查。



### 2.1 创建和操作表DDL

#### 2.1.1 表的创建

表的创建可以通过`create table`命令来完成：

```mysql
CREATE TABLE [IF NOT EXISTS] <表名> (
    <列名> <数据类型> [字段属性] [索引] [注释],
    <列名> <数据类型> [字段属性] [索引] [注释],
    ...
    <列名> <数据类型> [字段属性] [索引] [注释],
    PRIMARY KEY (主键字段)
) [驱动引擎] [字符集编码] [注释];
```

当表创建完毕之后我们可以通过`show`命令来查看创建这个表的MySQL语句，与之类似，对于数据库的创建实际上我们也可以使用`show`命令来进行查看，如下所示：

```mysql
-- 查看创建表的SQL语句
SHOW CREATE TABLE class;

-- 查看创建数据库的SQL语句
SHOW CREATE DATABASE crashcourse;
```



下面以一个实例进行演示：

```mysql 
CREATE TABLE IF NOT EXISTS class (
    student_id INT NOT NULL AUTO_INCREMENT comment '学号',
    student_name CHAR(64) NOT NULL,
    student_country CHAR(32) NOT NULL DEFAULT 'China',
    student_gender CHAR(12) NOT NULL,
    student_home_addr CHAR(255) NULL,
    PRIMARY KEY (student_id, student_name)
) ENGINE = InnoDB default charset=utf8;
```

在上面的MySQL语句中我们使用了如下的一些常用关键字：

1. `if not exists`指出这个表只有在同名表不存在的情况下创建；
2. `auto_increment`指出在每一次添加一行新的数据的时候，这个id号会自动增一，所以我们对于这一列的值我们可以不加以指定，让你自动生成即可；
3. `comment`是用在建表时使用的内置注释，方便后续查看；
4. `null`和`not null`指出表中的列在添加的时候是否可以为空；
5. `default`表明了在添加新行时若对这一行该列中的值加以指定，那么该列就会被指定默认值；
6. `primary key`指定了表的主键，它必须与其他行同列中的值不同，且主键可以由多个列名共同组成；
7. `engine`指出这个表由哪一个引擎进行驱动。
8. `deafult charset`用来指定表使用何种字符编码。

> 在阿里巴巴的数据库规范中必须会包含如下几个字段：主键id、版本号version（用于乐观锁）、伪删除is_delete、创建时间gmt_create、修改时间gmt_update。
>
> 同时还有一个建议就是在创建表的时候尽可能使用"``"这两个符号来对表名、字段名进行括定，这样对于中间含有空格的字符串就可变得合法。



#### 2.1.2 数据类型

在MySQL中主要有如下4种数据类型，包括①串数据类型；②数值数据类型；③日期和时间类型；④二进制数据类型。

- 串数据类型，具体可以分成定长串数据类型和可变长串数据类型，常用的如下所示：

|   数据类型   |          描述           |
| :----------: | :---------------------: |
|    `char`    | 可指定1~255长度的定长串 |
|  `varchar`   |   最大为255的可变长串   |
|    `text`    |   最长可达64k的文本串   |
|  `tinytext`  |  最大长度为255的文本串  |
| `mediumtext` |  最大长度为16k的文本串  |
|  `longtext`  | 最大长度可为4G的文本串  |

- 数值数据类型，常用的如下所示：

|  数据类型   |                          描述                          |
| :---------: | :----------------------------------------------------: |
|  `boolean`  |                        布尔类型                        |
|  `tinyint`  |                      1字节整数值                       |
| `smallint`  |                      2字节整数值                       |
| `mediumint` |                      3字节整数值                       |
|    `int`    |                      4字节整数值                       |
|  `bigint`   |                      8字节整数值                       |
|   `float`   |                      单精度浮点数                      |
|  `double`   |                      双精度浮点数                      |
|  `decimal`  | 字符串形式的浮点数，精度可变，<br />在金融领域经常使用 |

- 日期和时间数据类型，常见的如下所示：

|  数据类型   |            描述            |
| :---------: | :------------------------: |
|   `date`    |   日期，格式为YYYY-MM-DD   |
|   `time`    |    时间，格式为HH:MM:SS    |
| `datetime`  |          日期时间          |
| `timestamp` | 时间戳，自1970.1.1的微秒数 |
|   `year`    |             年             |

- 二进制数据类型，常见的如下所示：

|   数据类型   |            描述             |
| :----------: | :-------------------------: |
|    `blob`    |    最长64K二进制数据类型    |
|  `tinyblob`  | 最长255字节的二进制数据类型 |
| `mediumblob` |   最长64M的二进制数据类型   |
|  `longblob`  |   最长4G的二进制数据类型    |

> 还有一个需要注意的就是NULL这个特殊值，它并没有“值”，任何数据类型的值与之运算的结果都是NULL！



#### 2.1.3 表的字段属性

在创建表的时候常会使用一些字段属性来限制表中的列（或者称其为字段），常见的如下所示：

|     字段属性      |                             描述                             |
| :---------------: | :----------------------------------------------------------: |
|    `unsigned`     |               无符号整数，适用于整数值类型字段               |
|    `zerofill`     | 对不足的位数进行零填充，适用于整数值类型。<br />例如定义int(5)，并将值设为34，则前面的位会填充0 |
| `auto_increment`  | 对数值在插入新行时进行自增，默认+1，适用于作为主键的整数类型 |
| `not null`/`null` |                 插入新行时必须设值/不必设值                  |
|     `default`     |       新行中未显式设定的字段默认设定建表时指定的默认值       |

同时我们可以在MySQL中可以通过`desc`命令来查看某一张指定表的字段属性值，如下：

```mysql
DESC <表名>;
```



#### 2.1.4 数据库引擎

在驱动表的过程中起到最为重要的东西就是创建表时设置（或默认设置）的数据库引擎。在MySQL中有多种引擎可以选择，但是最使用的主要有InnoDB、MyISAM、MEMORY和其他的一些。

其中InnoDB是一个可靠的事务处理引擎，但缺点就是不支持全文本搜索，同时它也是MySQL的默认数据库引擎；而MyISAM是早先年MySQL的默认引擎，性能极高，支持全文本搜索，不过不支持时下热门的事务处理；最后就是MEMORY，它的功能类似于MyISAM，特点就是数据存储在内存而不是硬盘当中。下面对比了InnoDB和MyISAM的区别：

|    功能    | MyISAM |    InnoDB     |
| :--------: | :----: | :-----------: |
|  事务支持  |   ❌    |       ✔       |
| 数据行锁定 |   ❌    |       ✔       |
|  外键约束  |   ❌    |       ✔       |
| 全文本搜索 |   ✔    |       ❌       |
| 表空间大小 |  较小  | 较大，约为2倍 |

对于两者主要根据它们各自的特点进行选择：如果希望节约空间且较快的速度，那么可以选择MyISAM；如果希望有着较高的安全性、支持事务处理且能够多表多用户操作，那么选择InnoDB。而且两者产生的实际存储文件也各不相同，这些不同引擎驱动的表都位于mysql的data目录下，相比之下MyISAM会产生更多的文件。

> 随便提一句，MySQL的全局默认字符集编码设定是放在MySQL安装目录下的my.ini文件中，配置为character-set-server，一般被设置为utf8。



#### 2.1.5 表的更新

表的更新可以通过`alter table`来完成：

```mysql
-- 修改表名
ALTER TABLE <old_tablename> RENAME AS <new_tablename>;

-- 添加新列（或字段）
ALTER TABLE <tablename>
ADD <column_name> <type>;

-- 修改表中字段的属性或约束
ALTER TABLE <tablename>
MODIFY <column_name> <新的字段属性>;

-- 修改表中字段名以及它的字段属性
ALTER table <tablename>
CHANGE <old_column_name> <new_column_name> <新的字段属性>;

-- 删除列（或字段）
ALTER TABLE <tablename>
DROP <column_name>;

-- 定义表的外键（使得该列与另一个表中的列进行关联）
ALTER TABLE <tablename>
ADD CONSTRAINT fk_<tablename>_<another_tablename>
FOREIGN KEY (column_name) REFERENCES <another_tablename> (column_name);
```

一般来说我们并不建议在定义表之后再去对表进行更新修改，而是更建议在创建表的时候就对表的结构进行良好的设计。

除此之外，修改表名的命令我们还可以通过如下命令完成：

```mysql
RENAME TALBE <old_tablename> TO <new_tablename>;
```



#### 2.1.6 表的删除

表的删除可以通过`drop table`命令来完成：

```mysql
DROP TABLE IF EXISTS <tablename>;
```

> 随便提示下，对于这些表的创建和删除最好添加一个if exists这样的判断语句，这样可以防止报错发生。



### 2.2 数据的操纵

#### 2.2.1 外键约束

在MySQL中外键约束的作用就是将表中的某个或某些字段对另一个表中的字段进行引用，使得两个表形成关联，这样就可以控制存储在外键表中的数据，使得该表外键所在列的数据要不就是所引用表中字段已存在的值，要不就是NULL，从而维护数据的一致性和完整性。其中引用另一个表中的字段的表称为从表，而在被引用的表被称为主表。

一个典型的案例就是员工的部门，若存在两个表，一个是员工信息表，另一个是部门信息表，那么员工表中的部门信息一定存在于部门信息表之中，因此两者存在着一定的关联。如果愿意，我们就可以使用关键约束来限制员工信息表中的部门信息字段。

我们可以通过如下两种方式来向一个表中的某个或某些字段添加外键约束：

- 一种方式是在**创建表时添加外键约束**，使得其中的某些字段对另一个表中的字段进行引用。不过这种方式略显笨重，不是经常使用。其具体语法如下：

```mysql
CREATE TABLE [IF NOT EXISTS] <tablename> (
    ...
	[外键字段名] <数据类型> [字段属性] [索引] [注释] ,
    ...
    KEY <外键约束名> (外键字段名) ,
    CONSTRAINT <外键约束名> FOREIGN KEY (外键字段名) REFERENCES <另一个表>(外键字段名)
) ENGINE = InnoDB DEFAULT CHARSET = utf8;
```

上面的constraint一行的意思就是为当前表添加一个外键约束，使得当前表中的外键字段引用另一个表中的外键字段，这样当前表中该外键字段列的值必须是主表中该字段值中之一或者NULL，否则无法添加新的数据。我们以如下实例来展示外键约束的使用：

```mysql
-- 创建主表
CREATE TABLE IF NOT EXISTS grade (
    gradeID INT NOT NULL AUTO_INCREMENT ,
    gradeName CHAR(16) NOT NULL ,
    PRIMARY KEY (gradeID)
) ENGINE = InnoDB DEFAULT CHARSET = utf8;

-- 创建从表，并在创建之时添加外键约束
CREATE table IF NOT EXISTS student (
    id INT NOT NULL AUTO_INCREMENT ,
    name CHAR(32) NOT NULL ,
    gender CHAR(8) NOT NULL ,
    gradeID INT NOT NULL ,
    address VARCHAR(64) NULL,
    email CHAR(32)  NULL ,
    PRIMARY KEY (id) ,
    KEY fk_gradeID (gradeID) ,
    CONSTRAINT fk_gradeID FOREIGN KEY (gradeID) REFERENCES grade(gradeID)
) ENGINE = InnoDB DEFAULT CHARSET = utf8;
```

- 另一种方式就是在表创建之后通过ALTER命令来添加外键约束。这也是最常见的实现外键约束的方式，其格式在上面<创建和操作表>一节中已经讲述：

```mysql
ALTER TABLE <tablename>
ADD CONSTRAINT <外键约束名>
FOREIGN KEY (column_name) REFERENCES <another_tablename> (column_name);
```



> 上述的外键称为物理外键，属于数据库级别的外键，不建议使用。因为这种关联关系很容易造成不必要的困扰。最佳的实践就是数据库中存储单纯的表，只用来存储数据，单单含有行（数据）和列（字段），一般在实际中都是通过程序来实现逻辑上的外键关系。



#### 2.2.2 数据的插入

向表中插入数据（新的一行）的基本语法如下：

```mysql
INSERT INTO <tablename> (
    ...
    [字段n],
    [字段m],
    ...
)
VALUES (
	...
    [字段n所对应的值],
    [字段m所对应的值],
    ...
);
```

一般来说上面的字段名可以选择性添加，除非这些字段具有非空约束或默认值，这样我们在添加新的一行时必须给它指定的数据。同时上面给定的值必须与字段一一对应，否则就无法向其添加数据。更甚至我们可以将第一个`()`取消，只留下VALUES后面的部分，那么此时我们就必须从第一个字段开始提供每一个值，除非最后几个字段可以为空或具有默认值。

如果我们想添加多条数据，那么最好的方式如下所示：

```mysql
INSERT INTO <tablename> (
    ...
    [字段n],
    [字段m],
    ...
)
VALUES (
	...
    [字段n所对应的值],
    [字段m所对应的值],
    ...
)
...
, (
	...
    [字段n所对应的值],
    [字段m所对应的值],
    ...
);
```



####  2.2.3 数据的更新和删除

对于数据（准确点说就是表中的某一行）的更新我们可以通过如下的SQL语句来更新：

```mysql
UPDATE <tablename>
SET <column1> = <newVal1>,
    ...
	<columnN> = <newValN>
WHERE <指定条件>;
```

而对于数据的删除则可以通过如下的命令

```mysql
DELETE FROM <tablename>
WHERE <指定条件>;
```

其中where子句中常常涉及到一些逻辑/比较操作符，如下展示了一些比较常见的：

|        操作符         |     描述     |
| :-------------------: | :----------: |
|          `=`          |     相等     |
|      `!=`或`<>`       |     不等     |
|          `<`          |     小于     |
|         `<=`          |   小于等于   |
|          `<`          |     大于     |
|         `>=`          |   大于等于   |
| `between ... and ...` | 在指定范围内 |
|    `in (..,..,..)`    |  在集合之内  |
|         `and`         |      与      |
|         `or`          |      或      |
|         `not`         |      非      |

如果我们在上面的delete子句中没有指定删除的条件的话，那么这个delete子句就会将整个表中的数据全部删除，不过它并不会删除表本身。如果用户的目的就是为了清空表中的所有数据，那么最好的方法就是使用truncate语句而不是delete，如下所示：

```mysql
TRUNCATE TABLE <tablename>; -- TABLE可以省略
```

`TRUACATE`与`DELETE`子句的不同之处在于`TRUACATE`会复位自增列计数器，使其归零；同时还不影响事务的处理。
