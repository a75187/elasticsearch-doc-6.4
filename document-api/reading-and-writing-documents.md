# 读取和写入文档

### 介绍

Elasticsearch中的每个索引都分为碎片，每个碎片可以有多个副本。 这些副本称为复制组，在添加或删除文档时必须保持同步。 如果我们不这样做，从一个副本中读取将导致与从另一个副本读取的结果截然不同。 保持分片副本同步并从中提供读取的过程就是我们所说的**数据复制模型**。

Elasticsearch的数据复制模型基于主备份模型，并在Microsoft Research的[PacificA](https://www.microsoft.com/en-us/research/publication/pacifica-replication-in-log-based-distributed-storage-systems/)论文中得到了很好的描述。该模型基于单一复制的副本组，该副本充当主分片（primary shard）。 其他的副本称为备份分片（replica shards）。主分片作为全部索引操作的主入口，它负责验证它们并确保它们是正确的。当主分片接受到一个索引操作请求，它还负责将操作复制到其他副本。

本节的目的是对Elasticsearch复制模型进行高级概述，并讨论它对写入和读取操作之间的各种交互的影响。

### 写模型

Elasticsearch中的每个索引操作首先使用[路由](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html#index-routing)解析为复制组，通常基于文档ID。确定复制组后，操作将在内部转发到组的当前主分片。主分片负责验证操作并将其转发到其他副本。由于副本可以脱机，因此不需要将主副本复制到所有副本。相反，Elasticsearch维护应该接收操作的分片副本列表。此列表称为同步副本，由主节点维护。顾名思义，这些是“好”分片副本的集合，保证已经处理了已经向用户确认的所有索引和删除操作。主要负责维护此不变量，因此必须将所有操作复制到此集合中的每个副本。

主分片遵循以下基本流程：

1. 验证传入操作并在结构无效时拒绝它（例如：在数字字段中传入对象）
2. 在本地执行操作，即索引或删除相关文档。 这也将验证字段的内容并在需要时拒绝（例如：关键字值太长，无法在Lucene中进行索引）。
3. 将操作转发到当前同步副本集中的每个副本。 如果有多个副本，则这是并行完成的。
4. 一旦所有副本成功执行了操作并响应主服务器，主服务器就会确认成功完成对客户端的请求。

### 失败处理

在索引编制过程中可能会出现许多问题 - 磁盘可能会损坏，节点可能会相互断开连接，或者某些配置错误可能导致复制副本上的操作失败，尽管它在主服务器上成功。这些是罕见的，但主分片必须回应它们。在主分片本身发生故障的情况下，托管主分片服务器的节点将向主服务器（the master?）发送有关它的消息。索引操作将等待（默认情况下最多1分钟），以便主服务器将其中一个副本提升为新的主分片。然后，该操作将被转发到新的主分片处理。请注意，主服务器还会监控节点的运行状况，并可能决定主动降级主分片。当通过网络问题将拥有主分片的节点与群集隔离时，通常会发生这种情况。 有关详细信息，请参见[此处](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-replication.html#demoted-primary)。

一旦在主服务器上成功执行了操作，主服务器就必须在副本分片上执行它时处理潜在的故障。 这可能是由副本上的实际故障或由于网络问题导致操作无法到达副本（或阻止副本响应）引起的。所有这些都具有相同的最终结果：作为同步副本集的一部分的副本错过了即将被确认的操作。为了避免违反不变量，主分片向主服务器发送消息，请求从同步副本集中删除有问题的分片。只有在主节点确认删除了分片后，主分片才会确认操作。 请注意，主服务器还将指示另一个节点开始构建新的分片副本，以便将系统还原到正常状态。

在将操作转发到副本时，主分片将使用副本来验证它仍然是活动的主分片。 如果主要由于网络分区（或长GC）而被隔离，则它可能会在意识到它已被降级之前继续处理传入的索引操作。复制分片将拒绝来自陈旧主分片的操作。 当主分片收到来自副本的拒绝其请求的响应，因为它不再是主分片，那么它将联系主服务器并将知道它已被替换。 然后将操作路由到新主服务器。

#### 如果没有副本会怎么样？

这是一个有效的方案，可能由于索引配置或仅因为所有副本都已失败而发生。 在这种情况下，主分片处理操作而没有任何外部验证，这可能看起来有问题。 另一方面，主分片本身不能使其他分片失败，除非请求主服务器代表它执行此操作。 这意味着主节点知道主分片是唯一的好副本。 因此，我们保证主节点不会将任何其他（过时的）分片副本提升为新的主分区，并且任何索引到主分区的操作都不会丢失。 当然，由于此时我们只使用单个数据副本运行，因此物理硬件问题可能导致数据丢失。 请参阅[Wait For Active Shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html#index-wait-for-active-shards)以获取一些缓解选项。

### 读模型

Elasticsearch中的读取可以是ID非常轻量级的查找，也可以是具有复杂聚合的大量搜索请求，这些聚合会占用非常重要的CPU能力。

###  




