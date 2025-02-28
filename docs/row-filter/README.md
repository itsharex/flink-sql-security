# FlinkSQL的行级权限解决方案及源码

支持面向用户级别的行级数据访问控制，即特定用户只能访问授权过的行，隐藏未授权的行数据。此方案是实时领域Flink的解决方案，类似于离线数仓Hive中Ranger Row-level Filter方案。

<br/>
源码地址: https://github.com/HamaWhiteGG/flink-sql-security 

> 注: 如果用IntelliJ IDEA打开源码，请提前安装 **Manifold** 插件。


## 一、基础知识
### 1.1 行级权限
行级权限(Row-Level Security)是一种数据权限控制机制，它允许系统管理员或数据所有者对数据库中的数据行进行细粒度的访问控制。
行级权限可以限制用户对数据库中某些行的读取或修改，以确保敏感数据只能被授权人员访问。行级权限可以基于多种条件来定义，如用户角色、组织结构、地理位置等。通过行级权限控制，可以有效地防止未经授权的数据访问和泄露，提高数据的安全性和保密性。
在大型企业和组织中，行级权限通常被广泛应用于数据库、电子表格和其他数据存储系统中，以满足安全和合规性的要求。

例如针对订单表，**用户A**只能查看到**北京**区域的数据，**用户B**只能查看到**杭州**区域的数据。
![Row-level filter example data.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/Row-level%20filter%20example%20data.png)

### 1.2 业务流程
#### 1.2.1 设置行级权限
管理员配置用户、表、行级权限条件，例如下面的配置。
| 序号 |  用户名 | 表名 | 行级权限条件 | 
| --- | --- | --- | --- | 
| 1 | 用户A | orders | region = 'beijing' | 
| 2 | 用户B | orders | region = 'hangzhou' | 


#### 1.2.2 用户查询数据
用户在系统上查询`orders`表的数据时，系统在底层查询时会根据该用户的行级权限条件来自动过滤数据，即让行级权限生效。

当用户A和用户B在执行下面相同的SQL时，会查看到不同的结果数据。

```sql
SELECT * FROM orders
```

**用户A查看到的结果数据是**:
| order_id |  order_date | customer_name | price |  product_id |  order_status |  region | 
| --- | --- | --- | --- |  --- |  --- |  --- | 
| 10001 | 2020-07-30 10:08:22 | Jack | 50.50 | 102 | false | beijing |
| 10002 | 2020-07-30 10:11:09 | Sally | 15.00 | 105 | false | beijing | 

<br/>

**用户B查看到的结果数据是**:
| order_id |  order_date | customer_name | price |  product_id |  order_status |  region |  
| --- | --- | --- | --- |  --- |  --- |  --- | 
| 10003 | 2020-07-30 12:00:30 | Edward | 25.25 | 106 | false | hangzhou | 
| 10004 | 2022-12-15 12:11:09 | John | 78.00 | 103 | false | hangzhou | 

## 二、Hive行级权限解决方案
在离线数仓工具Hive领域，由于发展多年已有Ranger来支持表数据的行级权限控制，详见参考文献[[2]](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.0/authorization-ranger/content/row_level_filtering_in_hive_with_ranger_policies.html)。下图是在Ranger里配置Hive表行级过滤条件的页面，供参考。

![Hive-Ranger row-level filter.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/Hive-Ranger%20row-level%20filter.png)

<br/>

但由于Flink实时数仓领域发展相对较短，Ranger还不支持FlinkSQL，以及要依赖Ranger会导致系统部署和运维过重，因此开始**自研实时数仓的行级权限解决工具**。

