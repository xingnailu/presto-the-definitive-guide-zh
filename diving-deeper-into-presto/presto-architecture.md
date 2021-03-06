# 4. Presto 技术架构

在前面的章节介绍了 Presto 以及进行了初始安装和使用之后，我们现在讨论 Presto 架构。 我们将深入探讨相关概念，因此您可以了解 Presto 查询执行模型，查询计划和基于代价的优化器。

在本章中，我们首先讨论 Presto 高级架构组件。大致了解 Presto 的工作方式非常重要，特别是如果您打算自己安装和管理 Presto 集群，如第5章所述。

在本章的下半部分，当我们讨论 Presto 的查询执行模型时，我们将更深入地研究这些组件。 如果您需要诊断或调优性能低下的查询（在第8章中进行了讨论），或者您打算为 Presto 开源项目做出贡献，那么这是非常重要的。

## 集群中的 Coordinator 和 Worker

如第2章中所述，首次安装 Presto 时，仅使用一台计算机来运行。这种部署方式无法满足可伸缩性和性能的要求。

Presto 是一个分布式 SQL 查询引擎，类似于大规模并行处理（MPP）的数据库和查询引擎。它无需依赖于运行 Presto 的服务器的垂直扩展，而是能够以水平方式在服务器集群中分布所有处理。这意味着用户可以添加更多节点以获得更多处理能力。

利用这种体系结构，Presto 查询引擎能够跨集群或节点并行处理大数据量的 SQL 查询。 Presto 在每个节点上作为单服务进程运行。运行 Presto 的多个节点（配置为彼此协作）组成一个 Presto 集群。

图4-1描述了由一个协调节点和多个工作节点组成的 Presto 群集。 Presto 用户通过客户端连接到协调节点，例如使用 JDBC 驱动程序或 Presto CLI 工具。然后，协调节点与工作节点协作，工作节点访问数据源。

