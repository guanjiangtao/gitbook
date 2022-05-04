# ES原理总结

* Cluster：包含多个Node的集群。
* Node：集群服务单元。
* Index：一个ES索引包含一个或多个物理分片，它只是这些分片的逻辑命名空间。
* Type：一个index的不同分类，6.x后只能配置一个type，以后将移除。
* Document：最基础的可被索引的数据单元，如一个JSON串。
* Shards：一个分片是一个底层的工作单元，它仅保存全部数据中的一部分，它是一个Lucence实例 (一个lucene索引最大包含2,147,483,519 (= Integer.MAX\_VALUE-128)个文档数量)。
* Replicas：分片备份，用于保障数据安全与分担检索压力。

![](<../.gitbook/assets/Untitled Diagram.drawio.png>)



// todo
