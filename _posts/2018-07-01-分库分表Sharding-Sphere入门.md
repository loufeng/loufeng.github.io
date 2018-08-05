---
layout:     post
title:      Sharding-sphere简单入门使用
subtitle:   入门
date:       2018-07-01
author:     lou
header-img: img/aboout.jpg
catalog: true
tags:
    - jdbc
---
### Sharding-sphere简单入门使用

一、Sharding-Sphere是一套开源的分布式数据库中间件解决方案组成的生态圈，它由Sharding-JDBC、Sharding-Proxy和Sharding-Sidecar

- 官方地址

- souche示例地址
- demo地址

---

二、功能

1. 数据分片
   - 支持分库分表
   - 分片算法使用，简单的可以用inline表达式代替：
     - 精确分片算法，PreciseShardingAlgorithm(=和IN)，需要配合StandardShardingStrategy使用
     - 范围分片算法，RangeShardingAlgorithm(BETWEEN AND)，需要配合StandardShardingStrategy使用
     - 复合分片算法，ComplexKeysShardingAlgorithm，需要配合ComplexShardingStrategy使用
     - Hint分片算法，需要配合HintShardingStrategy使用。
   - 分布式主键生成
   - 强制分片路由(Hint)
2. 读写分离
   - 目前仅支持单主库，多从库
   - 负载均衡策略
   - 可同时配合分库分表使用
   - 同一线程内，如有写入操作，以后的读操作均从主库读取，保持数据一致性
   - 基于Hint的强制主库路由
3. 数据治理
4. 事务
   - 柔性事务：
     - 最大努力送达型事务
     - TCC
   - 弱XA（支持逻辑异常型跨库事务）

---

三、案例

1. Sharding-JDBC（jar包，需要写配置文件，性能最好适合线上使用）
   - 分库
     - inline表达式，分库sharding-column，选定字段来做数据分片，如下按照用户id奇偶来分数据库
           <sharding:inline-strategy id="databaseStrategy" sharding-column="user_id" algorithm-expression="ds$->{u_id % 2}" />
   - 分表
     - inline表达式，sharding-column，选定字段按照一定规则来做数据分片，例如
           <sharding:inline-strategy id="shareTableStrategy" sharding-column="site_id" algorithm-expression="share_record$->{site_id % 2}" />
       对逻辑表share_record按照字段site_id按2取模，即分成2个分片分表
   - 查询路由(先查找库路由，再确定表路由ds.table)：
         -- 先按照分库路由，确定数据源，再按照分表路由确定查询表，如下sql由于查询site_id则按照奇偶数确定表
         select * from share_record where site_id=? and saler_phone=?
         -- 先按照分库路由，确定数据源，再按照分表路由确定查询表，如下sql会在数据源下查询所有数据表，最终结果集merge
         select * from share_record where date_create between ? and ?
   - 源码分析
     -     //sql执行，ShardingPreparedStatement.java
           //包含两个核心逻辑，route(路由)和MergeEngine(结果集合并)
           @Override
           public ResultSet executeQuery() throws SQLException {
               ResultSet result;
               try {
                   Collection<PreparedStatementUnit> preparedStatementUnits = route();
                   List<ResultSet> resultSets = new PreparedStatementExecutor(
                           getConnection().getShardingContext().getExecutorEngine(), routeResult.getSqlStatement().getType(), preparedStatementUnits).executeQuery();
                   List<QueryResult> queryResults = new ArrayList<>(resultSets.size());
                   for (ResultSet each : resultSets) {
                       queryResults.add(new JDBCQueryResult(each));
                   }
                   MergeEngine mergeEngine = MergeEngineFactory.newInstance(connection.getShardingContext().getShardingRule(), queryResults, routeResult.getSqlStatement());
                   result = new ShardingResultSet(resultSets, mergeEngine.merge(), this);
               } finally {
                   clearBatch();
               }
               currentResultSet = result;
               return result;
           }
     -     //route核心逻辑，PreparedStatement
           private Collection<PreparedStatementUnit> route() throws SQLException {
               Collection<PreparedStatementUnit> result = new LinkedList<>();
               routeResult = routingEngine.route(getParameters());
               for (SQLExecutionUnit each : routeResult.getExecutionUnits()) {
                   PreparedStatement preparedStatement = generatePreparedStatement(each);
                   routedStatements.add(preparedStatement);
                   replaySetParameter(preparedStatement, each.getSqlUnit().getParameterSets().get(0));
                   result.add(new PreparedStatementUnit(each, preparedStatement));
               }
               return result;
           }
     -     //MergeEngine逻辑
           MergeEngine mergeEngine = MergeEngineFactory.newInstance(connection.getShardingContext().getShardingRule(), queryResults, routeResult.getSqlStatement());
     -     //读写分离,MasterSlaveRouter.java
           public Collection<String> route(final SQLType sqlType) {
               if (isMasterRoute(sqlType)) {
                   MasterVisitedManager.setMasterVisited();
                   return Collections.singletonList(masterSlaveRule.getMasterDataSourceName());
               } else {
                   return Collections.singletonList(masterSlaveRule.getLoadBalanceAlgorithm().getDataSource(
                           masterSlaveRule.getName(), masterSlaveRule.getMasterDataSourceName(), new ArrayList<>(masterSlaveRule.getSlaveDataSourceNames())));
               }
           }
           //主库，1.非查询、2.当前线程中有在master session、3.强制路由到主库
           private boolean isMasterRoute(final SQLType sqlType) {
               return SQLType.DQL != sqlType 
                 || MasterVisitedManager.isMasterVisited() 
                 || HintManagerHolder.isMasterRouteOnly();
           }
       
2. Sharding-Proxy（独立的代理服务）
   - config.yaml配置文件
   - 默认启动端口3307
   
3. Sharding-Sidecar（仍在规划中）
4. 比较

       	Sharding-JDBC	Sharding-Proxy	Sharding-Sidecar
  数据库  	任意           	MySQL         	MySQL           
  连接消耗数	高            	低             	高               
  异构语言 	仅Java        	任意            	任意              
  性能   	损耗低          	损耗略高          	损耗低             
  无中心化 	是            	否             	是               
  静态入口 	无            	有             	无               

---

四、涉及技术

- 全局ID
- 分库分表
- 分布式事务

---

五、感想

    	业务发展到一定规模，拆分是必经的阶段，垂直拆分或者业务拆分，一般而言垂直拆分是将一个复杂的大型应用分拆成多个逻辑服务，如用户、订单等，服务化治理；而水平拆分则是在某一具体逻辑服务中对大数据表进行拆分(如订单表)，受限于单表性能瓶颈，以及单库连接数限制，用一些工具对大表进行分库分表。
    	而这次出发点是对公司分享业务进行改造，目前单表数据达到了近4000万，查询上几乎索引能做的优化都已经做了，当前其实面临的主要问题反而是做数据迁移，按时间将表分段迁移到备份历史表中，而目前场景下只用到最近3个月的数据，数据量大概在1200万，对这部分数据需要进行分片。


