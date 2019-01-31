# RDS迁移到MaxCompute实现动态分区 {#concept_gxd_crn_4fb .concept}

本文向您详细介绍如何使用DataWorks数据集成同步功能，自动创建分区，动态的将RDS中的数据，迁移到MaxCompute大数据计算服务上。

## 准备工作 {#section_od5_frc_pfb .section}

1.  MaxCompute

    [开通MaxCompute](../../../../../cn.zh-CN/准备工作/开通MaxCompute.md#)服务并创建工作空间，本文选择的区域是华北2（北京）。同时启动DataWorks相关服务。如下图所示。

    ![](images/14287_zh-CN.jpeg)

    **说明：** 如果您是第一次使用DataWorks，请确认已经根据[准备工作](../../../../../cn.zh-CN/准备工作/准备阿里云账号.md#)，准备好账号和项目角色、项目空间等内容。

2.  新增数据源

    1.  新增RDS数据源

        进入DataWorks[数据集成](../../../../../cn.zh-CN/使用指南/数据集成/数据集成简介/数据集成概述.md#)控制台，新增[MySQL数据源](../../../../../cn.zh-CN/使用指南/数据集成/数据源配置/配置MySQL数据源.md#)。 如下图。

        ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846133491_zh-CN.png)

    2.  新增ODPS数据源。详细描述请参考文档[配置MaxCompute数据源](https://www.alibabacloud.com/help/zh/faq-detail/74280.htm)。
    配置完成后，如下图所示。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846114385_zh-CN.png)

3.  确认作为目标的ODPS数据库中有表

    在ODPS数据库中创建RDS对应目标表ods\_user\_info\_d。在**数据开发**选项下右键单击**新建ODPS SQL节点**create\_table\_ddl，输入建表语句。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846133545_zh-CN.png)

    SQL如下：

    ```
    
    CREATE TABLE IF NOT EXISTS ods_user_info_d (
    uid STRING COMMENT '用户ID',
    gender STRING COMMENT '性别',
    age_range STRING COMMENT '年龄段',
    zodiac STRING COMMENT '星座'
    )
    PARTITIONED BY (
    dt STRING
    );
    
    ```

    您也可以选择在**业务流程** \> **表**下**新建表**。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846133547_zh-CN.png)

    详细的信息请参见[建表并上传数据](https://www.alibabacloud.com/help/zh/doc-detail/84670.htm)。

4.  新建业务流程

    进入DataWorks管理控制台，单击对应项目后的**进入数据开发**，开始数据开发操作。选择**数据开发** \> **业务流程**下的**新建业务流程**workshop，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846133546_zh-CN.png)

5.  新建并配置同步任务节点

    在新创建的业务流程workshop下，新建同步节点rds\_sync。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214388_zh-CN.png)


详细的数据同步任务的操作和配置请参见[DataWorks数据开发和运维](https://www.alibabacloud.com/help/zh/doc-detail/84669.htm)。

## 自动创建分区 {#section_sfm_3rc_pfb .section}

准备工作完成后，需要将RDS中的数据定时每天同步到MaxCompute中，自动创建按天日期的分区。

1.  选择数据来源和数据去向，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214391_zh-CN.png)

2.  参数配置，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214396_zh-CN.png)

    一般配置到这个地方的时候，默认是系统自带的时间参数：`${bizdate}`，格式为yyyymmdd，对应也**数据去向**下的**分区信息**。就是说在调度执行这个任务的时候，这个分区会被自动替换为任务执行日期的前一天，一般用户会在当前跑前一天的业务数据，这个日期也叫业务日期。如果用户要使用当天任务运行的日期作为分区值，需要自定义这个参数。选择**调度配置**。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214394_zh-CN.png)

    自定义参数设置，格式非常灵活，日期是当天日期，用户可以自由选择哪一天以及格式。可供参考的变量参数配置方式如下：

    后N年：`$[add_months(yyyymmdd,12*N)]`

    前N年：`$[add_months(yyyymmdd,-12*N)]`

    后N月：`$[add_months(yyyymmdd,N)]`

    前N月：`$[add_months(yyyymmdd,-N)]`

    后N周：`$[yyyymmdd+7*N]`

    前N周：`$[yyyymmdd-7*N]`

    后N天：`$[yyyymmdd+N]`

    前N天：`$[yyyymmdd-N]`

    后N小时：`$[hh24miss+N/24]`

    前N小时：`$[hh24miss-N/24]`

    后N分钟：`$[hh24miss+N/24/60]`

    前N分钟：`$[hh24miss-N/24/60]`

    **说明：** 

    -   请以中括号 \[\] 编辑自定义变量参数的取值计算公式，例如 `key1=$[yyyy-mm-dd]`。
    -   默认情况下，自定义变量参数的计算单位为天。例如 `$[hh24miss-N/24/60]`表示`(yyyymmddhh24miss-(N/24/60 * 1天))`的计算结果，然后按 hh24miss 的格式取时分秒。
    -   使用 add\_months 的计算单位为月。例如 `$[add_months(yyyymmdd,12 N)-M/24/60]` 表示`(yyyymmddhh24miss-(12 N 1月))-(M/24/60 1天)`的结果，然后按 `yyyymmdd` 的格式取年月日。
    详细的参数设置请参见[参数配置](https://www.alibabacloud.com/help/zh/doc-detail/74450.htm)。

3.  测试运行。

    **保存**所有配置，单击**运行**，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846233548_zh-CN.png)

    查看运行日志，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214399_zh-CN.png)

    可以看到日志中，MaxCompute（日志中打印原名ODPS）的信息中partition分区，dt=20181025，自动替换成功。检查下实际的数据有没有转移到ODPS表中，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214400_zh-CN.png)

    **说明：** 在MaxCompute2.0中分区表查询需要添加分区筛选，不支持全量查询。SQL语句如下。

    ```
    --查看是否成功写入MaxCompute
    select count(*) from ods_user_info_d where dt=业务日期;
    
    ```

    SELECT命令详情请参见[Select操作](../../../../../cn.zh-CN/用户指南/SQL/SELECT操作/Select语法介绍.md#)。

    此时看到数据已经迁移到ODPS表中，并且成功创建了一个分区值。那么这个任务在执行定时调度的时候，会每天将RDS中的数据同步到MaxCompute中按照日期自动创建的分区中。


## 补数据实验 {#section_cbp_cqd_pfb .section}

如果用户的数据有很多运行日期之前的历史数据，想要实现自动同步和自动分区，此时您可以进入DataWorks的**运维中心**，选择当前的同步数据节点，选择**补数据**功能来实现。

1.  首先，我们需要在RDS端把历史数据按照日期筛选出来，比如历史数据2018-09-13这天的数据，我们要自动同步到MaxCompute的20180825的分区中。在迁移阶段可以设置where过滤条件，如图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846214401_zh-CN.png)