![&#x56FE; 4-1](../.gitbook/assets/figure-4-1-presto-architecture-overview-with-coordinator-and-workers.png)

协调节点（coordinator）负责接收所有的查询，并且管理工作节点去执行查询。

工作节点（worker）负责执行任务并且处理数据。

发现服务（discovery service）通常运行在协调节点上，允许工作节点注册并参与到集群中。

所有节点、客户端之间的通信使用了基于 HTTP/HTTPS 的 REST 协议。

图 4-2 展示了节点间是如何通信的。协调节点与工作节点通信，以分配任务，更新状态，获取最外层的结果返回给用户。工作节点之间也会互相通信，以获取来自其他节点的上游任务的数据。所有的节点都可以从数据源抽取数据。

![&#x56FE; 4-2](../.gitbook/assets/figure-4-2-communication-between-coordinator-and-workers-in-a-presto-cluster.png)

## Coordinator 协调节点

协调节点负责接收用户的 SQL 查询请求，解析语法，构建执行计划，管理工作节点。它可以说是 Presto 的大脑。用户可以通过 Presto 命令行工具访问；应用程序可以通过 JDBC/ODBC 驱动访问；其他语言编写的程序也可以通过不同的客户端工具访问。协调节点从客户端接收 SQL 语句，比如 `select`。

Presto 运行环境必须包含一个协调节点，以及一个或多个工作节点。在开发及测试环境，这两个角色可以部署在一台机器上。

协调节点持续追踪工作节点的状态，以及捕获查询的执行情况。协调节点创建了一个多阶段的执行模型。

图 4-3 展示了客户端、协调节点、工作节点之间的通信。

一旦接收到 SQL 语句，协调节点就会开始解析、分析、计划，并在工作节点之间分发调度。查询语句会被翻译为一系列运行在整个集群的、相互关联的任务。工作节点处理完成数据后，协调节点将会通过输出缓冲区将数据返回给客户端。一旦客户端完全读取输出缓冲，协调节点就会代表客户端向工作节点请求更多数据。 另外，工作节点与数据源进行交互以从中获取数据。最终，客户端不断请求数据并由工作节点从数据源提供数据，直到查询执行完成。

协调节点和工作节点使用 HTTP 协议通信。

![&#x56FE; 4-3](../.gitbook/assets/figure-4-3-client-coordinator-and-worker-communication-processing-a-sql-statement.png)

## Discovery Service 发现服务

Presto 使用了发现服务来找到集群中所有的节点。每个 Presto 实例在启动时都会向发现服务注册，并定期发送心跳信号。这允许协调节点拥有可用工作节点的最新列表，并将该列表用于计划查询执行。

如果工作节点无法报告心跳信号，则发现服务将触发故障检测，并且该工作节点将无法再执行其他任务。

为了简化部署并避免运行其他服务，Presto 的发现服务通常是嵌入部署在协调节点。 它与 Presto 共享 HTTP 服务，因此使用相同的端口。

因此，发现服务的工作程序配置通常指向协调节点的主机名和端口。

## Worker 工作节点

Presto 工作节点是 Presto 集群中的服务者。 它负责执行协调器分配的任务并处理数据。工作节点通过使用连接器从数据源获取数据，然后彼此交换中间数据。 最终得到的数据将传递给协调节点。协调节点负责从工作节点那里收集结果并将最终结果提供给客户端。

在安装过程中，工作节点被配置为需要知道集群发现服务的主机名或IP地址。当工作节点的进程启动时，它会向发现服务发布信息，这将使协调程序可以使用它来执行任务。

工作节点使用基于 HTTP 的协议与其他工作节点和协调节点进行通信。

图 4-4 展示了工作节点是如何获取数据，以及相互之间如何协作处理数据，直到将数据传递给协调节点才停止。

![&#x56FE; 4-4](../.gitbook/assets/figure-4-4-workers-in-a-cluster-collaborate-to-process-sql-statements-and-data.png)

## 基于 Connector 的架构

基于 connector 的架构是 Presto 计算存储的核心。connector 使得 Presto 可以连接任意的数据源。

每一个 connector 都对它底层的数据源提供了表级别的抽象。只要一种数据可以被映射成 Presto 中表、列、行的概念，那么就可以创建一个 connector 对该数据进行查询。

Presto 提供了一种服务发现接口（Service Provider Interface，SPI），一种可以实现 connector 的 API。通过在 connector 中实现该接口，Presto 可以在内部连接任意的数据源并执行操作。connector 内部负责处理连接的细节。

每一个 connector 都要实现 API 的三个部分：

* 获取 table/view/schema 元信息的操作
* 使数据根据逻辑单元分区的操作，可以让 Presto 并行读写数据
* 定义数据的 source 和 sink，用于将源数据转换为查询引擎期望的内存格式或从内存格式转换为源数据

Presto 提供了非常多的连接器，比如 HDFS/Hive，MySQL，PostgreSQL，MS SQL Server，Kafka，Cassandra，Redis 等。在第6、第7章，你将接触到这些连接器。可用的连接器种类正在被不断的添加。

Presto 的 SPI 机制同样允许用户创建自己的定制化 connector。当你需要连接的数据源没有已经实现的 connector，这会变得极为有用。当你创建完一个自己的连接器时，我们强烈建议你多和 Presto 开源社区沟通，贡献你的连接器。当你的公司有一个定制化的数据源时，定制连接器也会很重要。这就是 Presto 如何实现 “SQL 查一切“ 功能的过程。

图4-5 展示了 Presto SPI 在 coordinator 中的接口：元数据，数据分析，数据位置；同样，在 worker 中，也有数据处理的接口。

![&#x56FE; 4-5](../.gitbook/assets/figure-4-5-overview-of-the-presto-service-provider-interface-spi.png)

在 Presto 服务启动时，连接器是以插件的形式加载的。在  catalog 的配置文件中配置好以后，就可以从插件的文件夹被加载到。我们在第6章会详细讨论。

{% hint style="info" %}
提示：Presto 插件化的架构体现在很多方面：事件监听器，权限控制，函数、类型管理，等等。
{% endhint %}

## Catalogs，Schemas 和 Tables

之前已经讨论过，Presto 使用了基于连接器的架构来处理所有的查询。每一个  catalog 都被配置到了一种数据源上。一个  catalog 中可能有多个 schema。一个 schema 中有多个 table，每个 table 又会提供不同的行、数据类型。我们在第8章会详细讨论。

## 查询执行模型

现在你已经了解了 Presto 集群的构成，以及协调节点和工作节点各自的作用。现在我们看一下 SQL 语句是如何被处理的。

{% hint style="info" %}
提示：第8、第9章详细描述了 Presto 对 SQL 的支持。
{% endhint %}

了解查询执行模型可以让你拥有足够的 SQL 查询调优知识。

回想一下，协调节点使用 ODBC 或 JDBC 驱动程序或其他客户端，从命令行接受最终用户的 SQL 语句。然后，协调节点触发工作节点从数据源获取所有数据，创建结果数据集，并将其提供给客户端。

让我们先仔细研究一下协调节点内部发生的情况。将 SQL 语句提交给协调节点后，它将以文本格式接收。协调节点获取该文本并进行解析和分析。 然后，它使用 Presto 中的内部数据结构（称为查询计划）创建执行计划。该流程如图4-6 所示。查询计划概括地表示了处理数据并根据 SQL 语句返回结果所需的步骤。

![&#x56FE; 4-6](../.gitbook/assets/figure-4-6-processing-a-sql-query-statement-to-create-a-query-plan.png)

如图4-7，查询计划生成器使用元数据 SPI 和数据分析 SPI 来创建查询计划。所以协调节点是通过 SPI 直接获取数据源中表的元信息。

![&#x56FE; 4-7](../.gitbook/assets/figure-4-7-the-service-provider-interfaces-for-query-planning-and-scheduling.png)

协调节点使用元数据 SPI 获取表、列、类型的信息。这些信息被用来验证查询在语义上是合法的，同时被用来在安全检查中进行类型、表达式的验证。

分析 SPI 被用来获取表的行数、表的尺寸大小，以用于构造基于代价的查询计划。

数据位置 SPI 可以加快分布式查询计划的生成。它用来生成表的逻辑切分（split）计划。切片（splits）是并行工作的最小单元。

{% hint style="info" %}
Presto 中不同的 SPI 更像是概念上的分离； 实际的底层 Java API 以更细粒度的方式由不同的 Java 包分隔。
{% endhint %}

分布式查询计划是由一个或多个阶段（stage）组成的简单查询计划的扩展。简单查询计划分为多个计划片段。阶段（stage）是计划片段的运行时状态，它包含该阶段的计划片段描述的工作的所有任务。

协调节点分解执行计划，以允许在集群上处理查询，从而使工作节点可以并行执行，从而加快了整体查询的速度。具有多个阶段的查询会导致需要创建阶段的依赖关系树。阶段数取决于查询的复杂性。例如，查询表，返回的列，JOIN 语句，WHERE 条件，GROUP BY 操作和其他 SQL 语句都会影响创建的阶段数。

图 4-8 展示了在集群的协调节点中，逻辑查询计划是如何被转化成分布式执行计划的。

![&#x56FE; 4-8](../.gitbook/assets/figure-4-8-transformation-of-the-query-plan-to-a-distributed-query-plan.png)

分布式查询计划定义了在 Presto 群集上执行查询的阶段和方式。协调节点使用它来进一步计划和安排整个工作节点的任务。一个阶段包含一个或多个任务。通常，查询过程会涉及许多任务，每个任务处理一部分数据。

协调节点从阶段（stage）中拆解出具体的任务（task），再分配给工作节点，如图 4-9所示。

![&#x56FE; 4-9](../.gitbook/assets/figure-4-9-task-management-performed-by-the-coordinator.png)

任务处理的数据单位称为分割（Split）。分割是工作节点可以检索和处理的基础数据段的描述符。它是并行和计算工作分配的基础单位。连接器对数据执行的特定操作取决于基础数据源。

例如，Hive 连接器以路径的形式描述文件的分割，其偏移量和长度指示文件的哪一部分需要处理。

源阶段的任务以页面（page）形式生成数据，这些数据是列格式的行的集合。这些页面流向其他中间下游阶段。交换操作器在阶段之间转移页面（page），交换操作器从上游阶段的任务中读取数据。

源任务使用数据源 SPI 在连接器的帮助下从基础数据源中获取数据。该数据以页面（page）的形式显示给Presto，并在查询引擎中流动。处理器（operator）根据其语义来处理和生成页面。例如，过滤操作可以删除行，投影操作可以生成带有新列的页面（page），等等。任务中运算符的序列称为管道。管道的最后一个运算符通常将其输出页放在任务的输出缓冲区中。下游任务中的交换（exchange\) 操作器会消费上游任务输出缓冲区中的页面（page）。所有这些操作在不同的工作线程上并行发生，如图4-10所示。

![&#x56FE; 4-10](../.gitbook/assets/figure-4-10-data-in-splits-is-transferred-between-tasks-and-processed-on-different-workers.png)

当查询计划被分配到具体的工作节点时，它的运行时的表现内容，就叫做 task。当一个 task 创建后，会为每一个分割操作实例化一个 driver。每一个 driver 又是数据在分割（split）时的操作和转化的表现内容。一个 task 可能有一个或多个 driver，具体的数量根据 Presto 的配置而定，如图 4-11。一旦所有 driver 的工作都完成，数据会被传输到下一个分割工作流，当前阶段的 driver 和 task 将会在完成任务后被销毁。

![&#x56FE; 4-11](../.gitbook/assets/figure-4-11-parallel-drivers-in-a-task-with-input-and-output-splits.png)

处理器（operator）的作用是处理输入的数据，并发送到下一个处理器。扫描表（table scan），过滤（filter），关联（joins），聚合（aggregations）等等都是处理器。一系列的处理器构成了处理器的工作流（pipeline）。举个例子，一个处理器的工作流可能是先扫描和读取表，然后过滤数据，最后进行聚合。

为了进行查询，协调节点根据数据的元信息构造了一系列的分割操作。通过这些分割操作，协调节点开始在工作节点之间调度任务，以获取分割的数据。在查询执行时，协调节点会追踪到所有的可用分割，以及数据位置，另外还有任务的运行节点。当一个任务执行结束，并且向下游构造更多的数据分割时，协调节点持续调度任务，直到没有新的数据分割产生。

一旦所有的分割完成，所有的数据都可以用，那么协调节点就可以让结果对用户可见了。

## 查询计划

在深入研究 Presto 查询计划的实现和基于代价的优化工作之前，让我们先提供一个示例查询作为我们探索的引子，以帮助您了解查询计划的生成过程。

示例代码 4-1 使用了 TPC-H 的数据集（参见：Presto TPC-H 和 TPC-DS 连接器），查询订单的汇总值，并根据国家区分，同时列出前五的国家。

{% code title="示例4-1" %}
```sql
SELECT
    (SELECT name FROM region r WHERE regionkey = n.regionkey) AS region_name,
    n.name AS nation_name,
    sum(totalprice) orders_sum
FROM nation n, orders o, customer c
WHERE n.nationkey = c.nationkey
  AND c.custkey = o.custkey
GROUP BY n.nationkey, regionkey, n.name
ORDER BY orders_sum DESC
LIMIT 5;
```
```
{% endcode %}

让我们试着了解这个 SQL 的结构以及它的目的：

* 首先，`SELECT` 操作查询了 `FROM` 后面的三张表，隐含地表明需要对 nation，orders，customer 这三张表做 `CROSS JOIN`
* `WHERE` 条件用来过滤保留 nation，orders，customer 三张表中的匹配数据
* `GROUP BY regionkey` 的操作用来根据国家聚合 order 的值
* 子查询 `(SELECT name FROM region WHERE regionkey = n.regionkey)`从 region 表中获取国家名；注意这个操作是相关的，就像要对结果集

  的每一行都要做这个操作

* 排序语句 `ORDER BY orders_sum DESC` ，在返回结果前排序
* `limit` 语句定义了只返回数据最大的5条记录

### 词法分析、语法和语义分析

当查询可以被执行前，查询语句需要被解析和分析。查询语句的构造可以在第8章、第9章中了解更多细节。Presto 首先会验证一些语法的正确性，然后，将会分析查询语句：

_**识别查询了哪些表**_

 根据 catalog、schema 可以确定一个表，所以多个表可以有相同的 table 名。举例，TPC-H 的数据中，提供了同名不同 schema 的多个表，比如 `sf10.orders`，`sf100.orders`，等等。

_**识别查询涉及的列**_

`orders.totalprice` 这样的语句明确的指出了这是在查询 `orders` 表的 `totalprice` 字段。然而，某些情况下我们也可以只指定 `totalprice` 字段，而不用指定来自哪个表，如查询 4-1。Presto 的分析器可以决定使用哪个表的字段。

_**识别字段的归属**_

表达式 `c.bonus` 可能是表示一个表名为 `c` ，也可能是一个表的别名为 `c`，又或者表示一种 `Row` 的类型。如何定义字段的性质和归属，以免产生歧义，是 Presto 分析器的工作。分析需要遵循 SQL 语言的范围和可见性规则。 之后将在计划过程中使用收集到的信息（例如标识符歧义消除）。因此执行计划生成器无需再次了解查询语言的各种规则。

如你所见，查询分析器具有复杂且跨领域的职责。它的作用是非常技术性的，并且从查询的角度来看，只要查询正确，它就对用户不可见。每当查询违反 SQL 语言规则，超出用户权限或由于其他某些原因而报错时，分析器就会让人感觉到它的存在。

分析完查询，并处理和解析了查询中的所有标识符后，Presto 进入下一阶段，即生成查询计划。

### 初始化查询计划

查询计划可以认为是一种获取查询结果的程序。回顾一下 SQL 的定义：用户通过 SQL 查询，从系统中获取想要的数据。和命令式编程不同，用户并不需要指定如何去获取数据。这些获取数据的具体细节，被包裹在查询计划生成器和优化器中，它们共同决定了如何返回想要的结果。

获取查询结果的步骤，就被称作：查询计划（query plan）。从理论上讲，指数级别的查询计划可以产生相同的查询结果。不同查询计划的性能差异很大，这就是为什么 Presto 查询计划器和优化器试图确定最佳的计划。总是产生相同结果的计划称为等效计划。

让我们看看一下前面示例4-1中显示的查询。此查询最直接的查询计划是与查询的 SQL 语法结构最相似的查询计划。 该计划如示例4-2所示。 为了便于讨论，查询计划应该是不言自明的。你只需要知道计划是一棵树，并且它的执行就从叶节点开始并沿着树结构向上进行。

{% code title="示例4-2:简洁直接、易于理解的文本查询计划" %}
```sql
- Limit[5]
  - Sort[orders_sum DESC]
    - LateralJoin[2]
      - Aggregate[by nationkey...; orders_sum := sum(totalprice)]
       - Filter[c.nationkey = n.nationkey AND c.custkey = o.custkey]
          - CrossJoin
            - CrossJoin
              - TableScan[nation]
              - TableScan[orders]
            - TableScan[customer]
      - EnforceSingleRow[region_name := r.name]
        - Filter[r.regionkey = n.regionkey]
          - TableScan[region]
```
{% endcode %}

查询计划中的每个元素都可以转化成一个直接的操作命令。比如，`TableScan` 这个元素的工作，就是在某个存储介质上获取表，并且返回包含这张表所有行的数据。元素 `Filter` 获取这些行，在每一行上执行一次过滤条件，只返回满足条件的行。元素 `CrossJoin` 操作它从两个子节点获取到的两份数据集。它处理这些数据集中的所有内容，可能是把数据集放入内存，以使得我们不用多次读取数据的原生存储介质。

{% hint style="warning" %}
警告：最新版的 Presto 修改了执行计划中的命名。比如，`TableScan` 改为了 `ScanProject`，`Filter` 改为了 `FilterProject`。但它们背后的语义还是一致的。
{% endhint %}



## 优化规则

## 执行规则

## 基于代价的优化器

## 处理表分析

## 总结