## 三、FlinkSQL行级权限解决方案
### 3.1 解决方案
#### 3.1.1 FlinkSQL执行流程
可以参考作者文章[[FlinkSQL字段血缘解决方案及源码]](https://github.com/HamaWhiteGG/flink-sql-lineage/blob/main/README_CN.md)，本文根据Flink1.16修正和简化后的执行流程如下图所示。
![FlinkSQL simple-execution flowchart.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/FlinkSQL%20simple-execution%20flowchart.png)

在`CalciteParser`进行`parse()`和`validate()`处理后会得到一个SqlNode类型的抽象语法树(`Abstract Syntax Tree`，简称AST)，本文会针对此抽象语法树来组装行级过滤条件后生成新的AST，以实现行级权限控制。

#### 3.1.2 Calcite对象继承关系
下面章节要用到Calcite中的SqlNode、SqlCall、SqlIdentifier、SqlJoin、SqlBasicCall和SqlSelect等类，此处进行简单介绍以及展示它们间继承关系，以便读者阅读本文源码。

| 序号 | 类 | 介绍 |
| --- | --- | --- |
| 1 | SqlNode | A SqlNode is a SQL parse tree. |
| 2 | SqlCall | A SqlCall is a call to an SqlOperator operator. |
| 3 | SqlIdentifier | A SqlIdentifier is an identifier, possibly compound. |
| 4 | SqlJoin | Parse tree node representing a JOIN clause. |
| 5 | SqlBasicCall | Implementation of SqlCall that keeps its operands in an array. |
| 6 | SqlSelect | A SqlSelect is a node of a parse tree which represents a select statement, the parent class is SqlCall |

<img src="https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/Calcite%20SqlNode%20diagrams.png" alt="Calcite SqlNode diagrams.png"  style="width:70%;">

#### 3.1.3 解决思路

如果输入SQL包含对表的查询操作，则一定会构建Calcite SqlSelect对象。因此限制表的行级权限，只要对Calcite SqlSelect对象的Where条件进行修改即可，而不需要解析用户执行的各种SQL来查找配置过行级权限条件约束的表。在`CalciteParser`进行语法解析(parse)和语法校验(validate)后生成抽象语法树AST，其会构造出SqlSelect对象，采用自定义`Calcite SqlBasicVisitor`来重新生成新的SqlSelect Where条件。

首先通过执行用户和表名来查找配置的行级权限条件，系统会把此条件用CalciteParser提供的`parseExpression(String sqlExpression)`方法解析生成一个SqlBasicCall再返回。然后结合用户执行的SQL和配置的行级权限条件重新组装Where条件，即生成新的带行级过滤条件Abstract Syntax Tree，最后基于新AST(即新SQL)再执行。
![FlinkSQL row-level filter solution.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/FlinkSQL%20row-level%20filter%20solution.png)

### 3.2 详细方案
主要通过Calcite提供的访问者模式自定义RowFilterVisitor来实现，遍历AST中所有的SqlSelect对象重新生成Where子句。
下面详细描述替换Where子句的步骤，整体流程如下图所示。

![Row-level filter-rewrite the main process.png](https://github.com/HamaWhiteGG/flink-sql-security/blob/dev/docs/images/Row-level%20Filter-Rewrite%20the%20main%20process.png)

1. 遍历AST中的SELECT语句。
2. 判断SELECT语句中的FROM类型，按照不同类型对应执行下面的步骤3、4和10。
3. 如果FROM是SqlJoin类型，则分别遍历其左Left和Right右节点，即执行当前步骤3和步骤5。由于可能是三张表及以上的Join，因此进行递归处理，即针对其左节点跳回到步骤2。
4. 如果FROM是SqlBasicCall类型，还需要判断是否来自子查询，是则跳转到步骤10继续遍历AST，后续步骤1会对子查询中的SELECT语句进行处理。否则跳转到步骤6。
5. 递归处理Join的右节点，即跳回到步骤2。
6. 根据当前执行SQL的用户名和表名来查找已配置的行级约束条件，并调用Calcite进行解析表达式操作，生成permissions(类型是上文提到的SqlBasicCall)。
7. 给行级权限解析后的permissions增加别名，例如行级约束条件是region = '北京'，来自于orders表，别名是o。则此步骤处理后的结果是o.region = '北京'。
8. 组装旧where和行级权限permissions来生成新的where，即把两个约束用and联合起来，然后执行步骤9。
9. 用新where替换掉旧where。
10. 继续遍历AST，找到里面的SELECT语句进行处理，跳回到步骤1。

## 四、用例测试
用例测试数据来自于CDC Connectors for Apache Flink
[[6]](https://ververica.github.io/flink-cdc-connectors/master/content/%E5%BF%AB%E9%80%9F%E4%B8%8A%E6%89%8B/mysql-postgres-tutorial-zh.html)官网，
本文给`orders`表增加一个region字段，再增加`'connector'='print'`类型的print_sink表，其字段和`orders`表的一样。数据库建表及初始化SQL位于data/database目录下。

测试用例中的catalog名称是`hive`，database名称是`default`。

下载本文源码后，可通过Maven运行单元测试。

```shell
$ cd flink-sql-security
$ mvn test
```
详细测试用例可查看源码中的单测`RewriteRowFilterTest`和`ExecuteRowFilterTest`，下面只描述三个案例。

### 4.1 测试SELECT

#### 4.1.1 输入SQL
```sql
SELECT order_id, customer_name, product_id, region FROM orders
```
#### 4.1.2 输出SQL
```sql
SELECT
    orders.order_id,
    orders.customer_name,
    orders.product_id,
    orders.region
FROM
    hive.default.orders AS orders
WHERE
    orders.region = 'beijing' 
```
#### 4.1.3 测试小结
输入SQL是一个简单SELECT语句，经过语法分析和语法校验后FROM类型是SqlBasicCall，SQL中的表名orders会被替换为完整的`hive.default.orders`，别名是`orders`。由于输入SQL中没有WHERE条件，只需要把行级过滤条件`region = 'beijing'`追加到WHERE后，同时带上表别名`orders`。

### 4.2 两表JOIN

#### 4.2.1 输入SQL
```sql
SELECT 
    o.order_id, 
    o.customer_name,
    o.product_id,
    o.region,
    p.name,
    p.description 
FROM 
    orders AS o 
LEFT JOIN 
    products AS p 
ON 
    o.product_id = p.id 
WHERE 
    o.price > 45.0 OR o.customer_name = 'John'
```
#### 4.2.2 输出SQL
```sql
SELECT 
    o.order_id, 
    o.customer_name, 
    o.product_id, 
    o.region, 
    p.name, 
    p.description 
FROM 
    hive.default.orders AS o 
LEFT JOIN 
    hive.default.products AS p 
ON 
    o.product_id = p.id 
WHERE 
    (o.price > 45.0 OR o.customer_name = 'John') 
    AND o.region = 'beijing'
```
#### 4.2.3 测试小结
两张表进行JOIN时，左表order配置有行级约束条件`region = 'beijing'`，而且WHERE子句后已有约束条件`o.price > 45.0 OR o.customer_name = 'John'`.

因此先把`region = 'beijing'`增加左表的别名o得到 o.region = 'beijing'，然后在组装的时候给已有的`price > 45.0 OR customer_name = 'John'`两侧增加括号。

### 4.3 INSERT来自带子查询的SELECT
#### 4.3.1 输入SQL
```sql
INSERT INTO print_sink SELECT * FROM orders
```
#### 4.3.2 输出SQL
```sql
INSERT INTO print_sink (
    SELECT 
        orders.order_id, 
        orders.order_date, 
        orders.customer_name, 
        orders.product_id, 
        orders.price, 
        orders.order_status, 
        orders.region 
    FROM 
        hive.default.orders AS orders 
    WHERE 
        orders.region = 'beijing'
)
```
#### 4.3.3 测试小结
无论运行SQL类型是INSERT、SELECT或者其他，只会找到查询`orders`表的子句，然后对其组装行级权限条件。


## 五、参考文献
1. [数据管理DMS-敏感数据管理-行级管控](https://help.aliyun.com/document_detail/161149.html)
2. [Apache Ranger Row-level Filter](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.0/authorization-ranger/content/row_level_filtering_in_hive_with_ranger_policies.html)
3. [OpenLooKeng的行级权限控制](https://www.modb.pro/db/212124)
4. [PostgreSQL中的行级权限/数据权限/行安全策略](https://www.kankanzhijian.com/2018/09/28/PostgreSQL-rowsecurity)
5. [FlinkSQL字段血缘解决方案及源码](https://github.com/HamaWhiteGG/flink-sql-lineage/blob/main/README_CN.md)
6. [基于 Flink CDC 构建 MySQL 和 Postgres 的 Streaming ETL](https://ververica.github.io/flink-cdc-connectors/master/content/%E5%BF%AB%E9%80%9F%E4%B8%8A%E6%89%8B/mysql-postgres-tutorial-zh.html)
7. [重新认识访问者模式：从实践到本质](https://www.51cto.com/article/703150.html)
8. [github-manifold-systems/manifold](https://github.com/manifold-systems/manifold)
