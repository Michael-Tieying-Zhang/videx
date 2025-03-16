

# VIDEX

<p align="center">
  <a href="./README.md">English</a> |
  <a href="./README_zh.md">简体中文</a>
</p>


**VIDEX** 为 MySQL 提供了一个解耦的、可扩展的开源虚拟索引引擎 (**\[VI\]**rtual in**\[DEX\]**)。🚀

- **虚拟索引**：不需要真实数据、仅基于统计信息和算法模型，即可高精度地模拟 VIDEX 产生的查询计划、模拟表连接顺序、模拟索引选择；
- **解耦**：VIDEX 支持在单独的实例上部署，而不必须在原始库 MySQL 上安装；
- **可拓展**：VIDEX提供了便捷的接口，用户可以将 基数估计（Cardinality）、独立值估计（NDV） 等算法模型应用于 MySQL 的下游任务中（例如索引推荐）；


“虚拟索引” 旨在模拟 SQL 查询计划中使用索引的代价（cost）， 从而向用户展示索引对 SQL 计划的影响，而无需在原始实例上创建实际索引。
这项技术广泛应用于各种 SQL 优化任务，包括索引推荐和表连接顺序优化。
业界许多数据库已经以官方或第三方的方式提供了虚拟索引功能，
例如 [Postgres](https://github.com/HypoPG/hypopg)、
[Oracle](https://oracle-base.com/articles/misc/virtual-indexes) 和
[IBM DB2](https://www.ibm.com/docs/en/db2-for-zos/12?topic=tables-dsn-virtual-indexes)。

> **注意**：此处使用的“虚拟索引”一词与
> [MySQL 官方文档](https://dev.mysql.com/doc/refman/8.4/en/create-table-secondary-indexes.html) 中提及的“虚拟索引”不同，
> 后者指的是在虚拟生成列上构建的索引。

此外，VIDEX 封装了一组用于成本估算的标准化接口，
解决了学术研究中的热门话题，如 **基数估计** 和 **不同值数量（NDV）估计**。
研究人员和数据库开发人员可以轻松地将自定义算法集成到 VIDEX 中以用于优化任务。
默认情况下，VIDEX 可以以 `ANALYZE TABLE` 的方式收集统计信息，或者基于少量采样数据构建统计信息。

VIDEX 提供两种启动模式：
1. **作为插件安装到生产数据库**：将 VIDEX 作为插件安装到生产数据库实例。
2. **独立实例**：此模式可以完全避免影响在线运行实例的稳定性，在工业环境中很实用。

在功能方面，VIDEX 支持创建和删除索引（单列索引、复合索引、EXTENDED_KEYS 索引）。
目前暂不支持函数索引（`functional indexes`）、全文索引（`FULL-Text`）和空间索引（`Spatial Indexes`）。

在**拟合精度**方面，我们已经在 `TPC-H`、`TPC-H-Skew` 和 `JOB` 等复杂分析基准测试上对 VIDEX 进行了测试。
<font color="red">给定准确的 ndv 和 cardinality 信息，**VIDEX 可以 100% 模拟 MySQL InnoDB 的查询计划**。</font>
（更多详细信息请参考 [3. Example: TPCH Tiny](#3-example-tpch-tiny) 章节）。

我们期望 VIDEX 能为用户提供一个更好的平台，以便更轻松地测试基数和 NDV 算法的有效性，并将其应用于 SQL 优化任务。


---

## 1. 概览

<p align="center">
  <img src="doc/videx-structure.png" width="600">
</p>

VIDEX 包含两部分：

- **VIDEX-MySQL**：全面梳理了 MySQL handler 的超过90个接口函数，并实现与索引（Index）相关的部分。
- **VIDEX-Statistic-Server**（简称 **VIDEX-Server**）：根据收集的统计信息和估算算法计算独立值（NDV） 和基数（Cardinality），并将结果返回给 VIDEX-MySQL 实例。

VIDEX 根据原始实例中指定的目标数据库（`target_db`）创建一个虚拟数据库，并创建相同结构的关系表（具有相同的 DDL，但将引擎从 `InnoDB` 更换为 `VIDEX`）。

## 2. VIDEX Environment Setup

### 2.1 从 Docker 镜像启动

考虑到编译 VIDEX-MySQL 的复杂性，我们已提供了一个 Docker 镜像。
该镜像已经包含了编译好的 VIDEX-MySQL 和 VIDEX-Server。

镜像基于 [Percona-MySQL release-8.0.34-26](https://github.com/percona/percona-server/tree/release-8.0.34-26)
（Percona-MySQL 是 MySQL 的兼容增强版本）。


```shell
# 注意，这是字节跳动的镜像，下一步会替换为 dockerhub 的镜像。即将推出。
docker run -itd --name videx -p 13308:3306 -p 5001:5001 \
--entrypoint=/bin/bash hub.byted.org/boe/toutiao.mysql.sqlbrain_parse_80:54a3bf649b5c6e0795954669ee4447b9 \
-c "cd /opt/tiger/mysql-server && bash init_start.sh"
```

### 2.2 从源代码编译 VIDEX-MySQL

参考 [文档](doc/compile_zh.md) ，基于 MySQL 源码来编译 VIDEX-MySQL。

### 2.3 启动 Videx-Server

VIDEX-Server 和 VIDEX-MySQL 是解耦的；用户可以添加新的代价估计算法（NDV，基数，索引缓存百分比），
启动自己的 VIDEX-Server。若如此做，用户只需要在执行查询前指定 VIDEX-Server 的 IP。

建议使用 Anaconda 或 Miniconda 创建一个独立的 Python 环境，并安装 VIDEX 环境。

```bash
VIDEX_HOME=videx

git clone git@github.com:bytedance/videx.git $VIDEX_HOME

cd $VIDEX_HOME

conda create -n videx_py39 python=3.9

conda activate videx_py39

python3.9 -m pip install -e . --use-pep517
```

设置 Videx-Stats-Server 的端口并启动服务。

```bash
cd $VIDEX_HOME/src/sub_platforms/sql_opt/videx/scripts
python start_videx_server.py --port 5001   

```

### 2.4. 导入 VIDEX Metadata 

指定原始数据库和 videx-stats-server 的连接方式。 从原始数据库收集统计信息，保存到一个中间文件中， 然后将它们导入到 VIDEX 数据库。

> - 如果 VIDEX-MySQL 是单独启动、而非在原库（target-MySQL）上安装插件，用户可以通过 `--videx` 参数单独指定 `VIDEX-MySQL` 地址。
> - 如果 VIDEX-Server 是单独启动、而非部署在 VIDEX-MySQL 所在机器上，用户可以通过 `--videx_server` 参数单独指定 `VIDEX-Server` 地址。
> - 如果用户已经生成了元数据文件、可以指定 `--meta_path` 参数，跳过采集过程。

```bash
cd $VIDEX_HOME/src/sub_platforms/sql_opt/videx/scripts
python videx_build_env.py --target 127.0.0.1:13308:tpch_sf1:user:password \
[--videx 127.0.0.1:13309:videx_tpch_sf1:user:password] \
[--videx_server 127.0.0.1:5001] \
[--meta_path /path/to/file]

```


至此，用户可以用 MySQL 原生语法来执行创建索引、删除索引、EXPLAIN 等操作。



## 3. Example: TPCH Tiny

在这个例子中，我们以 `TPC-H Tiny`  为例，展示 VIDEX 的使用全流程。
`TPC-H-tiny` 从 `TPC-H sf1(1g)` 中随机采样了 1% 的数据。


### Step 0: 一个就绪的 VIDEX 环境

我们已经准备了一个导入  数据、导入 TPCH 元数据的实例。用户可以跳过 Step 1~3，直接进入 step 4 EXPLAIN sql：
```shell
mysql -h10.37.59.194 -P13308 -ubytebrain -pbytebrain@2023 -Dvidex_tpch_tiny
```

### Step 1: 准备 VIDEX 环境

我们假设用户的生产实例和 VIDEX 的连接信息如下：

- `target-MySQL`：目标实例（生产库）。连接信息为 127.0.0.1:13308:tpch_tiny:user:password
- `VIDEX-MySQL`：以插件形式安装在 `target-MySQL` 中，因此连接信息同上。 
- `VIDEX-Server`：与 `VIDEX-MySQL` 安装在同一个节点，开启默认端口。地址为 127.0.0.1:5001。

通过 `Docker` 启动是最简单的方式，可以一次性启动 VIDEX 所有组件。当然，用户完全可以自定义启动 `VIDEX-MySQL` 和 `VIDEX-Server`。  

```shell
cd $VIDEX_HOME
   
docker run -d  -p 13308:13308 -p 5001:5001 --name videx  videx:latest
```

### Step 2: 导入 TPCH-Tiny 库表

将我们准备的 `TPCH-tiny.sql` 导入目标实例。

```shell
cd $VIDEX_HOME

mysql -h127.0.0.1 -P13308 -uvidex -ppassword -e "create database tpch_tiny;"
tar -zxf data/tpch_tiny/tpch_tiny.sql.tar.gz
mysql -h127.0.0.1 -P13308 -uvidex -ppassword -Dtpch_tiny < tpch_tiny.sql
```

### Step 3: 采集并导入 VIDEX 元数据

请确保 VIDEX 环境已经安装好。若尚未安装，请参考 [2.3 启动 Videx-Server](#23-启动-videx-server)

```shell
cd $VIDEX_HOME
python src/sub_platforms/sql_opt/videx/scripts/videx_build_env.py \
 --target 127.0.0.1:13308:tpch_tiny:videx:password \
 --videx 127.0.0.1:13308:videx_tpch_tiny:videx:password

```

输出如下：
```log
2025-02-17 13:46:48 [2855595:140670043553408] INFO     root            [videx_build_env.py:178] - Build env finished. Your VIDEX server is 127.0.0.1:5001.
You are running in non-task mode.
To use VIDEX, please set the following variable before explaining your SQL:
--------------------
-- Connect VIDEX-MySQL: mysql -h127.0.0.1 -P13308 -uvidex -ppassowrd -Dvidex_tpch_tiny
USE videx_tpch_tiny;
SET @VIDEX_SERVER='127.0.0.1:5001';
-- EXPLAIN YOUR_SQL;
```

现在元数据已经收集完毕、并导入 VIDEX-Server。json 文件已经写入 `videx_metadata_tpch_tiny.json`。

如果用户预先准备了元数据文件，则可以指定 `--meta_path` ，跳过采集阶段，直接导入。

**Step 4: EXPLAIN SQL**

连接到 `VIDEX-MySQL` 上，执行 EXPLAIN。

为了展示 VIDEX 的有效性，我们对比了 TPC-H Q21 的 EXPLAIN 细节，这是一个包含四表连接的复杂查询，涉及 `WHERE`、`聚合`、`ORDER BY`、
`GROUP BY`、`EXISTS` 和 `自连接` 等多种部分。MySQL 可以选择的索引有 11 个，分布在 4 个表上。

由于 VIDEX-Server 部署在 VIDEX-MySQL 所在节点、并且开启了默认端口（5001），因此我们不需要额外设置 `VIDEX_SERVER`。
如果 VIDEX-Server 部署在其他节点，则需要先执行 `SET @VIDEX_SERVER`。

```sql
-- SET @VIDEX_SERVER='127.0.0.1:5001'; -- 以 Docker 启动，则不需要额外设置 
EXPLAIN
FORMAT = JSON
SELECT s_name, count(*) AS numwait
FROM supplier,
     lineitem l1,
     orders,
     nation
WHERE s_suppkey = l1.l_suppkey
  AND o_orderkey = l1.l_orderkey
  AND o_orderstatus = 'F'
  AND l1.l_receiptdate > l1.l_commitdate
  AND EXISTS (SELECT *
              FROM lineitem l2
              WHERE l2.l_orderkey = l1.l_orderkey
                AND l2.l_suppkey <> l1.l_suppkey)
  AND NOT EXISTS (SELECT *
                  FROM lineitem l3
                  WHERE l3.l_orderkey = l1.l_orderkey
                    AND l3.l_suppkey <> l1.l_suppkey
                    AND l3.l_receiptdate > l3.l_commitdate)
  AND s_nationkey = n_nationkey
  AND n_name = 'IRAQ'
GROUP BY s_name
ORDER BY numwait DESC, s_name;
```

我们对比了 VIDEX 和 InnoDB。我们使用 `EXPLAIN FORMAT=JSON`，这是一种更加严格的格式。
我们不仅比较表连接顺序和索引选择，还包括查询计划的每一个细节（例如每一步的行数和代价）。

如下图所示，VIDEX（左图）能生成一个与 InnoDB（右图）几乎 100% 相同的查询计划。
完整的 EXPLAIN 结果文件位于 `data/tpch_tiny`。

![explain.png](doc/explain_tpch_tiny_compare.png)

请注意，VIDEX 的准确性依赖于如下三个关键的算法接口：
- `ndv`
- `cardinality`
- `pct_cached`（索引数据加载到内存中的百分比）。未知的情况下可以设为 0（冷启动）或 1（热数据），但生产实例的 `pct_cached` 可能会不断变化。

VIDEX 的一个重要作用是模拟索引代价。我们额外新增一个索引。VIDEX 增加索引的代价是 `O(1)` ：

```sql
ALTER TABLE tpch_tiny.orders ADD INDEX idx_o_orderstatus (o_orderstatus);
ALTER TABLE videx_tpch_tiny.orders ADD INDEX idx_o_orderstatus (o_orderstatus);
```

再次执行 EXPLAIN，我们看到 MySQL-InnoDB 和 VIDEX 的查询计划发产生了相同的变化，两个查询计划均采纳了新索引。

![img.png](doc/explain_tpch_tiny_compare_alter_index.png)

> VIDEX 的行数估计 (7404) 与 MySQL-InnoDB (7362) 相差约为 `0.56%`，这个误差来自于基数估计算法的误差。

最后，我们移除索引：

```sql
ALTER TABLE tpch_tiny.orders DROP INDEX idx_o_orderstatus;
ALTER TABLE videx_tpch_tiny.orders DROP INDEX idx_o_orderstatus;
```

**Appendix: TPC-H sf1 (1g)**

我们额外为 TPC-H sf1 准备了元数据文件：`data/videx_metadata_tpch_sf1.json`，无需采集，直接导入即可体验 VIDEX。

```shell
cd $VIDEX_HOME
python src/sub_platforms/sql_opt/videx/scripts/videx_build_env.py \
 --target 127.0.0.1:13308:tpch_sf1:user:password \
 --meta_path data/tpch_sf1/videx_metadata_tpch_sf1.json

```

与 TPCH-tiny 一致，VIDEX 可以为 `TPCH-sf1 Q21` 产生与 InnoDB 几乎完全一致的查询计划，详见 `data/tpch_sf1`。

![explain.png](doc/explain_tpch_sf1_compare.png)


## 🚀 集成自定义模型

### Method 1：在 VIDEX-Statistic-Server 中添加一种新方法

实现 `VidexModelBase` 后，重启 `VIDEX-Statistic-Server`。

用户可以完整地实现 `VidexModelBase`。

如果用户只关注 cardinality 和 ndv（两个研究热点），他们也可以选择继承 `VidexModelInnoDB`（参见 `VidexModelExample`）。
`VidexModelInnoDB` 为用户屏蔽了系统变量、索引元数据格式等复杂细节，并提供了一个基本的（启发式的）ndv 和 cardinality 算法。


```python
class VidexModelBase(ABC):
    """
    Abstract cost model class. VIDEX-Statistic-Server receives requests from VIDEX-MySQL for Cardinality
    and NDV estimates, parses them into structured data for ease use of developers.

    Implement these methods to inject Cardinality and NDV algorithms into MySQL.
    """

    @abstractmethod
    def cardinality(self, idx_range_cond: IndexRangeCond) -> int:
        """
        Estimates the cardinality (number of rows matching a criteria) for a given index range condition.

        Parameters:
            idx_range_cond (IndexRangeCond): Condition object representing the index range.

        Returns:
            int: Estimated number of rows that match the condition.

        Example:
            where c1 = 3 and c2 < 3 and c2 > 1, ranges = [RangeCond(c1 = 3), RangeCond(c2 < 3 and c2 > 1)]
        """
        pass

    @abstractmethod
    def ndv(self, index_name: str, table_name: str, column_list: List[str]) -> int:
        """
        Estimates the number of distinct values (NDV) for specified fields within an index.

        Parameters:
            index_name (str): Name of the index.
            table_name (str): Table Name
            column_list (List[str]): List of columns(aka. fields) for which NDV is to be estimated.

        Returns:
            int: Estimated number of distinct values.

        Example:
            index_name = 'idx_videx_c1c2', table_name= 't1', field_list = ['c1', 'c2']
        """
        raise NotImplementedError()
```

### Method 2: 全新实现 VIDEX-Statistic-Server

VIDEX-MySQL 将基于用户指定的地址，通过 `HTTP` 请求索引元数据、 NDV 和基数估计结果。
因此，用户可以用任何编程语言实现 HTTP 响应、并在任意位置启动 VIDEX-Server。


## License

本项目采用双重许可协议：

- MySQL 引擎实现采用 GNU 通用公共许可证第 2 版（GPL - 2.0）许可。
- 所有其他代码和脚本采用 MIT 许可。

详情请参阅 [LICENSE](./LICENSES) 目录。

## Authors
SQLBrain Group, ByteBrain, 字节跳动

## Contact
如果您有任何疑问，请随时通过电子邮件联系我们（kangrong.cn@bytedance.com, kr11thss@gmail.com）。