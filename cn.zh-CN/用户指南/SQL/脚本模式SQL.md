# 脚本模式SQL {#concept_f4h_hnc_bfb .concept}

脚本模式SQL即Script Mode SQL。

MaxCompute当前的SQL引擎支持脚本模式。在脚本模式下编译脚本时，一个多语句的SQL脚本文件将被作为一个整体进行编译，而不是一个一个语句的编译。在提交运行时，它也会被作为一个整体提交，生成一个执行计划，保证一次排队，一次执行，从而充分利用MaxCompute的资源。

Script Mode的SQL书写便利，您只需要按照业务逻辑，用类似于普通编程语言的方式书写，无需考虑如何组织语句。

工具支持

-   MaxCompute Studio，详情请参见[认识Studio](../../../../cn.zh-CN/工具及下载/MaxCompute Studio/认识Studio.md#)。
-   MaxCompute CMD命令行工具，详情请参见[安装并配置客户端](../../../../cn.zh-CN/准备工作/安装并配置客户端.md#)。
-   DataWorks，详情请参见[ODPS SCRIPT节点](../../../../cn.zh-CN/使用指南/数据开发/节点类型/ODPS SCRIPT节点.md#)。

## 语法结构 {#section_mrl_g4c_bfb .section}

```
--set
set odps.sql.type.system.odps2=true;
[set odps.stage.reducer.num=xxx;]
[...]
--ddl
create table table1 xxx;
[create table table2 xxx;]
[...]
--dml
@var1 := SELECT [ALL | DISTINCT] select_expr, select_expr, ...
        FROM table3
        [WHERE where_condition];
@var2 := SELECT [ALL | DISTINCT] select_expr, select_expr, ...
        FROM table4
        [WHERE where_condition];
@var3 := SELECT [ALL | DISTINCT] var1.select_expr, var2.select_expr, ...
        FROM @var1 join @var2 on ...;
INSERT OVERWRITE|INTO TABLE [PARTITION (partcol1=val1, partcol2=val2 ...)]
        SELECT [ALL | DISTINCT] select_expr, select_expr, ...
        FROM @var3;    
[@var4 := SELECT [ALL | DISTINCT] var1.select_expr, var.select_expr, ... FROM @var1 
        UNION ALL | UNION 
        SELECT [ALL | DISTINCT] var1.select_expr, var.select_expr, ... FROM @var2;    
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
        AS 
        SELECT [ALL | DISTINCT] select_expr, select_expr, ...
        FROM var4;]
[...]
```

-   脚本模式支持SET语句、部分DDL语句（不支持结果为屏显类型的语句如desc、show）、DML语句。
-   一个脚本的完整形式是先SET语句，然后DDL语句，最后DML语句。每个部分都可以有0到多个语句，但是不同类型的语句不能交错。
-   多个语句以`@`开始的表示变量连接。
-   一个脚本最多支持一个屏显结果的语句（如单独的SELECT语句），否则会发生报错。不建议在脚本中执行屏显的SELECT语句。
-   一个脚本最多支持一个CREATE TABLE AS语句并且必须是最后一句。推荐您将建表语句与INSERT语句分开写。
-   脚本模式下，一旦有一个语句失败，整个脚本的语句都不会执行成功。
-   脚本模式下，只有当所有输入的数据都准备好并到达，才会生成一个作业进行数据处理。
-   脚本模式下，如果一个表先被写再被读，则会发生报错，如下所示。

    ```
    insert overwrite table src2 select * from src where key > 0;
    @a := select * from src2;
    select * from @a;
    ```

    所以，为避免因表的先写后读产生的报错，应修改SQL脚本如下。

    ```
    @a := select * from src where key > 0;
    insert overwrite table src2 select * from @a;
    select * from @a;
    ```


**脚本模式SQL示例** 

```
create table if not exists dest(key string , value bigint) partitioned by (d string);
create table if not exists dest2(key string,value bigint) partitioned by (d string);
@a := select * from src where value >0;
@b := select * from src2 where key is not null;
@c := select * from src3 where value is not null;
@d := select a.key,b.value from @a left outer join @b on a.key=b.key and b.value>0;
@e := select a.key,c.value from @a inner join @c on a.key=c.key;
@f := select * from @d union select * from @e union select * from @a;
insert overwrite table dest partition (d='20171111') select * from @f;
@g := select e.key,c.value from @e join @c on e.key=c.key;
insert overwrite table dest2 partition (d='20171111') SELECT * from @g;
```

**适用场景** 

-   脚本模式适合用来改写本来要用层层嵌套子查询的单个语句，或者因为脚本复杂性而不得不拆成多个语句的脚本。
-   如果多个输入的数据源数据准备完成的时间间隔很长（例如一个凌晨1点可以准备好，一个上午7点可以准备好），则不适合通过table variable衔接，拼装为一个大的脚本模式SQL。

## 使用MaxCompute Studio执行脚本模式 {#section_od2_yqc_bfb .section}

使用MaxCompute Studio脚本模式，首先请保证MaxCompute Studio完成安装、添加项目链接、建立MaxCompute SQL脚本文件，详情请参见[安装IntelliJ IDEA](../../../../cn.zh-CN/工具及下载/MaxCompute Studio/工具安装与版本信息/安装IntelliJ IDEA.md#)、[项目空间连接管理](../../../../cn.zh-CN/工具及下载/MaxCompute Studio/项目空间连接管理.md#)、[创建MaxCompute Script Module](../../../../cn.zh-CN/工具及下载/MaxCompute Studio/开发SQL程序/创建MaxCompute Script Module.md#)。编辑脚本页面如下。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20231/155558698744793_zh-CN.png)

脚本编译后提交运行，查看执行计划图。虽然脚本上是多个语句，但执行计划图是一个相通的DAG图。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20231/155558698711441_zh-CN.png)

## 通过客户端 （odpscmd）提交脚本模式 {#section_arx_3xc_bfb .section}

您需要使用0.27上版本的odpscmd提交脚本。建议您[下载](http://odps.alibaba-inc.com/official_downloads/odpscmd/)安装最新版本。安装后，请使用`-s`参数提交脚本。

编辑脚本模式的源码myscript.sql文件，调用odpscmd命令执行如下。

 `odpscmd -s myscript.mxql;` 

**说明：** `-s`为odpscmd的命令行选项，类似于-f, -e，而非交互环境中的命令。odpscmd的交互环境中暂不支持脚本模式与表变量。

## 通过DataWorks使用脚本模式 {#section_p91_9ik_im6 .section}

在DataWorks中可以建立脚本模式的节点，如下图所示。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20231/155558698744790_zh-CN.png)

在此节点中进行脚本模式的脚本编辑，编辑好后点击工具栏的运行按钮，提交脚本到MaxCompute执行。执行计划图和结果可以从输出信息的logview url中得到。

