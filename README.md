# FlinkSQL数据脱敏和行级权限解决方案及源码

支持面向用户级别的数据脱敏和行级数据访问控制，即特定用户只能访问到脱敏后的数据或授权过的行。此方案是实时领域Flink的解决方案，类似于离线数仓Hive Ranger中的Row-level Filter和Column Masking方案。

<br/>

| 序号 | 作者 | 版本 | 时间 | 备注 |
| -- | --- | --- | --- | --- |
| 1 | HamaWhite | 1.0.0 | 2022-12-15 | 1. 支持行级权限 |
| 2 | HamaWhite | 1.0.1 | 2023-04-11 | 1. 通过 [manifold-ext](https://github.com/manifold-systems/manifold/tree/master/manifold-deps-parent/manifold-ext) 扩展Flink ParserImpl类的方法</br> 2. 自定义calcite visitor来增加行级权限，不再改SqlSelect源码 |
| 3 | HamaWhite | 2.0.0 | 2023-04-23 | 1. 支持数据脱敏 |
| 4 | HamaWhite | 2.0.1 | 2023-05-07 | 1. 语法校验后再增加权限约束 |


<br/>
源码地址: https://github.com/HamaWhiteGG/flink-sql-security 

> 注: 如果用IntelliJ IDEA打开源码，请提前安装 **Manifold** 插件。

<br/>

**如果希望进一步阅读技术细节，请查看系列文章**:
1. [FlinkSQL的行级权限解决方案及源码](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/row-filter/README.md)
2. [FlinkSQL的数据脱敏解决方案及源码](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/data-mask/README.md)


## 一、基础知识
### 1.1 数据脱敏
数据脱敏(Data Masking)是一种数据安全技术，用于保护敏感数据，以防止未经授权的访问。该技术通过将敏感数据替换为虚假数据或不可识别的数据来实现。
例如可以使用数据脱敏技术将信用卡号码、社会安全号码等敏感信息替换为随机生成的数字或字母，以保护这些信息的隐私和安全。

### 1.2 行级权限
行级权限(Row-Level Security)是一种数据权限控制机制，它允许系统管理员或数据所有者对数据库中的数据行进行细粒度的访问控制。
行级权限可以限制用户对数据库中某些行的读取或修改，以确保敏感数据只能被授权人员访问。行级权限可以基于多种条件来定义，如用户角色、组织结构、地理位置等。通过行级权限控制，可以有效地防止未经授权的数据访问和泄露，提高数据的安全性和保密性。
在大型企业和组织中，行级权限通常被广泛应用于数据库、电子表格和其他数据存储系统中，以满足安全和合规性的要求。

### 1.3 简单案例
例如针对订单表，在数据脱敏方面，**用户A**查看到的顾客姓名(`customer_name`字段)全部被掩盖掉，**用户B**查看到顾客姓名只会显示前4位，剩下的用`x`代替。
在行级权限方面，**用户A**只能查看到**北京**区域的数据，**用户B**只能查看到**杭州**区域的数据。
![Data mask and Row-level filter example data.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/Data%20mask%20and%20Row-level%20filter%20example%20data.png)

### 1.4 组件版本
| 组件名称 | 版本 | 备注 |
| --- | --- | --- |
| Flink | 1.16.1 |  |
| Flink-connector-mysql-cdc | 2.3.0 |  |


## 二、 FlinkSQL执行流程介绍
可以参考作者文章[[FlinkSQL字段血缘解决方案及源码]](https://github.com/HamaWhiteGG/flink-sql-lineage/blob/main/README_CN.md)，本文根据Flink1.16修正和简化后的执行流程如下图所示。
![FlinkSQL simple-execution flowchart.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/FlinkSQL%20simple-execution%20flowchart.png)

在`CalciteParser`进行`parse()`和`validate()`处理后会得到一个SqlNode类型的抽象语法树(`Abstract Syntax Tree`，简称AST)，本文会针对此抽象语法树来组装行级过滤条件后生成新的AST，以实现行级权限控制。

## 三、解决方案
### 3.1 数据脱敏
针对输入的Flink SQL，在`CalciteParser`进行语法解析(parse)和语法校验(validate)后生成抽象语法树(`Abstract Syntax Tree`，简称AST)后，采用自定义
`Calcite SqlBasicVisitor`的方法遍历AST中的所有`SqlSelect`，获取到里面的每个输入表。如果输入表中字段有配置脱敏条件，则针对输入表生成子查询语句，
并把脱敏字段改写成`CAST(脱敏函数(字段名) AS 字段类型) AS 字段名`,再通过`CalciteParser.parseExpression()`把子查询转换成SqlSelect，
并用此SqlSelect替换原AST中的输入表来生成新的AST，最后得到新的SQL来继续执行。
![FlinkSQL data mask solution.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/FlinkSQL%20data%20mask%20solution.png)

### 3.2 行级权限
如果输入SQL包含对表的查询操作，则一定会构建Calcite SqlSelect对象。因此限制表的行级权限，只要对Calcite SqlSelect对象的Where条件进行修改即可，而不需要解析用户执行的各种SQL来查找配置过行级权限条件约束的表。在`CalciteParser`进行语法解析(parse)和语法校验(validate)后生成抽象语法树AST，其会构造出SqlSelect对象，采用自定义`Calcite SqlBasicVisitor`来重新生成新的SqlSelect Where条件。

首先通过执行用户和表名来查找配置的行级权限条件，系统会把此条件用CalciteParser提供的`parseExpression(String sqlExpression)`方法解析生成一个SqlBasicCall再返回。然后结合用户执行的SQL和配置的行级权限条件重新组装Where条件，即生成新的带行级过滤条件Abstract Syntax Tree，最后基于新AST(即新SQL)再执行。
![FlinkSQL row-level filter solution.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/FlinkSQL%20row-level%20filter%20solution.png)

### 3.3 整体执行流程
针对输入的Flink SQL，由`CalciteParser`进行语法解析和语法校验后生成抽象语法树AST。由于行级权限会修改SELECT语句中的Where子句，为避免修改数据脱敏生成子SELECT语句中的WHERE，因此先根据行级权限方案替换AST中的Where子句，然后再根据数据脱敏方案把AST中的输入表改为子查询，最后得到新的SQL来继续执行。
![Data mask and Row-level filter overall execution flowchart.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/Data%20mask%20and%20Row-level%20filter%20overall%20execution%20flowchart.png)


## 四、案例讲解
项目源码中有比较多的单元测试用例，可用于学习和测试，下面只描述部分测试点。

测试用例中的catalog名称是`hive`，database名称是`default`。

```shell
$ cd flink-sql-security
$ mvn test
```
用户A和用户B的权限策略配置如1.3小节所述，即:
- **用户A**只能查看到**北京**区域的数据，且顾客姓名(`customer_name`字段)全部被掩盖掉;
- **用户B**只能查看到**杭州**区域的数据，且顾客姓名只会显示前4位，剩下的用`x`代替。

### 4.1 输入SQL
```sql
SELECT order_id, customer_name, product_id, region FROM orders
```

### 4.2 用户A的最终执行SQL
```sql
SELECT
    orders.order_id,
    orders.customer_name,
    orders.product_id,
    orders.region
FROM (
    SELECT 
         order_id,
         order_date,
         CAST(mask(customer_name) AS STRING) AS customer_name,
         product_id,
         price,
         order_status,
         region
    FROM 
         hive.default.orders
     ) AS orders
WHERE
    orders.region = 'beijing'
```

### 4.3 用户B的最终执行SQL
```sql
SELECT
    orders.order_id,
    orders.customer_name,
    orders.product_id,
    orders.region
FROM (
    SELECT 
         order_id,
         order_date,
         CAST(mask_show_first_n(customer_name, 4, 'x', 'x', 'x', -1, '1') AS STRING) AS customer_name,
         product_id,
         price,
         order_status,
         region
    FROM 
         hive.default.orders
     ) AS orders
WHERE
    orders.region = 'hangzhou'
```

## 五、下一步计划
1. FlinkSQL Access策略，即库、表、字段的权限控制。
2. ranger-flink-plugin。
