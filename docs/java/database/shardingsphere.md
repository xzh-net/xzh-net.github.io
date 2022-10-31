
# ShardingSphere

Sharding-JDBC采用无中心化架构，适用于Java开发的高性能的轻量级OLTP应用

Sharding-Proxy提供静态入口以及异构语言的支持，适用于OLAP应用以及对分片数据库进行管理和运维的场景。


数据处理大致可以分成两大类：联机事务处理OLTP（on-line transaction processing）、联机分析处理OLAP（On-Line Analytical Processing）。

- OLTP是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。
- OLAP是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。 
- OLTP 系统强调数据库内存效率，强调内存各种指标的命令率，强调绑定变量，强调并发操作；
- OLAP 系统则强调数据分析，强调SQL执行市场，强调磁盘I/O，强调分区等

<table> 
  <tbody> 
   <tr> 
    <th></th> 
    <th>Mycat</th> 
    <th>Sharding-JDBC</th> 
    <th>Sharding-Proxy</th> 
    <th>Sharding-Sidecar</th> 
   </tr> 
  </tbody> 
  <tbody> 
   <tr> 
    <td>官方网站</td> 
    <td><a href=" http://mycat.org.cn/" rel="nofollow" target="_blank">官方网站</a></td> 
    <td><a href=" https://shardingsphere.apache.org/index_zh.html" rel="nofollow" target="_blank">官方网站</a></td> 
    <td><a href=" https://shardingsphere.apache.org/index_zh.html" rel="nofollow" target="_blank">官方网站</a></td> 
    <td><a href=" https://shardingsphere.apache.org/index_zh.html" rel="nofollow" target="_blank">官方网站</a></td> 
   </tr> 
   <tr> 
    <td>源码地址</td> 
    <td><a href=" https://github.com/MyCATApache/Mycat-Server" rel="nofollow" target="_blank">GitHub</a></td> 
    <td><a href=" https://github.com/apache/shardingsphere" rel="nofollow" target="_blank">GitHub</a></td> 
    <td><a href=" https://github.com/apache/shardingsphere" rel="nofollow" target="_blank">GitHub</a></td> 
    <td><a href=" https://github.com/apache/shardingsphere" rel="nofollow" target="_blank">GitHub</a></td> 
   </tr> 
   <tr> 
    <td>官方文档</td> 
    <td><a href=" http://www.mycat.org.cn/document/mycat-definitive-guide.pdf" rel="nofollow" target="_blank">Mycat 权威指南</a></td> 
    <td><a href=" https://shardingsphere.apache.org/document/current/cn/overview/" rel="nofollow" target="_blank">官方文档</a></td> 
    <td><a href=" https://shardingsphere.apache.org/document/current/cn/overview/" rel="nofollow" target="_blank">官方文档</a></td> 
    <td><a href=" https://shardingsphere.apache.org/document/current/cn/overview/" rel="nofollow" target="_blank">官方文档</a></td> 
   </tr> 
   <tr> 
    <td>开发语言</td> 
    <td>Java</td> 
    <td>Java</td> 
    <td>Java</td> 
    <td>Java</td> 
   </tr> 
   <tr> 
    <td>开源协议</td> 
    <td>GPL-2.0/GPL-3.0</td> 
    <td>Apache-2.0</td> 
    <td>Apache-2.0</td> 
    <td>Apache-2.0</td> 
   </tr> 
   <tr> 
    <td>数据库</td> 
    <td>MySQL<br>Oracle<br>SQL Server<br>PostgreSQL<br>DB2<br>MongoDB<br>SequoiaDB<br></td> 
    <td>MySQL<br>Oracle<br>SQLServer<br>PostgreSQL<br>任何遵循 SQL92 标准的数据库<br></td> 
    <td>MySQL/PostgreSQL</td> 
    <td>MySQL/PostgreSQL</td> 
   </tr> 
   <tr> 
    <td>链接数</td> 
    <td>低</td> 
    <td>高</td> 
    <td>低</td> 
    <td>高</td> 
   </tr> 
   <tr> 
    <td>应用语言</td> 
    <td>任意</td> 
    <td>Java</td> 
    <td>任意</td> 
    <td>任意</td> 
   </tr> 
   <tr> 
    <td>代码入侵</td> 
    <td>无</td> 
    <td>须要修改代码</td> 
    <td>无</td> 
    <td>无</td> 
   </tr> 
   <tr> 
    <td>性能</td> 
    <td>损耗略高</td> 
    <td>损耗低</td> 
    <td>损耗略高</td> 
    <td>损耗低</td> 
   </tr> 
   <tr> 
    <td>无中心化</td> 
    <td>否</td> 
    <td>是</td> 
    <td>否</td> 
    <td>是</td> 
   </tr> 
   <tr> 
    <td>静态入口</td> 
    <td>有</td> 
    <td>无</td> 
    <td>有</td> 
    <td>无</td> 
   </tr> 
   <tr> 
    <td>管理控制台</td> 
    <td>Mycat-web</td> 
    <td>Sharding-UI</td> 
    <td>Sharding-UI</td> 
    <td>Sharding-UI</td> 
   </tr> 
   <tr> 
    <td>分库分表</td> 
    <td>单库多表/多库单表</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>多租户方案</td> 
    <td>✔️</td> 
    <td>--</td> 
    <td>--</td> 
    <td>--</td> 
   </tr> 
   <tr> 
    <td>读写分离</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>分片策略定制化</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>分布式主键</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>标准化事务接口</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>XA强一致事务</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>柔性事务</td> 
    <td>--</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>配置动态化</td> 
    <td>开发中</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>编排治理</td> 
    <td>开发中</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>数据脱敏</td> 
    <td>--</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>可视化链路追踪</td> 
    <td>--</td> 
    <td>✔️</td> 
    <td>✔️</td> 
    <td>✔️</td> 
   </tr> 
   <tr> 
    <td>弹性伸缩</td> 
    <td>开发中</td> 
    <td>开发中</td> 
    <td>开发中</td> 
    <td>开发中</td> 
   </tr> 
   <tr> 
    <td>多节点操做</td> 
    <td>分页<br>去重<br>排序<br>分组<br>聚合<br></td> 
    <td>分页<br>去重<br>排序<br>分组<br>聚合<br></td> 
    <td>分页<br>去重<br>排序<br>分组<br>聚合<br></td> 
    <td>分页<br>去重<br>排序<br>分组<br>聚合<br></td> 
   </tr> 
   <tr> 
    <td>跨库关联</td> 
    <td>跨库 2 表 Join <br> ER Join <br> 基于 caltlet 的多表 Join<br></td> 
    <td>--</td> 
    <td>--</td> 
    <td>--</td> 
   </tr> 
   <tr> 
    <td>IP 白名单</td> 
    <td>✔️</td> 
    <td>--</td> 
    <td>--</td> 
    <td>--</td> 
   </tr> 
   <tr> 
    <td>SQL 黑名单</td> 
    <td>✔️</td> 
    <td>--</td> 
    <td>--</td> 
    <td>--</td> 
   </tr> 
   <tr> 
    <td>存储过程</td> 
    <td>✔️</td> 
    <td>--</td> 
    <td>--</td> 
    <td>--</td> 
   </tr> 
  </tbody> 
 </table>