2.  补数据操作。然后**保存** \> **提交**。提交后到**运维中心** \> **任务列表** \> **周期任务**中的rds\_sync节点，单击**补数据** \> **当前节点**，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846220998_zh-CN.png)

3.  跳转至补数据节点页面。选择日期区间，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846220999_zh-CN.png)

4.  单击**确定**。此时会同时生成多个同步的任务实例按顺序执行，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846221000_zh-CN.png)

5.  查看运行的日志，可以看到运行过程中对RDS数据的抽取。此时MaxCompute已自动创建分区，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846221001_zh-CN.png)

    查看运行结果。数据写入的情况，以及是否自动创建了分区，数据是否已同步到分区表中，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846221002_zh-CN.png)

    查询对应分区信息，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846233549_zh-CN.png)

    **说明：** 在maxcompute2.0中分区表查询需要添加分区筛选，SQL语句如下。其中分区列需要更新为业务日期，如任务运行的日期为20180717，那么业务日期为20180716。

    ```
    --查看是否成功写入MaxCompute
    select count(*) from ods_user_info_d where dt=业务日期;
    
    ```


## Hash实现非日期字段分区 {#section_rr4_tgt_pfb .section}

如果用户数据量比较大，或者第一次全量的数据并不是按照日期字段进行分区，而是按照省份等非日期字段分区，那么此时数据集成操作就不能做到自动分区了。也就是说，可以按照RDS中某个字段进行hash，将相同的字段值自动存放到这个字段对应值的MaxCompute分区中。

流程如下：

1.  首先我们需要把数据全量同步到MaxCompute的一个临时表。创建一个SQL脚本节点。单击**运行** \> **保存** \> **提交**。

    sql命令如下。

    ```
    drop table if exists ods_user_t;
    CREATE TABLE ods_user_t ( 
            dt STRING,
    	uid STRING,
            gender STRING,
            age_range STRING,
            zodiac STRING);
    insert overwrite table ods_user_t select dt,uid,gender,age_range,zodiac from ods_user_info_d;--将ODPS表中的数据存入临时表。
    
    ```

2.  创建同步任务的节点mysql\_to\_odps，就是简单的同步任务，将RDS数据全量同步到MaxCompute，不需要设置分区。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846333591_zh-CN.png)

3.  使用sql语句进行动态分区到目标表。命令如下。

    ```
    drop table if exists ods_user_d;
    //创建一个ODPS分区表（最终目的表）
    	CREATE TABLE ods_user_d (
    	uid STRING,
            gender STRING,
            age_range STRING,
            zodiac STRING
    )
    PARTITIONED BY (
    	dt STRING
    );
    //执行动态分区sql，按照临时表的字段dt自动分区，dt字段中相同的数据值，会按照这个数据值自动创建一个分区值
    //例如dt中有些数据是20180913，会自动在ODPS分区表中创建一个分区，dt=20181025
    //动态分区sql如下
    //可以注意到sql中select的字段多写了一个dt，就是指定按照这个字段自动创建分区
    insert overwrite table ods_user_d partition(dt)select dt,uid,gender,age_range,zodiac from ods_user_t;
    //导入完成后，可以把临时表删除，节约存储成本
    drop table if exists ods_user_t;
    ```

    在MaxCompute中我们通过SQL语句来完成同步。详细的SQL语句介绍请参见[阿里云大数据利器MaxCompute学习之--分区表的使用](https://yq.aliyun.com/articles/81775?spm=a2c4e.11153940.blogcont182972.20.72856a07pHyeLb)。

4.  最后将三个节点配置成一个工作流，按顺序执行。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846321007_zh-CN.png)

5.  查看执行过程。我们可以重点观察最后一个节点的动态分区过程，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846321008_zh-CN.png)

    查看数据。动态的自动化分区完成。相同的日期数据迁移到了同一个分区中。如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846333594_zh-CN.png)

    对应查询分区信息，如下图。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/24452/154892846321013_zh-CN.png)

    如果是以省份字段命名分区，执行步骤参请考上述内容。


DataWorks数据同步功能可以完成大部分自动化作业，尤其是数据的同步迁移，调度等，了解更多的调度配置请参见调度配置[时间属性](https://www.alibabacloud.com/help/zh/doc-detail/74452.htm)。

